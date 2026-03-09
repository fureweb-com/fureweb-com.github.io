# SaaS 프로젝트를 위한 Stripe 구독 결제 연동 가이드

Stripe를 사용하여 SaaS 프로젝트에 구독 결제를 연동하는 방법을 단계별로 정리했습니다. Stripe Checkout, Customer Portal, Webhook까지 실제 운영에 필요한 전체 플로우를 다룹니다.

---

## 사전 준비

- [Stripe](https://dashboard.stripe.com/) 계정
- Node.js 백엔드 서버 (Express, Fastify 등)
- Stripe CLI (로컬 웹훅 테스트용)

```bash
# Stripe CLI 설치 (macOS)
brew install stripe/stripe-cli/stripe

# 로그인
stripe login
```

---

## 1. Sandbox(테스트 환경) 생성

Stripe Dashboard에서 Sandbox를 생성합니다. Sandbox는 실제 결제가 발생하지 않는 격리된 테스트 환경입니다.

1. Dashboard 좌측 상단의 환경 선택 드롭다운 클릭
2. **+ New sandbox** 또는 **Create sandbox** 클릭
3. 이름 지정 후 생성

생성 후 **Developers → API keys**에서 키를 확인합니다:

| 키 | 용도 |
|----|------|
| **Publishable key** (`pk_test_...`) | 프론트엔드용 (Checkout Session 방식이면 미사용) |
| **Secret key** (`sk_test_...`) | 백엔드 서버에서 사용 |

> Secret key는 절대 프론트엔드에 노출하면 안 됩니다.

---

## 2. 상품(Product) 및 가격(Price) 생성

Dashboard에서 **Product catalog → Add product**로 상품을 생성합니다.

### 상품 생성 예시

| 항목 | 값 |
|------|------|
| Name | Pro |
| Description | 상품 설명 |
| Pricing model | Standard pricing |
| Price | $10.00 USD |
| Billing period | Monthly |

### Price ID 확인

상품 생성 후 해당 상품 페이지의 **Pricing** 섹션에서 Price를 클릭하면 `price_`로 시작하는 ID를 확인할 수 있습니다.

> **Product ID (`prod_xxx`)와 Price ID (`price_xxx`)는 다릅니다.**
> Checkout Session을 생성할 때 사용하는 것은 **Price ID**입니다.

---

## 3. 백엔드 연동

### 패키지 설치

```bash
npm install stripe dotenv
```

### 환경변수

```env
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_PRO_PRICE_ID=price_xxx
```

### Stripe 초기화

```javascript
require('dotenv').config()
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY)
```

### Checkout Session 생성

사용자가 구독을 시작할 때 Stripe Checkout 페이지로 리다이렉트합니다. Stripe이 결제 UI를 제공하므로 직접 카드 입력 폼을 만들 필요가 없습니다.

```javascript
app.post('/billing/checkout', async (req, res) => {
  const { priceId } = req.body

  // Stripe Customer 생성 (또는 기존 Customer 재사용)
  const customer = await stripe.customers.create({
    email: req.user.email,
    metadata: { userId: req.user.id },
  })

  const session = await stripe.checkout.sessions.create({
    mode: 'subscription',
    customer: customer.id,
    line_items: [{ price: priceId, quantity: 1 }],
    success_url: 'https://yourapp.com?billing=success',
    cancel_url: 'https://yourapp.com?billing=cancel',
    subscription_data: {
      metadata: { userId: req.user.id },
    },
  })

  res.json({ url: session.url })
})
```

프론트엔드에서는 응답받은 `url`로 `window.location.href`를 설정하면 됩니다.

### Customer Portal 세션 생성

기존 구독자가 플랜을 변경하거나 취소할 때 Stripe Customer Portal로 보냅니다.

```javascript
app.post('/billing/portal', async (req, res) => {
  const session = await stripe.billingPortal.sessions.create({
    customer: req.user.stripeCustomerId,
    return_url: 'https://yourapp.com',
  })

  res.json({ url: session.url })
})
```

---

## 4. Webhook 설정

Stripe에서 발생하는 이벤트(결제 완료, 구독 변경, 취소 등)를 서버에서 수신하려면 Webhook을 설정해야 합니다.

### 왜 Webhook이 필요한가요?

Checkout 완료 후 `?billing=success`로 리다이렉트되지만, 이 시점에 실제 결제 처리가 완료되었다는 보장이 없습니다. Webhook은 Stripe 서버에서 직접 보내는 이벤트이므로, 이를 통해 DB를 업데이트하는 것이 안전합니다.

### 로컬 개발 (Stripe CLI)

```bash
stripe listen --forward-to localhost:4000/billing/webhook
```

실행 시 출력되는 `whsec_...` 값을 `.env`의 `STRIPE_WEBHOOK_SECRET`에 설정합니다.

### 프로덕션

Stripe Dashboard → **Developers → Webhooks → Add endpoint**:

| 항목 | 값 |
|------|------|
| Endpoint URL | `https://api.yourapp.com/billing/webhook` |
| Events | `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted` |

생성 후 **Signing secret** (`whsec_...`)을 프로덕션 환경변수에 설정합니다.

> 로컬 CLI의 `whsec_`와 Dashboard Webhook의 `whsec_`는 별개의 값입니다.

### Webhook 엔드포인트 구현

Webhook 요청의 body를 **raw Buffer 상태**로 받아야 서명 검증이 가능합니다. JSON으로 파싱된 body를 사용하면 서명 검증에 실패합니다.

#### Express

```javascript
// express.json()보다 먼저 등록해야 합니다
app.post('/billing/webhook',
  express.raw({ type: 'application/json' }),
  async (req, res) => {
    const sig = req.headers['stripe-signature']
    let event

    try {
      event = stripe.webhooks.constructEvent(
        req.body,  // raw Buffer
        sig,
        process.env.STRIPE_WEBHOOK_SECRET,
      )
    } catch (err) {
      console.error(`Webhook signature verification failed: ${err.message}`)
      return res.status(400).send('Invalid signature')
    }

    switch (event.type) {
      case 'checkout.session.completed': {
        const session = event.data.object
        // session.customer, session.subscription 등으로 DB 업데이트
        break
      }
      case 'customer.subscription.updated': {
        const subscription = event.data.object
        // subscription.items.data[0].price.id로 플랜 변경 반영
        break
      }
      case 'customer.subscription.deleted': {
        const subscription = event.data.object
        // 플랜을 free로 변경
        break
      }
    }

    res.json({ received: true })
  }
)
```

#### Fastify

Fastify는 기본적으로 `application/json`을 자동 파싱하므로, Webhook 엔드포인트에서 raw Buffer를 받으려면 **별도의 플러그인 스코프**에서 content type parser를 오버라이드해야 합니다.

```javascript
// 별도 플러그인 스코프로 등록해야 다른 라우터에 영향을 주지 않습니다
fastify.register(async function webhookPlugin(app) {
  app.addContentTypeParser(
    'application/json',
    { parseAs: 'buffer' },
    (_req, body, done) => done(null, body),
  )

  app.post('/billing/webhook', async (request, reply) => {
    const sig = request.headers['stripe-signature']
    let event

    try {
      event = stripe.webhooks.constructEvent(
        request.body,  // Buffer 그대로 전달
        sig,
        process.env.STRIPE_WEBHOOK_SECRET,
      )
    } catch (err) {
      fastify.log.error(`Webhook signature verification failed: ${err.message}`)
      return reply.code(400).send({ error: 'Invalid signature' })
    }

    // 이벤트 처리 (위 Express 예시와 동일)

    reply.send({ received: true })
  })
})
```

> **주의**: `req.rawBody`에 저장하고 나중에 `request.raw.rawBody`로 접근하는 방식은 Fastify의 플러그인 스코프에서 동작하지 않을 수 있습니다. `request.body`에 Buffer를 직접 전달하는 것이 가장 확실한 방법입니다.

---

## 5. Customer Portal 설정

사용자가 플랜 변경, 결제 수단 변경, 구독 취소 등을 직접 관리할 수 있는 페이지를 Stripe이 제공합니다.

Stripe Dashboard → **Settings → Billing → Customer portal**

### Subscriptions

| 설정 | 권장값 |
|------|--------|
| Cancel subscriptions | 활성화 |
| Update subscriptions | 활성화 |
| When customers change plans | **Prorate charges and credits** |
| Charge timing | **Invoice prorations immediately at the time of the update** |
| Products | 전환 가능한 모든 상품 추가 |

### 기타

| 설정 | 설명 |
|------|------|
| Return URL | Portal에서 돌아올 URL (예: `https://yourapp.com`) |
| Payment methods | 결제 수단 변경 허용 |
| Invoices | 청구서 내역 조회 허용 |

> **"Update subscriptions" 또는 "Switch plans" 옵션이 보이지 않을 때:**
> - "Subscriptions" 섹션이 비활성화되어 있으면 먼저 토글을 켜주세요.
> - Products가 등록되어 있지 않으면 플랜 전환 옵션이 나타나지 않습니다.
> - Sandbox에서는 메뉴명이 다를 수 있습니다 ("Update subscriptions", "Switching plans", "Plan changes" 등).

---

## 6. 플랜 변경 흐름 설계

| 변경 방향 | 처리 방식 |
|-----------|-----------|
| 무료 → 유료 | Stripe Checkout으로 새 구독 생성 |
| 유료 → 다른 유료 (업그레이드/다운그레이드) | Customer Portal에서 플랜 변경 |
| 유료 → 무료 (취소) | Customer Portal에서 구독 취소 |

업그레이드/다운그레이드 시:
- **업그레이드**: 차액이 즉시 과금됩니다 (prorate 설정 시)
- **다운그레이드**: 남은 기간에 대한 크레딧이 발생하고 다음 결제에 반영됩니다

플랜 변경 시 Stripe가 `customer.subscription.updated` 웹훅을 발생시키므로, 이 이벤트에서 DB의 플랜 정보를 업데이트하면 됩니다.

---

## 7. 테스트

### 테스트 카드 번호

| 시나리오 | 카드 번호 |
|----------|-----------|
| 결제 성공 | `4242 4242 4242 4242` |
| 결제 실패 (거부) | `4000 0000 0000 0002` |
| 3D Secure 인증 필요 | `4000 0025 0000 3155` |
| 잔액 부족 | `4000 0000 0000 9995` |

- 만료일: 미래 아무 날짜 (예: `12/34`)
- CVC: 아무 3자리 (예: `123`)

### 테스트 플로우

1. 구독하기 클릭 → Stripe Checkout으로 이동
2. 테스트 카드로 결제 → 성공 URL로 리다이렉트
3. `stripe listen` 터미널에서 Webhook 이벤트 수신 확인
4. DB에서 플랜 변경 확인
5. Customer Portal에서 플랜 변경/취소 테스트

### Webhook 수동 트리거

```bash
# checkout.session.completed 이벤트
stripe trigger checkout.session.completed

# 구독 변경 이벤트
stripe trigger customer.subscription.updated

# 구독 취소 이벤트
stripe trigger customer.subscription.deleted
```

---

## 8. 프로덕션 체크리스트

라이브 전환 전에 확인해야 할 사항입니다:

- [ ] Stripe Dashboard에서 Live 모드로 전환
- [ ] Live 모드의 API keys로 환경변수 교체 (`sk_live_...`)
- [ ] Live 모드에서 상품/가격 재생성 (테스트 모드 데이터는 이관되지 않음)
- [ ] Live 모드의 Price ID로 환경변수 교체
- [ ] Dashboard에서 프로덕션 Webhook endpoint 등록 및 Signing secret 설정
- [ ] Customer Portal 설정 확인 (Live 모드에서 별도 설정 필요)
- [ ] Checkout의 `success_url`, `cancel_url`이 프로덕션 도메인인지 확인
- [ ] Portal의 `return_url`이 프로덕션 도메인인지 확인
- [ ] HTTPS 적용 확인 (Stripe는 HTTPS를 권장)

---

## 9. 자주 겪는 문제

### "No webhook payload was provided"

Webhook 서명 검증에 실패하는 가장 흔한 원인입니다. `stripe.webhooks.constructEvent()`에 전달하는 body가 raw Buffer가 아닌 JSON 파싱된 객체이면 발생합니다.

**해결**: Webhook 엔드포인트에서 body를 JSON으로 파싱하지 않고 raw Buffer 상태로 받아야 합니다. 위 4번의 코드 예시를 참고해주세요.

### Webhook은 수신되는데 DB가 업데이트되지 않음

- `checkout.session.completed` 이벤트에서 `session.subscription`으로 구독 정보를 조회하고 있는지 확인해주세요.
- `metadata`에 `userId`를 포함시켰는지 확인해주세요. Checkout Session의 `subscription_data.metadata`에 설정해야 Subscription 객체에서도 접근 가능합니다.

### Customer Portal에서 플랜 변경 옵션이 없음

- Customer Portal 설정에서 "Update subscriptions"가 활성화되어 있는지 확인해주세요.
- 전환 가능한 Products가 등록되어 있는지 확인해주세요.
- 상품이 1개뿐이면 전환할 대상이 없으므로 옵션이 나타나지 않습니다.

### Sandbox의 데이터를 Live로 이관하고 싶음

Stripe는 테스트 모드와 라이브 모드의 데이터를 완전히 분리합니다. 상품, 가격, 고객, 구독 등 모든 데이터를 라이브 모드에서 새로 생성해야 합니다.

---

## 참고 자료

- [Stripe Docs - Checkout](https://docs.stripe.com/checkout)
- [Stripe Docs - Customer Portal](https://docs.stripe.com/customer-management/activate-no-code-customer-portal)
- [Stripe Docs - Webhooks](https://docs.stripe.com/webhooks)
- [Stripe Docs - Testing](https://docs.stripe.com/testing)
- [Stripe CLI](https://docs.stripe.com/stripe-cli)
