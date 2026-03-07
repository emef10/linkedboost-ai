# 📧 LinkedBoost AI — Email Sistemi Kılavuzu

## Kullanılacak Servis: Resend (En kolay + ücretsiz 3,000 email/ay)

---

## ADIM 1: Resend Hesabı Aç (3 dakika)

1. https://resend.com → "Start for free"
2. Email doğrula
3. Dashboard → **API Keys → Create API Key**
4. Key'i kopyala: `re_xxxxx`
5. `.env` dosyasına ekle:

```env
RESEND_API_KEY=re_BURAYA_KOPYALA
EMAIL_FROM=noreply@linkedboost.ai
EMAIL_FROM_NAME=LinkedBoost AI
```

> **Domain yoksa:** Resend'in kendi test domain'ini kullanabilirsin (`onboarding@resend.dev`)
> **Domain varsa:** Dashboard → Domains → Add Domain → DNS kayıtları ekle (5 dakika)

---

## ADIM 2: Backend'e Resend Ekle

```bash
cd linkedboost-backend
npm install resend
```

`server.js` dosyasına ekle:

```javascript
const { Resend } = require('resend');
const resend = new Resend(process.env.RESEND_API_KEY);
```

---

## ADIM 3: Email Gönderme Fonksiyonları

`emails.js` adında yeni bir dosya oluştur:

```javascript
const { Resend } = require('resend');
require('dotenv').config();

const resend = new Resend(process.env.RESEND_API_KEY);

const FROM = `${process.env.EMAIL_FROM_NAME} <${process.env.EMAIL_FROM}>`;

// ─────────────────────────────────────────
// 1. HOŞ GELDİN EMAIL (Kayıt sonrası)
// ─────────────────────────────────────────
async function sendWelcomeEmail({ name, email, plan }) {
  const firstName = name.split(' ')[0];

  await resend.emails.send({
    from: FROM,
    to: email,
    subject: `Welcome to LinkedBoost AI, ${firstName}! 🚀`,
    html: getWelcomeHTML({ firstName, plan }),
  });

  console.log(`✅ Welcome email sent to ${email}`);
}

// ─────────────────────────────────────────
// 2. ÖDEME MAKBUZU (Stripe webhook'tan)
// ─────────────────────────────────────────
async function sendReceiptEmail({ name, email, plan, amount, date, invoiceId }) {
  const firstName = name.split(' ')[0];

  await resend.emails.send({
    from: FROM,
    to: email,
    subject: `Your LinkedBoost AI receipt — $${amount}`,
    html: getReceiptHTML({ firstName, plan, amount, date, invoiceId }),
  });

  console.log(`✅ Receipt email sent to ${email}`);
}

// ─────────────────────────────────────────
// 3. HAFTALIK POST ÖNERİLERİ
// ─────────────────────────────────────────
async function sendWeeklyPostsEmail({ name, email, posts }) {
  const firstName = name.split(' ')[0];

  await resend.emails.send({
    from: FROM,
    to: email,
    subject: `Your 5 LinkedIn posts for this week ✍️`,
    html: getWeeklyPostsHTML({ firstName, posts }),
  });
}

// ─────────────────────────────────────────
// 4. ÖDEME BAŞARISIZ UYARISI
// ─────────────────────────────────────────
async function sendPaymentFailedEmail({ name, email }) {
  const firstName = name.split(' ')[0];

  await resend.emails.send({
    from: FROM,
    to: email,
    subject: `Action required: Update your payment method`,
    html: getPaymentFailedHTML({ firstName }),
  });
}

module.exports = {
  sendWelcomeEmail,
  sendReceiptEmail,
  sendWeeklyPostsEmail,
  sendPaymentFailedEmail,
};
```

---

## ADIM 4: Route'lara Email Ekle

`server.js` içinde ilgili yerlere ekle:

```javascript
const {
  sendWelcomeEmail,
  sendReceiptEmail,
  sendWeeklyPostsEmail,
  sendPaymentFailedEmail
} = require('./emails');

// Kayıt route'una ekle:
app.post('/api/register', async (req, res) => {
  const { name, email, password, plan } = req.body;
  // ... kullanıcıyı veritabanına kaydet ...

  // Hoş geldin emaili gönder
  await sendWelcomeEmail({ name, email, plan: plan || 'free' });

  res.json({ success: true });
});

// Webhook'a ekle (invoice.payment_succeeded):
case 'invoice.payment_succeeded': {
  const invoice = event.data.object;
  const amount = (invoice.amount_paid / 100).toFixed(2);
  const date = new Date(invoice.created * 1000).toLocaleDateString('en-US', {
    year: 'numeric', month: 'long', day: 'numeric'
  });

  // Kullanıcıyı veritabanından bul
  // const user = await db.users.findOne({ stripeCustomerId: invoice.customer });

  await sendReceiptEmail({
    name: invoice.customer_name || 'Valued Customer',
    email: invoice.customer_email,
    plan: 'Pro',   // veya DB'den al
    amount,
    date,
    invoiceId: invoice.id,
  });
  break;
}

// Webhook'a ekle (invoice.payment_failed):
case 'invoice.payment_failed': {
  const invoice = event.data.object;
  await sendPaymentFailedEmail({
    name: invoice.customer_name || 'there',
    email: invoice.customer_email,
  });
  break;
}
```

---

## ADIM 5: HTML Şablonları

`email-templates.js` dosyası oluştur:

```javascript
// ─── HOŞ GELDİN ───────────────────────────
function getWelcomeHTML({ firstName, plan }) {
  return `<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width">
<title>Welcome to LinkedBoost AI</title>
</head>
<body style="margin:0;padding:0;background:#f5f3ee;font-family:'DM Sans',Helvetica,Arial,sans-serif;">

<table width="100%" cellpadding="0" cellspacing="0" style="background:#f5f3ee;padding:40px 20px;">
<tr><td align="center">
<table width="560" cellpadding="0" cellspacing="0" style="background:#ffffff;border-radius:20px;overflow:hidden;border:1px solid #e5e2da;">

  <!-- HEADER -->
  <tr><td style="background:#0a0a0f;padding:32px 40px;text-align:center;">
    <div style="font-family:Georgia,serif;font-size:22px;font-weight:900;color:#ffffff;letter-spacing:-0.03em;">
      Linked<span style="color:#0057ff;">Boost</span> AI
    </div>
  </td></tr>

  <!-- HERO -->
  <tr><td style="padding:48px 40px 32px;text-align:center;">
    <div style="font-size:48px;margin-bottom:16px;">🚀</div>
    <h1 style="font-family:Georgia,serif;font-size:28px;font-weight:900;color:#0a0a0f;margin:0 0 12px;letter-spacing:-0.03em;">
      Welcome aboard, ${firstName}!
    </h1>
    <p style="font-size:16px;color:#6b7280;line-height:1.6;margin:0;">
      Your LinkedBoost AI account is active and ready.<br>
      Let's grow your LinkedIn presence starting today.
    </p>
  </td></tr>

  <!-- DIVIDER -->
  <tr><td style="padding:0 40px;"><hr style="border:none;border-top:1px solid #e5e2da;margin:0;"></td></tr>

  <!-- STEPS -->
  <tr><td style="padding:32px 40px;">
    <h2 style="font-size:14px;font-weight:700;text-transform:uppercase;letter-spacing:0.1em;color:#6b7280;margin:0 0 24px;">
      Get started in 3 steps
    </h2>

    <table width="100%" cellpadding="0" cellspacing="0">
      <tr>
        <td style="width:36px;vertical-align:top;padding-top:2px;">
          <div style="width:28px;height:28px;background:#0057ff;border-radius:50%;text-align:center;line-height:28px;font-size:13px;font-weight:700;color:#fff;">1</div>
        </td>
        <td style="padding:0 0 20px 12px;">
          <div style="font-size:15px;font-weight:600;color:#0a0a0f;margin-bottom:4px;">Analyze your profile</div>
          <div style="font-size:14px;color:#6b7280;">Get your LinkedIn profile score and see exactly what to fix.</div>
        </td>
      </tr>
      <tr>
        <td style="width:36px;vertical-align:top;padding-top:2px;">
          <div style="width:28px;height:28px;background:#0057ff;border-radius:50%;text-align:center;line-height:28px;font-size:13px;font-weight:700;color:#fff;">2</div>
        </td>
        <td style="padding:0 0 20px 12px;">
          <div style="font-size:15px;font-weight:600;color:#0a0a0f;margin-bottom:4px;">Generate your first post</div>
          <div style="font-size:14px;color:#6b7280;">Use the AI generator to write a scroll-stopping post in 10 seconds.</div>
        </td>
      </tr>
      <tr>
        <td style="width:36px;vertical-align:top;padding-top:2px;">
          <div style="width:28px;height:28px;background:#0057ff;border-radius:50%;text-align:center;line-height:28px;font-size:13px;font-weight:700;color:#fff;">3</div>
        </td>
        <td style="padding:0 0 4px 12px;">
          <div style="font-size:15px;font-weight:600;color:#0a0a0f;margin-bottom:4px;">Post and watch your reach grow</div>
          <div style="font-size:14px;color:#6b7280;">Copy to LinkedIn, post, and track your growth in the dashboard.</div>
        </td>
      </tr>
    </table>
  </td></tr>

  <!-- CTA BUTTON -->
  <tr><td style="padding:0 40px 40px;text-align:center;">
    <a href="https://linkedboost.ai/dashboard"
       style="display:inline-block;background:#0057ff;color:#ffffff;text-decoration:none;font-size:16px;font-weight:700;padding:16px 40px;border-radius:100px;letter-spacing:0.01em;">
      Go to Dashboard →
    </a>
    <p style="font-size:13px;color:#9ca3af;margin-top:16px;">
      Your plan: <strong style="color:#0a0a0f;">${plan === 'free' ? 'Free Starter' : plan === 'pro' ? 'Pro (7-day trial)' : 'Agency'}</strong>
    </p>
  </td></tr>

  <!-- FOOTER -->
  <tr><td style="background:#f9f8f5;border-top:1px solid #e5e2da;padding:24px 40px;text-align:center;">
    <p style="font-size:12px;color:#9ca3af;margin:0 0 8px;">
      LinkedBoost AI · support@linkedboost.ai
    </p>
    <p style="font-size:12px;color:#9ca3af;margin:0;">
      <a href="#" style="color:#9ca3af;">Unsubscribe</a> · 
      <a href="#" style="color:#9ca3af;">Privacy Policy</a>
    </p>
  </td></tr>

</table>
</td></tr>
</table>
</body>
</html>`;
}

// ─── MAKBUZ ───────────────────────────────
function getReceiptHTML({ firstName, plan, amount, date, invoiceId }) {
  return `<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"><meta name="viewport" content="width=device-width"></head>
<body style="margin:0;padding:0;background:#f5f3ee;font-family:'DM Sans',Helvetica,Arial,sans-serif;">

<table width="100%" cellpadding="0" cellspacing="0" style="background:#f5f3ee;padding:40px 20px;">
<tr><td align="center">
<table width="560" cellpadding="0" cellspacing="0" style="background:#ffffff;border-radius:20px;overflow:hidden;border:1px solid #e5e2da;">

  <!-- HEADER -->
  <tr><td style="background:#0a0a0f;padding:28px 40px;text-align:center;">
    <div style="font-family:Georgia,serif;font-size:20px;font-weight:900;color:#fff;">
      Linked<span style="color:#0057ff;">Boost</span> AI
    </div>
    <div style="font-size:12px;color:rgba(255,255,255,0.35);margin-top:4px;letter-spacing:0.1em;text-transform:uppercase;">Payment Receipt</div>
  </td></tr>

  <!-- AMOUNT -->
  <tr><td style="padding:48px 40px 32px;text-align:center;background:linear-gradient(135deg,#f0f4ff,#f5f3ee);">
    <div style="font-size:13px;font-weight:600;text-transform:uppercase;letter-spacing:0.1em;color:#6b7280;margin-bottom:8px;">Amount Paid</div>
    <div style="font-family:Georgia,serif;font-size:52px;font-weight:900;color:#0a0a0f;letter-spacing:-0.04em;line-height:1;">$${amount}</div>
    <div style="font-size:14px;color:#6b7280;margin-top:8px;">${date}</div>
  </td></tr>

  <!-- DETAILS -->
  <tr><td style="padding:32px 40px;">
    <table width="100%" cellpadding="0" cellspacing="0">
      <tr>
        <td style="font-size:14px;color:#6b7280;padding:10px 0;border-bottom:1px solid #e5e2da;">Customer</td>
        <td style="font-size:14px;font-weight:600;color:#0a0a0f;text-align:right;padding:10px 0;border-bottom:1px solid #e5e2da;">${firstName}</td>
      </tr>
      <tr>
        <td style="font-size:14px;color:#6b7280;padding:10px 0;border-bottom:1px solid #e5e2da;">Plan</td>
        <td style="font-size:14px;font-weight:600;color:#0a0a0f;text-align:right;padding:10px 0;border-bottom:1px solid #e5e2da;">LinkedBoost ${plan}</td>
      </tr>
      <tr>
        <td style="font-size:14px;color:#6b7280;padding:10px 0;border-bottom:1px solid #e5e2da;">Invoice ID</td>
        <td style="font-size:12px;font-family:monospace;color:#6b7280;text-align:right;padding:10px 0;border-bottom:1px solid #e5e2da;">${invoiceId}</td>
      </tr>
      <tr>
        <td style="font-size:14px;color:#6b7280;padding:10px 0;">Status</td>
        <td style="text-align:right;padding:10px 0;">
          <span style="background:#d1fae5;color:#065f46;font-size:12px;font-weight:700;padding:4px 12px;border-radius:100px;">✓ PAID</span>
        </td>
      </tr>
    </table>
  </td></tr>

  <!-- CTA -->
  <tr><td style="padding:0 40px 40px;text-align:center;">
    <a href="https://linkedboost.ai/dashboard"
       style="display:inline-block;background:#0057ff;color:#fff;text-decoration:none;font-size:15px;font-weight:700;padding:14px 36px;border-radius:100px;">
      Go to Dashboard →
    </a>
    <p style="font-size:13px;color:#9ca3af;margin-top:16px;line-height:1.5;">
      Questions? Email us at <a href="mailto:support@linkedboost.ai" style="color:#0057ff;">support@linkedboost.ai</a><br>
      Cancel anytime in <a href="https://linkedboost.ai/dashboard/settings" style="color:#0057ff;">Settings</a>
    </p>
  </td></tr>

  <!-- FOOTER -->
  <tr><td style="background:#f9f8f5;border-top:1px solid #e5e2da;padding:20px 40px;text-align:center;">
    <p style="font-size:12px;color:#9ca3af;margin:0;">
      LinkedBoost AI · <a href="#" style="color:#9ca3af;">Privacy</a> · <a href="#" style="color:#9ca3af;">Terms</a>
    </p>
  </td></tr>

</table>
</td></tr>
</table>
</body>
</html>`;
}

// ─── HAFTALIK POSTLAR ─────────────────────
function getWeeklyPostsHTML({ firstName, posts }) {
  const postCards = posts.map((post, i) => `
    <tr><td style="padding:0 0 16px;">
      <table width="100%" cellpadding="0" cellspacing="0"
             style="background:#f9f8f5;border-radius:14px;border:1px solid #e5e2da;overflow:hidden;">
        <tr>
          <td style="padding:16px 20px;background:#f0f4ff;border-bottom:1px solid #e5e2da;">
            <span style="font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:0.1em;color:#0057ff;">
              Post ${i+1} · ${post.type}
            </span>
            <span style="float:right;font-size:12px;color:#6b7280;">📅 ${post.bestDay} ${post.bestTime}</span>
          </td>
        </tr>
        <tr>
          <td style="padding:16px 20px;font-size:14px;color:#374151;line-height:1.65;">
            ${post.text.replace(/\n/g, '<br>')}
          </td>
        </tr>
      </table>
    </td></tr>
  `).join('');

  return `<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"></head>
<body style="margin:0;padding:0;background:#f5f3ee;font-family:Helvetica,Arial,sans-serif;">
<table width="100%" cellpadding="0" cellspacing="0" style="background:#f5f3ee;padding:40px 20px;">
<tr><td align="center">
<table width="600" cellpadding="0" cellspacing="0" style="background:#fff;border-radius:20px;overflow:hidden;border:1px solid #e5e2da;">

  <tr><td style="background:#0a0a0f;padding:28px 40px;">
    <div style="font-family:Georgia,serif;font-size:20px;font-weight:900;color:#fff;">
      Linked<span style="color:#0057ff;">Boost</span> AI
    </div>
  </td></tr>

  <tr><td style="padding:40px 40px 24px;">
    <h1 style="font-family:Georgia,serif;font-size:24px;color:#0a0a0f;margin:0 0 8px;letter-spacing:-0.03em;">
      Your 5 posts for this week, ${firstName} ✍️
    </h1>
    <p style="font-size:15px;color:#6b7280;margin:0;">
      Copy, paste, and post. Best times are included.
    </p>
  </td></tr>

  <tr><td style="padding:0 40px 40px;">
    <table width="100%" cellpadding="0" cellspacing="0">
      ${postCards}
    </table>
  </td></tr>

  <tr><td style="padding:0 40px 40px;text-align:center;border-top:1px solid #e5e2da;padding-top:32px;">
    <a href="https://linkedboost.ai/dashboard"
       style="display:inline-block;background:#0057ff;color:#fff;text-decoration:none;font-size:15px;font-weight:700;padding:14px 36px;border-radius:100px;">
      View in Dashboard →
    </a>
  </td></tr>

  <tr><td style="background:#f9f8f5;border-top:1px solid #e5e2da;padding:20px 40px;text-align:center;">
    <p style="font-size:12px;color:#9ca3af;margin:0;">
      <a href="#" style="color:#9ca3af;">Unsubscribe from weekly posts</a>
    </p>
  </td></tr>

</table>
</td></tr>
</table>
</body>
</html>`;
}

// ─── ÖDEME BAŞARISIZ ─────────────────────
function getPaymentFailedHTML({ firstName }) {
  return `<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"></head>
<body style="margin:0;padding:0;background:#f5f3ee;font-family:Helvetica,Arial,sans-serif;">
<table width="100%" cellpadding="0" cellspacing="0" style="background:#f5f3ee;padding:40px 20px;">
<tr><td align="center">
<table width="560" cellpadding="0" cellspacing="0" style="background:#fff;border-radius:20px;overflow:hidden;border:1px solid #e5e2da;">

  <tr><td style="background:#0a0a0f;padding:28px 40px;text-align:center;">
    <div style="font-family:Georgia,serif;font-size:20px;font-weight:900;color:#fff;">
      Linked<span style="color:#0057ff;">Boost</span> AI
    </div>
  </td></tr>

  <tr><td style="padding:48px 40px 32px;text-align:center;">
    <div style="font-size:48px;margin-bottom:16px;">⚠️</div>
    <h1 style="font-family:Georgia,serif;font-size:24px;color:#0a0a0f;margin:0 0 12px;">
      Payment failed, ${firstName}
    </h1>
    <p style="font-size:15px;color:#6b7280;line-height:1.6;margin:0 0 32px;">
      We couldn't process your payment. Please update your payment method to keep your Pro access.
    </p>
    <a href="https://linkedboost.ai/dashboard/settings"
       style="display:inline-block;background:#ff4757;color:#fff;text-decoration:none;font-size:15px;font-weight:700;padding:14px 36px;border-radius:100px;">
      Update Payment Method →
    </a>
    <p style="font-size:13px;color:#9ca3af;margin-top:20px;">
      Need help? <a href="mailto:support@linkedboost.ai" style="color:#0057ff;">Contact support</a>
    </p>
  </td></tr>

  <tr><td style="background:#f9f8f5;border-top:1px solid #e5e2da;padding:20px 40px;text-align:center;">
    <p style="font-size:12px;color:#9ca3af;margin:0;">LinkedBoost AI</p>
  </td></tr>

</table>
</td></tr>
</table>
</body>
</html>`;
}

module.exports = {
  getWelcomeHTML,
  getReceiptHTML,
  getWeeklyPostsHTML,
  getPaymentFailedHTML,
};
```

---

## ADIM 6: Haftalık Email Otomasyonu (Cron Job)

Her Pazartesi sabahı otomatik post emaili gönder:

```bash
npm install node-cron
```

`cron.js` dosyası:

```javascript
const cron = require('node-cron');
const { sendWeeklyPostsEmail } = require('./emails');
// const db = require('./db');  // veritabanın

// Her Pazartesi sabah 08:00'de çalışır
cron.schedule('0 8 * * 1', async () => {
  console.log('📧 Sending weekly posts emails...');

  // Pro kullanıcıları getir
  // const users = await db.users.findAll({ where: { plan: 'pro' } });

  // Demo: tek kullanıcı
  const users = [{ name: 'John Doe', email: 'john@example.com' }];

  for (const user of users) {
    try {
      // Claude API ile post üret (her kullanıcı için)
      const posts = await generateWeeklyPosts(user);
      await sendWeeklyPostsEmail({ name: user.name, email: user.email, posts });
    } catch (err) {
      console.error(`Failed for ${user.email}:`, err.message);
    }
  }

  console.log(`✅ Weekly emails sent to ${users.length} users`);
}, {
  timezone: 'America/New_York'
});
```

---

## Test Et

```javascript
// test-email.js — terminalde çalıştır: node test-email.js
require('dotenv').config();
const { sendWelcomeEmail, sendReceiptEmail } = require('./emails');

async function test() {
  // Hoş geldin testi
  await sendWelcomeEmail({
    name: 'Test User',
    email: 'senin@email.com',   // ← kendi emailin
    plan: 'pro'
  });

  // Makbuz testi
  await sendReceiptEmail({
    name: 'Test User',
    email: 'senin@email.com',
    plan: 'Pro',
    amount: '29.00',
    date: 'June 15, 2025',
    invoiceId: 'in_test_123456'
  });

  console.log('Emails sent! Check your inbox.');
}

test();
```

```bash
node test-email.js
```

---

## Özet: Hangi email ne zaman gider?

| Email | Ne zaman | Kim alır |
|---|---|---|
| 🎉 Hoş geldin | Kayıt olunca | Tüm kullanıcılar |
| 💳 Makbuz | Ödeme başarılı | Pro + Agency |
| ✍️ Haftalık postlar | Her Pazartesi 08:00 | Pro + Agency |
| ⚠️ Ödeme başarısız | Kart reddedilince | Pro + Agency |

---

## Resend Ücretsiz Limit

| Plan | Email/ay | Fiyat |
|---|---|---|
| Free | 3,000 | $0 |
| Pro | 50,000 | $20/mo |
| Business | 100,000 | $45/mo |

**100 kullanıcı × 5 email/hafta × 4 hafta = 2,000 email/ay → tamamen ücretsiz** ✅

---

*LinkedBoost AI — Email System Guide v1.0*
