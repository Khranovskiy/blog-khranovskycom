---
title: "How to Add Checkout Payments in 3 Weeks as a Solo Engineer"
date: "2026-04-08"
description: "How to narrow the problem down to a real payment MVP for e-commerce: one PSP, hosted checkout, idempotency, webhooks, and reconciliation without trying to build a payment platform."
draft: false
url: "/checkout-payments-mvp/"
tags:
  - payments
  - checkout
  - mvp
  - psp
  - ecommerce
  - system-design
categories:
  - architecture
series:
  - payment-mvp
---

I was recently asked a question:

Suppose you need to add payments to a simple e-commerce site. MVP in 3 weeks, one engineer. How would you do it?

Your reaction to that question depends a lot on where you’re coming from.

If you’ve spent a long time working on large payment systems, the whole grown-up payment domain immediately comes to mind: PCI, 3DS, fraud, refunds, chargebacks, ledger, accounting, reconciliation, support tooling, provider outages, and the long tail of operational complexity. Against that backdrop, “one engineer in 3 weeks” sounds almost unrealistic.

But if you’ve ever added payments to a small online store or a simple website, the problem looks much more grounded: pick a PSP, embed checkout, and get the happy path working.

That’s exactly where the confusion starts. Some people hear **“payment platform”**, others hear **“checkout on top of a PSP.”** This article is about the second case.
<!-- 
![](img/2026-payments-plans.jpg)
-->

## How I narrowed the problem down

For problems like this, it helps not to jump straight into “payment architecture,” but to walk through a short path:

**problem space → rough scope → candidate design-driving axes → scope tightening → active axes → archetypes → chosen envelope.**

The idea is simple: you can’t narrow scope blindly. First you need to see the space of architectural forks, and only then can you consciously cut away the unnecessary parts. **Candidate axes help narrow scope, and scope determines which axes remain active.**

For this problem, my rough scope looked like this: not “the whole payment platform,” but a **checkout payments slice for a simple e-commerce MVP**. Then I looked at candidate axes - hosted checkout vs custom form, one PSP vs multi-PSP, webhook-only vs webhook + reconciliation, manual ops vs automation, and others - and fixed a narrower envelope: **one-time payment, one PSP, hosted checkout, no custom card handling, no multi-PSP routing, and no platform-level payment complexity**.

Once you narrow it down like that, the big forks are already closed. Inside the chosen MVP, only the axes that still materially change the design remain. Those are what become the **active axes**.

I moved **candidate axes** and **scope tightening** into a separate write-up in [payments-design-axes]({{< ref "/posts/2026-04-checkout-payments-mvp-p2" >}}). From here on, what matters more is not the whole solution space, but **what the chosen checkout MVP actually looks like**.

## Active axes

After narrowing scope like that, the big forks are already fixed: this is **not a payment platform**, but a **checkout MVP**; not multi-PSP, but **one provider**; not a custom card flow, but **hosted checkout**. After that, only a handful of **active architectural axes** remain - forks that still materially change the design inside the chosen boundaries.

The first axis is **PSP-led flow vs app-owned payment lifecycle**.  
You can build a very thin integration where the PSP effectively owns the payment flow, and your backend only creates a checkout session and updates the order based on the result. Or you can take internal control over the payment lifecycle: create a separate `payment` entity, your own create-payment flow, your own state machine, and treat webhook processing as an inbound event stream. This axis more than anything else separates a simple checkout integration from a real internal payment orchestration layer.

The second axis is **webhook-only vs webhook + reconciliation**.  
You can build a “clean” design and trust webhooks entirely, or you can add a repair path up front for lost webhooks and ambiguous state. For payments, I would almost always choose the second option: it is less elegant, but much more reliable. **Trade-off:** repairability matters more than architectural purity. The price is higher operational complexity.

The third axis is **simple vs richer state machine**.  
For an MVP, you want to keep the set of states minimal and avoid pulling the whole mature payment domain into the system. But a model that is too coarse is also harmful: later it becomes hard to investigate incidents, stuck states, and unclear transitions. You need balance here, not an extreme.

The fourth axis is **manual ops vs automation**.  
You can initially handle rare tail cases manually through a simple admin/support flow, or you can start building a more automated recovery path immediately. For an MVP, I would consciously leave some rare scenarios to manual handling, but I would not cut corners on correctness-critical things: idempotency, webhook verification, and repair path.

Those are the axes that shape the actual solution. Not “how payments work in general,” but **what the checkout MVP looks like inside an already tightly narrowed frame**.

## Archetypes

Once scope is narrowed, it helps not to jump straight to one “correct architecture,” but first to assemble a few **archetypes** - a few coherent forms of the solution that naturally emerge from the chosen active axes.

I would draw the boundaries not by component count, but by three questions:

1. Who owns the payment lifecycle - the PSP or our system?  
2. Where does the source of truth live for intermediate and final states?  
3. Which failure cases does the system handle itself, and which does it leave to manual handling?

That gives three archetypes.

**Thin checkout integration** is the variant where we primarily **connect PSP checkout to the order flow**, rather than build our own payment domain. The PSP effectively owns the payment lifecycle, while our backend does the minimum necessary: creates the checkout session, stores a link to the provider payment/session, receives the webhook, and moves the order to `paid` or `failed`. In this model, payment usually does not become a rich standalone entity, reconciliation is either absent or manual, and rare ambiguous cases are handled through the provider dashboard and manual ops. This is the fastest path, but it mostly relies on the happy path plus minimal guardrails.

**Lean Payment Orchestrator** is still a checkout MVP around one PSP, but here payment becomes a **first-class internal entity**, not just an order attribute. Our system explicitly owns the payment lifecycle: it creates a payment attempt, keeps its own state machine, processes webhooks as inbound events, deduplicates them, stores the `order ↔ payment` relationship, supports idempotency on the create-payment path, and can clean up lost webhooks or ambiguous state through reconciliation. The PSP still handles the actual money movement and hosted checkout, but the internal system is now responsible not just for the happy path, but for correctness and repairability in the payment flow.

**Early payment platform** is a broader variant where we start designing not just a checkout slice, but the beginnings of an internal payment platform. This is where a more general abstraction over PSPs, a richer payment domain model, groundwork for multi-PSP routing, more complex payment flows, and a gradual move beyond a single checkout use case tend to appear.

These archetypes are useful not because one of them is “correct,” but because they help avoid mixing up three very different classes of solution: **connect payments**, **take control of the payment lifecycle**, **start building a platform**.

## Chosen envelope

For the case “simple e-commerce, one engineer, 3 weeks,” I would choose **Lean Payment Orchestrator**, but in a very restrained form.

That means I am still designing not a payment platform, but a **checkout payments slice** around a single PSP. The payment is **one-time**, the checkout is **hosted**, and card data stays on the provider side. But I also do not want the system to end at “we created a checkout session and hope everything arrives.” So my backend still takes responsibility for a few things: creating and storing the payment attempt, linking payment to order, idempotency on the create-payment path, webhook processing, a simple internal state machine, and a minimal reconciliation/repair path for lost webhooks or ambiguous state.

At the same time, I deliberately leave multi-PSP routing, stored cards, refunds, disputes, complex auth/capture flows, and the rest of platform-level payment complexity outside the boundaries of this version.

## What we consciously will not build

For this problem to fit at all into the “one engineer, three weeks” format, you need to make explicit not only what you are building, but also what you are **not** building. This version is not a payment platform, and not an attempt to pull the whole payment domain in-house.

We deliberately do not build a custom payment form and do not collect card data ourselves. Instead, we use the provider’s hosted checkout. We do not take on multi-PSP routing, fallback across acquirers, stored cards, recurring payments, refunds, disputes, antifraud, a ledger/accounting-first model, or the rest of platform-level payment complexity. Not because those things are unimportant, but because that is already a different class of problem.

## Where you cannot cut corners

A narrow scope does not mean you can be sloppy in correctness-critical places. In payments, there are a few things you really cannot save effort on.

You cannot rely only on the user redirect as confirmation of a successful payment - the source of truth needs to be on the server side. You cannot leave the create-payment path without idempotency, or the first retry or double click becomes a duplicate risk. You cannot treat webhooks as a perfectly reliable channel: you need verification, deduplication, and a repair path for lost webhooks or ambiguous state. And you cannot lose the relationship between payment and order or leave the system without a clear state machine, or every failure quickly turns into support pain and manual chaos.

In short: **we cut scope, but we do not cut correctness**. You can simplify almost everything except the places where an error becomes a double charge, a lost status, or a broken order state.

In practice, the most expensive mistakes here are usually very mundane. A user can click “Pay” twice, the frontend can retry after a timeout, and it is very easy to write a webhook handler too naively - without signature verification, deduplication, and safe reprocessing. That is why even in a very narrow MVP you cannot save effort on idempotency, server-side confirmation, and repair path.

## What I would realistically build in 3 weeks

If the task is “add payments to an e-commerce site in 3 weeks as a solo engineer,” I would not try to build a payment platform in that time. I would aim for a much narrower result: a **working checkout MVP** around a single PSP.

In the first week, I would build the **happy path**: define a simple `order` and `payment` model, decide on a minimal state machine, implement a create-payment endpoint, and integrate with the provider’s hosted checkout. By the end of that week, the user should already be able to click Pay, and the backend should be able to create a payment attempt and send the user to the PSP checkout page.

In the second week, I would work on the things that turn an integration into a real payment system rather than a demo: webhook endpoint, server-side confirmation, idempotency, payment and order updates, duplicate protection, and basic operational visibility. That is where the real source of truth for payment outcome appears.

I would spend the third week not on new features, but on **repairability**. I would add reconciliation for lost webhooks and ambiguous state, walk through the most expensive failure cases - double click, retry, duplicate webhook, a second payment attempt for an already paid order - and get the system to a state where it can not only be shown, but also supported.

The outcome of the first version, for me, would look like this: one PSP, hosted checkout, create payment, webhook-driven completion, idempotency, a clear state machine, reconciliation, and a minimal support/debugging path.

Everything else I would deliberately leave **after the first version**: a second PSP, refunds, stored cards, recurring payments, a richer payment state machine, a platform abstraction over providers, antifraud, and the rest of platform-level complexity. Not because they are unimportant, but because that is already the next class of problem.

### What else helps make 3 weeks realistic

For an MVP like this to fit into such a short timeline at all, you need to save effort not only on scope, but also on the surrounding service layer. I would not rush to build my own payment backoffice: PSPs usually already provide a basic operational UI - payment history, statuses, search, manual refunds. For a first release, that is a good way not to spend time on an internal interface that does not yet create new business value.

The same applies to the checkout flow. Hosted checkout does not just reduce the amount of work, it also avoids opening a whole separate stream of work around card data handling, PCI DSS, 3DS, and your own payment UI. In a large company, that kind of surface can take a long time and separate teams to build. In an MVP, we consciously leave it outside the system boundary.

## Which PSP I would choose for the European market

If the market is Europe, in the first version I would not look at “the most famous global PSP,” but at whoever best covers **hosted checkout**, **SCA/3DS**, and **local payment methods**.

My first candidate would be **Mollie**. Their flow is short and straightforward: create a payment, send the user to hosted checkout, process the webhook. For European e-commerce, this is especially convenient because of their strong focus on local payment methods: iDEAL, Bancontact, EPS, SEPA Bank Transfer, SEPA Direct Debit, Trustly, TWINT, Vipps, and others. Mollie also has public pricing, which is useful at the MVP stage.

- Docs: [Accepting payments](https://docs.mollie.com/docs/accepting-payments)  
- Docs: [Hosted checkout](https://docs.mollie.com/docs/hosted-checkout)  
- Pricing: [Mollie pricing](https://www.mollie.com/pricing)

If I were looking at the next step after MVP, I would then consider **Adyen**. Their basic **Sessions flow** is still reasonably friendly, but overall it is already a more mature European payment stack. They have separate documentation for **PSD2 / SCA**, and the pricing model is closer to enterprise from the start.

- Docs: [Build your integration](https://docs.adyen.com/online-payments/build-your-integration)  
- Docs: [PSD2 / SCA compliance and implementation guide](https://docs.adyen.com/online-payments/psd2-sca-compliance-and-implementation-guide)  
- Pricing: [Adyen pricing](https://www.adyen.com/pricing)

I would place **Checkout.com** nearby as another Europe-relevant hosted-page option. Their Hosted Payments Page fits well when sensitive payment data should not pass through your backend. It is already a more enterprise-oriented path, and the pricing looks like it as well.

- Docs: [Hosted Payments Page](https://www.checkout.com/docs/payments/accept-payments/accept-a-payment-on-a-hosted-page)  
- Pricing: [Checkout.com pricing](https://www.checkout.com/pricing)

I would not remove **Stripe** entirely, but I would not make it the only default in a Europe-focused article. Stripe still has a very strong hosted checkout, very good developer experience, and a clear integration path. If you need a global devex-first option, it is still a strong candidate. But in an EU context, I would put it alongside Mollie and Adyen, rather than automatically above them.

- Docs: [Online payments / Checkout](https://docs.stripe.com/payments/online-payments)  
- Docs: [Stripe Checkout](https://stripe.com/payments/checkout)  
- Pricing: [Stripe pricing](https://stripe.com/pricing)

Very briefly, my shortlist for Europe would look like this: **Mollie for the first MVP**, **Adyen when acceptance and economics start to matter**, **Checkout.com as a hosted-page enterprise alternative**, **Stripe as a strong global option, but not necessarily the European default**.

I would not consider direct acquiring at all as the “next step after MVP.” It only makes sense when processing cost and payment ownership have already become a distinct business problem.

## Why real payment systems still grow into many teams

At the start, the problem is small not because payments are simple, but because we hold almost all of the axes of complexity fixed. A large team appears when those axes start moving again: more markets, more payment methods, higher correctness requirements, fraud control, and operational reliability. At that point, “accept a payment” turns into a real money platform.

## What’s next

I want to keep this text as a **living document** that will keep getting refined and edited over time. The more I think about this problem and run it through both interview and practical contexts, the more I want not to “close the topic,” but to gradually turn it into a more precise working model.

My next steps are to separately break down the archetypes, look in more detail at integrations with specific PSPs, and describe the directions in which the architecture evolves: at what point a checkout MVP stops being enough and starts growing into a broader payment system.
