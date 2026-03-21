---
layout: post
title: "How Your Phone Knows It's in a Different Country"
date: 2026-03-20 00:00:01 -0500
categories: everyday-tech networking
---

I drove from Seattle to Vancouver one week ago. Right after the border crossing, my phone went quiet for about 30 seconds — no signal, nothing. Then it came back. The carrier indicator at the top of the screen, which had said **AT&T** the whole drive up, now said **Bell**. I hadn't touched anything.

That 30-second gap, and then the automatic handoff to a completely different carrier in a different country, is actually a pretty interesting piece of distributed systems. Let me walk through it.

## What's on Your SIM

Your SIM identity (whether physical SIM or eSIM profile) includes an **IMSI** — International Mobile Subscriber Identity. It looks something like this:

```
310 410 1234567890
 ^   ^       ^
 |   |       └─ your subscriber ID within AT&T
 |   └─────────── AT&T's network code
 └─────────────── country code (310 = USA)
```

This is the identity your phone announces to every cell tower it talks to. The first six digits (country code + network code) are globally standardized, so any carrier in the world can look at your IMSI and know exactly which home network you belong to.

Your phone number isn't what the network uses internally. The IMSI is.

## How Your Phone Connects to a Tower

When your phone has signal, it's in a constant back-and-forth with the nearest tower. It's registered in two places:

- **HLR (Home Location Register)** — AT&T's central database. Knows your subscription tier, what services you're allowed to use, and which network you're currently on.
- **VLR (Visitor Location Register)** — a local database maintained by whatever network your phone is currently connected to. Holds a temporary record of you while you're in their coverage area.

In normal operation (you're in Seattle, on AT&T), both are in sync. The VLR is just AT&T's own local record of you being in a particular region.

## What Happens at the Border

As I drove toward the crossing, my phone was happily registered on an AT&T tower somewhere south of the border. Then a few things happened in sequence:

**1. Signal drops.**
AT&T's last tower fell out of range. My phone lost its registration. The 30-second clock starts here.

**2. The phone scans for available networks.**
With no registered carrier, my phone starts broadcasting and listening for nearby towers. It picks up several — some might still be AT&T towers just across the border, some are Bell, some are Rogers or Telus. It sees them all, signal strength and all.

**3. The phone picks a network.**
My phone knows it belongs to AT&T (that's in the SIM). It checks a priority list: first, any AT&T towers (there weren't any). Then, preferred roaming partners. AT&T and Bell have a **roaming agreement** — a business contract that says "our customers can use your towers when they're in Canada, and we'll settle the billing later." Bell is on AT&T's preferred list, so my phone latches onto Bell.

**4. Bell authenticates my SIM.**
Bell doesn't know me. But it can see my IMSI starts with `310 410`, which means I'm an AT&T subscriber. Bell sends a message to AT&T's HLR — essentially: "Hey, subscriber IMSI 310-410-XXXX just showed up on our network. Are they allowed to roam?"

This message travels over a protocol called **SS7** (Signaling System No. 7), a decades-old global backbone that carriers use to talk to each other. It's clunky and has well-documented security issues, but it's still the thing that makes international roaming work.

**5. AT&T responds.**
AT&T's HLR says yes, sends over my profile — what data speeds I'm entitled to, whether international voice is enabled, etc. It also updates its own record: "this subscriber is now on Bell's network in Canada."

**6. Bell creates a VLR record.**
Bell now has a local record of me. It assigns me a temporary roaming number (called an MSRN) so that incoming calls and data can be routed to me. My phone re-registers, signal bars come back, and the indicator flips to **Bell**.

The whole sequence looks like this:

```
[Phone]          [Bell Tower]          [AT&T HLR]
   |                   |                    |
   |── "I'm here" ────►|                    |
   |   (IMSI sent)     |                    |
   |                   |─── "Can this ─────►|
   |                   |     sub roam?"     |
   |                   |                    |
   |                   |◄── "Yes, here ─────|
   |                   |     is profile"    |
   |                   |                    |
   |◄── Registered ────|                    |
   |    (Bell VLR set) |                    |
```

That round-trip to AT&T's HLR — happening over SS7, across international carrier infrastructure — is most of those 30 seconds.

## Why Does AT&T Still Appear Anywhere?

On many phones, the status bar shows the **home carrier** (AT&T) even while roaming. On others it shows the **serving carrier** (Bell). It depends on the phone and carrier settings. Some show both: "AT&T / Bell" or "Roaming - Bell". The underlying mechanism is the same regardless.

## The Billing Side

This is where roaming agreements matter on the money side. Bell is providing the towers, but you're paying AT&T. Bell doesn't bill you directly — it bills AT&T after the fact. AT&T either passes that through to you as a roaming charge, bundles it into an international day pass, or absorbs it if you're on a plan that includes Canada. The whole settlement process happens separately from the real-time signaling.

## Why Is SS7 Still Running This?

This is the thing that surprised me when I first looked into it. SS7 was designed in the 1970s. It has no authentication between carriers — if you can connect to the SS7 network, you can send messages pretending to be any carrier. Security researchers have used this to redirect SMS messages, track phone locations, and intercept calls. It's genuinely bad.

The replacement is a protocol called **Diameter** (and more recently, the **5G Core** uses HTTP/2-based signaling). Modern networks are slowly migrating. But SS7 is still the glue between legacy carriers worldwide, including for the AT&T-to-Bell handshake I described above. It'll be around for a while.

## Wrapping Up

The 30 seconds of silence crossing into Canada isn't dead air — it's your phone negotiating entry into a foreign network on your behalf, with your home carrier vouching for you over a protocol older than most smartphones. The business contract (roaming agreement), the identity system (IMSI + HLR), and the signaling backbone (SS7) all have to line up for the handoff to work. When it does, it's almost invisible. When it doesn't, you're the person in Vancouver frantically turning airplane mode on and off.
