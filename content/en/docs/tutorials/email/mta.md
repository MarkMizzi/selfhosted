---
title: Setting up an MTA
description: Tutorial for setting up Postfix as an MTA server
date: 2017-01-04
weight: 20
---

An MTA server is responsible for actually sending and receiving emails. It does this via the Simple Mail Transfer Protocol (SMTP). User agent programs (such as Thunderbird) send mail by communicating with an MTA server via SMTP. Incoming email is received by the MTA server from other MTAs using SMTP, and then stored in the appropriate user mailbox where the MAA server can access it and make it available to MAA clients.

For more information on the SMTP protocol, see [here](https://www.cloudflare.com/learning/email-security/what-is-smtp) or see [our reference material on email](/docs/reference/email).

As an MTA, we will use [`postfix`](https://www.postfix.org).

Milters (portmanteau for mail filters) are special programs which are used to filter received or sent emails. They can be used with the `postfix` and `sendmail` MTAs, and communicate with them over TCP. The MTA server informs and provides data about each stage of the email receipt/sending to milters, and milter programs can choose to terminate sending/receiving the email at any point.

It is recommended to configure `postfix` with some milter programs (see [Setting up DKIM](/docs/tutorials/dkim) and [Spam filters](/docs/tutorials/email/spam)).

## Installation

``` bash
sudo apt-get update
sudo apt-get install -y postfix
```

When prompted with this screen:
``` text
  ┌───────────────────────────┤ Postfix Configuration ├───────────────────────────┐
  │ Please select the mail server configuration type that best meets your needs.  │
  │                                                                               │
  │  No configuration:                                                            │
  │   Should be chosen to leave the current configuration unchanged.              │
  │  Internet site:                                                               │
  │   Mail is sent and received directly using SMTP.                              │
  │  Internet with smarthost:                                                     │
  │   Mail is received directly using SMTP or by running a utility such           │
  │   as fetchmail. Outgoing mail is sent using a smarthost.                      │
  │  Satellite system:                                                            │
  │   All mail is sent to another machine, called a 'smarthost', for              │
  │   delivery.                                                                   │
  │  Local only:                                                                  │
  │   The only delivered mail is the mail for local users. There is no            │
  │   network.                                                                    │
  │                                                                               │
  │ General mail configuration type:                                              │
  │                                                                               │
  │                           No configuration                                    │
  │                           Internet Site                                       │
  │                           Internet with smarthost                             │
  │                           Satellite system                                    │
  │                           Local only                                          │
  │                                                                               │
  │                                                                               │
  │                     <Ok>                         <Cancel>                     │
  │                                                                               │
  └───────────────────────────────────────────────────────────────────────────────┘
```
choose `Internet Site` as the configuration that should be installed.

Then enter the domain name you want to use in your email addresses in the following screen:
``` text
 ┌────────────────────────────┤ Postfix Configuration ├────────────────────────────┐
 │ The 'mail name' is the domain name used to 'qualify' _ALL_ mail addresses       │
 │ without a domain name. This includes mail to and from <root>: please do not     │
 │ make your machine send out mail from root@example.org unless root@example.org   │
 │ has told you to.                                                                │
 │                                                                                 │
 │ This name will also be used by other programs. It should be the single, fully   │
 │ qualified domain name (FQDN).                                                   │
 │                                                                                 │
 │ Thus, if a mail address on the local host is foo@example.org, the correct       │
 │ value for this option would be example.org.                                     │
 │                                                                                 │
 │ System mail name:                                                               │
 │                                                                                 │
 │ _______________________________________________________________________________ │
 │                                                                                 │
 │                      <Ok>                          <Cancel>                     │
 │                                                                                 │
 └─────────────────────────────────────────────────────────────────────────────────┘
```

To check that it is running correctly, use:
``` bash
sudo service postfix status
```

## Reloading configuration

`postfix` is configured mainly through a single config file at `/etc/postfix/main.cf`. After configuration changes, `postfix` needs to be reloaded with:
``` bash
sudo service postfix reload
```
in order for the changes to take effect.

## Basic Configuration

Ensure that `/etc/postfix/main.cf` contains the following:
``` text
# public DNS name of this MTA server
myhostname = mail.<domain-name>

# The list of domains that are delivered via the $local_transport mail delivery transport.
# In other words this is the list of domains for which postfix will send/receive mail
mydestination = mail.<domain-name>, <domain-name>, localhost, localhost.localdomain

# The next-hop destination(s) for non-local mail
# In this case we do not want to use postfix as a relay, so we leave it empty
relayhost =

# The list of "trusted" remote SMTP clients that have more privileges than "strangers".
# In particular, "trusted" SMTP clients are allowed to relay mail through Postfix.
# We limit these to local clients only. 
# NOTE that I don't think this value matters for our setup.
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128

# The maximal size of any local individual mailbox or maildir file, or zero (no limit).
mailbox_size_limit = 0

# The set of characters that can separate an email address localpart, user name, or a 
# .forward file name from its extension. 
#
# For example, with "recipient_delimiter = +", the software tries user+foo@example.com
# before trying user@example.com, user+foo before trying user, and .forward+foo before 
# trying .forward.
recipient_delimiter = +

# The local network interface addresses that this mail system receives mail on. 
# Specify "all" to receive mail on all network interfaces (default)
inet_interfaces = all

# specify that received mail should be stored using Maildir format, this matches our dovecot installation.
# Each mail user will have a mailbox at ~/Maildir.
home_mailbox = Maildir/
```

### Receiving mail

We need to specify recipients (the part before `@` in an email address) to `postfix` so that it knows how to route incoming mail to the right mail user's mailbox. Up until now we have treated mail users and recipients as interchangeable, but `postfix` allows for multiple recipients to map to the same mail user. This means that emails sent to `postmaster@markmizzi.dev` and `root@markmizzi.dev`, for e.g. may both be routed into the `root` user's mailbox.

To specify recipients, we use the following configuration parameter:
``` text
local_recipient_maps = proxy:unix:passwd.byname $alias_maps
```
- `proxy:unix:passwd.byname` ensures that each system user in `/etc/passwd` is also a recipient; this is important as our [`dovecot` setup](/docs/tutorial/email/maa) assumes that each system user is also a mail user. This option hence ensures that mail can be sent to each mail user defined in `dovecot`'s user database, using their username as the recipient.

- `$alias_maps` are optional lookup tables that are searched in order to determine the mail user corresponding to a particular recipient.

Because of the order in which we specified the options, the `/etc/passwd` file will be searched first.

One file in `$alias_maps` by default is `/etc/aliases`, which we can modify to map recipients to mail users. These are some common mappings added:
``` text
mailer-daemon: postmaster
postmaster: root
nobody: root
hostmaster: root
usenet: root
news: root
webmaster: root
www: root
ftp: root
abuse: root
```

With this setup emails sent to `postmaster@<domain-name>`, `www@<domain-name>`, `abuse@<domain-name>`, and so on will all be routed to `root`'s mailbox.

