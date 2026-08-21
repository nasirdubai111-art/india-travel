# BusHub Pro - Architecture & Mobile Mockups

---

## SYSTEM ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  Web App         │  │  iOS App         │  │  Android App     │   │
│  │  React.js        │  │  React Native    │  │  React Native    │   │
│  │  TypeScript      │  │  Swift Bridge    │  │  Java Bridge     │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           │                     │                     │              │
│           └─────────────────────┼─────────────────────┘              │
│                                 │                                     │
│                        API Gateway (REST/GraphQL)                     │
│                        Rate Limiting | Auth Proxy                     │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────────┐
│                    BACKEND SERVICE LAYER                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────────────┐  ┌──────────────────┐  ┌──────────────────┐│
│  │ AUTH SERVICE        │  │ BOOKING SERVICE  │  │ PAYMENT SERVICE  ││
│  ├─────────────────────┤  ├──────────────────┤  ├──────────────────┤│
│  │ • OTP Verification  │  │ • Search Routes  │  │ • Payment Init   ││
│  │ • JWT Management    │  │ • Seat Selection │  │ • Gateway Intg   ││
│  │ • Token Refresh     │  │ • Booking Logic  │  │ • Refund Process ││
│  │ • Role Validation   │  │ • Discount Apply │  │ • Invoice Gen    ││
│  └─────────────────────┘  └──────────────────┘  └──────────────────┘│
│                                                                       │
│  ┌─────────────────────┐  ┌──────────────────┐  ┌──────────────────┐│
│  │ HOTEL SERVICE       │  │ TOUR SERVICE     │  │ NOTIFICATION SVC ││
│  ├─────────────────────┤  ├──────────────────┤  ├──────────────────┤│
│  │ • Inventory Mgmt    │  │ • Package Build  │  │ • Email (SendGrid)│
│  │ • Rate Management   │  │ • Pricing Logic  │  │ • SMS (Twilio)   ││
│  │ • Partnership API   │  │ • Event Mgmt     │  │ • Push Notif     ││
│  │ • Booking Intg      │  │ • Logistics Plan │  │ • In-app Messages││
│  └─────────────────────┘  └──────────────────┘  └──────────────────┘│
│                                                                       │
│  ┌─────────────────────┐  ┌──────────────────┐  ┌──────────────────┐│
│  │ DISCOUNT ENGINE     │  │ ANALYTICS SERVICE│  │ OPERATOR SERVICE ││
│  ├─────────────────────┤  ├──────────────────┤  ├──────────────────┤│
│  │ • Tier Calculation  │  │ • Revenue Stats  │  │ • Route Mgmt     ││
│  │ • Dynamic Pricing   │  │ • Occupancy Data │  │ • Driver Assign  ││
│  │ • Bulk Discounts    │  │ • User Analytics │  │ • Fleet Tracking ││
│  │ • Seasonal Offers   │  │ • Dashboard Data │  │ • Report Gen     ││
│  └─────────────────────┘  └──────────────────┘  └──────────────────┘│
│                                                                       │
└───────────────────┬──────────────────────────────────┬────────────────┘
                    │                                  │
┌───────────────────────────────────────────────────────────────────────┐
│              CACHING & MESSAGE LAYER                                   │
├───────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐ │
│  │ Redis Cache      │  │ RabbitMQ Queue   │  │ Elasticsearch       │ │
│  ├──────────────────┤  ├──────────────────┤  ├─────────────────────┤ │
│  │ • Seat Avail     │  │ • Booking Events │  │ • Activity Logs    │ │
│  │ • Route Data     │  │ • Notif Queue    │  │ • Audit Trails     │ │
│  │ • Discount Cache │  │ • Email Queue    │  │ • Search Indexing  │ │
│  │ • Session Store  │  │ • SMS Queue      │  │ • Analytics Events │ │
│  └──────────────────┘  └──────────────────┘  └─────────────────────┘ │
│                                                                         │
└────────────────────────────┬──────────────────────────────────────────┘
                             │
┌────────────────────────────────────────────────────────────────────────┐
│           DATABASE LAYER                                                │
├────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Primary Database: MySQL 8.0 (Main Read/Write)                  │   │
│  │ Tables:                                                         │   │
│  │ • Users, Routes, Schedules, Seats                              │   │
│  │ • Bookings, Discounts, Bulk Bookings                           │   │
│  │ • Hotels, Partnerships, Tours, Events                          │   │
│  │ • Payments, Invoices                                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Read Replicas (For Analytics & Reporting)                       │   │
│  │ • Cross-region replication for failover                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└────────────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────────────────────────────────────────────────┐
│         EXTERNAL INTEGRATIONS & STORAGE                                 │
├────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │
│  │ Payment Gateways │  │ Cloud Storage    │  │ Maps & Location     │  │
│  ├──────────────────┤  ├──────────────────┤  ├─────────────────────┤  │
│  │ • Razorpay       │  │ • AWS S3         │  │ • Google Maps API   │  │
│  │ • PhonePe        │  │ • CloudFront CDN │  │ • Location Services │  │
│  │ • Stripe         │  │ • CloudWatch Logs│  │ • Geofencing        │  │
│  └──────────────────┘  └──────────────────┘  └─────────────────────┘  │
│                                                                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │
│  │ Communication    │  │ Monitoring       │  │ Document Gen        │  │
│  ├──────────────────┤  ├──────────────────┤  ├─────────────────────┤  │
│  │ • Twilio SMS     │  │ • New Relic      │  │ • PDFKit            │  │
│  │ • SendGrid Email │  │ • Datadog        │  │ • QR Code Gen       │  │
│  │ • Firebase Push  │  │ • Sentry Logs    │  │ • Barcode Gen       │  │
│  └──────────────────┘  └──────────────────┘  └─────────────────────┘  │
│                                                                          │
└────────────────────────────────────────────────────────────────────────┘
```

---

## DATA FLOW DIAGRAMS

### Booking Flow
```
User Search
    ↓
[Route Search Service] → Redis Cache check
    ↓
DB Query (Routes + Schedules)
    ↓
Apply Discounts [Discount Engine]
    ↓
Return Available Routes
    ↓
User Selects Route
    ↓
[Seat Service] → Real-time seat availability
    ↓
User Selects Seats
    ↓
Calculate final price with tax
    ↓
[Payment Service] → Initiate payment
    ↓
Payment Gateway (Razorpay/PhonePe)
    ↓
Payment Success?
    ├─ YES → [Booking Service] Create booking
    │          Lock seats in DB
    │          Send confirmation SMS/Email
    │          Clear cache
    │          Queue booking event
    │          Return e-ticket
    │
    └─ NO → Retry or cancel
```

### Bulk Booking Flow (Travel Agent)
```
Travel Agent Login
    ↓
[Auth Service] - Validate agent credentials
    ↓
Access Bulk Booking Dashboard
    ↓
Select Multiple Routes/Dates
    ↓
Enter Passenger Count
    ↓
[Discount Engine] - Calculate tier discount
    ├─ 10-24 passengers → 15% discount
    ├─ 25-39 passengers → 25% discount
    └─ 40+ passengers → 35% discount
    ↓
[Pricing Service] - Generate invoice
    ↓
Confirm Booking
    ↓
[Bulk Booking Service] - Create booking group
    ↓
Generate commission breakdown
    ↓
Send group confirmation
    ↓
Queue individual e-tickets
```

### Hotel Integration Flow
```
User Books Bus Ticket
    ↓
[Booking Service] Extracts destination
    ↓
[Hotel Service] → Query nearby hotels in destination
    ↓
Apply partnership discounts
    ↓
Show hotel options to user
    ↓
User books hotel (optional)
    ↓
[Combined Invoice Service]
    │
    ├─ Bus ticket charges (5% GST)
    ├─ Hotel accommodation (5% GST)
    ├─ Service charges (18% GST)
    │
    └─ Generate combined invoice
    ↓
Payment for combined package
```

---

## MOBILE APP SCREEN MOCKUPS

### CUSTOMER APP - SCREEN FLOW

#### Screen 1: Login/OTP
```
┌─────────────────────────────┐
│    Welcome to BusHub Pro    │
│                             │
│  📱 Enter your phone number │
│  ┌───────────────────────┐  │
│  │ +91 [_ _ _ _ _____]   │  │
│  └───────────────────────┘  │
│                             │
│  ☑ I agree to Terms        │
│                             │
│   ┌─────────────────────┐   │
│   │  Send OTP           │   │
│   └─────────────────────┘   │
│                             │
│  Powered by BusHub Pro      │
└─────────────────────────────┘
```

#### Screen 2: Route Search & Results
```
┌─────────────────────────────┐
│ ← Search Results            │
│                             │
│  From: BNG  ↔️  To: HYD      │
│  📅 24 Jul, Tuesday          │
│                             │
│  ▯ Filters [2] | Sort by ⬇  │
│                             │
│  🚌 SafariGold Express       │
│  BNG 08:00 → 14:00 HYD      │
│  ⭐ 4.8 | 45 seats           │
│  ₹1020 → ₹867 Save ₹153     │
│  ┌──────────────────────┐   │
│  │  Select seats        │   │
│  └──────────────────────┘   │
│                             │
│  🚌 TravelKing Sleeper      │
│  BNG 22:00 → 06:30 HYD      │
│  ⭐ 4.6 | 42 seats           │
│  ₹1500 → ₹1275 Save ₹225    │
│  ┌──────────────────────┐   │
│  │  Select seats        │   │
│  └──────────────────────┘   │
└─────────────────────────────┘
```

#### Screen 3: Seat Selection
```
┌─────────────────────────────┐
│ ← SafariGold Express         │
│                             │
│  BNG 08:00 → 14:00 HYD      │
│  Seat Layout | Price | Info  │
│                             │
│      FRONT (DRIVER)          │
│  ▢ 1   ▢ 2  |  ◼ 3  ▢ 4     │
│  ▢ 5   ▢ 6  |  ▢ 7  ▢ 8     │
│  ✓ 9   ▢ 10 | ◼ 11 ▢ 12    │
│  ▢ 13  ▢ 14 | ▢ 15 ✓ 16    │
│  ▢ 17  ▢ 18 | ◼ 19 ▢ 20    │
│  ▢ 21  ◼ 22 | ▢ 23 ▢ 24    │
│                             │
│      BACK (EXIT)             │
│                             │
│ ✓ = Selected (2 seats)       │
│ ◼ = Booked                   │
│ ▢ = Available                │
│                             │
│ Base fare: ₹1200 × 2         │
│ Discount (15%): -₹360        │
│ Total: ₹1320 + ₹118 GST      │
│                             │
│   ┌──────────────────────┐   │
│   │  Proceed to Payment  │   │
│   └──────────────────────┘   │
└─────────────────────────────┘
```

#### Screen 4: Payment
```
┌─────────────────────────────┐
│ ← Payment                    │
│                             │
│  Order Summary               │
│  ┌─────────────────────────┐ │
│  │ SafariGold Express      │ │
│  │ Seats: 9, 16            │ │
│  │ BNG → HYD               │ │
│  │ ₹1020 × 2 = ₹2040       │ │
│  │ Discount (15%): -₹306   │ │
│  │ Subtotal: ₹1734         │ │
│  │ GST (5%): ₹86.70        │ │
│  │ ─────────────────────── │ │
│  │ Total: ₹1820.70         │ │
│  └─────────────────────────┘ │
│                             │
│  Payment Method              │
│  ◉ Credit/Debit Card        │
│  ○ Net Banking              │
│  ○ UPI                       │
│  ○ Wallet                    │
│                             │
│  ┌─────────────────────────┐ │
│  │ 💳 Card Number          │ │
│  │ [____________]          │ │
│  └─────────────────────────┘ │
│                             │
│   ┌──────────────────────┐   │
│   │  Pay ₹1820.70        │   │
│   └──────────────────────┘   │
└─────────────────────────────┘
```

#### Screen 5: Booking Confirmation
```
┌─────────────────────────────┐
│    ✓ Booking Confirmed      │
│                             │
│    Booking Reference         │
│    BHP-2607-8947            │
│                             │
│    SafariGold Express        │
│    BNG → HYD                │
│    24 July 2026             │
│    08:00 AM departure       │
│                             │
│    Seats: 9A, 16A           │
│                             │
│    Passenger Details         │
│    John Doe                  │
│    Phone: 98765 43210        │
│    Email: john@email.com     │
│                             │
│    Amount Paid               │
│    ₹1820.70                  │
│                             │
│  ┌──────────────────────────┐│
│  │ 🎫 View E-Ticket         ││
│  └──────────────────────────┘│
│                             │
│  ┌──────────────────────────┐│
│  │ 📋 View Invoice          ││
│  └──────────────────────────┘│
│                             │
│  ┌──────────────────────────┐│
│  │ 🏨 Book Hotel in HYD    ││
│  └──────────────────────────┘│
│                             │
│  Share | Download | Close   │
└─────────────────────────────┘
```

---

### TRAVEL AGENT APP - SCREEN FLOW

#### Screen 1: Agent Dashboard
```
┌─────────────────────────────┐
│ 👋 Hi, Rajesh!              │
│                             │
│ 📊 Your Performance          │
│ ┌─────────────────────────┐ │
│ │ Total Bookings          │ │
│ │ 2,847                   │ │
│ └─────────────────────────┘ │
│ ┌─────────────────────────┐ │
│ │ This Month Revenue      │ │
│ │ ₹45.2 Lakhs             │ │
│ └─────────────────────────┘ │
│ ┌─────────────────────────┐ │
│ │ Commission Earned       │ │
│ │ ₹9.8 Lakhs              │ │
│ └─────────────────────────┘ │
│ ┌─────────────────────────┐ │
│ │ Active Customers        │ │
│ │ 542                     │ │
│ └─────────────────────────┘ │
│                             │
│  ⭐ Tier: Gold              │
│  Next tier: Platinum (3000) │
│                             │
│  [Create Group Booking]     │
│  [View Commissions]         │
│  [Customer Database]        │
│  [Reports]                  │
└─────────────────────────────┘
```

#### Screen 2: Bulk Booking
```
┌─────────────────────────────┐
│ ← Create Bulk Booking        │
│                             │
│  Group Name                  │
│  ┌───────────────────────┐  │
│  │ IIM Alumni Meetup     │  │
│  └───────────────────────┘  │
│                             │
│  Number of Passengers        │
│  ┌───────────────────────┐  │
│  │ 50                    │  │
│  └───────────────────────┘  │
│                             │
│  Route 1                     │
│  From: BNG  To: HYD          │
│  Date: 24 Jul | Qty: 25      │
│  Discount Applied: 35%       │
│  Price: ₹18,250             │
│                             │
│  + Add Another Route         │
│                             │
│  ┌─────────────────────────┐│
│  │ Discount Summary        ││
│  │ 35% (50+ passengers)    ││
│  │ Total Savings: ₹22,000  ││
│  └─────────────────────────┘│
│                             │
│   ┌──────────────────────┐   │
│   │ Generate Invoice     │   │
│   └──────────────────────┘   │
└─────────────────────────────┘
```

---

### OPERATOR APP - SCREEN FLOW

#### Screen 1: Operator Dashboard
```
┌─────────────────────────────┐
│ Fleet Status                │
│                             │
│ Fleet Utilization           │
│ ████████░░ 87%              │
│                             │
│ Active Routes: 24           │
│ Buses Deployed: 18          │
│ Buses Available: 6          │
│                             │
│ Today's Revenue             │
│ ₹3.2 Lakhs                  │
│                             │
│ Average Occupancy           │
│ 78%                         │
│                             │
│ Rating                      │
│ ⭐⭐⭐⭐⭐ 4.8                │
│ (5,234 reviews)             │
│                             │
│ [Manage Routes]             │
│ [Set Discounts]             │
│ [Driver Assignments]        │
│ [Analytics]                 │
│                             │
│ Pending Actions: 3          │
│ • 5 cancellation requests   │
│ • 2 route changes needed    │
└─────────────────────────────┘
```

#### Screen 2: Discount Management
```
┌─────────────────────────────┐
│ ← Discount Management        │
│                             │
│ Dynamic Discount Control     │
│                             │
│ BNG → HYD Route              │
│ ├─ Base Occupancy: 65%      │
│ ├─ Time to Departure: 18h   │
│ ├─ Suggested Discount: 18%  │
│ │                           │
│ └─ Set Discount             │
│    [Slider: 0─50%]          │
│    Current: 15%             │
│                             │
│ BNG → Mysore Route           │
│ ├─ Occupancy: 92%           │
│ ├─ Suggested: 0%            │
│ └─ Set Discount: 0%         │
│                             │
│ Bulk Booking Tiers          │
│ ┌─────────────────────────┐ │
│ │ 10-24 passengers: 15%   │ │
│ │ 25-39 passengers: 25%   │ │
│ │ 40+ passengers: 35%     │ │
│ └─────────────────────────┘ │
│                             │
│ [Edit Tiers]                │
│                             │
│ Active Promotions            │
│ • Early bird (7+ days): 5%  │
│ • Weekend: +10%             │
│ • Peak hours: +8%           │
└─────────────────────────────┘
```

---

### TOUR OPERATOR APP - SCREEN FLOW

#### Screen 1: Tour Package Builder
```
┌─────────────────────────────┐
│ + Create Tour Package        │
│                             │
│ Package Name                 │
│ ┌───────────────────────┐  │
│ │ Kerala Backwaters...  │  │
│ └───────────────────────┘  │
│                             │
│ Duration                     │
│ ◉ 3 days 2 nights           │
│ ○ 4 days 3 nights           │
│ ○ 5 days 4 nights           │
│                             │
│ Destinations (Multi-select)  │
│ ☑ Kochi ☑ Munnar             │
│ ☑ Thekkady ☐ Varkala        │
│                             │
│ Include Services             │
│ ☑ Bus Transportation         │
│ ☑ Hotel Accommodation (3*)   │
│ ☑ Meal Package               │
│ ☑ Tour Guide                 │
│ ☑ Activity Passes            │
│                             │
│ Base Cost: ₹8,500            │
│ Discount: 15% (Group offer) │
│ Final Price: ₹7,225          │
│                             │
│ Max Passengers: 50           │
│ Minimum Booking: 10          │
│                             │
│   ┌──────────────────────┐   │
│   │  Create Package      │   │
│   └──────────────────────┘   │
└─────────────────────────────┘
```

---

## RESPONSIVE DESIGN BREAKPOINTS

### Mobile First Approach
- **Mobile (320px - 767px):** Single column, touch-optimized
- **Tablet (768px - 1024px):** 2-3 columns, larger touch targets
- **Desktop (1025px+):** Full multi-column layout

### Key UI/UX Principles

1. **Search Component**
   - Sticky top on mobile
   - Auto-complete suggestions
   - Recent searches cache
   - Location detection

2. **Seat Selection**
   - Responsive grid layout
   - Touch-friendly seat sizes (min 40x40px)
   - Color-coded legend
   - Clear selection count

3. **Payment**
   - Simplified mobile checkout
   - Autofill from saved cards
   - One-click payment
   - Secure enclosure

4. **Navigation**
   - Bottom tab bar on mobile
   - Hamburger menu for secondary options
   - Clear back button hierarchy

---

## ACCESSIBILITY STANDARDS

- WCAG 2.1 AA compliance
- Screen reader optimization
- Keyboard navigation support
- Color contrast ratios ≥ 4.5:1
- Font sizes ≥ 12px on mobile

---

## PERFORMANCE OPTIMIZATION

### Mobile App
- Image lazy loading
- Code splitting by route
- Service worker for offline capability
- Local caching of search results
- Compressed assets

### Web App
- Server-side rendering (Next.js)
- Static asset caching (1 year)
- API response caching (Redis)
- Bundle size < 250KB (gzipped)
- Lighthouse score ≥ 90

---

**Mockup Version:** 1.0  
**Last Updated:** July 2026
