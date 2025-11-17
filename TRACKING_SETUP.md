# Tracking & Analytics Setup Guide

This document provides guidance for setting up conversion tracking for the Last Seat Ticket website.

## Google Analytics Setup

1. Create a Google Analytics 4 property
2. Add the Google Analytics tracking code to `index.html`:

```html
<!-- Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=GA_MEASUREMENT_ID"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'GA_MEASUREMENT_ID');
</script>
```

## Google Ads Conversion Tracking

### Phone Call Conversions
Track phone clicks in the following components:
- `FloatingCTA.tsx` - Floating call button
- `TopDeals.tsx` - Call buttons on deal cards
- `Header.tsx` - Phone number in header
- `ContactForm.tsx` - Call button in contact section

Add this code where phone links are clicked:

```javascript
gtag('event', 'conversion', {
  'send_to': 'AW-CONVERSION_ID/CONVERSION_LABEL'
});
```

### Form Submission Conversions
Track form submissions in `ContactForm.tsx` after successful submission:

```javascript
gtag('event', 'conversion', {
  'send_to': 'AW-CONVERSION_ID/CONVERSION_LABEL',
  'transaction_id': ''
});
```

## Facebook Pixel (Optional)

Add Facebook Pixel for remarketing:

```html
<!-- Facebook Pixel Code -->
<script>
  !function(f,b,e,v,n,t,s)
  {if(f.fbq)return;n=f.fbq=function(){n.callMethod?
  n.callMethod.apply(n,arguments):n.queue.push(arguments)};
  if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';
  n.queue=[];t=b.createElement(e);t.async=!0;
  t.src=v;s=b.getElementsByTagName(e)[0];
  s.parentNode.insertBefore(t,s)}(window, document,'script',
  'https://connect.facebook.net/en_US/fbevents.js');
  fbq('init', 'YOUR_PIXEL_ID');
  fbq('track', 'PageView');
</script>
```

## Key Conversion Actions

1. **Phone Clicks** - Primary conversion (tel: links)
2. **Form Submissions** - Secondary conversion
3. **WhatsApp Clicks** - Secondary conversion
4. **Search Interactions** - Micro-conversion

## Testing

Use Google Tag Assistant and Google Ads Preview mode to verify tracking is working correctly before launching campaigns.
