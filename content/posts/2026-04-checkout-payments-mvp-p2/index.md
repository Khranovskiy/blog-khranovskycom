---
title: "How to Narrow a Payment Problem: Candidate Axes, Tightened Scope"
date: "2026-04-15"
description: "A practical walkthrough of how to move from the vague prompt 'add payments' to a meaningful scope"
draft: false
url: "/payment-design-axes/"
tags:
  - payments
  - system-design
  - architecture
  - requirements
  - design-axes
  - scope
categories:
  - architecture
series:
  - payment-mvp
---

This article is a continuation of [the first part about a payment MVP for checkout]({{< ref "/posts/2026-04-checkout-payments-mvp" >}}).  
That one is a more practical breakdown of the case itself.  
This one looks at the same problem through a system design lens: candidate axes, tightened scope, active axes, archetypes, and chosen envelope.

## Candidate axes

### 1. System slice

X: checkout integration / thin orchestration layer<br>
Y: payment platform / internal payments domain

This is the highest-level axis. It determines whether we are building a narrow payment slice around a PSP, or immediately thinking in terms of routing, ledger, fraud, disputes, payouts, and platform concerns.

### 2. Payment lifecycle ownership

X: PSP-led flow<br>
Y: app-owned payment lifecycle

This axis asks who really owns the payment lifecycle.  
With X, the PSP remains the main owner of payment state, and the application mostly creates checkout sessions, stores provider references, and updates orders based on callbacks or webhooks.  
With Y, the application introduces its own payment entity, its own create-payment flow, its own state machine, and treats provider events as inputs into an internal payment domain.

This axis is important because it separates a thin checkout integration from a real internal payment orchestration layer.

### 3. Payment data collection

X: hosted checkout / provider-hosted page<br>
Y: custom payment form / direct card collection

With X, most of the complexity is pushed into the PSP.<br>
With Y, integration complexity, PCI surface, 3DS/SCA, tokenization, and UX ownership all grow sharply.

### 4. PSP topology

X: one PSP<br>
Y: multi-PSP / routing / fallback

This axis determines whether this is one integration with a simple flow, or orchestration across providers with routing rules, fallback, and provider-specific abstractions.

### 5. Payment lifecycle richness

X: immediate charge / sale<br>
Y: richer lifecycle: auth → capture / void / partial capture

If money is charged immediately, the flow is simpler.<br>
If auth/capture is needed, you get a more complex state machine and more failure/compensation logic.

### 6. Confirmation model

X: sync/redirect-first confirmation<br>
Y: async confirmation via webhook as source of truth

With X, UX feels simpler, but correctness is weaker.<br>
With Y, the system is more honest about the real payment state, but the flow becomes more complex.

### 7. Repair path

X: webhook-only<br>
Y: webhook + reconciliation / polling / repair jobs

This axis is about repairability.<br>
Do we want one “clean” event path, or must the system actively recover from lost webhooks, ambiguous timeouts, and stuck states?

### 8. Workflow coupling

X: order + payment close-coupled / one DB or nearly one transactional boundary<br>
Y: distributed workflow across order, inventory, payment, and fulfillment

This axis determines whether local transactions and retries are enough, or whether you already need orchestration-level thinking, and later maybe saga.

### 9. Money model

X: payment record as source of truth for checkout<br>
Y: ledger/accounting-first model

With X, a payment/order state machine is enough for a checkout MVP.<br>
With Y, you are already thinking about double-entry, accounting correctness, balance semantics, and financial reporting.

### 10. State model richness

X: simple state machine<br>
Y: rich payment domain model

With X, `created / pending / succeeded / failed / expired` is enough.<br>
With Y, you get `authorized`, `captured`, `partially_refunded`, `chargeback_pending`, `manual_review`, and so on.

### 11. Refund handling

X: no refunds or manual refunds<br>
Y: full refund lifecycle in-system

This axis changes the size of the problem very quickly.<br>
Refunds are no longer “just checkout”; they are already a separate chunk of payment operations.

### 12. Ops model

X: manual ops for tail cases<br>
Y: full automation of edge cases

With X, rare complex cases can be fixed manually through admin/support flows.<br>
With Y, the system handles more of the tail itself: retries, compensations, escalations, recovery workflows.

### 13. Abstraction depth

X: thin PSP-specific adapter<br>
Y: generic payment abstraction for many providers

With X, delivery is faster and there is less accidental complexity.<br>
With Y, portability is higher and growth into multi-PSP is easier, but the upfront cost is higher.

### 14. Infra style

X: relational DB + background jobs<br>
Y: queue/event-driven pipeline

With X, reasoning is simpler and MVP delivery is faster.<br>
With Y, decoupling, burst handling, and extensibility are better, but system complexity is higher.

### 15. Geography / market complexity

X: single country / few payment methods / simple currency setup<br>
Y: multi-country / many local payment methods / regulatory variation

This axis is not only about scale, but also about integration shape.<br>
Local methods, country-specific flows, and regulatory differences break the illusion of “just checkout” very quickly.

## Tightened scope

* **system slice:** checkout integration, not a payment platform  
* **payment lifecycle ownership:** not fully PSP-led; we keep a minimal internal payment lifecycle  
* **payment data collection:** hosted checkout  
* **PSP topology:** one PSP  
* **payment lifecycle:** immediate charge  
* **confirmation model:** async/webhook truth  
* **workflow coupling:** no saga at the start  
* **money model:** not ledger-first  
* **refunds:** manual or out of scope  
* **infra style:** Postgres + jobs

We are not designing a full payment platform, only a **checkout payments slice** for a simple e-commerce MVP. The task is not to build our own payment stack with the whole tail of domain complexity, but to let a user pay for an order through an external PSP and reflect the result correctly in our own system.

So I would narrow the scope immediately to a **one-time payment through one PSP with hosted checkout**. No card storage, no custom payment form, no multi-PSP routing, no refunds, disputes, or other platform-level payment complexity.

What stays inside the scope is only the necessary core: create a payment attempt, link it to the order, process the webhook, update statuses, and have a repair path for duplicates and lost events.

## Go back

Продолжение про активные архитектурные оси и выбранный design envelope находится в [первой части, начиная с Active axes]({{< ref "/posts/2026-04-checkout-payments-mvp" >}})
