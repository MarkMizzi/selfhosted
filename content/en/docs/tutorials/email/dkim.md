---
title: Setting up DKIM
description: Tutorial for setting up DKIM signing/verification on Postfix using OpenDKIM
date: 2017-01-04
weight: 30
---

Email headers contain a `FROM:` line that identifies the sender. This line is a part of the data sent, and is not used when the email is being routed to its destination. This means that an email can be successfully sent with an address in its `FROM:` line that does not match the actual sender; a flaw of SMTP used by malicious actors in order to send emails that appear to come from a legitimate source. This is called [email spoofing](https://www.cloudflare.com/learning/email-security/what-is-email-spoofing/).

Modern mail servers contain several protections against email spoofing, with one of the more advanced being DomainKeys Identified Mail (DKIM). MTA servers configured with DKIM sign outgoing emails. The signature is then verified against a public [DNS TXT record](https://www.cloudflare.com/learning/dns/dns-records/dns-dkim-record/) that has a special format.

It is highly recommended to set up DKIM on any self hosted mail server, as otherwise a lot of mail services such as Gmail will automatically assume that mail sent from your mail server is spam.

We will configure DKIM for `postfix` using [OpenDKIM](http://www.opendkim.org), which is set up as a milter. I found [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-dkim-with-postfix-on-debian-wheezy) helpful when setting this up.

OpenDKIM can be installed using:
``` bash
sudo apt-get update
sudo apt-get install -y opendkim opendkim-tools
```

## Reloading configuration

OpenDKIM is configured via a config file at `/etc/opendkim.conf`. After changes to this file, you must restart the service:
``` bash
sudo service opendkim restart
```
for the changes to take effect.

## Basic Configuration

A basic configuration can be added using the following
``` bash
echo '
# auto restart the DKIM filter on failures
AutoRestart             Yes
# specifies the filter’s maximum restart rate, if restarts begin to happen faster than this rate, the filter will terminate; 10/1h - 10 restarts/hour are allowed at most
AutoRestartRate         10/1h

# gives all access permissions to the user group defined by UserID and allows other users to read and execute files
UMask                   002
# the opendkim process runs under this user and group
UserID                  opendkim:opendkim

# these parameters enable detailed logging via calls to syslog
Syslog                  Yes
SyslogSuccess           Yes
LogWhy                  Yes

# defines the canonicalization methods used at message signing
# - the simple method allows almost no modification while 
# - the relaxed one tolerates minor changes such as whitespace replacement; 
# - relaxed/simple: the message header will be processed with the relaxed algorithm and the body with the simple one
Canonicalization        relaxed/simple

# specifies the external hosts that can send mail through the server as one of the signing domains without credentials
ExternalIgnoreList      refile:/etc/opendkim/TrustedHosts
# defines a list of internal hosts whose mail should not be verified but signed instead
InternalHosts           refile:/etc/opendkim/TrustedHosts
# maps key names to signing keys
KeyTable                refile:/etc/opendkim/KeyTable
# lists the signatures to apply to a message based on the address found in the From: header field
SigningTable            refile:/etc/opendkim/SigningTable

# declares operating modes; in this case the milter acts as a signer (s) and a verifier (v)
Mode                    sv

# the path to the Pid file which contains the process identification number
PidFile                 /var/run/opendkim/opendkim.pid

# selects the signing algorithm to use when creating signatures
SignatureAlgorithm      rsa-sha256

# the milter will listen on the socket specified here, 
# Postfix will send messages to opendkim for signing and verification through this socket;
# 12301@localhost defines a TCP socket that listens on localhost, port 12301
Socket                  inet:12301@localhost' | sudo tee -a /etc/opendkim.conf
```

Make sure that port `12301` is free for OpenDKIM to use.

Open `/etc/postfix/main.cf` and add the following lines if they are not already present:
``` text
# The mail filter protocol version and optional protocol extensions for communication with a Milter application
# 6 specifies that we want to use Sendmail 8 mail filter protocol version 6
milter_protocol = 6
# The default action when a Milter (mail filter) response is unavailable (for example, bad Postfix configuration or Milter failure).
# accept specifies that we should proceed as if the mail filter was not present.
milter_default_action = accept

# A list of Milter (mail filter) applications for new mail that arrives via the Postfix smtpd server.
smtpd_milters = inet:localhost:12301
# A list of Milter (mail filter) applications for new mail that does not arrive via the Postfix smtpd server.
non_smtpd_milters = inet:localhost:12301
```

If `smtpd_milters` or `non_smtpd_milters` are already present and populated, add `, inet:localhost:12301` to the end of both of them. For example, if using [SpamAssassin](/docs/tutorials/email/spam) as a milter, the current configuration will look like this:
``` text
smtpd_milters = unix:/spamass/spamass.sock
non_smtpd_milters = unix:/spamass/spamass.sock
```
and must be changed to this
``` text
smtpd_milters = unix:/spamass/spamass.sock, inet:localhost:12301
non_smtpd_milters = unix:/spamass/spamass.sock, inet:localhost:12301
```

## Setting trusted hosts

Trusted hosts are hosts where mail originating from them does not require DKIM verification. They are defined in the `/etc/opendkim/TrustedHosts` file as per our configuration. We recommend limiting trusted hosts to local network hosts and public hosts with `<domain-name>`; this can be done using the following:
``` bash
# create dir to store TrustedHosts file.
sudo mkdir -p /etc/opendkim
sudo chown opendkim:opendkim /etc/opendkim

# create and populate the TrustedHosts file
echo '
127.0.0.1
localhost
192.168.0.1/24

*.<domain-name>' | sudo tee /etc/opendkim/TrustedHosts

# change ownership of newly created hosts file to opendkim
sudo chown opendkim:opendkim /etc/opendkim/TrustedHosts
```

## Creating the signing keys

The [DKIM DNS record](https://www.cloudflare.com/learning/dns/dns-records/dns-dkim-record/) is a TXT record for `<selector>._domainkey.<domain-name>`, where `<selector>` is any alphanumeric string chosen during the setup of DKIM on the MTA.

Key tables map a selector/domain pair to a private key; they consist of lines in the following format:
``` text
<selector>._domainkey.<domain-name> <domain-name>:<selector>:<private-key-file>
```

In our case we will use `mail` as the `<selector>` and we will populate the key table (contained at `/etc/opendkim/KeyTable` as per our configuration) using:
``` bash
echo 'mail._domainkey.<domain-name> <domain-name>:mail:/etc/opendkim/keys/<domain-name>/mail.private' | sudo tee /etc/opendkim/KeyTable

# change ownership of key table file to opendkim
sudo chown opendkim:opendkim /etc/opendkim/KeyTable
```

OpenDKIM also requires a signing table with lines in the following format:
``` text
*@<domain-name> <selector>._domainkey.<domain-name>
```

This will identify the key to use when signing emails outgoing from the MTA with domain name `<domain-name>`.

For our purposes we can populate the signing table (contained at `/etc/opendkim/SigningTable` as per our configuration) using:
``` bash
echo '*@<domain-name> mail._domainkey.<domain-name>' | sudo tee /etc/opendkim/SigningTable

# change ownership of signing table file to opendkim
sudo chown opendkim:opendkim /etc/opendkim/SigningTable
```

We can finally generate a private key for OpenDKIM to use when signing outgoing emails, this can be done using the following commands:
``` bash
sudo mkdir -p /etc/opendkim/keys/<domain-name>
cd /etc/opendkim/keys/<domain-name>

# create the signing private key for OpenDKIM
#   -s specifies the selector, in our case mail
#   -d specifies the domain name to be used with emails.
#   -b specifies the key size (in bytes), 2048 or greater does not work with Gmail in my experience.
sudo opendkim-genkey -s mail -d <domain-name> -b 1024

# change ownership of created key to opendkim
sudo chown -R opendkim:opendkim /etc/opendkim/keys/<domain-name>
```

## Creating a DKIM DNS record

As mentioned above, the [DKIM DNS record](https://www.cloudflare.com/learning/dns/dns-records/dns-dkim-record/) must contain the public key corresponding to the private key used by OpenDKIM to sign outgoing emails at the MTA. The public key and its information required for this DNS record can be accessed by running the following command:
``` bash
sudo cat /etc/opendkim/keys/<domain-name>/mail.txt
```

To create the DKIM DNS record, create a TXT record for `<selector>._domainkey.<domain-name>` with content in the following format:
``` text
v=DKIM1; k=<public-key-type>; p=<public-key>;
```
The `<selector>` is chosen during OpenDKIM configuration (see above), and the values to put for `k=` and `p=` can be obtained from the DKIM configuration as well (see above).

In my case, for example, `<selector>` is `mail`, and the contents of the DKIM record are:
``` text
v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDFMMPBxFVmYv+UqWG6YZUrpPq7P9NKovp6jw1m8rp/QQTFV5Ar8seCarbP7Rl7Q3UjIpO5JisKyfp9RjzkEZ8fnGpyZhOhzN9D4X5wf3ug37RbS4j8XF1zYx1m1POrtGHKiSa0BKBFSX15REs3RnkaiIRHNFmj+XIwTId5MyIOoQIDAQAB;
```
