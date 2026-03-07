# 💳 LinkedBoost AI — Stripe Entegrasyon Kılavuzu

## Bu dosyayı oku — Stripe'ı çalıştırmak için her şey burada.

---

## ADIM 1: Stripe Hesabı Aç (5 dakika)

1. https://stripe.com adresine git
2. "Start now" → hesap oluştur (emailin yeterli)
3. Dashboard → **Developers → API keys**
4. İki key kopyala:
   - `pk_test_...` → Publishable Key (frontend'de kullanılır)
   - `sk_test_...` → Secret Key (sadece backend'de, HİÇBİR ZAMAN frontend'e koyma!)

---

## ADIM 2: Ürünleri Stripe'ta Oluştur (10 dakika)

Stripe Dashboard → **Products → Add product**

### Pro Plan Oluştur:
- Name: `LinkedBoost Pro`
- Pricing: Recurring
  - Monthly: $29.00 / month
  - Annual: $228.00 / year (= $19/mo)
- Her oluşturduktan sonra **Price ID** kopyala: `price_xxxxx`

### Agency Plan Oluştur:
- Name: `LinkedBoost Agency`
- Monthly: $79.00 / month
- Annual: $612.00 / year

**Kopyala ve kaydet:**
```
STRIPE_PRICE_PRO_MONTHLY    = price_xxxxx
STRIPE_PRICE_PRO_ANNUAL     = price_xxxxx
STRIPE_PRICE_AGENCY_MONTHLY = price_xxxxx
STRIPE_PRICE_AGENCY_ANNUAL  = price_xxxxx
```

---

## ADIM 3: Backend Kur (Node.js/Express)

### 3a. Proje klasörü oluştur:
```bash
mkdir linkedboost-backend
cd linkedboost-backend
npm init -y
npm install express stripe cors dotenv
```

### 3b. `.env` dosyası oluştur:
```env
STRIPE_SECRET_KEY=sk_test_BURAYA_SECRET_KEY
STRIPE_WEBHOOK_SECRET=whsec_BURAYA_WEBHOOK_SECRET

STRIPE_PRICE_PRO_MONTHLY=price_BURAYA
STRIPE_PRICE_PRO_ANNUAL=price_BURAYA
STRIPE_PRICE_AGENCY_MONTHLY=price_BURAYA
STRIPE_PRICE_AGENCY_ANNUAL=price_BURAYA

FRONTEND_URL=http://localhost:3000
PORT=4000
```

### 3c. `server.js` dosyası:

```javascript
const express = require('express');
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
const cors = require('cors');
require('dotenv').config();

const app = express();

// Webhook için raw body gerekli — önce tanımla
app.post('/api/webhook', express.raw({ type: 'application/json' }), handleWebhook);

// Diğer route'lar için JSON parser
app.use(express.json());
app.use(cors({ origin: process.env.FRONTEND_URL }));

// ─────────────────────────────────────────────
// ROUTE 1: Subscription başlat (checkout session)
// ─────────────────────────────────────────────
app.post('/api/create-checkout', async (req, res) => {
  try {
    const { email, plan, billing } = req.body;
    // plan: 'pro' | 'agency'
    // billing: 'monthly' | 'annual'

    // Price ID'yi seç
    const priceMap = {
      'pro-monthly':     process.env.STRIPE_PRICE_PRO_MONTHLY,
      'pro-annual':      process.env.STRIPE_PRICE_PRO_ANNUAL,
      'agency-monthly':  process.env.STRIPE_PRICE_AGENCY_MONTHLY,
      'agency-annual':   process.env.STRIPE_PRICE_AGENCY_ANNUAL,
    };

    const priceId = priceMap[`${plan}-${billing}`];
    if (!priceId) return res.status(400).json({ error: 'Invalid plan/billing' });

    // Stripe Checkout Session oluştur
    const session = await stripe.checkout.sessions.create({
      mode: 'subscription',
      payment_method_types: ['card'],
      customer_email: email,
      line_items: [{ price: priceId, quantity: 1 }],
      subscription_data: {
        trial_period_days: 7,  // 7 gün ücretsiz deneme!
      },
      success_url: `${process.env.FRONTEND_URL}/checkout-success?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${process.env.FRONTEND_URL}/checkout`,
    });

    res.json({ url: session.url });
  } catch (err) {
    console.error('Checkout error:', err);
    res.status(500).json({ error: err.message });
  }
});

// ─────────────────────────────────────────────
// ROUTE 2: Stripe Checkout'a yönlendir
// ─────────────────────────────────────────────
// Frontend'den çağır:
// const res = await fetch('/api/create-checkout', {
//   method: 'POST',
//   headers: { 'Content-Type': 'application/json' },
//   body: JSON.stringify({ email, plan: 'pro', billing: 'monthly' })
// });
// const { url } = await res.json();
// window.location.href = url;  // Stripe'ın kendi ödeme sayfasına git

// ─────────────────────────────────────────────
// ROUTE 3: Subscription iptal
// ─────────────────────────────────────────────
app.post('/api/cancel-subscription', async (req, res) => {
  try {
    const { subscriptionId } = req.body;

    // Dönem sonunda iptal (anında değil)
    const subscription = await stripe.subscriptions.update(subscriptionId, {
      cancel_at_period_end: true,
    });

    res.json({ success: true, subscription });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ─────────────────────────────────────────────
// ROUTE 4: Customer portal (faturaları gör, plan değiştir)
// ─────────────────────────────────────────────
app.post('/api/customer-portal', async (req, res) => {
  try {
    const { customerId } = req.body;

    const session = await stripe.billingPortal.sessions.create({
      customer: customerId,
      return_url: `${process.env.FRONTEND_URL}/dashboard`,
    });

    res.json({ url: session.url });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ─────────────────────────────────────────────
// ROUTE 5: Webhook — Stripe olaylarını yakala
// Bu ÇOOOK önemli. Ödeme başarılı olunca kullanıcıyı
// Pro yap, iptal olunca Free'ye döndür.
// ─────────────────────────────────────────────
async function handleWebhook(req, res) {
  const sig = req.headers['stripe-signature'];
  let event;

  try {
    event = stripe.webhooks.constructEvent(
      req.body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET
    );
  } catch (err) {
    console.error('Webhook signature error:', err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  // Olayları işle
  switch (event.type) {

    case 'customer.subscription.created':
    case 'customer.subscription.updated': {
      const sub = event.data.object;
      const customerId = sub.customer;
      const status = sub.status; // 'active', 'trialing', 'past_due', etc.
      const planId = sub.items.data[0].price.id;

      // Hangi plan?
      let planName = 'free';
      if (planId === process.env.STRIPE_PRICE_PRO_MONTHLY ||
          planId === process.env.STRIPE_PRICE_PRO_ANNUAL) {
        planName = 'pro';
      } else if (planId === process.env.STRIPE_PRICE_AGENCY_MONTHLY ||
                 planId === process.env.STRIPE_PRICE_AGENCY_ANNUAL) {
        planName = 'agency';
      }

      // VERİTABANINDA GUNCELLE:
      // await db.users.update(
      //   { stripeCustomerId: customerId },
      //   { plan: planName, subscriptionStatus: status, subscriptionId: sub.id }
      // );

      console.log(`✅ Subscription ${event.type}: customer=${customerId}, plan=${planName}, status=${status}`);
      break;
    }

    case 'customer.subscription.deleted': {
      const sub = event.data.object;
      // VERİTABANINDA GUNCELLE:
      // await db.users.update(
      //   { stripeCustomerId: sub.customer },
      //   { plan: 'free', subscriptionStatus: 'canceled' }
      // );
      console.log(`❌ Subscription canceled: customer=${sub.customer}`);
      break;
    }

    case 'invoice.payment_succeeded': {
      const invoice = event.data.object;
      console.log(`💰 Payment succeeded: $${invoice.amount_paid/100} from ${invoice.customer_email}`);
      // Email gönder: "Ödemeniz alındı" makbuzu
      break;
    }

    case 'invoice.payment_failed': {
      const invoice = event.data.object;
      console.log(`⚠️ Payment failed: ${invoice.customer_email}`);
      // Email gönder: "Ödemeniz başarısız oldu, kartınızı güncelleyin"
      break;
    }
  }

  res.json({ received: true });
}

// ─────────────────────────────────────────────
app.listen(process.env.PORT || 4000, () => {
  console.log(`🚀 Backend running on port ${process.env.PORT || 4000}`);
});
```

---

## ADIM 4: Webhook Kur

### Test için (local):
```bash
# Stripe CLI indir: https://stripe.com/docs/stripe-cli
stripe listen --forward-to localhost:4000/api/webhook
# Bu komut STRIPE_WEBHOOK_SECRET'i terminalde gösterir, .env'e kopyala
```

### Production için (Vercel/Railway'e deploy ettikten sonra):
Stripe Dashboard → **Developers → Webhooks → Add endpoint**
- URL: `https://senin-domain.com/api/webhook`
- Events: `customer.subscription.*`, `invoice.payment_*`

---

## ADIM 5: Frontend'i Stripe Checkout'a Bağla

`checkout.html` içindeki `handlePayment()` fonksiyonunu güncelle:

```javascript
async function handlePayment() {
  if (!validateForm()) return;

  // Loading state...
  setLoading(true);

  try {
    // Backend'e istek at
    const response = await fetch('http://localhost:4000/api/create-checkout', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        email: document.getElementById('pay-email').value,
        plan: currentPlan,      // 'pro' veya 'agency'
        billing: currentBilling // 'monthly' veya 'annual'
      })
    });

    const { url, error } = await response.json();

    if (error) throw new Error(error);

    // Stripe'ın kendi checkout sayfasına yönlendir
    window.location.href = url;

  } catch (err) {
    showError(err.message);
    setLoading(false);
  }
}
```

---

## ADIM 6: Test Et

### Test Kart Numaraları (gerçek para çekilmez):

| Durum | Kart No |
|---|---|
| ✅ Başarılı ödeme | `4242 4242 4242 4242` |
| ❌ Reddedildi | `4000 0000 0000 0002` |
| 🔐 3D Secure | `4000 0025 0000 3155` |
| ⚠️ Yetersiz bakiye | `4000 0000 0000 9995` |

**Tüm test kartları için:** Herhangi bir gelecek tarih + herhangi 3 haneli CVC

---

## ADIM 7: Canlıya Geç

1. Stripe Dashboard → **Activate your account** (kimlik doğrulama)
2. `sk_test_` → `sk_live_` ile değiştir
3. `pk_test_` → `pk_live_` ile değiştir
4. Yeni webhook endpoint ekle (production URL'in ile)
5. Gerçek ürün Price ID'lerini oluştur

---

## Deploy (Backend)

### Railway (En kolay, ücretsiz başlar):
```bash
npm install -g @railway/cli
railway login
railway init
railway up
# Otomatik URL alırsın: https://xxx.railway.app
```

### Vercel (Next.js ile kullanıyorsan):
`/api/create-checkout.js`, `/api/webhook.js` şeklinde Vercel Functions kullan.

---

## Gelir Özeti

| Plan | Aylık | Yıllık |
|---|---|---|
| Pro | $29/kullanıcı | $228/kullanıcı |
| Agency | $79/kullanıcı | $612/kullanıcı |

**100 Pro kullanıcısı = $2,900/ay = $34,800/yıl** 🚀

---

## Önemli Notlar

- `sk_live_` key'ini ASLA frontend'e koyma, GitHub'a push etme
- `.env` dosyasını `.gitignore`'a ekle
- Stripe test modunda sonsuz test yapabilirsin (gerçek para gitmez)
- Webhook'lar olmadan ödeme durumu güncellenmez — MUTLAKA kur

---

*LinkedBoost AI — Stripe Integration Guide v1.0*
