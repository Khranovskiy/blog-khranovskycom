---
title: 'Payment mvp для checkout за 3w'
date: '2026-04-08'
description: todo
draft: false
url: /payments/
---

Недавно мне задали вопрос: "Допустим, нужно добавить оплаты на простой e-commerce сайт. MVP за 3 недели, один инденер. Как бы ты это сделал?"

Если вы долго работали в большой платежной системе, такой вопрос звучит почти издевательски. 
В голове сразу всплывает не "checkout на сайт", а весь взрослый payment domain: PCI, 3DS, антифрод, refunds, chargebacks, ledger, accounting, reconciliation, support tooling, provider outages и ручной разбор хвоста edge-cases.
И на этом фоне формулировка "один инженер, 3 недели" выглядит не как MVP, а как что-то заведомо невозможное. Но потом понимаешь: проблема не в вопросе.
Проблема в том, что мы по привычке слышим "платежная платформа", а задача на самом деле про "checkout MVP поверх PSP".
<!-- 
![](img/2026-payments-plans.jpg)
-->

## Главный тезис

Далее расскажу какие есть design axes
на основе них выделил важные architectual characteristics
мы сузим scope
обозначим где нельзя срезать углы
проведем tradeoff analysis, 
какие задать вопросы на уточнение требований
рассмотрим пару реальных psp

## Короткий ответ 

todo

## Процесс сужения scope

Проведем мыслительный процесс: problem space → rough scope → candidate design-driving axes → scope tightening → active axes → archetypes → chosen envelope.

Мы не можем сузить scope вслепую, нужно сначала увидеть пространство архитектурных развилок. Сandidate design-driving axes помогают сузить scope, а scope определяет, какие axes останутся active.

Сначала задаём rough scope. Нужно сначала выбрать, какой кусок системы вообще проектируется.
Не: "вся платёжная платформа", а "checkout payments slice для simple e-commerce MVP".

Дальше нужно быстро посмотреть на пространство архитектурных развилок
- какие оси здесь вообще могут форкать архитектуру?
- где находятся границы упрощения?
- что именно мы можем сознательно отрезать?

Это ещё не active axes. Это candidate axes — возможные архитектурно значимые измерения, оси которые могли бы быть архитектурно значимыми в данной problem space.

Candidate axes
- hosted checkout vs custom form
- one PSP vs multi-PSP
- immediate charge vs auth/capture
- webhook-only vs webhook + reconciliation
- local transaction style vs saga/distributed workflow
- strong correctness vs looser UX-first approach
- manual ops vs automation

Tightened scope
Зная оси, можно осознанно выбрать:

- hosted checkout
- one PSP
- no custom card handling
- no saga on start
- no advanced refunds/disputes

Теперь scope становится не просто "узким", а осмысленно узким.

Active axes
- webhook-only vs webhook + reconciliation
- simpler vs richer payment state machine
- more manual ops vs more automation
- how hard we lean into idempotency/retry/recovery design

Остаются только те axes, которые внутри выбранного scope всё ещё реально меняют дизайн.

## Candidate axes

Выделил в отдельную статью [payments-design-axes]({{< ref "/posts/2026-payments-3w-p2" >}}).

## Active axes

После такого сужения scope большие развилки уже зафиксированы: это **не payment platform**, а **checkout MVP**, не multi-PSP, а **один провайдер**, не custom card flow, а **hosted checkout**. После этого у меня остаются только несколько **активных архитектурных осей** — то есть развилки, которые всё ещё реально меняют дизайн.

Первая ось — **webhook-only vs webhook + reconciliation**. Можно сделать “чистую” схему и полностью верить webhook-ам, а можно сразу закладывать repair path для lost webhook и ambiguous state. Для платежей я бы почти всегда выбирал второй вариант: он менее элегантный, но гораздо надёжнее. **Trade-off**: В платежах repairability важнее архитектурной чистоты. Цена решения — больше operational complexity.

Вторая ось — **простая vs более богатая state machine**. Для MVP хочется держать минимальный набор состояний и не тащить весь взрослый payment domain внутрь системы. Но слишком грубая модель тоже вредна: потом трудно разбирать инциденты и неясные статусы. Здесь нужен баланс, а не крайность.

Третья ось — **тонкий PSP-specific adapter vs более универсальная абстракция**. Если делать всё совсем “в лоб”, MVP выйдет быстрее. Если чуть отделить внутренний payment flow от конкретного API провайдера, система получится здоровее и дешевле в развитии. Для первой версии я бы выбрал тонкий, но всё-таки отдельный adapter — без попытки строить платформу под десять PSP.

Четвёртая ось — **manual ops vs automation**. Редкие хвостовые кейсы можно сначала чинить руками через простой admin/support flow, а можно сразу строить more automated recovery. Для MVP я бы сознательно оставил часть редких сценариев на manual handling, но не экономил бы на idempotency, webhook verification и reconciliation.

Именно эти оси уже формируют реальный shape решения. Не “как устроены платежи вообще”, а **каким именно получится checkout MVP внутри уже жёстко суженной рамки**.

## Archetypes

После того как scope сужен, полезно не прыгать сразу к одной “правильной архитектуре”, а сначала собрать несколько **archetype** — несколько связных форм решения, которые естественно получаются из выбранных active axes.

Для этой задачи я бы видел три archetype.

**Thin checkout integration** — самый узкий и быстрый вариант: hosted checkout одного PSP, минимум своей логики и ставка в основном на happy path.

**Robust checkout orchestrator** — всё ещё MVP, но уже с отдельным payment lifecycle, idempotency, webhook handling и reconciliation.

**Early payment platform** — более широкий вариант с ранней платформенной абстракцией, более богатой state machine и заделом под multi-PSP или более сложные payment flows.

Эти archetype полезны не потому, что один из них “правильный”, а потому что они помогают не смешивать разные классы решений в одну кучу.

## Chosen envelope

Для кейса “простой e-commerce, один инженер, 3 недели” я бы выбрал **robust checkout orchestrator**.

Это значит, что я всё ещё проектирую не payment platform, а **checkout payments slice** вокруг одного PSP. Платёж — **one-time**, checkout — **hosted**, карточные данные остаются на стороне провайдера. Наш backend отвечает за создание payment attempt, связь с order, webhook processing, idempotency, понятную state machine и reconciliation на случай lost webhook или ambiguous state.

При этом я сознательно оставляю за границей этой версии multi-PSP routing, stored cards, refunds, disputes, сложный auth/capture flow и прочую platform-level payment complexity.

## Как я сузил задачу до реального MVP

Изначально формулировка “добавить оплаты” звучит обманчиво просто. Если долго работал с большими платёжными системами, мозг мгновенно дорисовывает весь взрослый payment domain: PCI, антифрод, ledger, accounting, reconciliation, refunds, disputes, provider outages, support tooling и длинный хвост operational complexity. В такой рамке задача “один инженер, три недели” выглядит почти нереальной.

Поэтому первым делом я не пытался рисовать архитектуру, а сузил problem space. Мне нужно было ответить не на вопрос “как устроены платежи вообще”, а на вопрос “какой минимальный payment slice действительно нужен простому e-commerce сайту”.

В результате я зафиксировал более узкий scope: это не платёжная платформа, а **checkout payments slice**. Пользователь должен оплатить заказ через внешний PSP, а наша система — создать payment attempt, связать его с order, дождаться подтверждения и корректно перевести заказ в финальное состояние.

После этого стало понятно, какие большие развилки можно закрыть сразу. Я сознательно выбрал **one-time payment**, **одного PSP** и **hosted checkout**. Это означает: мы не собираем и не храним карточные данные, не строим custom payment form, не делаем multi-PSP routing и не тянем в первую версию refunds, disputes, stored cards и прочую platform-level complexity.

Когда scope сузился, вместе с ним сузилось и пространство trade-off’ов. Большие компромиссы уровня problem space — вроде *payment platform vs checkout MVP* или *hosted checkout vs custom form* — перестали быть предметом дальнейшего выбора: они уже стали частью рамки задачи. Зато остались более локальные архитектурные развилки внутри выбранного MVP: полагаться только на webhook или сразу добавлять reconciliation, делать совсем простую state machine или чуть более внятную, оставлять редкие хвостовые кейсы на manual ops или автоматизировать их с самого начала.

Из этих развилок для меня естественно сложились три archetype: **thin checkout integration**, **robust checkout orchestrator** и **early payment platform**. Первый слишком хрупкий для платежей, третий слишком широкий для такого срока. Поэтому разумный выбор для кейса “простой e-commerce, один инженер, три недели” — **robust checkout orchestrator**: всё ещё MVP, но уже с idempotency, webhook processing, понятной state machine и reconciliation.

Именно в этот момент абстрактная задача “добавить оплаты” превращается в конкретный выбранный envelope. Мы больше не обсуждаем весь payment domain целиком. Мы проектируем вполне определённую систему: checkout MVP вокруг одного PSP, с hosted checkout, webhook-driven confirmation, защитой от дублей и repair path для lost webhook и ambiguous state.

## Что мы сознательно не будем делать

custom card flow, multi-PSP, refunds/disputes, stored cards, antifraud, ledger/platform complexity

Чтобы задача вообще помещалась в формат “один инженер, три недели”, нужно не только понять, что строим, но и явно зафиксировать, **чего мы не строим**. В этой версии это не платёжная платформа и не попытка забрать весь payment domain под себя. Мы сознательно не делаем custom payment form и не собираем карточные данные у себя, а используем hosted checkout провайдера. Мы не берём multi-PSP routing, fallback между эквайерами, stored cards, recurring payments, refunds, disputes, antifraud, ledger/accounting-first модель и прочую platform-level payment complexity. Это не потому, что всё это неважно, а потому, что для checkout MVP это другой класс задачи.

## Где нельзя срезать углы

idempotency, server-side payment confirmation, webhook verification/dedup, payment↔order linkage, state machine, reconciliation.

При этом “узкий scope” не означает, что можно халтурить в correctness-critical местах. В платежах есть несколько вещей, на которых как раз нельзя экономить. Нельзя полагаться только на redirect пользователя как на подтверждение успешной оплаты — source of truth должен быть на серверной стороне. Нельзя оставлять create-payment path без idempotency, иначе первый же retry или двойной клик превратится в риск дубля. Нельзя относиться к webhook-ам как к идеально надёжному каналу: нужны верификация, deduplication и repair path для lost webhook или ambiguous state. И нельзя терять связь между payment и order или оставлять систему без внятной state machine, иначе любой сбой быстро превращается в support pain и ручной хаос.

Если совсем коротко, то граница такая: **мы режем scope, но не режем correctness**. Упрощать можно почти всё, кроме тех мест, где ошибка превращается в двойное списание, потерянный статус или сломанный order state.

## Roadmap на 3 недели

На первой неделе я бы собрал **happy path**. Зафиксировал бы простую модель `order` и `payment`, определил бы минимальную state machine, сделал бы create-payment endpoint и интеграцию с hosted checkout у провайдера. К концу этой недели пользователь уже должен уметь нажать Pay, а backend — создать payment attempt и отправить его на checkout page PSP.

На второй неделе я бы занялся тем, что превращает интеграцию в реальную платёжную систему, а не в демо. То есть webhook endpoint, server-side confirmation, idempotency, обновление `payment` и `order`, защита от дублей и базовая операционная видимость. Именно здесь появляется настоящий source of truth для результата оплаты.

Третью неделю я бы потратил не на новые фичи, а на **repairability**. Добавил бы reconciliation для lost webhook и ambiguous state, прошёлся бы по самым дорогим failure cases — double click, retry, duplicate webhook, повторная попытка оплаты уже оплаченного заказа — и довёл бы систему до состояния, где её можно не только показать, но и поддерживать.

Итог первой версии для меня выглядел бы так: один PSP, hosted checkout, create payment, webhook-driven completion, idempotency, внятная state machine, reconciliation и минимальный support/debugging path.

Всё остальное я бы сознательно оставил **после первой версии**: второй PSP, refunds, stored cards, recurring payments, более богатую payment state machine, платформенную абстракцию над провайдерами, antifraud и прочую platform-level complexity. Не потому, что это неважно, а потому, что это уже следующий класс задачи.

## Какой PSP я бы выбрал для европейского рынка

Если рынок — Европа, я бы в первой версии смотрел не на “самый известный глобальный PSP”, а на то, кто лучше закрывает **hosted checkout \+ SCA/3DS \+ локальные методы оплаты**. Мой первый кандидат — **Mollie**: у них базовый flow очень короткий — создать payment, отправить пользователя на hosted checkout, обработать webhook — а в наборе методов оплаты сразу виден сильный европейский фокус: iDEAL, Bancontact, EPS, SEPA Bank Transfer, SEPA Direct Debit, Trustly, TWINT, Vipps и другие локальные методы.

У Mollie это ещё и удобно с точки зрения прозрачности: на pricing page есть публичные тарифы. Для **domestic consumer debit & credit cards** у них указаны **1.20%** в pay-as-you-go и **€0.10 + 0.85%** на Pro-плане; для части EEA consumer card scenarios на странице также видны уровни вроде **1.80% \+ €0.25**. Это не универсальная “истина про стоимость эквайринга”, а полезный ориентир, который сразу делает разговор не абстрактным, а прикладным.

Если смотреть на следующий шаг после MVP, я бы уже рассматривал **Adyen**. У них базовый **Sessions flow**остаётся достаточно дружелюбным: в integration comparison он помечен как **Light** effort и требует **one request from your server**. При этом у Adyen уже явно чувствуется более зрелый европейский платежный контур: у них есть отдельный PSD2/SCA guide, где прямо сказано, что PSD2 требует **strong customer authentication** для онлайн-платежей в EEA, а рекомендуемый способ — **3D Secure**. По pricing у Adyen уже enterprise-style модель: на публичной pricing page фигурирует **Interchange++** и примеры вроде **$0.13 \+ Interchange++ \+ 0.60%**.

**Checkout.com** я бы поставил рядом как ещё один европейски релевантный путь, если нужен hosted-page подход без своей формы оплаты. В их docs Hosted Payments Page описан прямо: покупатель уходит на secure page, hosted by Checkout.com, а sensitive payment details не проходят через тебя. По pricing у них нет простой публичной flat-rate карточки, зато есть enterprise-style pricing без setup fees и account maintenance fees.

**Stripe** я бы не убирал совсем, но в европейской статье не делал бы его единственным дефолтом. У Stripe по-прежнему очень сильный hosted checkout: в docs **Checkout page** помечен как **Recommended**, а **Checkout Sessions API** — как путь для complete checkout flows. На pricing side у них есть standard pricing, а для больших объёмов — **custom pricing**, **IC+ pricing** и **volume discounts**. Если нужен глобальный devex-first вариант, Stripe остаётся сильным кандидатом; просто в EU-контексте я бы ставил его рядом с Mollie и Adyen, а не поверх них.

Если совсем коротко, мой shortlist для Европы выглядел бы так: **Mollie для первого MVP**, **Adyen когда уже важны acceptance и economics**, **Checkout.com как hosted-page enterprise alternative**, **Stripe как сильный глобальный вариант, но не обязательно как европейский дефолт**. А на прямой эквайринг я бы вообще не смотрел как на “следующий шаг после MVP”: до него имеет смысл доходить только тогда, когда processing cost и payment ownership уже стали отдельной бизнес-задачей.

## Ещё несколько практических замечаний

Для первого релиза я бы не спешил строить свой payment backoffice. У Stripe Dashboard уже есть базовый operational UI: поиск по ресурсам вроде customers, invoices и payouts, а возвраты можно делать прямо из Dashboard. Для MVP это важный способ не тратить неделю на служебный интерфейс, который пока не даёт новой бизнес-ценности. 

Если бы речь шла не про обычный e-commerce, а про SaaS или digital goods, я бы отдельно посмотрел на **Paddle** и **Lemon Squeezy** как на Merchant of Record. У Paddle они прямо пишут, что берут на себя sales tax/VAT там, где это требуется, а у Lemon Squeezy сказано, что как merchant of record они обычно сами закрывают sales tax reporting для продаж через их платформу. **PayPal** я бы рассматривал не как основной card-PSP path, а как отдельный wallet/button-style checkout: у них стандартный flow строится вокруг `createOrder` и buyer approval в PayPal Checkout. 

Есть и пара неочевидных кусков, которые новички часто недооценивают. Первый — **double payment**. Пользователь может нажать “Оплатить” дважды, фронт может повторить запрос после таймаута, сеть может оборваться в неудобный момент. У Stripe для этого есть официальный механизм idempotency keys: повторный запрос с тем же ключом возвращает тот же результат, а в Payment Intents docs они отдельно рекомендуют создавать ровно один PaymentIntent на order или customer session. Второй — **webhook handler**. Его недостаточно просто “принять JSON-ом”: у Stripe нужно проверять `Stripe-Signature` и endpoint secret, а при ошибке верификации отклонять событие. В платёжной интеграции это уже не техническая деталь, а часть correctness story. 

## Evolution

На старте задача маленькая не потому, что платежи простые, а потому, что мы жёстко фиксируем почти все оси сложности. Большая команда появляется тогда, когда эти оси снова начинают двигаться: добавляются рынки, способы оплаты, требования к correctness, fraud-control и operational reliability. В этот момент “принять оплату” превращается в полноценную денежную платформу.
