# When your Medallia Survey Is A Phishing Link

**How a scammer used a compromised Medallia survey account to create a phishing campaign**

*Analysis date: 2026-10-30 · Email received: 2026-10-24 (Sat) 20:43:59 PDT / 2026-10-25 03:43:59 UTC*

> **Handling note:** All identifying details (recipient address, per-recipient tokens, third-party contacts) have been redacted. All URLs are defanged. No landing page was ever rendered in a browser or interacted with; network requests were `curl` requests (headers-only, or plain GETs) with redirects not followed, and responses were read as plain text. No wallets were harmed in the making of this analysis.

---

## TL;DR

An email titled **"Updated Generation Standard Now Open for Migration"** told the recipient to "migrate" a crypto wallet by Monday, 26 October, or risk "exposure of seeds to brute-force." It was sent through a legitimate survey platform and passed SPF, DKIM, and DMARC, but it is phishing.

- It went out through a **real, legitimate survey platform** (Medallia), from what the headers indicate was a **United Rentals survey sandbox account**.
- The visible (HTML) body was a crypto lure. The hidden plain-text body was the **original, legitimate survey invitation**. Someone redecorated the living room and forgot there was a completely different basement.
- The "Start Migration" button bounces through a SendGrid click tracker to **`calbopat[.]com`**: a domain registered about three weeks earlier, hidden behind Cloudflare, and picky about who it talks to.
- The likely goal is **seed phrase theft**. That's inferred from the lure; the final landing page was never observed.

**Confidence meter**

| Claim | Confidence |
|---|---|
| This is phishing | High |
| The survey account was compromised or abused | Moderate (inferred from headers, not confirmed by the vendors) |
| Who did it | None (nothing here identifies anyone, and this writeup won't guess) |

---

## Timeline (UTC)

| When | Event |
|---|---|
| 2026-10-02 19:51 | `calbopat[.]com` registered (big plans) |
| 2026-10-03 | TLS certificates issued for the apex, `www`, and a wildcard (Cloudflare-managed). The site was set up the very next day. Efficient. |
| 2026-10-24 06:16 | Domain registration record updated (reason unknown) |
| 2026-10-25 03:43 | Phishing email sent; delivered 8 seconds later |
| 2026-10-26 | The "migration deadline" from the email body |
| 2026-10-30 06:38 | Tracking link **still live**, about five days after delivery |

---

## Finding 1: Authentication passed. Judgment not included.

| Check | Result | Domain |
|---|---|---|
| SPF | PASS | `sbx.sc4.medallia.com` (sending IP `199.102.178.227`) |
| DKIM | PASS | `express.medallia.com` (selector `medallia1`) |
| DMARC | PASS | `p=REJECT` |

These three checks prove the email really came from Medallia's infrastructure and wasn't altered on the way. They're bouncers who check that your ID is real. Nobody checks what you plan to do once you're inside.

The sandbox-style hostnames (`sbx`, `ursandbox2-unitedrentals`) and the `X-Medallia-Company-Name: United Rentals` header point to a United Rentals survey account as the source. The recipient is a personal Gmail address with no apparent connection to United Rentals.

## Finding 2: Two emails in a trench coat

The message is `multipart/alternative`, which means it carries two versions of the body. Mail clients show the HTML version. The plain-text version is the fallback, usually seen only by text-only clients and security tooling.

| | Plain-text part (hidden) | HTML part (what you see) |
|---|---|---|
| **Topic** | Survey invitation for an "Annual Management Meeting" workshop | Crypto wallet "migration" |
| **Ask** | Name two customers; complete a form by January 8th | Click "Start Migration" before Monday, 26 October |
| **Sign-off** | "Sincerely, United Rentals" | "Built and operated by our engineers · No third parties involved" |
| **Deadline** | Invitation expires October 31 | October 26 |
| **Greeting** | "Hello ," (the name placeholder was left empty) | None |

Evidence that the HTML was swapped into an existing template:

- The HTML `<title>` still says **"United Rentals AMM Preparation."** 
- The "Start Migration" button uses the **same SendGrid tracking token** as the "privacy policy" link in the plain-text part. A link in the template was most likely repointed.

Is this deliberate obfuscation? Probably not. It looks like someone edited the part they could see and never touched the part they couldn't. (Attackers *do* sometimes stuff the plain-text part with filler to dilute spam scoring, but this doesn't look like that. It looks like an oversight, not a plan.)

## Finding 3: Phishing bingo

| Technique | In this email |
|---|---|
| Manufactured urgency | Monday deadline, sent Saturday night. Nothing says "trustworthy engineering team" like a weekend ultimatum. |
| Fear | Unmigrated wallets "could face exposure of seeds to brute-force" |
| Money pressure | Miss the deadline and holders "cover their own migration fees" |
| Reassurance | "No fees," "all balances carry across exactly," "access is open to every user" |
| Authority | "Service Notice" banner, "Built and operated by our engineers" |
| Vague jargon | "Updated generation standard," "advanced applications," "Web3 partners." A wallet or protocol is never actually named. |
| Self-contradiction | Claims "no third parties involved," delivered by a third-party survey platform. Bold strategy. |

Free space in the middle of the card: the word "Web3."

The email never names a real product, which is convenient. It can go to everyone and be relevant to nobody.

## Finding 4: Four hops to nowhere

```
"Start Migration" button
  └─ hxxps://u2330170[.]ct[.]sendgrid[.]net/ls/click?upn=[REDACTED]
       └─ HTTP 302 Found
            └─ hxxps://calbopat[.]com
                 └─ HTTP 403 "Access Restricted" (custom block page)
                      └─ after 5 s: hxxp://kaspersky[.]fr/resource-center/definitions/what-are-bots
```

Observed via `curl -I` (first hop) and a plain GET with a browser user agent saved to a file (final hop). The SendGrid hop returned a `302` with `Location: hxxps://calbopat[.]com`.

## Finding 5: Bot detection with main-character energy

Requests to `calbopat[.]com` returned `403` both with curl's default user agent and with a Chrome-style one. The 403 body is a custom page generated by the site, not by Cloudflare:

- Title: **"Access Restricted."**
- Message: *"We've detected unusual traffic patterns from your connection,"* followed by a list of reasons: "VPN or proxy usage," "Automated browsing tools," "Suspicious network behavior."
- A script that redirects the visitor after five seconds to a Kaspersky article about bots.

That is a phishing site telling the person investigating the phishing site that *their* behavior is suspicious. The audacity is almost admirable.

Supporting details:

- A `PHPSESSID` cookie suggests a PHP application behind the response.
- Aggressive `no-store` and `noindex, nofollow, nosnippet, noarchive` headers.
- `Access-Control-Allow-Origin: *` with broad allowed methods, which is unusual for a landing page.

Interpretation: **visitor filtering ("cloaking")**. Anything that doesn't look like an intended victim gets turned away and sent somewhere harmless. The decoy is the digital equivalent of "go read a book." Kaspersky is an innocent bystander here, used purely as a prop.

*Limitation:* the real page was never seen, so its contents (and the seed-phrase goal) are inferred from the email.

## Finding 6: Infrastructure

| Attribute | Value |
|---|---|
| Registered | 2026-10-02 (about 3 weeks before the email, which in domain years is a toddler with a Cloudflare account) |
| Last updated | 2026-10-24 06:16 UTC (about 21 hours before the email; reason unknown) |
| Registrar | Dominet (HK) Limited (IANA ID 3775) |
| Registrant | Privacy-redacted; state/country shown as California, US (unverified) |
| Name servers | `nola.ns.cloudflare.com`, `aaden.ns.cloudflare.com` |
| Hosting | Behind Cloudflare; origin IP not exposed |
| Certificates | All issued 2026-10-03: apex, `www`, and a wildcard (via Cloudflare's certificate service) |

A new domain isn't proof of anything by itself. Combined with everything above, it fits the pattern. The wildcard certificate hides any sibling hostnames from the certificate logs. The Cloudflare name server pair is account-specific and could be a pivot for finding related domains, which was not pursued.

---

## Assessment

**Established from the evidence**
- The email was delivered through Medallia's infrastructure and passed authentication.
- The HTML and plain-text bodies describe two unrelated messages.
- The "Start Migration" button redirects via SendGrid to `calbopat[.]com`.
- `calbopat[.]com` is a recent registration, Cloudflare-fronted, and denies requests that look automated or proxied.

**Inferred, not confirmed**
- The survey account or template was compromised or abused.
- The landing page collects wallet credentials or seed phrases.

**Not determined (and not going to be guessed at)**
- Who is behind it, how they got access, and how many people received the email.

Nothing here suggests wrongdoing by Medallia or United Rentals. The evidence points to their infrastructure being misused, which makes them closer to victims than culprits.

---

## Indicators of compromise (defanged)

| Type | Value | Note |
|---|---|---|
| Domain | `calbopat[.]com` | Redirect target; Cloudflare-fronted |
| URL | `hxxps://u2330170[.]ct[.]sendgrid[.]net/ls/click?upn=[REDACTED]` | Click tracker; 302 to the above |
| Decoy URL | `hxxp://kaspersky[.]fr/resource-center/definitions/what-are-bots` | Benign; used as a decoy only |
| IP | `199.102.178.227` | Medallia sending MTA (legitimate infrastructure, abused) |
| Host | `mta-176-227[.]mx[.]medallia[.]com` | Same |
| Sender domain | `express[.]medallia[.]com` | Legitimate; DKIM selector `medallia1` |
| Display name | `SeedTrustWallet Migration` | Spoofed brand-style name |
| Subject | `Updated Generation Standard Now Open for Migration` | |
| Page title | `Access Restricted` | Block page on `calbopat[.]com` |
| Name servers | `nola[.]ns[.]cloudflare[.]com`, `aaden[.]ns[.]cloudflare[.]com` | Possible pivot |

---

## Recommendations

**If you received this email (don't be the sequel)**
- Don't click anything. Report it as phishing in your mail client and delete it.
- If you entered a seed phrase or connected a wallet, **move your funds to a brand-new wallet immediately**. A compromised seed phrase can't be fixed, only abandoned.
- **Rule 1:** your seed phrase goes in zero web forms, ever. **Rule 2:** see Rule 1.

**For the affected platforms (Medallia / United Rentals)**
- Review the survey account for unauthorized logins or API activity, and audit recently edited invitation templates.
- Check for invitations sent to recipients outside the expected distribution list.
- Rotate credentials and tokens for the affected account.

**For email and survey platform operators (detection ideas)**
- Flag messages where the plain-text and HTML parts diverge materially in topic. (This one was a gift.)
- Alert on link-target changes in existing templates, especially to newly registered domains.
- Treat sandbox accounts sending to external recipients as anomalous.
- Passing DMARC is the floor, not the ceiling.

**Where to report**

| Who | How |
|---|---|
| Medallia | Their security or abuse contact (check their website for the current address) |
| United Rentals | Their security team, as the account owner |
| SendGrid (Twilio) | Abuse reporting, to disable the tracking link |
| Registrar | `domainabuse@service.aliyun.com` (from the whois record) |
| Cloudflare | cloudflare.com/abuse (forwarded to the hosting provider) |
| Google | "Report phishing" in Gmail; submit the domain to Safe Browsing |

---

## Appendix: Redacted headers (relevant lines only)

```text
From: SeedTrustWallet Migration <SeedTrustWalletMigration@express.medallia.com>
To: [REDACTED]@gmail.com
Subject: Updated Generation Standard Now Open for Migration
Date: Sat, 24 Oct 2026 20:43:59 -0700 (PDT)
Message-ID: <[REDACTED]@sbx.sc4.medallia.com>
Return-Path: <void-bounce-[REDACTED]=gmail.com@sbx.sc4.medallia.com>
Received: from mta-176-227.mx.medallia.com (mta-176-227.mx.medallia.com. [199.102.178.227])
          by mx.google.com with ESMTPS (TLS1_3, TLS_AES_256_GCM_SHA384)
Received-SPF: pass (google.com: domain of void-bounce-[REDACTED]@sbx.sc4.medallia.com
          designates 199.102.178.227 as permitted sender)
Authentication-Results: mx.google.com;
          dkim=pass header.i=@express.medallia.com header.s=medallia1;
          spf=pass smtp.mailfrom="void-bounce-[REDACTED]@sbx.sc4.medallia.com";
          dmarc=pass (p=REJECT sp=REJECT dis=NONE) header.from=express.medallia.com
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=express.medallia.com;
          s=medallia1; [signature values omitted]
Content-Type: multipart/alternative
Feedback-ID: unitedrentals-[REDACTED]:ursandbox2-unitedrentals-unitedrentals.sbx.sc4.medallia.com-unitedrentals:MedalliaInc
List-Unsubscribe: [REDACTED]
X-Medallia-Company-Name: United Rentals
X-Medallia-Company-URL-Name: unitedrentals
```

---

*Redactions applied: recipient address and all variants, per-recipient survey/unsubscribe tokens, the SendGrid tracking token, the message ID and feedback UUID, and a third-party employee contact address. Gmail-internal routing IDs and cryptographic signature values were omitted as noise.*
