---
layout: post
title: Email Spoofing and Prevention
modified: 2019-08-28
tags: [email]
categories: [sysadmin]
---

TL;DR: No, Sender Policy Framework (SPF) won't fix all your spoofing problems and DKIM by itself will prevent nothing.

## Email Crash Course

I am not able to explain email and SMTP in detail end-to-end in the time it would take for this post but there are some core things that need to be understood before we can move on. The best illustration is to use telnet to send an email via an SMTP server (I have indented the server responses to make it more clear:)

    EHLO mycomputer.mydomain.com
        250- redacted smtp.mydomain.com [10.10.10.10], pleased to meet you
    MAIL FROM: noreply@mydomain.com
        250 2.1.0 noreply@mydomain.com ... Sender ok
    RCPT TO: user@example.com
        250 2.1.5 user@example.com ... Recipient ok
    DATA
        354 Enter mail, end with "." on a line by itself
    From: Bob Smith <bob.smith@mydomain.com>
    Reply-to: <noreply@mydomain.com>
    Subject: Hello World
    
    Message body contents.
    .
        250 2.0.0 wASDDusO0124297 Message accepted for delivery

The above example is not only how we send messages through SMTP but is how applications and MTAs do it as well. Let's break it down:

* `HELO`/`EHLO` In the HELO command, the host sending the command identifies itself; the command may be interpreted as saying "Hello, I am <domain>" (and, in the case of EHLO, "and I support service extension requests")  

* `MAIL FROM` The "MAIL"  command initiates transfer of mail and identifies the sender. The address specified here is where errors are sent and will typically appear in the message source as the 'return-path'.   

* `RCPT TO` This identifies the recipient(s) and may be repeated as many times as necessary for multiple recipients. (Cc: or Bcc: would be delineated under "DATA".)  

* `DATA` Everything following DATA is considered to be message text until the end of data indicator (`.` on its own line followed by a blank line.) This is also where header items are specified in accordance with RFC 5322.  

  * `From:` this is the "header from" address and is what **will appear in most mail clients like Outlook.** It is optional and will be equal to the "MAIL FROM" address if omitted.  

  * `Reply-to:` another optional header item which can direct replies to a specific address.  

  * `Subject:` The message's subject as it will appear in the mail client.  

  * The rest is the message body followed by the end of data indicator (`.<CRLF>`) 

Other header items could have been included or we could have omitted any of the ones that were included. The only required items are the HELO, MAIL FROM, RCPT TO, and DATA and everything else is optional.

### The "Envelope" vs The "Header"

The "MAIL FROM" address and "RCPT TO" address(es) are referred to as the "envelope" and you may hear "MAIL FROM" called the "envelope from." Other names for the "MAIL FROM"  address  may include "MailFrom", "RFC5321.From", "RFC5321.MailFrom", and a number of others. In contrast, our `From: Bob Smith <bob.smith@mydomain.com>` above is the "header from" or may be called the "RFC5322.From", "from", "display from", or any number of other names. The "Bob Smith" portion is typically referred to as the "display name" but is mostly cosmetic (more on that later.)

**For the rest of this document:** The "envelope from" will be called the "MailFrom" address and the "header from" address will be called simply the "from" address.

## That's Allowed!?

Some things that really surprise people,

- You can specify whatever you want in the MailFrom or From address. (SMTP servers ***can*** be configured to disallow certain domains or only allow "authoritative domains.")  
- Either the MailFrom or From address can be null (`<>`) and there are circumstances where this is actually required.  
- You can specify a name with no address in the From address (`From: Bob Smith <>`) which will show up in some mail clients as "Bob Smith" business as usual.  
- You can format a From address like this: `From: Bob Smith <bob.smith@example.com> <hacker@hacker.su>` which is legal with the message actually *from* "hacker@hacker.su" but what most mail clients will show is "Bob Smith <bob.smith@example.com>."  

All of this is SMTP working as intended and while it's easy to see how it can be abused for malicious purposes it's important that SMTP functions this way (why is something for another post on another day.) Which leads us into...

## Spoofing

When we talk about spoofing there are three main types:

- Envelope From Spoofing ... In envelope from spoofing the MailFrom address is declared in a way that is meant to look legitimate. Usually the header from is omitted and the spoofed address appears in both places.

- Header From Spoofing ... In header from spoofing the MailFrom is a real address the attacker controls but they declare an address in the header from that is intended to look legitimate since it's what will appear in most mail clients anyway. Header from spoofing is more likely to get through filters which we'll get into later.

- Display Name Spoofing ... In display name spoofing both the MailFrom and From are addresses the attack controls (or that have no spoofing controls) but they use the display name to make the message look like it came from someone legitimate. This is the most likely to make it through filters and while it's the easiest for a human to detect it still works way too often. It's usually used when the message requires some sort of response. If we use our example above `From: Bob Smith <bob.smith@example.com>`, "Bob Smith" is the display name. Usually Outlook and other clients will show "Bob Smith <bob.smith@example.com>" the first time you get an email from an unknown person but once you reply it will usually truncate it to "Bob Smith" and hide the address which makes it easy to miss if you don't notice on first contact.

When it comes to spoofing the actual email address an attacker might use one of a number of different options:

1. Use the actual address such as "bob.smith@megacorp.com" hoping it makes it through filters.

2. Use a misspelling of the address like "bob.smith@megac0rp.com" (notice the zero.)

3. Use a completely different domain but a valid name "bob.smith@yahoo.com" and claim it's a personal address of the person they're pretending to be.

4. Use whatever address they want and rely on user stupidity "CeoEmail@hackersite.ru", "MicrosoftSupport@92n3n33.com", etc.

The most difficult spoofing to deal with as mail administrators is display name spoofing or spoofing where nothing about the address is actually spoofed and just relies on the user to herp-derp through it (2-4 above.) 

**A note on compromised mailboxes:** Another big problem is when the mailbox of a real user is compromised (successfully credential phished, virus, etc) and is used to send further phishing messages, spam, or malicious attachments. A compromised mailbox is not "spoofed" since the attacker is using the actual user's credentials.

**A note to domain owners:** Sending mail from your own domain but specifying an address other than your own is not "spoofing" unless you're not authorized to do so or have malicious intent. If I am an admin for Mega Corp and send service messages from noreply@megacorp.com that's business as usual not "spoofing." Spoofing is unauthorized or malicious. 

## Protection Mechanisms

### Sender Policy Framework (SPF)

In any post about spoofing or mail security recommendation number one is "implement SPF." Which is good advice, everyone should have SPF implemented. So what is it and what does it do?

ELI5: SPF is a DNS record a domain owner publishes that contains a list of servers from which they send email. The idea is that a receiving server sees an email from their domain, checks the list of legitimate sources, and if the server it came from isn't on the list it knows it's not legitimate.

An SPF record looks something like this:

    v=spf1 include:_netblocks.google.com include:_netblocks2.google.com include:_netblocks3.google.com ~all

Which is gmail.com's if you follow the redirect. They're contained in a txt record at the top of the domain so can do `dig <domain> txt | grep v=spf1` on Linux or `Resolve-DnsName <domain> txt | ? {$_.Strings -like "v=spf1*"}` in PowerShell on Windows to see a domain's SPF record.

There are a bunch of rules about how they're structured and a number of different mechanisms that an SPF record can contain that I won't get in to here but ultimately they resolve to a list of IPs that can be compared to the origin of an email. 

SPF is both useful for protecting against MailFrom spoofing of your domain towards your users but also to external destinations where it could harm your brand. Ie, Microsoft's SPF record stops MailFrom spoofing from @microsoft.com not just to @microsoft.com but also to you which helps stop some fake Microsoft phishing emails.

**Caveats:**

* **SPF is only concerned with the MailFrom address.**  It is not checked against the Header From address so does not in any way protect against header from spoofing or display name spoofing.
* The SPF RFC (7208) uses the word "SHOULD" a lot and rarely uses the word "MUST" so different receiving servers/filters may handle SPF failures differently. Even if you specify "hard fail" in your SPF record they may accept it on failure. (Lots of receiving mail servers treat `-all` and `~all` exactly the same.)

### DomainKeys Identified Mail (DKIM)

DKIM is a key-pair signing mechanism for the header of mail messages. When you send mail you attach a signature to the message using a private key which is compared to a public key published in DNS for your domain. DKIM adds authenticity to a message and guards against tampering with the header by down-stream mail servers. One of the benefits to working on the header is it survives SMTP relaying and auto-forwarding.

**DKIM does not directly prevent abusive / malicious behaviour.** DKIM is just a signature... If I hand you a letter with my signature on it there's added authenticity; However, if I hand you a letter without my signature if there's no requirement for the letter to be signed there's no reason to be suspicious. It's like SSL, just because a website doesn't have SSL doesn't mean it's fake but it's preferred when SSL is used.

### Domain Message Authentication Reporting & Conformance (DMARC)

As the name suggests there are reporting and conformance components to DMARC. For this post we're only concerned with the conformance component which tries to make up for the weaknesses in both SPF and DKIM. **The DMARC record of the domain in the header from address is used if it exists.** Like the above records it exists as a TXT record in DNS.

DMARC's conformance check is called "alignment" and it checks that the header from is "aligned" with other authenticated domains on the message either via DKIM or SPF. If **either** DKIM or SPF alignment passes DMARC evaluates as a "PASS."

**SPF Alignment:** The domain in the header from and envelope from must be the same (or sub-domains of the same parent domain if "relaxed") and must pass SPF.

**DKIM Alignment:**  DMARC requires a valid signature where the domain specified in the `d=` tag aligns with the sender's domain from the header from field.

**Caveats:**

* DMARC alignment is only enforced when your policy (`p=`) is set to "reject" or "quarantine".
* Lots of receiving mail servers still do not evaluate DMARC, evaluate only for reporting, or evaluate but don't report (it's a crap shoot.)
* DMARC can mess with automated messages like Out of Office replies and/or messages where the two from addresses have different domains but are still legitimate if only SPF+DMARC are implimented. It's generally best to implement DKIM along with DMARC to avoid SPF alignment issues.

## Solutions...

* Envelope from spoofing... SPF
* Header from spoofing... SPF + DMARC, DKIM + DMARC, or SPF + DKIM + DMARC. No one mechanism alone will be sufficient.
* Display name spoofing... Advanced threat filters, transport rules, and user training. None of the mechanisms care about the display name.
* Compromised mailboxes or "legitimate" senders.... Advanced threat filters, transport rules, and user training.

It is the owner of the domain who implements these technologies and it's up to receivers to configure their filters to check them and take appropriate action. **If you're getting mail spoofed from someone else's domain and they don't have SPF, adding SPF to your own domain isn't going to do anything.**

## Useful Links

**RFCs**

- [Simple Mail Transfer Protocol - RFC 5321](https://tools.ietf.org/html/rfc5321) ... If anyone links you to RFC 821 tell them it was made obsolete by RFC 2821 in 2001 which was made obsolete by 5321 in 2008 and they need to update their information.
- [Internet Message Format - RFC 5322](https://tools.ietf.org/html/rfc5322) ... Governs how message headers are formatted (again 822 was made obsolete by 2822 which was made obsolete by 5322... I'm looking at you MX Toolbox.)
- [Sender Policy Framework \(SPF\) - RFC 7208](https://tools.ietf.org/html/rfc7208) ... Obsoleted RFC 4408.
- [DomainKeys Identified Mail \(DKIM\) Signatures - RFC 6376](https://tools.ietf.org/html/rfc6376)
- [Domain-based Message Authentication, Reporting, and Conformance \(DMARC\) - RFC 7489](https://tools.ietf.org/html/rfc7489)

**Tools**

- https://mxtoolbox.com/ ... Various mail tools like SPF lookup, blacklist lookup, header parser, etc.
- https://kitterman.com/spf/validate.html ... One of the better public SPF validation tools.
- http://dmarcian.com/ ... A useful tool for parsing DMARC reports.
- https://dmarc.org/resources/deployment-tools/ ... DMARC deployment tools.

**NOTE:** There are other forms of spoofing and other mitigation techniques but those are for another day...
