---
title: "Как сузить payment-задачу: candidate axes, tightened scope и chosen envelope"
date: "2026-04-08"
description: "Практический разбор того, как из размытой формулировки 'добавить оплаты' перейти к осмысленному scope, активным архитектурным осям и выбранному design envelope."
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

Эта статья — продолжение [первой части про payment MVP для checkout]({{< ref "/posts/2026-04-checkout-payments-mvp" >}}).  
Там — более практический разбор самого кейса.  
Здесь — system design взгляд: candidate axes, tightened scope, active axes, archetypes и chosen envelope.

## Candidate axes

### 1. System slice

X: checkout integration / thin orchestration layer<br>
Y: payment platform / внутренний платёжный контур

Это самая верхняя ось. Она определяет, строим ли мы узкий payment slice вокруг PSP или сразу думаем категориями routing, ledger, fraud, disputes, payouts и platform concerns.

### 2. Payment data collection

X: hosted checkout / provider-hosted page<br>
Y: custom payment form / direct card collection

При X большая часть сложности уезжает в PSP.<br>
При Y резко растут integration complexity, PCI-surface, 3DS/SCA, токенизация и контроль над UX.

### 3. PSP topology

X: one PSP<br>
Y: multi-PSP / routing / fallback

Ось определяет, это одна интеграция с простым flow или уже orchestration между провайдерами с routing rules, fallback и provider-specific abstractions.

### 4. Payment lifecycle richness

X: immediate charge / sale<br>
Y: richer lifecycle: auth → capture / void / partial capture

Если деньги списываются сразу, flow проще.<br>
Если нужен auth/capture, появляется более сложная state machine и больше failure/compensation logic.

### 5. Confirmation model

X: sync/redirect-first confirmation<br>
Y: async confirmation через webhook как source of truth

При X кажется проще UX, но correctness слабее.<br>
При Y система честнее по отношению к реальному состоянию платежа, но сложнее flow.

### 6. Repair path

X: webhook-only<br>
Y: webhook + reconciliation / polling / repair jobs

Эта ось про repairability.<br>
Нужен ли один "красивый" event path или система обязана сама дочищать lost webhook, ambiguous timeout и зависшие состояния.

### 7. Workflow coupling

X: order + payment close-coupled / одна БД или почти один транзакционный контур<br>
Y: distributed workflow между order, inventory, payment, fulfillment

Эта ось определяет, хватает ли local transactions и retries, или уже нужен orchestration-level thinking, а позже, возможно, saga.

### 8. Money model

X: payment record as source of truth for checkout<br>
Y: ledger/accounting-first model

При X достаточно payment/order state machine для checkout MVP.<br>
При Y уже думаем про double-entry, accounting correctness, balance semantics, financial reporting.

### 9. State model richness

X: simple state machine<br>
Y: rich payment domain model

При X хватает created / pending / succeeded / failed / expired.<br>
При Y появляются authorized, captured, partially_refunded, chargeback_pending, manual_review и т.д.

### 10. Refund handling

X: no refunds or manual refunds<br>
Y: full refund lifecycle in-system

Эта ось быстро меняет объём задачи.<br>
Refunds — это уже не просто checkout, а отдельный кусок payment operations.

### 11. Ops model

X: manual ops for tail cases<br>
Y: full automation of edge cases

При X редкие сложные случаи можно чинить руками через admin/support flow.<br>
При Y система сама обрабатывает больше хвоста: retries, compensations, escalations, recovery workflows.

### 12. Abstraction depth

X: thin PSP-specific adapter<br>
Y: generic payment abstraction for many providers

При X быстрее delivery и меньше accidental complexity.<br>
При Y выше переносимость и проще рост в multi-PSP, но выше upfront cost.


### 13. Infra style

X: relational DB + background jobs<br>
Y: queue/event-driven pipeline

При X проще reasoning и быстрее MVP.<br>
При Y лучше decoupling, bursts, extensibility, но выше system complexity.

### 14. Geography / market complexity

X: single country / few payment methods / simple currency setup<br>
Y: multi-country / many local payment methods / regulatory variation

Это axis не только про scale, но и про shape интеграции.<br>
Local methods, country-specific flows и regulatory differences быстро ломают “просто checkout”.

## Tightened scope

* **system slice:** checkout integration, не payment platform  
* **payment data collection:** hosted checkout  
* **PSP topology:** one PSP  
* **payment lifecycle:** immediate charge  
* **confirmation model:** async/webhook truth  
* **workflow coupling:** без saga на старте  
* **money model:** не ledger-first  
* **refunds:** manual or out of scope  
* **infra style:** Postgres + jobs

Мы не проектируем платёжную платформу целиком, а только **checkout payments slice** для простого e-commerce MVP. Задача — не строить свой payment stack со всем хвостом доменной сложности, а дать пользователю оплатить заказ через внешний PSP и корректно отразить результат оплаты у себя.

Поэтому scope я бы сразу сузил до **one-time payment через одного PSP с hosted checkout**. Без хранения карт, без custom payment form, без multi-PSP routing, refunds, disputes и прочего platform-level payment complexity.

Внутри scope остаётся только необходимое: создать payment attempt, связать его с order, обработать webhook, обновить статусы и иметь repair path для дублей и потерянных событий.

## Go обратно 

Если хочется вернуться от framework обратно к более практическому разбору, вот ссылка на [первую часть, начиная с Active axes]({{< ref "/posts/2026-04-checkout-payments-mvp#active-axes" >}})
