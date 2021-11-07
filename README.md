# A "simple", yet complete mail server setup
Setting up a proper mailserver is anything but simple, especially if you want to be considered reputable by other servers. In this guide I explain what I did for setting up and configuring my mailserver with Postfix and Dovecot. I'll assume you want to run everything on the same machine and you already own a domain.




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
- Certbot
- OpenDKIM
- OpenDMARC
- Dovecot

My server runs Arch Linux so I'll be referring to its conventions (file paths, services, etc.). However, as usual, you can easily adapt the guide to any Linux distro. Any configuration not mentioned is left as default.




## Postfix
Postfix is both an MTA and an MDA. We'll set it up for sending and mail for the domain *domain.tld*. Of course, replace it with your own domain if/when following the guide.

### Base configuration
Postfix configuration is contained in the directory `/etc/postfix` and is mostly based on two files: `/etc/postfix/main.cf`, which contains configuration parameters; and `/etc/postfix/master.cf`, which contains the services (and their options) Postfix shall run.

#### `main.cf`
Since Postfix is a very versatile MTA, it is possible to configure it to accept mail and relay it to other servers. Since it is not our case, we'll set it up to accept all incoming email directed to our domain:
```pfmain
# hostname of the server
myhostname = domain.tld

# domain of the server
mydomain = domain.tld

# domain of the outcoming mail
myorigin = $mydomain

# domains of received mail to accept
mydestination = $myhostname, $mydomain, localhost, localhost.$mydomain

# this host does all the work
mynetworks_style = host
relayhost =
```
See `postconf(5)`.

### Aliases
An alias `a: b` lets `b` receive mail sent to `a@domain.tld`. While on systems like Debian the alias file is usually located in `/etc/aliases`, Arch Linux prefers to create a Postfix-specific alias file in `/etc/postfix/aliases`. Of course, tweak it as you like by changing `/etc/postfix/main.cf` accordingly:
```pfmain
alias_maps = hash:/etc/postfix/aliases
alias_database = $alias_maps
```
See `aliases(5)`, `postalias(1)`, `newaliases(1)`.

When you create a new alias file make sure to build the database with the `postalias` command. Just run `newaliases` if you only need to update an existing alias database.

As for sending mail from an alias, we'll see that after configuring SASL authentication in Dovecot.

Tip: you can use `a: b,c,d` to make `a` a mailing list for `b`, `c`, and `d`.


### TLS
[WIP]




## Sources
- experience and trial-and-error
- manual pages
- Some [Arch Wiki](https://wiki.archlinux.org) pages, including [Mail server](https://wiki.archlinux.org/title/Mail_server)
- Postfix docs
- Dovecot docs
- Some Wikipedia pages, including [MX](https://en.wikipedia.org/wiki/MX_record), [SPF](https://en.wikipedia.org/wiki/Sender_Policy_Framework), [DKIM](https://en.wikipedia.org/wiki/DomainKeys_Identified_Mail), [DMARC](https://en.wikipedia.org/wiki/DMARC)

I highly recommend checking out the various Wikipedia pages regarding mail servers and protocols as they are a big source for learning.




## Other links
- https://www.appmaildev.com/en/dkim
