# دليل حل مشاكل NMI Payment

## المشكلة الأساسية
عند الضغط على "بطاقة الائتمان / Visa" في صفحة الدفع، الصفحة كانت ترجع لنفس الشاشة بدون ما توضح السبب.

## الأسباب المحتملة

### 1. نقص NMI Credentials في `.env`
**الحل:** تأكد من وجود المفاتيح التالية في ملف `.env`:

```env
NMI_PRIVATE_KEY=your_nmi_security_key_here
NMI_API_URL=https://secure.safewebservices.com/api/v2/three-step
NMI_WEBHOOK_SIGNING_KEY=6A3E05DB0E4D021EABAD4377ED0F4755
```

**خطوات الحصول على NMI Private Key:**
1. سجل دخول على [NMI Dashboard](https://secure.safewebservices.com/)
2. اذهب إلى Settings → Security Keys
3. انسخ الـ Private Security Key
4. ضعه في `.env` بدلاً من `your_nmi_security_key_here`

### 2. الـ Edge Function مش شغالة
**التحقق:**
```bash
curl https://nejnzgwqpbuooffivdqo.supabase.co/functions/v1/nmi-create-payment
```

**الحل:** تأكد من أن الـ Edge Function deployed صح:
- افتح Supabase Dashboard
- روح على Edge Functions
- تأكد من وجود `nmi-create-payment` و `nmi-webhook`

### 3. Environment Variables مش متزامنة
**الحل:**
1. بعد تحديث `.env`، أعد تشغيل الـ dev server:
```bash
# أوقف الـ server (Ctrl+C)
npm run dev
```

2. لو بتستخدم Supabase CLI:
```bash
supabase functions deploy nmi-create-payment
supabase functions deploy nmi-webhook
```

## التحسينات اللي تمت

### 1. تحسين Error Handling
- إضافة console.log للـ debugging
- رسائل خطأ أوضح وأفضل
- عرض الـ errors بشكل مرئي في الصفحة

### 2. تحسين User Experience
- Loading state واضح أثناء معالجة الدفع
- Disable الأزرار أثناء المعالجة
- رسائل توضيحية أفضل

### 3. إضافة Logging
تم إضافة console.log في:
- بداية عملية الدفع
- حالة الـ response
- البيانات المستلمة
- أي أخطاء تحدث

## كيفية الاختبار

### 1. Test في الـ Browser Console
1. افتح صفحة الدفع
2. افتح Developer Tools (F12)
3. روح على tab Console
4. اضغط على "بطاقة الائتمان / Visa"
5. راقب الرسائل اللي بتظهر

### 2. ما تتوقع تشوفه
```
Initiating NMI payment... {amount: 1, invoice: "INV-86120366-029"}
Response status: 200
Payment data received: {formUrl: "https://...", tokenId: "..."}
Redirecting to: https://...
```

### 3. لو في مشكلة
- هتشوف error message واضح في الصفحة
- هتشوف details في الـ console
- الأزرار هترجع تاني للضغط

## الـ Webhook Setup

تأكد من أن الـ webhook مُعدّ في NMI:

1. **URL:** `https://nejnzgwqpbuooffivdqo.supabase.co/functions/v1/nmi-webhook`
2. **Signing Key:** `6A3E05DB0E4D021EABAD4377ED0F4755`
3. **Events:** Transaction Response

## Contact Support

لو المشكلة مستمرة:
1. اجمع الـ console logs
2. خد screenshot للـ error message
3. تواصل مع الـ technical support

## ملاحظات مهمة

- الـ NMI Private Key **سري جداً** - لا تشاركه أبداً
- الـ webhook signing key يُستخدم للتحقق من صحة الطلبات
- كل transaction بيرجع `formUrl` للـ redirect
- الـ customer بيدخل بيانات البطاقة في صفحة NMI (مش عندنا)
