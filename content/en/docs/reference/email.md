---
title: How email works
description: Reference material about the internal workings of email systems.
weight: 1
---

# Mail system architecture

Sending mail in a typical email setup looks like this:

![](/docs/reference/email-arch.png)

The different components in this diagram are:
- A user agent is a command line or GUI program used to compose, read, reply to and forward messages. It also stores mail locally on an end user's computer.

- A mail server is a dedicated computer used to send and receive mail over the Internet.

- Message transfer agents (MTAs) perform the actual mail transfer (sending and receiving) using the Simple Mail Transfer Protocol (SMTP). Mail is always sent from an MTA client to an MTA server, with the latter running as a service and listening on ports 25 or 587 of the mail server machine.

- Mailboxes (marked as Boxes in our diagram) are files or directories on the mail server which are used to store received mail. A mail server for a particular domain will typically have multiple mailboxes, with the destination mailbox for an email specified as part of the email address: `<mailbox-name>@<domain-name>`.

- Message Access Agents (MAAs) allow a user agent program to access received mail from the mail server. These typically use the IMAP or the POP3 protocols (see below).

The steps involved in sending an email are as follows:
1. A user agent (UA) program (usually running on a laptop, PC, or smartphone) is used to write and initiate sending of an email.

2. The UA program then uses a mail transfer agent (MTA) client to transfer the outgoing mail to an MTA server via SMTP.

3. The MTA server queues the outgoing email in a spool.

4. The outgoing email is picked up by an MTA client on the mail server. This client usually runs continuously, waiting for new emails to appear on the spool.

5. The email is sent by the MTA client over the internet to an MTA server on the destination mail server. The destination mail server is located using the `<domain-name>` in an email address `<mailbox-name>@<domain-name>`, as well as an MX DNS record (see below).

6. The MTA server at the destination mail server stores the received mail in the appropriate mailbox. The mailbox name is determined from the email address: `<mailbox-name>@<domain-name>`.

7. On receiving a request from an MAA client, the MAA server at the destination mail server retrieves all received mail from the appropriate mailbox.

8. It then forwards it to the MAA client.

9. The MAA client makes newly received mail available to the UA program.

Note that SMTP is a push protocol: mail is pushed from a client to a server until it reaches the destination mailbox; while IMAP and POP3 are pull protocols: mail is requested from the mailbox of the destination mail server by a client. This architecture is well suited for the Internet, where end users typically operate machines (such as PCs and smartphones) that are intended to run client (and not server) software. This also explains the division into MTAs and MAAs within a mail system.

# Simple Mail Transfer Protocol (SMTP)

# IMAP/POP3 

Additional links of interest:
- [RFC 9051: IMAP v4, rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 2683: IMAP4 Implementation Recommendations](https://datatracker.ietf.org/doc/html/rfc2683)
- [RFC 2177: IMAP4 IDLE command](https://datatracker.ietf.org/doc/html/rfc2177)
- [RFC 1081: POP3](https://datatracker.ietf.org/doc/html/rfc1081)
- [RFC 1939: POP3 (STD 53)](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 1957: Some Observations on Implementations of the Post Office Protocol](https://datatracker.ietf.org/doc/html/rfc1957)
