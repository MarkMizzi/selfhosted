---
title: Creating new users
description: Tutorial for creating new mail users
date: 2017-01-04
weight: 60
---

Creating new mail users is not straightforward with our setup, and while instructions are given in the rest of this tutorial, they are spread over multiple different pages. This page attempts to collect all the instructions for creating a new mail user in one place; for administration of your mail server long after you have set it up.

## Creating a new system user

Each mail user must have a corresponding system user; this is created using the commands:
``` bash
sudo useradd -m <username> -p <password>
sudo usermod -L <username>
```

Users created with these commands will have a home directory to store their mailbox in Maildir format, but cannot login on the mail server. Preventing login ensures that users cannot access and edit their own mails on the server, thus corrupting `dovecot` state.

## Creating a password

For security purposes, the new mail user needs a password for their email which differs from the system password. This can be setup by running the following command:
``` bash
printf "<mail-username>:`doveadm pw -s BLF-CRYPT -p '<password>'`\n" | sudo tee -a /usr/local/etc/passwd.replica
```
which uses the `doveadm` tool to convert a given password to a stored format according to the specified scheme (in this case Blowfish crypt or `BLF-CRYPT`), and then stores the `<mail-username>:<password>` pair in the specified `passwd-file` (in this case `/usr/local/etc/passwd.replica`).

Make sure that you select a strong password.

For more information on using `doveadm pw` see [here](https://doc.dovecot.org/2.3/configuration_manual/authentication/password_schemes/#generating-encrypted-passwords).

## Creating additional recipients

By default, a new mail user with username `<mail-username>` will receive mail at `<mail-username>@<domain-name>`. However, you may wish to add additional recipients for the user.

This can be done by adding lines to `/etc/aliases` in the form:
``` text
<recipient>: <mail-username>
```

For example if you want a newly created mail user `suline` to also receive male sent to `wife@<domain-name>` in their mailbox, you would add a line
``` text
wife: suline
```

## Configuring sender addresses

A mail user must also be configured with email addresses that it is allowed to send mail from. This configuration is managed in the file `/etc/postfix/sender_login`.

There is one line for each email address and each mail user such that the mail user is allowed to send mail from that email address, in the following format:
``` text
<email-address> <mail-username>
```

It is usual to add at least the email address `<mail-username>@<domain-name>` to this file for new mail users. So for example for a new mail user `suline`, add
``` text
suline@<domain-name> suline
```
to allow `suline` to send mail from `suline@<domain-name>`.

`postfix` uses an [indexed version](https://www.postfix.org/DATABASE_README.html#types) of this file, which must be updated using
``` bash
sudo postmap hash:/etc/postfix/sender_login
```

Reload `postfix` to ensure these changes take effect:
``` bash
sudo service postfix reload
```