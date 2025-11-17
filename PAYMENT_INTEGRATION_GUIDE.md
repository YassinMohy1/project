# Payment Gateway Integration Guide for Last Seat Ticket

This guide explains how to integrate payment processing into your flight booking website.

## Overview

The payment system is built using:
- **Stripe** for payment processing
- **Supabase** for storing booking and payment records
- **Edge Functions** for secure server-side payment processing

## Setup Steps

### 1. Get Stripe API Keys

1. Create a Stripe account at [https://dashboard.stripe.com/register](https://dashboard.stripe.com/register)
2. Navigate to Developers → API Keys
3. Copy your **Publishable Key** and **Secret Key**

### 2. Configure Environment Variables

Add to your `.env` file:

```env
VITE_STRIPE_PUBLISHABLE_KEY=pk_test_your_publishable_key_here
```

The Stripe Secret Key is automatically configured in Supabase Edge Functions.

### 3. Database Tables Created

Two tables have been created in your Supabase database:

#### `bookings` table
Stores customer booking information:
- Customer details (name, email, phone)
- Flight details (from, to, dates, passengers)
- Booking status and amount
- Created/updated timestamps

#### `payments` table
Stores payment transactions:
- Links to booking via `booking_id`
- Stripe payment intent ID
- Payment amount, currency, status
- Payment method and customer email

### 4. Edge Function Deployed

**Function:** `create-payment-intent`
- **Endpoint:** `{SUPABASE_URL}/functions/v1/create-payment-intent`
- **Purpose:** Creates Stripe payment intents securely on the server
- **Security:** Keeps your Stripe secret key secure

### 5. Components Available

#### `PaymentModal.tsx`
A complete payment form component with:
- Credit card input fields with validation
- Support for multiple payment methods
- Secure Stripe integration
- Fallback to phone booking if Stripe not configured

## How to Use

### Add Payment Button to Your Components

```typescript
import { useState } from 'react';
import PaymentModal from './components/PaymentModal';

function YourComponent() {
  const [showPayment, setShowPayment] = useState(false);

  const bookingDetails = {
    destination: 'New York to London',
    amount: 459,
    description: 'Round trip flight booking'
  };

  return (
    <>
      <button onClick={() => setShowPayment(true)}>
        Pay Now
      </button>

      <PaymentModal
        isOpen={showPayment}
        onClose={() => setShowPayment(false)}
        bookingDetails={bookingDetails}
      />
    </>
  );
}
```

### Save Booking to Database

```typescript
import { supabase } from '../lib/supabase';

async function createBooking(formData) {
  const { data, error } = await supabase
    .from('bookings')
    .insert([
      {
        customer_name: formData.name,
        customer_email: formData.email,
        customer_phone: formData.phone,
        from_city: formData.from,
        to_city: formData.to,
        departure_date: formData.departDate,
        return_date: formData.returnDate,
        passengers: formData.passengers,
        trip_type: formData.tripType,
        status: 'pending',
        amount: 459,
        notes: formData.notes
      }
    ])
    .select()
    .maybeSingle();

  if (error) {
    console.error('Error creating booking:', error);
    return null;
  }

  return data;
}
```

### Record Payment

```typescript
async function recordPayment(bookingId, paymentIntentId, amount) {
  const { data, error } = await supabase
    .from('payments')
    .insert([
      {
        booking_id: bookingId,
        payment_intent_id: paymentIntentId,
        amount: amount,
        currency: 'usd',
        status: 'succeeded',
        payment_method: 'card'
      }
    ])
    .select()
    .maybeSingle();

  return data;
}
```

## Payment Flow

1. **Customer fills booking form** → Saves to `bookings` table
2. **Customer clicks "Pay Now"** → Opens PaymentModal
3. **Customer enters card details** → Submits to your Edge Function
4. **Edge Function** → Creates Stripe Payment Intent
5. **Stripe processes payment** → Returns result
6. **Save payment record** → Updates `payments` table
7. **Update booking status** → Marks as confirmed
8. **Send confirmation** → Email/SMS to customer

## Supported Payment Methods

- 💳 **Credit/Debit Cards:** Visa, Mastercard, Amex, Discover
- 💰 **PayPal:** Integration ready (requires additional setup)
- 🍎 **Apple Pay:** Works automatically on Safari/iOS
- 📱 **Google Pay:** Works automatically on Chrome/Android
- 📅 **Afterpay:** For installment payments (requires setup)

## Testing

### Test Card Numbers (Stripe Test Mode)

```
Success: 4242 4242 4242 4242
Declined: 4000 0000 0000 0002
Requires 3D Secure: 4000 0025 0000 3155

Use any future expiry date (e.g., 12/34)
Use any 3-digit CVV (e.g., 123)
```

## Security Best Practices

✅ **Never expose secret keys** - Keep them in Supabase Edge Functions
✅ **Use HTTPS only** - All payment forms must use secure connections
✅ **Validate on server** - Always verify payments server-side
✅ **Enable RLS** - Row Level Security on all database tables
✅ **Log transactions** - Keep audit trail in database
✅ **PCI Compliance** - Stripe handles card data, not your servers

## Phone Fallback

If Stripe is not configured or payment fails:
- Modal shows fallback message
- Redirects to phone call: `tel:888-602-6667`
- Customer can complete booking over the phone
- Agent can manually process payment

## Webhook Integration (Optional)

Set up webhooks to handle payment events:

1. Go to Stripe Dashboard → Developers → Webhooks
2. Add endpoint: `{SUPABASE_URL}/functions/v1/stripe-webhook`
3. Select events: `payment_intent.succeeded`, `payment_intent.failed`
4. Copy webhook secret to environment

## Admin Dashboard (Future Enhancement)

You can build an admin panel to:
- View all bookings
- Track payment status
- Process refunds
- Manage customer inquiries

Query example:
```sql
SELECT
  b.*,
  p.status as payment_status,
  p.amount as paid_amount
FROM bookings b
LEFT JOIN payments p ON b.id = p.booking_id
ORDER BY b.created_at DESC;
```

## Support

For payment integration support:
- Stripe Docs: [https://stripe.com/docs](https://stripe.com/docs)
- Supabase Docs: [https://supabase.com/docs](https://supabase.com/docs)
- Call: 888-602-6667 for custom implementation help

## Cost Estimation

**Stripe Fees:**
- 2.9% + $0.30 per successful card charge
- No monthly fees or setup costs

**Supabase:**
- Free tier includes 500MB database + 2GB bandwidth
- Paid plans start at $25/month for production use

---

**Note:** Payment integration is ready to use. Just add your Stripe keys and the system will automatically process payments securely.
