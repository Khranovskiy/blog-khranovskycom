---
title: "How to Add Checkout Payments in 3 Weeks as a Solo Engineer"
date: "2026-04-14"
description: "How to narrow the problem down to a real payment MVP for e-commerce: one PSP, hosted checkout, idempotency, webhooks, and reconciliation without trying to build a payment platform."
draft: false
url: "/checkout-payments-mvp/"
tags:
  - payments
  - system-design
params:
  fontsize: small
---

I was recently asked a question:

Suppose you need to add payments to a simple e-commerce site. MVP in 3 weeks, one engineer. How would you do it?

Your reaction depends a lot on where you're coming from.

If you've spent years inside a large payment system, the whole grown-up domain immediately floods in: PCI, 3DS, fraud, refunds, chargebacks, ledger, accounting, reconciliation, support tooling, provider outages, and the long tail of operational complexity. Against that backdrop, "one engineer, 3 weeks" sounds almost absurd.

But if you've ever added payments to a small online store, the problem looks much more grounded: pick a PSP, embed checkout, get the happy path working.

That's exactly where the confusion starts. Some people hear **"payment platform"**, others hear **"checkout on top of a PSP."** This article is about the second case.
<!--
![](img/2026-payments-plans.jpg)
-->

## How I narrowed the problem down

For problems like this, it helps not to jump straight into "payment architecture," but to walk a short path first:

**problem space → rough scope → candidate design axes → scope tightening → active axes → archetypes → chosen envelope.**

The idea: you can't narrow scope blindly. First you need to see the space of architectural forks — then you can consciously cut the unnecessary parts. Candidate axes help narrow scope, and scope determines which axes stay active.

My rough scope was: not "the whole payment platform," but a **checkout payments slice for a simple e-commerce MVP**. From there I looked at candidate axes — hosted checkout vs custom form, one PSP vs multi-PSP, webhook-only vs webhook + reconciliation, and others — and landed on a narrower envelope: one-time payment, one PSP, hosted checkout, no custom card handling, no multi-PSP routing, none of the platform-level complexity.

Once you do that, the big forks are already closed. What's left are the axes that still materially change the design inside the chosen boundaries — the **active axes**.

I moved the full candidate axes breakdown and scope tightening into [a separate write-up]({{< ref "/posts/2026-04-checkout-payments-mvp-p2" >}}). Here I care more about what the chosen checkout MVP actually looks like.

## Active axes

After narrowing scope, the major forks are fixed: this is **not a payment platform**, but a **checkout MVP**; one provider, not multi-PSP; hosted checkout, not a custom card flow. A handful of **active axes** remain — forks that still meaningfully change the design inside these boundaries.

**PSP-led flow vs app-owned payment lifecycle.**
You can build a thin integration where the PSP effectively owns the payment flow — your backend creates a checkout session and updates the order based on the result. Or you take internal control: a separate `payment` entity, your own create-payment flow, your own state machine, webhook processing as an inbound event stream. More than anything else, this fork separates a simple checkout integration from a real internal payment orchestration layer.

**Webhook-only vs webhook + reconciliation.**
You can trust webhooks entirely, or you can add a repair path for lost webhooks and ambiguous state. For payments, I'd almost always pick the second option. It's less elegant, but much more reliable. Repairability matters more than architectural purity — the price is higher operational complexity.

**Simple vs richer state machine.**
For an MVP, keep the set of states minimal. But too coarse a model is also harmful: later it becomes hard to investigate incidents, stuck states, unclear transitions. You need balance, not an extreme.

**Manual ops vs automation.**
You can handle rare tail cases manually through a simple admin/support flow, or build a more automated recovery path from the start. For an MVP, I'd consciously leave some rare scenarios to manual handling — but not cut corners on correctness-critical things: idempotency, webhook verification, repair path.

## Archetypes

Once scope is narrowed, it helps not to jump straight to one "correct architecture," but to sketch a few **archetypes** — coherent forms of the solution that naturally emerge from the active axes.

I'd draw the boundaries with three questions:

1. Who owns the payment lifecycle — the PSP or our system?
2. Where does the source of truth live for intermediate and final states?
3. Which failure cases does the system handle itself, and which does it leave to manual handling?

That gives three archetypes.

**Thin checkout integration.** We primarily connect PSP checkout to the order flow, rather than build our own payment domain. The PSP owns the payment lifecycle; our backend does the minimum: creates the checkout session, stores a link to the provider payment, receives the webhook, moves the order to `paid` or `failed`. Payment doesn't become a rich standalone entity, reconciliation is absent or manual, and rare ambiguous cases go to the provider dashboard. Fastest path — but mostly relies on the happy path plus minimal guardrails.

**Lean Payment Orchestrator.** Still a checkout MVP around one PSP, but payment becomes a **first-class internal entity**, not just an order attribute. Our system explicitly owns the payment lifecycle: creates a payment attempt, keeps its own state machine, processes webhooks as inbound events, deduplicates them, stores the `order ↔ payment` relationship, supports idempotency on the create-payment path, and can repair lost webhooks or ambiguous state through reconciliation. The PSP still handles the actual money movement and hosted checkout — but the internal system is now responsible for correctness and repairability, not just the happy path.

**Early payment platform.** A broader variant where we start designing not just a checkout slice, but the beginnings of an internal payment platform: a more general abstraction over PSPs, a richer payment domain model, groundwork for multi-PSP routing, more complex payment flows.

These archetypes are useful not because one is "correct," but because they help avoid mixing up three very different classes of solution: **connect payments**, **take control of the payment lifecycle**, **start building a platform**.

## Chosen envelope

For "simple e-commerce, one engineer, 3 weeks," I'd choose **Lean Payment Orchestrator** — but in a restrained form.

Still a **checkout payments slice** around a single PSP. One-time payment, hosted checkout, card data stays on the provider side. But I also don't want the system to end at "we created a checkout session and hope everything arrives." So the backend still takes responsibility for a few things: creating and storing the payment attempt, linking payment to order, idempotency on the create-payment path, webhook processing, a simple internal state machine, and a minimal repair path for lost webhooks or ambiguous state.

Multi-PSP routing, stored cards, refunds, disputes, complex auth/capture flows — all of that stays outside this version.

## What we won't build

For this to fit into "one engineer, three weeks," you need to be explicit about what's out of scope, not just what's in.

No custom payment form, no collecting card data ourselves — we use the provider's hosted checkout. No multi-PSP routing, fallback across acquirers, stored cards, recurring payments, refunds, disputes, antifraud, ledger/accounting, or any of the rest of platform-level payment complexity.

## Where you can't cut corners

A narrow scope doesn't mean you can be sloppy in correctness-critical places.

Don't rely on the user redirect as confirmation of a successful payment — the source of truth needs to be server-side. Don't leave the create-payment path without idempotency, or the first retry or double-click becomes a duplicate risk. Don't treat webhooks as a perfectly reliable channel: you need verification, deduplication, and a repair path. Don't lose the relationship between payment and order, or leave the system without a clear state machine — every failure quickly turns into support pain and manual chaos.

We cut scope. We don't cut correctness.

The most expensive mistakes here are usually mundane. A user clicks "Pay" twice. The frontend retries after a timeout. The webhook handler is written too naively — without signature verification, deduplication, safe reprocessing. That's why even in a narrow MVP you can't skip idempotency, server-side confirmation, and a repair path.

## What I'd realistically build in 3 weeks

I wouldn't try to build a payment platform. I'd aim for a **working checkout MVP** around a single PSP.

**Week one: the happy path.** Define a simple `order` and `payment` model, pick a minimal state machine, implement a create-payment endpoint, integrate with the provider's hosted checkout. By the end of the week, a user should be able to click Pay, and the backend should create a payment attempt and send them to the PSP checkout page.

**Week two: the things that make it real.** Webhook endpoint, server-side confirmation, idempotency, payment and order updates, duplicate protection, basic operational visibility. This is where the actual source of truth for payment outcome appears.

**Week three: repairability.** Reconciliation for lost webhooks and ambiguous state, walkthrough of the most expensive failure cases — double click, retry, duplicate webhook, a second payment attempt for an already-paid order. Getting the system to a state where it can be supported, not just demoed.

The outcome of the first version: one PSP, hosted checkout, create payment, webhook-driven completion, idempotency, a clear state machine, reconciliation, and a minimal support/debugging path.

Everything else — a second PSP, refunds, stored cards, recurring payments, a richer payment state machine, a platform abstraction over providers, antifraud — I'd leave for after the first version. That's already the next class of problem.

### What helps make 3 weeks realistic

Save effort not only on scope, but on the surrounding service layer. Don't rush to build your own payment backoffice: PSPs usually provide a basic operational UI — payment history, statuses, search, manual refunds. For a first release, that's a good way not to spend time on an internal interface that doesn't yet create new business value.

Same for the checkout flow. Hosted checkout doesn't just reduce the amount of work — it avoids opening a whole separate stream around card data handling, PCI DSS, 3DS, and your own payment UI. In a large company, that surface takes separate teams to build. In an MVP, you leave it outside the system boundary.

## Which PSP I'd choose for the European market

If the market is Europe, I wouldn't default to "the most famous global PSP." I'd look at whoever best covers **hosted checkout**, **SCA/3DS**, and **local payment methods**.

My first candidate would be **Mollie**. Create a payment, send the user to hosted checkout, process the webhook — that's the whole flow. For European e-commerce, their focus on local payment methods is especially useful: iDEAL, Bancontact, EPS, SEPA Bank Transfer, SEPA Direct Debit, Trustly, TWINT, Vipps, and others. Public pricing is a nice bonus at the MVP stage.

- [Accepting payments](https://docs.mollie.com/docs/accepting-payments)
- [Hosted checkout](https://docs.mollie.com/docs/hosted-checkout)
- [Mollie pricing](https://www.mollie.com/pricing)

For the next step after MVP, I'd look at **Adyen**. Their basic **Sessions flow** is still reasonably friendly, but overall it's a more mature European payment stack — separate documentation for **PSD2/SCA**, pricing closer to enterprise from the start.

- [Build your integration](https://docs.adyen.com/online-payments/build-your-integration)
- [PSD2/SCA compliance](https://docs.adyen.com/online-payments/psd2-sca-compliance-and-implementation-guide)
- [Adyen pricing](https://www.adyen.com/pricing)

**Checkout.com** is worth a look as another hosted-page option. Their Hosted Payments Page fits well when sensitive payment data shouldn't pass through your backend. More enterprise-oriented.

- [Hosted Payments Page](https://www.checkout.com/docs/payments/accept-payments/accept-a-payment-on-a-hosted-page)
- [Checkout.com pricing](https://www.checkout.com/pricing)

**Stripe** — I wouldn't take it off the list, but I also wouldn't make it the default in a Europe-focused article. Very strong hosted checkout, excellent developer experience, clear integration path. A strong global option. But in an EU context I'd put it alongside Mollie and Adyen, not automatically above them.

- [Online payments / Checkout](https://docs.stripe.com/payments/online-payments)
- [Stripe Checkout](https://stripe.com/payments/checkout)
- [Stripe pricing](https://stripe.com/pricing)

My shortlist for Europe: **Mollie for the first MVP**, **Adyen when acceptance and economics start to matter**, **Checkout.com as a hosted-page enterprise alternative**, **Stripe as a strong global option, but not the European default**.

Direct acquiring isn't a "next step after MVP." It only makes sense when processing cost and payment ownership have become a distinct business problem.

## Why real payment systems still grow into many teams

At the start, the problem is small — not because payments are simple, but because we hold almost all axes of complexity fixed. A large team appears when those axes start moving again: more markets, more payment methods, higher correctness requirements, fraud control, operational reliability. At that point, "accept a payment" turns into a real money platform.

## What's next

I plan to separately break down the archetypes, look in more detail at integrations with specific PSPs, and describe how the architecture evolves — at what point a checkout MVP stops being enough and starts growing into a broader payment system.
