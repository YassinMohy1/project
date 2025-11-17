# دليل سريع: تغيير طريقة الدفع من Stripe لأي حاجة تانية

## 🚀 مثال عملي: التحويل لـ PayPal

### الخطوة 1: إزالة Stripe وتثبيت PayPal

```bash
npm uninstall @stripe/react-stripe-js @stripe/stripe-js
npm install @paypal/react-paypal-js
```

### الخطوة 2: تحديث `.env`

```env
# احذف Stripe Keys
# VITE_STRIPE_PUBLISHABLE_KEY=pk_test_...
# STRIPE_SECRET_KEY=sk_test_...

# أضف PayPal Keys
VITE_PAYPAL_CLIENT_ID=your_paypal_client_id_here
```

احصل على الـ Client ID من: https://developer.paypal.com/dashboard/

### الخطوة 3: تحديث `src/App.tsx`

استبدل الكود ده:

```tsx
// القديم - Stripe
import { Elements } from '@stripe/react-stripe-js';
import { loadStripe } from '@stripe/stripe-js';

const stripePromise = loadStripe(import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY || '');

// ... في الـ Route
<Route
  path="/checkout"
  element={
    <Elements stripe={stripePromise}>
      <Checkout />
    </Elements>
  }
/>
```

بالكود ده:

```tsx
// الجديد - PayPal
import { PayPalScriptProvider } from '@paypal/react-paypal-js';

const paypalOptions = {
  clientId: import.meta.env.VITE_PAYPAL_CLIENT_ID || '',
  currency: 'USD',
};

// ... في الـ Route
<Route
  path="/checkout"
  element={
    <PayPalScriptProvider options={paypalOptions}>
      <Checkout />
    </PayPalScriptProvider>
  }
/>
```

### الخطوة 4: تحديث `src/components/BillingPayment.tsx`

احذف الـ imports بتاعة Stripe:

```tsx
// احذف دول
import { CardElement, useStripe, useElements } from '@stripe/react-stripe-js';
```

أضف PayPal:

```tsx
// أضف ده
import { PayPalButtons } from '@paypal/react-paypal-js';
```

استبدل الـ payment section في الكود:

```tsx
// احذف CardElement section واستبدله بده:

<div className="mb-6">
  <h3 className="text-lg font-bold text-gray-900 mb-4">Payment Information</h3>

  <PayPalButtons
    style={{ layout: 'vertical' }}
    createOrder={(data, actions) => {
      // Calculate total from props
      return actions.order.create({
        purchase_units: [
          {
            amount: {
              value: '399.00', // Replace with actual total
            },
          },
        ],
      });
    }}
    onApprove={async (data, actions) => {
      const order = await actions.order?.capture();
      console.log('Payment successful:', order);
      // Handle success
    }}
    onError={(err) => {
      console.error('Payment error:', err);
      // Handle error
    }}
  />
</div>
```

### الخطوة 5: تحديث `src/pages/Checkout.tsx`

احذف الـ Stripe imports والـ hooks:

```tsx
// احذف دول
import { useStripe, useElements, CardElement } from '@stripe/react-stripe-js';

const stripe = useStripe();
const elements = useElements();
```

غير دالة `handlePayment` لتبسيطها:

```tsx
const handlePayment = async (billingData: BillingData) => {
  if (!passengerData) {
    setPaymentError('Please fill in passenger details.');
    return;
  }

  setIsProcessing(true);
  setPaymentError(null);

  try {
    const protectionCost = selectedPlanData?.price || 0;
    const baggageCost = baggageProtection ? pricingData.addons.baggage_protection.price : 0;
    const subtotal = pricingData.product.base_fare + protectionCost + baggageCost + pricingData.fees.service_fee;
    const tax = subtotal * (pricingData.fees.tax_rate / 100);
    const total = subtotal + tax;

    // PayPal handles payment automatically through PayPalButtons
    // Just save to database after successful payment
    const { error: dbError } = await supabase.from('bookings').insert({
      passenger_name: `${passengerData.firstName} ${passengerData.lastName}`,
      email: billingData.email,
      phone: `${passengerData.countryCode}${passengerData.phoneNumber}`,
      flight_details: pricingData.product.name,
      total_amount: total,
      payment_intent_id: 'paypal_order_id', // Will be updated by PayPal
      protection_plan: selectedPlan,
      baggage_protection: baggageProtection,
      status: 'confirmed',
    });

    if (!dbError) {
      setCurrentStep(4);
    }
  } catch (err: any) {
    setPaymentError(err.message || 'Payment failed.');
  } finally {
    setIsProcessing(false);
  }
};
```

---

## 🇪🇬 مثال تاني: Paymob (للسوق المصري)

### الخطوة 1: إضافة Paymob Keys

```env
VITE_PAYMOB_API_KEY=your_api_key
VITE_PAYMOB_INTEGRATION_ID=your_integration_id
VITE_PAYMOB_IFRAME_ID=your_iframe_id
```

### الخطوة 2: إنشاء Paymob Component

أنشئ ملف: `src/components/PaymobButton.tsx`

```tsx
import { useState } from 'react';

interface PaymobButtonProps {
  amount: number;
  customerEmail: string;
  customerName: string;
  customerPhone: string;
  onSuccess: () => void;
  onError: (error: string) => void;
}

export default function PaymobButton({
  amount,
  customerEmail,
  customerName,
  customerPhone,
  onSuccess,
  onError,
}: PaymobButtonProps) {
  const [loading, setLoading] = useState(false);

  const handlePayment = async () => {
    setLoading(true);

    try {
      // Call your backend to create Paymob payment
      const response = await fetch(
        `${import.meta.env.VITE_SUPABASE_URL}/functions/v1/create-paymob-payment`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${import.meta.env.VITE_SUPABASE_ANON_KEY}`,
          },
          body: JSON.stringify({
            amount: amount * 100, // Convert to cents
            email: customerEmail,
            name: customerName,
            phone: customerPhone,
          }),
        }
      );

      const data = await response.json();

      if (data.iframeUrl) {
        // Redirect to Paymob payment page
        window.location.href = data.iframeUrl;
      } else {
        throw new Error('Failed to create payment');
      }
    } catch (error) {
      onError('فشل الاتصال بنظام الدفع');
      setLoading(false);
    }
  };

  return (
    <button
      onClick={handlePayment}
      disabled={loading}
      className="w-full bg-gradient-to-r from-green-500 to-green-600 text-white py-4 rounded-xl font-bold text-lg hover:shadow-2xl transition-all duration-300 disabled:opacity-50"
    >
      {loading ? 'جاري التحويل للدفع...' : 'ادفع الآن'}
    </button>
  );
}
```

### الخطوة 3: إنشاء Paymob Edge Function

أنشئ ملف: `supabase/functions/create-paymob-payment/index.ts`

```typescript
import 'jsr:@supabase/functions-js/edge-runtime.d.ts';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'POST, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization, X-Client-Info, Apikey',
};

Deno.serve(async (req: Request) => {
  if (req.method === 'OPTIONS') {
    return new Response(null, { status: 200, headers: corsHeaders });
  }

  try {
    const { amount, email, name, phone } = await req.json();
    const apiKey = Deno.env.get('PAYMOB_API_KEY');
    const integrationId = Deno.env.get('PAYMOB_INTEGRATION_ID');
    const iframeId = Deno.env.get('PAYMOB_IFRAME_ID');

    // 1. Get authentication token
    const authRes = await fetch('https://accept.paymob.com/api/auth/tokens', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ api_key: apiKey }),
    });
    const { token } = await authRes.json();

    // 2. Create order
    const orderRes = await fetch('https://accept.paymob.com/api/ecommerce/orders', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        auth_token: token,
        delivery_needed: 'false',
        amount_cents: amount,
        currency: 'EGP',
        items: [],
      }),
    });
    const order = await orderRes.json();

    // 3. Get payment key
    const paymentKeyRes = await fetch('https://accept.paymob.com/api/acceptance/payment_keys', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        auth_token: token,
        amount_cents: amount,
        expiration: 3600,
        order_id: order.id,
        billing_data: {
          email,
          first_name: name.split(' ')[0] || 'Customer',
          last_name: name.split(' ').slice(1).join(' ') || 'Name',
          phone_number: phone,
          apartment: 'NA',
          floor: 'NA',
          street: 'NA',
          building: 'NA',
          shipping_method: 'NA',
          postal_code: 'NA',
          city: 'Cairo',
          country: 'EG',
          state: 'NA',
        },
        currency: 'EGP',
        integration_id: integrationId,
      }),
    });
    const { token: paymentToken } = await paymentKeyRes.json();

    const iframeUrl = `https://accept.paymob.com/api/acceptance/iframes/${iframeId}?payment_token=${paymentToken}`;

    return new Response(
      JSON.stringify({ iframeUrl, orderId: order.id }),
      { headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    console.error('Paymob error:', error);
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});
```

---

## 🎯 الفكرة العامة لأي Payment Gateway:

### المكونات الأساسية:

1. **Frontend Component** - زر أو form للدفع
2. **Backend Function** - Edge Function يتصل بالـ API
3. **Success Handler** - كود يشتغل بعد الدفع الناجح

### الخطوات العامة:

```
1. اختار الـ Payment Gateway
   ↓
2. سجل حساب واحصل على API Keys
   ↓
3. ثبت الـ SDK أو Library
   ↓
4. أنشئ Component للدفع
   ↓
5. أنشئ Edge Function
   ↓
6. استبدل في BillingPayment.tsx
   ↓
7. جرب في Test Mode
   ↓
8. خلي الموقع Live!
```

---

## 🌍 أشهر Payment Gateways:

| الاسم | الدول | التكلفة | السهولة |
|------|-------|---------|---------|
| **Stripe** | عالمي | 2.9% + $0.30 | ⭐⭐⭐⭐⭐ |
| **PayPal** | عالمي | 3.49% + fixed | ⭐⭐⭐⭐⭐ |
| **Paymob** | مصر، MENA | 2.5% | ⭐⭐⭐⭐ |
| **Razorpay** | الهند | 2% | ⭐⭐⭐⭐ |
| **Fawry** | مصر | 2% | ⭐⭐⭐ |
| **HyperPay** | السعودية، الخليج | 2.75% | ⭐⭐⭐⭐ |
| **Tap** | الخليج | 2.5% | ⭐⭐⭐⭐ |

---

## ❓ أي واحد تختار؟

- **لو شغلك عالمي** → Stripe أو PayPal
- **لو في مصر** → Paymob أو Fawry
- **لو في السعودية/الخليج** → HyperPay أو Tap
- **لو في الهند** → Razorpay

---

## 📞 محتاج مساعدة؟

**قولي اسم الـ Payment Gateway اللي عايز تستخدمه** وأنا هعملك الكود كامل جاهز!

مثلاً قولي:
- "عايز Paymob"
- "عايز PayPal"
- "عايز Fawry"
- "عايز HyperPay"

وأنا هديك الكود كامل! 🚀
