# 0.1% DEV — System Design

## Product

**0.1% DEV** is a developer ed-tech platform selling deep CS fundamentals courses — "Build a Programming Language", "JavaScript Engine Internals", "Build a Real-Time System", etc. The tagline is "Become *THAT* Developer." Target audience: developers who want to understand systems at the level most engineers never reach.

Courses are video-based. Access is lifetime after a one-time purchase. Preview mode exposes the first 3 chapters of every course for free.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  CLIENT (Browser)                                               │
│  Vue 3 / Nuxt 3 · Tailwind CSS · Pinia (+ persistedState)      │
│  Video.js · Razorpay checkout.js                                │
└──────────────────────┬──────────────────────────────────────────┘
                       │ SSR (first load) / SPA (navigation)
┌──────────────────────▼──────────────────────────────────────────┐
│  SERVER (Nitro — Nuxt's built-in server)                        │
│  /api/paymentRZ  →  Razorpay Node SDK  →  Razorpay API         │
│  /api/payments   →  PhonePe SHA256 HMAC  →  PhonePe API        │
└──────────────────────┬──────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────────┐
│  FIREBASE (Google)                                              │
│  Auth     — Google Sign-In (signInWithPopup)                    │
│  Firestore — users/{uid} · PrevCourses[] · purchase history     │
│  Storage  — MP4 video files (range-request streaming)           │
│  App Check — reCAPTCHA Enterprise token validation              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Rationale |
|---|---|---|
| Frontend framework | Nuxt 3 (Vue 3) | SSR for SEO, SPA for app feel, Nitro for API routes |
| Styling | Tailwind CSS | Utility-first, no stylesheet bloat |
| State | Pinia + persistedstate | Purchase state survives refresh without re-auth |
| Auth | Firebase Auth (Google) | One-click sign-in, no password management |
| Database | Firestore | Serverless, real-time, scales to zero |
| Video storage | Firebase Storage | CDN-backed, Range requests, App Check protected |
| Video player | Video.js | Mature, customizable, supports adaptive streaming |
| Payments (primary) | Razorpay | Dominant in India, supports UPI/card/netbanking |
| Payments (secondary) | PhonePe | UPI-native, high conversion for Indian users |
| Server runtime | Nitro (Nuxt built-in) | API routes colocated with frontend, no separate backend |
| Analytics | Google Analytics 4 + Meta Pixel | Course purchase attribution |
| Security | Firebase App Check (reCAPTCHA Enterprise) | Blocks unauthorized Storage access |

---

## Data Model

### Firestore: `users/{uid}`

```json
{
  "uid": "aBc123XyZ",
  "user": {
    "name": "Swanand Kadam",
    "email": "user@gmail.com",
    "photoURL": "https://...",
    "accessToken": "eyJhbGci...",
    "creationTime": "Mon, 01 Jan 2024 00:00:00 GMT",
    "lastSignInTime": "Mon, 01 Jan 2024 00:00:00 GMT"
  },
  "PrevCourses": [2, 6],
  "currentCourse": []
}
```

`PrevCourses[]` is the single source of truth for access control. `arrayUnion` ensures atomic concurrent writes.

### Local data files (not in Firestore)

- `data/tracks.js` — full course catalog with video chapter URLs (used in course player and detail pages)
- `data/trackspreview.js` — same structure, used for public course listing

Course metadata is static — no CMS, no API. New courses require a code deploy.

---

## Core Flows

### 1. Authentication

```
User clicks "Buy Track"
  → BuyTrack() checks Pinia: isUserLoggedIn?
  → No → router.push('/signin')
  → signInWithPopup(auth, GoogleAuthProvider)
  → onAuthStateChanged fires → uid received
  → _FBisNewUser(uid) → getDoc(users/{uid})
  → New user: _FBNewSignup() → setDoc with empty PrevCourses
  → Existing: _FBgetLoggedUserandStoreHistory() → fetch doc
  → Pinia store updated: isUserLoggedIn=true, PrevCourses loaded
  → persistedState → localStorage
  → router.push('/')
```

### 2. Purchase (Razorpay)

```
BuyTrack() → user logged in
  → handlePayRZ()
  → $fetch POST /api/paymentRZ { amount: 179900, currency: 'INR' }
  → [Nitro] new Razorpay(keys).orders.create(...)
  → Razorpay API returns { id: 'order_xyz', amount: 179900 }
  → Client: new window.Razorpay(options).open()
  → User pays (card / UPI / netbanking)
  → handler(response) fires: { payment_id, order_id, signature }
  → handlePaymentSuccess()
  → addCoursetoUserAccount(courseId)
  → updateDoc(users/{uid}, { PrevCourses: arrayUnion(courseId) })
  → Pinia: PrevCourses.push(courseId)
  → UI: "Buy Track" → "Start Track"
```

### 3. Video Access

```
User navigates /course/:id
  → isCourseBought(courseId) → Pinia.PrevCourses.includes(id)
  → Chapter list renders:
    - index < 3 → always unlocked (preview)
    - index ≥ 3 → locked unless isCourseBought()
  → User clicks chapter:
    → decodeSrc(chapter.src) → XOR decode → real Firebase URL
    → App Check token generated (reCAPTCHA Enterprise)
    → Video.js initialized with decoded src
    → Browser: GET Storage URL + Range header
    → Firebase validates App Check token
    → 206 Partial Content → chunks stream to Video.js
```

---

## Security

### Payment security
- Razorpay order created **server-side** (Nitro route) — amount cannot be tampered from client
- Live API keys in server-side code only; secret key never exposed to browser

### Video protection
- All video URLs are **XOR-obfuscated** with key `thisisasecretofus` and base64-encoded in the source data
- URLs are only decoded client-side at playback time
- **Firebase App Check** (reCAPTCHA Enterprise) validates every Storage request — raw URLs are useless without a valid token
- Access gate is enforced client-side via `isCourseBought()` on Pinia state

### Authentication
- Google OAuth only — no password storage, no credential management
- Firebase Auth manages token refresh automatically
- `onAuthStateChanged` is the single source of truth for auth state

### Known trade-offs
- XOR obfuscation is **not true DRM** — a determined attacker with valid App Check credentials could capture URLs at runtime. For a course platform at this scale it's an acceptable trade-off vs. full DRM complexity and cost.
- `isCourseBought()` is client-side only — Pinia state can be manually edited in DevTools. A production hardening would add a server-side check before streaming.
- Live Razorpay and PhonePe keys are hardcoded in server API files — should be moved to environment variables.

---

## Geo-Aware Pricing

```
User signs in → currency store field set based on country detection
  → 'INR' → priceINR (e.g. 179900 paise)
  → 'USD' → priceELSE (e.g. 2500 cents)

// In handlePayRZ()
const courseprice = userSessionStore.getCurrency === '$'
  ? courseToShow.priceELSE
  : courseToShow.priceINR;
```

---

## Scaling Considerations

| Concern | Current approach | At scale |
|---|---|---|
| Course catalog | Static local JS file | Move to Firestore or CMS; cache at CDN edge |
| Video delivery | Firebase Storage direct | Firebase Storage + Cloud CDN handles this natively |
| Payment webhooks | Not implemented — relies on client callback | Add Razorpay webhook → server-side purchase verification |
| Auth token refresh | Firebase SDK handles automatically | No action needed |
| Concurrent purchases | Firestore `arrayUnion` is atomic | Already correct |
| New courses | Code deploy required | Add CMS layer; keep video URLs in Firestore |

---

## File Structure (key files)

```
01dev/
├── pages/
│   ├── index.vue              — Landing page
│   ├── tracks.vue             — Course listing
│   ├── Coursedetails/[id].vue — Course detail + buy button
│   └── Course/[courseid].vue  — Video player + chapter list
├── components/
│   ├── Tracks/List.vue        — Course cards grid
│   └── Tracks/Main.vue        — Course detail + purchase flow
├── server/api/
│   ├── paymentRZ.js           — Razorpay order creation (Nitro)
│   └── payments.js            — PhonePe payment (Nitro)
├── stores/
│   └── userSession.js         — Pinia: auth, purchases, currency
├── data/
│   ├── tracks.js              — Full course catalog (gated)
│   └── trackspreview.js       — Public course listing
├── firebase.client.js         — Firebase init (client-only)
└── nuxt.config.ts             — Modules, GA, Meta Pixel, App Check
```
