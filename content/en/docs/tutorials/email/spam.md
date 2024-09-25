---
title: Setting up a spam filter
description: Tutorial for setting up a spam filter on Postfix using SpamAssassin
date: 2017-01-04
weight: 40
status: draft
---

I found [this tutorial](https://www.linuxbabe.com/mail-server/block-email-spam-check-header-body-with-postfix-spamassassin) helpful while writing this article.

## Installation

To install [SpamAssassin](https://spamassassin.apache.org), run the following commands:
``` bash
sudo apt-get update
sudo apt-get install -y spamassassin spamc spamass-milter
```

Add a system user for SpamAssassin to use (this is not done automatically):
``` bash
sudo useradd -m spamd
sudo usermod -L spamd
```

Enable and start SpamAssassin using:
``` bash
sudo systemctl enable spamassassin
sudo service spamassassin start
sudo systemctl enable spamass-milter
sudo service spamass-milter start
```

You can check that SpamAssassin is running correctly using
``` bash
sudo service spamassassin status
sudo service spamass-milter status
```

## Reloading Configuration

Whenever you modify SpamAssassin's configuration, make sure to restart it using
``` bash
sudo service spamassassin reload
sudo service spamass-milter reload
```
for the changes to take effect.

## Basic Configuration

SpamAssassin is configured using the file at `/etc/default/spamassassin`. Open this file and ensure that it contains the following values (you may have to edit or add values depending on whether they are already present):
``` text
# Options
# See man spamd for possible options. The -d option is automatically added.
#   '--create-prefs' specifies that user preferences files should be created.
#   '--max-children 5' specifies that a maximum of 5 children workers should be spawned.
#   '--username spamd' specifies that the user that spamd should run as is spamd
#   '--helper-home-dir /home/spamd/' specifies that the home dir used by spamd should be /home/spamd/
#   '-s /home/spamd/spamd.log' specifies the log file used by spamd.
OPTIONS="--create-prefs --max-children 5 --username spamd --helper-home-dir /home/spamd/ -s /home/spamd/spamd.log"

# Cronjob
# Set to anything but 0 to enable the cron job to automatically update
# spamassassin's rules on a nightly basis
CRON=1
```

## Configuring `postfix` to work with SpamAssassin

Open `/etc/postfix/main.cf` and ensure that the following values are set:
``` text
# Milter configuration
milter_default_action = accept
milter_protocol = 6
smtpd_milters = local:spamass/spamass.sock
non_smtpd_milters = local:spamass/spamass.sock
```

If you are using the [OpenDKIM extension](/docs/tutorials/email/dkim), the values of `smtpd_milters=` and `non_smtpd_milters=` should already be set:
``` text
smtpd_milters = inet:localhost:12301
non_smtpd_milters = inet:localhost:12301
```
Change this to:
``` text
smtpd_milters = inet:localhost:12301,  local:spamass/spamass.sock
non_smtpd_milters = inet:localhost:12301, local:spamass/spamass.sock
```

Run
``` bash
sudo service spamassassin reload
sudo service spamass-milter reload
```
for the changes to take effect.

## Spam Rules

The filtering rules for SpamAssassin are highly configurable, and are specified by the user in the file `/etc/spamassassin/local.cf`.

We will set up the following configuration:
``` text
# By default, suspected spam messages will not have the Subject, From or To lines tagged to indicate spam. 
# By setting this option, the header will be tagged with STRING to indicate that a message is spam. 
# For the Subject header, this will be prepended to the original subject. 
rewrite_header Subject [***** SPAM _SCORE_ *****]

# Set the score required before a mail is considered spam. 
# n.nn can be an integer or a real number. 
# 5.0 is the default setting, and is quite aggressive; it would be suitable for a single-user setup, 
# but if you're an ISP installing SpamAssassin, you should probably set the default to be more conservative, like 8.0 or 10.0.
required_score          6.0

# Whether to use the naive-Bayesian-style classifier built into SpamAssassin.
use_bayes               1
# Whether SpamAssassin should automatically feed high-scoring mails (or low-scoring mails, for non-spam) into its learning systems.
bayes_auto_learn        1
```

Run
``` bash
sudo service spamassassin reload
sudo service spamass-milter reload
```
to make sure the changes to take effect.

For a full list of configuration options see [here](https://spamassassin.apache.org/full/4.0.x/doc/Mail_SpamAssassin_Conf.html)

## Verifying that SpamAssassin is working

To check that SpamAssassin is working, send an email to one of the mailboxes configured on your mail server, and then open it in a UA program, such as Thunderbird. There should be an option to `View Source` of the email. Click it, which will show you the raw text actually sent over SMTP. If SpamAssassin is working you should find two lines that look like the following:
``` text
X-Spam-Status: No, score=-0.9 required=6.0 tests=DKIM_SIGNED,DKIM_VALID,
	DKIM_VALID_AU,DKIM_VALID_EF,FREEMAIL_ENVFROM_END_DIGIT,FREEMAIL_FROM,
	HTML_MESSAGE,RCVD_IN_DNSWL_NONE,RCVD_IN_MSPIKE_H2,SPF_HELO_NONE,
	SPF_PASS autolearn=unavailable autolearn_force=no version=3.4.6
X-Spam-Checker-Version: SpamAssassin 3.4.6 (2021-04-09) on mail.markmizzi.dev
```

The `X-Spam-Status` header also gives useful information about the spam score of an email, the threshold spam score set, and any tests used to determine the spaminess of the email.

## Debugging

Logs for SpamAssassin are found `/home/spamd/spamd.log`, this is helpful to check when running into issues.