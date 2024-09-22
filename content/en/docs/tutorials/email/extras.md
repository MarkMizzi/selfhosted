---
title: Extras
description: Extra information/setup that is good to know about for your self hosted mail server.
date: 2017-01-04
weight: 50
---

## Verifying that SPF and DKIM are working

To verify that SPF and DKIM are working on mail sent from your self hosted e-mail setup, send a test mail to a Gmail account.

Go to the received mail on Gmail (check spam as it may be there), and click the three dots on the top right, then click `Show original`:

![](/docs/tutorials/email/test-email.png)

This will open a page which contains the raw content of the sent mail. At the top of the page there is a header with information, including details about whether the sent mail passed SPF, DKIM and DMARC checks:

![](/docs/tutorials/email/test-email-original.png)

Make sure that all of these are marked as `PASS`.

## How to stop mail from your server being marked as spam

Verifying that SPF and DKIM are working correctly (see above) is a good first step to ensuring mail from your server is not marked as spam. However it is not the end-all-be-all, as even email which passes SPF, DKIM and DMARC checks can still be marked as spam.

What can you do if this is happening to mail sent from your server?