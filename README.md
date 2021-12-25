# Mailserver
Setting up a proper mailserver is anything but simple, especially if you want to be considered reputable by other servers. In this guide I explain what I did for setting up and configuring my mailserver with Postfix and Dovecot. I'll assume you want to run everything on the same machine and you already own a domain.




## Table of Contents
- [Introduction](#Introduction)
	- [Steps](#Steps)
	- [Needed software](#Needed-software)
- [Postfix](#Postfix)
	- [Base configuration](#Base-configuration)
	- [MX DNS record](#MX-DNS-record)
	- [Aliases](#Aliases)
- [Secure mail](#Secure-mail)
- [Verified mail](#Verified-mail)
	- [SPF record](#SPF-record)
	- [PTR record](#PTR-record)
	- [DKIM](#DKIM)
		- [DKIM service](#DKIM-service)
		- [DKIM record](#DKIM-record)
	- [DMARC](#DMARC)
		- [DMARC service](#DMARC-service)
		- [DMARC record](#DMARC-record)
- [Dovecot](#Dovecot)
	- [SASL](#SASL)
	- [TLS for SASL](#TLS-for-SASL)
	- [Sender restrictions](#Sender-restrictions)
- [Incoming SPF](#Incoming-SPF)




## Introduction
As a disclaimer, I am not an expert on the matter by any means. I'm just reporting as a guide what I have done on my server and what I found to work.

For a complete mail server setup you'll need a Mail Transfer Agent (MTA), which sends and receives mail via the SMTP protocol; a Mail Delivery Agent (MDA), which stores mail in mailbox formats such as mbox or Maildir; an IMAP/POP3 server, which lets users access their mail remotely from a Mail User Agent (MUA, clients such as Thunderbird). For server reputability you'll need to add some DNS records and relative services on the server: SPF, DKIM, DMARC, PTR (reverse DNS). Last but not least is of course TLS/SSL for secure mail.


### Steps
- install and configure Postfix
- add MX and SPF DNS records to your domain
- add PTR reverse DNS record
- create and configure TLS/SMTPS certificates (SSL/TLS or STARTTLS)
- install and configure DKIM (opendkim) and relative DNS record (keys)
- install and configure DMARC (opendmarc) and relative DNS record
- install and configure Dovecot (IMAP): Maildir/mbox, TLS, SASL socket for Postfix
- configure sender security on Postfix (FROM address) and aliases


### Needed software
Find and install from the package distribution of your server the appropriate packages which contain the following software:
- Postfix
- Certbot (optional)
- OpenDKIM
- OpenDMARC
- Dovecot

My server runs Arch Linux so I'll be referring to its conventions (file paths, services, etc.). However, as usual, you can easily adapt the guide to any Linux distro. Any configuration not mentioned is left as default.




## Postfix
Postfix is both an MTA and an MDA. Out of the box Postfix should already be able to send and receive mail locally (between system users). You can test this via the `sendmail` command or by using an implementation of the mailx POSIX standard, e.g. `s-nail` (`mail` command).

Anyway, we'll set up Postfix for sending and receiving mail internally *or* externally for the domain *domain.tld*. Of course, replace it with your own domain if/when following the guide. We'll also set up local users, i.e. each mail user is a system user. You can alternatively set up virtual users, i.e. mail users don't have a corresponding system user (though we'll not cover this method in this guide).


### Base configuration
Postfix configuration is contained in the directory `/etc/postfix` and is mostly based on two files: `/etc/postfix/main.cf`, which contains configuration parameters; and `/etc/postfix/master.cf`, which contains the services (and their options) Postfix will run. See `postconf(5)` and `master(5)`.

#### `main.cf`
Since Postfix is a very versatile MTA, it is possible to configure it to accept mail and relay it to other servers. Since it is not our case, we'll set it up to accept all incoming email directed to our domain:
```pfmain
# hostname of the server
myhostname = domain.tld

# domain of the server
mydomain = domain.tld

# domain of the outgoing mail
myorigin = $mydomain

# domains of received mail to accept
mydestination = $myhostname, $mydomain, localhost, localhost.$mydomain

# this host does all the work
mynetworks_style = host
relayhost =
```

Of course, enable/start Postfix service, e.g., if using systemd:
```sh
systemctl start postfix.service
```


### MX DNS record
It is now time to add an MX record to your DNS. An MX record tells a server which wants to send mail to *domain.tld* the IP of the server which will receive the mail. In more complex setups, you may specify more than one server via multiple MX records and different priorities, or specify just one and let your server relay the mail to various hosts. In our simple setup we'll add just one record and the server will keep the received mail.

Find out how to add an MX record with your DNS provider. For our usage the record should have `@` as host/name/subdomain (i.e. `domain.tld` and not `something.domain.tld`), priority 1 (although it is not really relevant since there's only one MX record) and of course the IP of your server as value. Don't worry if you already have a record for `@`, say of type A, this does not conflict as it is queried only when sending mail.

As the addition of the record takes effect your server should effectively send and receive (unsecure and unverified) mail.


### Aliases
An alias `a: b` lets `b` receive mail sent to `a@domain.tld`. While on systems like Debian the alias file is usually located in `/etc/aliases`, Arch Linux prefers to create a Postfix-specific alias file in `/etc/postfix/aliases`. Of course, tweak it as you like by changing `/etc/postfix/main.cf` accordingly:
```pfmain
alias_maps = hash:/etc/postfix/aliases
alias_database = $alias_maps
```
See `aliases(5)`, `postalias(1)`, `newaliases(1)`.

When you create a new alias file make sure to build the database with the `postalias` command. Just run `newaliases` if you only need to update an existing alias database:
```sh
postalias /etc/postfix/aliases
```

Remember to alias the root user as it's not a good idea to receive mail as root:
```
root: <you>
```

As for sending mail from an alias, we'll see that after configuring SASL authentication in Dovecot.

Tip: you can use `a: b,c,d` to make `a` a mailing list for `b`, `c`, and `d`.




## Secure mail (TLS)
Get a certificate for *domain.tld*. If you don't have one already, an easy way is using `certbot`, for example:
```sh
sudo certbot certonly --standalone --email youremail@example.com -d domain.tld
```

In main.cf:
```pfmain
# sending
smtp_tls_security_level = may

# receiving
smtpd_tls_security_level = may
smtpd_use_tls = yes
smtpd_tls_cert_file = /etc/letsencrypt/live/domain.tld/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/domain.tld/privkey.pem
smtpd_tls_protocols = !SSLv2, !SSLv3
```

There are two ways to accept secure mail: STARTTLS over SMTP (port 587) and SMTPS aka SSL/TLS (port 465). I chose SMTPS, which means I had to uncomment the following lines in `master.cf`:
```pfmaster
smtps     inet  n       -       n       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_recipient_restrictions=
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

You might also need to add the well known ports of SMTPS to `/etc/services` if not present:
```services
smtps		465/tcp
smtps		465/udp
```



## Verified mail
If you think about it, we didn't set up any mechanism that certifies to the receiver that mail signed with our domain actually comes from a legitimate server. In fact, anyone could set up their MTA to send mail under *domain.tld*. In this section we set up a series of mechanisms to certify that mail sent from our server is legitimately under our domain.

I suggest [this page](https://www.appmaildev.com/en/dkim) for testing the various protocols, however you could of course make the verifications yourself manually.


### SPF record
The first and simplest mechanism is a Sender Policy Framework (SPF) DNS record. An SPF record contains information about the IPs that can send mail under the name of the requested domain. When a server receives mail from *domain.tld*, it checks for SPF records with the DNS of *domain.tld* to see if the sender's IP is certified as legitimate by the domain administrators.

SPF records are actually special TXT records with a particular syntax. You can specify legitimate servers via various methods, including explicit IP and web list. You can also specify the policy the receiver should adopt when receiving mail from specified senders. You can find all the information you need in the [official documentation](http://www.open-spf.org/SPF_Record_Syntax). For our simple use case, we'll add a TXT record for the host/name/subdomain `@` with the following value:
```
v=spf1 +ip4:XXX.XXX.XXX.XXX -all
```
which tells the receiver to accept mail from the specified IPv4 and reject mail from any other host.

For more information on how to add TXT records, look for a guide by your DNS provider.


### PTR record
A PTR record is a reverse DNS record, i.e. a record that given an IP assigns a domain name. PTR records are usually assigned by requesting it to your ISP or to your server provider. This step is considered optional.


### DKIM
DomainKeys Identified Mail (DKIM) is a method for authentication of hosts that can send mail under a certain domain. Basically, a service in the sender host digitally signs the outgoing message with a private key, then the receiver decrypts the signature by using the public key specified on the DKIM DNS record of the sending domain. We then need to set up a DKIM DNS record for our domain and a DKIM service on our sending host.

#### DKIM service
The DKIM service we'll set up on our host is an OpenDKIM service. We'll need a couple of tweaks in the configuration file `/etc/opendkim/opendkim.conf` (you may copy the template from `/usr/share/doc/opendkim/opendkim.conf.sample`) (see `opendkim(8)` and `opendkim.conf(5)`):
```conf
Canonicalization	relaxed/simple
Domain			domain.tld
KeyFile			/etc/opendkim/default.private
Selector		default
Socket                  local:/run/opendkim/opendkim.sock
UMask			002
UserID			opendkim:postfix
```
This setup assumes the presence of a private key in `/etc/opendkim/default.private` and builds a UNIX socket for communication with Postfix (you may use a network socket if OpenDKIM and Postfix services reside on different machines, but UNIX sockets are more performant and more secure if on the same host). We then need to configure Postfix accordingly by adding, in `main.cf`:
```pfmain
smtpd_milters = unix:/run/opendkim/opendkim.sock
non_smtpd_milters = $smtpd_milters
```

We set up OpenDKIM to create a socket in `/run/opendkim/opendkim.sock`, but OpenDKIM does not create directories, so we need to create `/run/opendkim` manually:
```
sudo mkdir /run/opendkim
sudo chown opendkim:postfix /run/opendkim
```
You can do this automatically at boot by adding a file `/lib/tmpfiles.d/opendkim.conf` with the following content:
```
 D /run/opendkim 0750 opendkim postfix
```

Use the following command to generate the key pair:
```sh
opendkim-genkey -r -d domain.tld
```
See `opendkim-genkey(8)`.
Move the private key in the directory specified by the `KeyFile` option in `opendkim.conf`, in our case:
```sh
mv default.private /etc/opendkim/
```

#### DKIM record
DKIM records are actually TXT records, similar to SPF. In the `.txt` file generated by `opendkim-genkey` you'll find all needed info (expressed in Bind 9 format):
```
default._domainkey      IN      TXT     ( "v=DKIM1; k=<KEYTYPE>; s=email; "
          "p=<PUBLICKEY" )  ;
```
The record should then look like this:
```
v=DKIM1; k=<KEYTYPE>; s=email; p=<PUBLICKEY>
```
with the host/name/subdomain `default._domainkey`.

Enable/start the OpenDKIM service and that's it.


### DMARC
Domain-based Message Authentication, Reporting and Conformance (DMARC) is another mail authentication protocol based on SPF and DKIM that specifies a policy to take when receiving mail from a sender under a particular domain name, including reporting to the [postmaster](https://en.wikipedia.org/wiki/Postmaster_(computing\)).

#### DMARC service
We'll set up an OpenDMARC service with a similar yet simpler procedure to the one we used for OpenDKIM. In `/etc/opendmarc/opendmarc.conf` set:
```conf
Socket unix:/run/opendmarc/opendmarc.sock
```
See `opendmarc(8)` and `opendmarc.conf(5)`.

As for OpenDKIM, OpenDMARC does not create the socket directory, so we need to create `/run/opendmarc` manually:
```
sudo mkdir /run/opendmarc
sudo chown opendmarc:postfix /run/opendmarc
```
And at boot by adding the file `/lib/tmpfiles.d/opendmarc.conf` with:
```
 D /run/opendmarc 0750 opendmarc postfix
```

Tell Postfix to use DMARC by updating its `main.cf`:
```pfmain
smtpd_milters = unix:/run/opendkim/opendkim.sock, unix:/run/opendmarc/opendmarc.sock
```

The systemd service unit included in my repositories run OpenDMARC as `opendmarc:mail`, but we want it to run as group `postfix`. Let's fix that by adding a drop-in ovverride file `/etc/systemd/system/opendmarc.service.d/override.conf`:
```systemd
[Service]
Group=
Group=postfix
 ```

#### DMARC record
DMARC records are TXT records, similarly to SPF and DKIM. The DMARC record in particular contains information on the addresses to report malicious mail practices ([spoofing](https://en.wikipedia.org/wiki/Email_spoofing)) to. See the [official documentation](https://dmarc.org/overview) for the detailed syntax. Our DMARC record will look like this:
```
v=DMARC1; rua=mailto:postmaster@domain.tld; ruf=mailto:reports@domain.tld; adkim=s; fo=1
```
for the host/name/subdomain `_dmarc`.
You may choose the `ruf` (reporting URI for forensic reports) and `rua` (reporting URI for aggregate reports) addresses you prefer, but keep in mind the conventions and make sure it is accessible on your server (system/virtual user or alias).

Enable/start the OpenDMARC service and that's it.




## Dovecot
Dovecot is an IMAP and POP3 server: it allows email clients (MUAs) to access mail remotely. Start by copying the standard configuration:
```
cp /usr/share/doc/dovecot/example-config/dovecot.conf /etc/dovecot/
cp -r /usr/share/doc/dovecot/example-config/conf.d /etc/dovecot/
```
In `/etc/dovecot/conf.d/10-mail.conf` set:
```conf
# using mbox format instead of Maildir
mail_location = mbox:~/mail:INBOX=/var/spool/mail/%u
```


### SASL
We'll now set up Simple Authentication and Security Layer (SASL) between Postfix and Dovecot. Firstly, in `/etc/dovecot/conf.d/10-master.conf` set:
```conf
service auth {
	# communication with Postfix via UNIX socket
	unix_listener /var/spool/postfix/private/auth {
		group = postfix
		mode = 0660
		user = postfix
	}
}
```
then set in Postfix's `main.cf`:
```pfmain
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous, noplaintext
```


### TLS for SASL
Since we use the same domain for SMTP and IMAP/POP3, we can use the same certificate as before. In `/etc/dovecot/conf.d/10-ssl.conf` put:
```conf
ssl = required
ssl_cert = </etc/letsencrypt/live/domain.tld/fullchain.pem
ssl_key = </etc/letsencrypt/live/domain.tld/privkey.pem
```
In `/etc/dovecot/conf.d/
```conf
# safe since we enforce TLS
disable_plaintext_auth = no
```
And in `/etc/postfix/main.cf`:
```pfmain
smtpd_sasl_tls_security_options = noanonymous
smtpd_tls_auth_only = yes
```

Of course, enable/start Dovecot when done configuring.

Users may now access the IMAP/SMTP server with the following settings on their MUA:
- user/pw: system ones
- SMTP (outgoing): port 465, security SSL/TLS, server domain.tld
- IMAP (incoming): port 993, security SSL/TLS, server domain.tld



### Sender restrictions
Right now, every authenticated user can send mail from any address via our SMTP server (FROM field of SMTP). We want to limit the range of addresses they can send mail from to `username@domain.tld` and maybe some aliases. We can do this by specifying, in `/etc/postfix/main.cf`:
```pfmain
smtpd_sender_restrictions = reject_non_fqdn_sender, reject_unknown_sender_domain, reject_sender_login_mismatch
smtpd_sender_login_maps = pcre:/etc/postfix/sendas.pcre
```
We use PCRE for regex-based alias definitions. Keep in mind that you may need to install an appropriate package (possibly `postfix-pcre`).

The first two options of the first line limit FROM addresses to valid domains and to domains recognised by our Postfix configuration (`$myorigin`). The third option, together with the second line, tells Postfix to accept an address only if matches the rules provided by the `sendas.pcre` file. The file maps candidate FROM addresses to their owner(s) (similarly to an alias file):
```pcre
# Users can only send mail from their own user or from their aliases

/^postmaster@domain\.tld/     <you>
/^admin@domain\.tld/          <you>, <another-admin>

# match any other address with the user defined by the part before the @
/^(.*)@domain\.tld/ ${1}
```
You don't need to launch `postalias` or `newaliases` for PCRE files.




## Incoming SPF
An optional upgrade to your server is filter received mail to only accept from IPs verified with SPF. You can do that with a couple of tools, including `python-postfix-policyd-spf`. After installing (you can find it on the AUR), we need to configure Postfix accordingly. In `main.cf`:
```pfmain
policy-spf_time_limit = 3600s
smtpd_recipient_restrictions=
     permit_sasl_authenticated
     permit_mynetworks
     reject_unauth_destination
     check_policy_service unix:private/policy-spf
```
In `master.cf`:
```pfmaster
policy-spf  unix  -       n       n       -       0       spawn
  user=nobody argv=/usr/bin/policyd-spf
```
You can also configure the policy in `/etc/python-policyd-spf/policyd-spf.conf` (see the `.commented` version for reference), although defaults are sane. Remember to reload the Postfix service and you're done.




## Sources
- experience and trial-and-error
- manual pages
- Some [Arch Wiki](https://wiki.archlinux.org) pages, including [Mail server](https://wiki.archlinux.org/title/Mail_server), [Postfix](https://wiki.archlinux.org/title/Postfix), [OpenDKIM](https://wiki.archlinux.org/title/OpenDKIM), [OpenDMARC](https://wiki.archlinux.org/title/OpenDMARC), [Dovecot](https://wiki.archlinux.org/title/Dovecot).
- [Postfix docs](http://www.postfix.org/documentation.html) (especially [SASL with Dovecot](http://www.postfix.org/SASL_README.html#server_dovecot))
- [Dovecot docs](https://www.dovecot.org/documentation)
- Some Wikipedia pages, including [MX](https://en.wikipedia.org/wiki/MX_record), [SPF](https://en.wikipedia.org/wiki/Sender_Policy_Framework), [DKIM](https://en.wikipedia.org/wiki/DomainKeys_Identified_Mail), [DMARC](https://en.wikipedia.org/wiki/DMARC), [SASL](https://en.wikipedia.org/wiki/Simple_Authentication_and_Security_Layer)

I highly recommend checking out the various Wikipedia pages regarding mail servers and protocols as they are a big source for learning.




## Other links
- [Authentication tester](https://www.appmaildev.com/en/dkim.md)
- [SMTP reference](https://www.samlogic.net/articles/smtp-commands-reference.htm)




## License
This work is licensed under a [CC BY-NC-SA 4.0 license](https://creativecommons.org/licenses/by-nc-sa/4.0/legalcode)




## Contribution
If you find wrong or incomplete information, please report it via issue or pull request.
