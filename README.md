# A "simple", yet complete Mailserver setup
Setting up a proper mailserver is anything but simple, especially if you want to be considered reputable by other servers. In this guide I explain what I did for setting up and configuring my mailserver with Postfix and Dovecot. I'll assume you want to run everything on the same machine and you already own a domain.

## Steps
- install and configure Postfix
- add MX and SPF DNS records to your domain
- add PTR reverse DNS record
- create and configure TLS/SMTPS certificates (SSL/TLS or STARTTLS)
- install and configure DKIM (opendkim) and relative DNS record (keys)
- install and configure DMARC (opendmarc) and relative DNS record
- install and configure Dovecot (IMAP): Maildir/mbox, TLS, SASL socket for Postfix
- configure sender security on Postfix (FROM address) and aliases
