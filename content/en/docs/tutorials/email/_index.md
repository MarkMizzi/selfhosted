---
title: Email
description: Tutorial for self hosting your own mail server
date: 2017-01-04
weight: 8
---

This tutorial will show you how to self host your own mail server on Ubuntu Server 22.04 LTS.

The following software will be used:

- [Dovecot](https://www.dovecot.org) is used as a Mail Access Agent (MAA)
- [Postfix](https://www.postfix.org) is used as a Mail Delivery Agent (MDA)
- [OpenDKIM](http://www.opendkim.org) is used for DKIM signing and verification of emails.
- [SpamAssassin](https://spamassassin.apache.org) is used as a spam filter.