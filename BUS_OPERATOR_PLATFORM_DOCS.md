# BusHub Pro - Bus Operator Platform
## Complete Technical Documentation

---

## 1. PLATFORM OVERVIEW

### Vision
A comprehensive multi-role bus booking and transportation platform enabling seamless ticket sales, bulk discounts, hotel partnerships, and event/tour management across web and mobile platforms.

### Key Features
- **Real-time seat reservation** with dynamic pricing
- **Role-based dashboards** for 4 user types
- **Bulk booking discounts** with tiered systems
- **Hotel & event tie-ups** integration
- **Payment processing** with GST/invoicing
- **Mobile-responsive design** for all platforms

---

## 2. USER ROLES & PERMISSIONS

### A. Individual Customer
**Primary Needs:**
- Browse available routes
- Real-time seat availability
- Instant booking & e-tickets
- Hotel offers in destination cities
- Payment via multiple gateways

**Permissions:**
- View routes & prices
- Book tickets
- Cancel with refund policy
- View booking history
- Download e-tickets & invoices

---

### B. Travel Agent
**Primary Needs:**
- Bulk booking for groups
- Tiered discounts (10+ to 40+ passengers)
- Commission tracking
- Customer management
- Invoice generation

**Permissions:**
- View all routes (agent pricing)
- Create bulk bookings
- Access agent dashboard
- View commission structure
- Generate group invoices
- Manage customer database

---

### C. Bus Operator
**Primary Needs:**
- Fleet management
- Dynamic discount management
- Real-time revenue tracking
- Booking confirmations
- Vehicle & driver management

**Permissions:**
- Create/edit routes
- Set dynamic discounts
- View real-time fleet status
- Manage driver assignments
- View revenue analytics
- Accept/reject bookings

---

### D. Tour Operator
**Primary Needs:**
- Multi-day tour packages
- Integrated hotel bookings
- Group coordination
- Event/function catering
- Combined pricing

**Permissions:**
- Create tour packages
- Bundle bus + hotel deals
- Access hotel inventory
- Manage group bookings
- Event booking (marriage, corporate)
- Combined invoicing

---

## 3. DATABASE SCHEMA

```sql
-- USERS TABLE
CREATE TABLE users (
  id VARCHAR(36) PRIMARY KEY,
  phone VARCHAR(10) UNIQUE,
  email VARCHAR(100) UNIQUE,
  name VARCHAR(100),
  role ENUM('customer', 'agent', 'operator', 'tour'),
  password_hash VARCHAR(255),
  kyc_verified BOOLEAN DEFAULT FALSE,
  gst_number VARCHAR(15),
  agency_name VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ROUTES TABLE
CREATE TABLE routes (
  id VARCHAR(36) PRIMARY KEY,
  operator_id VARCHAR(36) REFERENCES users(id),
  from_city VARCHAR(50),
  to_city VARCHAR(50),
  departure_time TIME,
  arrival_time TIME,
  base_fare DECIMAL(8,2),
  bus_type ENUM('AC', 'NonAC', 'Premium', 'Sleeper'),
  total_seats INT,
  status ENUM('active', 'inactive', 'scheduled'),
  created_at TIMESTAMP,
  UNIQUE(operator_id, from_city, to_city, departure_time)
);

-- SCHEDULES TABLE (Route instances by date)
CREATE TABLE schedules (
  id VARCHAR(36) PRIMARY KEY,
  route_id VARCHAR(36) REFERENCES routes(id),
  travel_date DATE,
  available_seats INT,
  discount_percent DECIMAL(5,2),
  is_holiday BOOLEAN,
  status ENUM('bookable', 'cancelled', 'full')
);

-- SEATS TABLE
CREATE TABLE seats (
  id VARCHAR(36) PRIMARY KEY,
  schedule_id VARCHAR(36) REFERENCES schedules(id),
  seat_number VARCHAR(5),
  status ENUM('available', 'booked', 'reserved', 'blocked'),
  booked_by VARCHAR(36) REFERENCES users(id),
  boarding_point VARCHAR(100),
  unique(schedule_id, seat_number)
);

-- BOOKINGS TABLE
CREATE TABLE bookings (
  id VARCHAR(36) PRIMARY KEY,
  booking_ref VARCHAR(20) UNIQUE,
  customer_id VARCHAR(36) REFERENCES users(id),
  agent_id VARCHAR(36) REFERENCES users(id),
  schedule_id VARCHAR(36) REFERENCES schedules(id),
  route_id VARCHAR(36) REFERENCES routes(id),
  seat_ids JSON,
  passenger_details JSON,
  base_amount DECIMAL(10,2),
  discount_amount DECIMAL(10,2),
  total_amount DECIMAL(10,2),
  gst DECIMAL(10,2),
  final_amount DECIMAL(10,2),
  status ENUM('pending', 'confirmed', 'cancelled'),
  payment_status ENUM('unpaid', 'paid', 'refunded'),
  created_at TIMESTAMP,
  cancelled_at TIMESTAMP,
  cancellation_reason VARCHAR(255)
);

-- DISCOUNTS TABLE
CREATE TABLE discounts (
  id VARCHAR(36) PRIMARY KEY,
  operator_id VARCHAR(36) REFERENCES users(id),
  discount_name VARCHAR(100),
  discount_type ENUM('percentage', 'fixed', 'bulk', 'seasonal'),
  discount_value DECIMAL(5,2),
  min_passengers INT,
  max_passengers INT,
  min_travel_date DATE,
  max_travel_date DATE,
  applicable_routes JSON,
  status ENUM('active', 'inactive'),
  created_at TIMESTAMP
);

-- DISCOUNT_TIERS TABLE
CREATE TABLE discount_tiers (
  id VARCHAR(36) PRIMARY KEY,
  discount_id VARCHAR(36) REFERENCES discounts(id),
  min_count INT,
  max_count INT,
  discount_percent DECIMAL(5,2)
);

-- BULK BOOKINGS TABLE
CREATE TABLE bulk_bookings (
  id VARCHAR(36) PRIMARY KEY,
  bulk_ref VARCHAR(20) UNIQUE,
  agent_id VARCHAR(36) REFERENCES users(id),
  total_passengers INT,
  routes_included JSON,
  discount_tier VARCHAR(50),
  applied_discount_percent DECIMAL(5,2),
  total_value DECIMAL(12,2),
  individual_booking_ids JSON,
  status ENUM('draft', 'confirmed', 'partial_confirmed', 'cancelled'),
  created_at TIMESTAMP
);

-- HOTELS TABLE
CREATE TABLE hotels (
  id VARCHAR(36) PRIMARY KEY,
  hotel_name VARCHAR(100),
  city VARCHAR(50),
  category ENUM('1star', '2star', '3star', '4star', '5star'),
  base_rate DECIMAL(10,2),
  star_rating DECIMAL(3,1),
  address VARCHAR(255),
  phone VARCHAR(10),
  email VARCHAR(100),
  created_at TIMESTAMP
);

-- HOTEL_PARTNERSHIPS TABLE
CREATE TABLE hotel_partnerships (
  id VARCHAR(36) PRIMARY KEY,
  operator_id VARCHAR(36) REFERENCES users(id),
  hotel_id VARCHAR(36) REFERENCES hotels(id),
  commission_percent DECIMAL(5,2),
  discount_for_passengers DECIMAL(5,2),
  contract_start_date DATE,
  contract_end_date DATE,
  active BOOLEAN DEFAULT TRUE
);

-- TOUR_PACKAGES TABLE
CREATE TABLE tour_packages (
  id VARCHAR(36) PRIMARY KEY,
  operator_id VARCHAR(36) REFERENCES users(id),
  package_name VARCHAR(100),
  description TEXT,
  duration_days INT,
  destination_cities JSON,
  included_services JSON,
  base_cost DECIMAL(12,2),
  discount_percent DECIMAL(5,2),
  final_cost DECIMAL(12,2),
  start_date DATE,
  end_date DATE,
  max_passengers INT,
  booked_passengers INT,
  status ENUM('draft', 'published', 'cancelled'),
  created_at TIMESTAMP
);

-- PAYMENTS TABLE
CREATE TABLE payments (
  id VARCHAR(36) PRIMARY KEY,
  booking_id VARCHAR(36) REFERENCES bookings(id),
  amount DECIMAL(10,2),
  payment_method ENUM('credit_card', 'debit_card', 'net_banking', 'upi', 'wallet'),
  transaction_id VARCHAR(50),
  status ENUM('pending', 'success', 'failed'),
  gateway_response JSON,
  created_at TIMESTAMP
);

-- INVOICES TABLE
CREATE TABLE invoices (
  id VARCHAR(36) PRIMARY KEY,
  invoice_number VARCHAR(20) UNIQUE,
  booking_id VARCHAR(36) REFERENCES bookings(id),
  bulk_booking_id VARCHAR(36) REFERENCES bulk_bookings(id),
  issued_by_id VARCHAR(36) REFERENCES users(id),
  issued_to_name VARCHAR(100),
  issued_to_gst VARCHAR(15),
  issue_date DATE,
  due_date DATE,
  subtotal DECIMAL(10,2),
  discount DECIMAL(10,2),
  gst DECIMAL(10,2),
  total DECIMAL(10,2),
  payment_status ENUM('unpaid', 'partial', 'paid'),
  pdf_url VARCHAR(255)
);

-- EVENTS TABLE (Marriage, Corporate, etc.)
CREATE TABLE events (
  id VARCHAR(36) PRIMARY KEY,
  organizer_id VARCHAR(36) REFERENCES users(id),
  event_name VARCHAR(100),
  event_type ENUM('marriage', 'corporate', 'conference', 'festival', 'other'),
  event_date DATE,
  duration_days INT,
  expected_guests INT,
  budget DECIMAL(12,2),
  location VARCHAR(100),
  logistics_requirements TEXT,
  bus_requirement INT,
  hotel_requirement INT,
  status ENUM('draft', 'published', 'confirmed', 'completed')
);
```

---

## 4. API ENDPOINTS

### Authentication
- `POST /api/auth/register` - User registration
- `POST /api/auth/login` - Login with OTP
- `POST /api/auth/verify-otp` - Verify phone OTP
- `POST /api/auth/refresh-token` - Refresh JWT

### Routes & Schedules
- `GET /api/routes/search?from=&to=&date=` - Search routes
- `GET /api/schedules/{scheduleId}` - Get schedule details
- `GET /api/seats/{scheduleId}` - Get seat availability
- `POST /api/routes` - Create route (operator only)

### Bookings
- `POST /api/bookings` - Create booking
- `GET /api/bookings/{bookingId}` - Get booking details
- `GET /api/bookings/user/{userId}` - Get user bookings
- `PUT /api/bookings/{bookingId}/cancel` - Cancel booking
- `POST /api/bookings/bulk` - Bulk booking (agent)
- `GET /api/bookings/confirmation/{bookingRef}` - Get confirmation

### Discounts
- `GET /api/discounts/{scheduleId}` - Get applicable discounts
- `POST /api/discounts` - Create discount (operator)
- `PUT /api/discounts/{discountId}` - Update discount
- `GET /api/discounts/bulk/tiers` - Get bulk discount tiers

### Hotels
- `GET /api/hotels/city/{cityName}` - Get hotels by city
- `GET /api/hotels/{hotelId}/rates` - Get rates & availability
- `POST /api/hotels/book` - Book hotel room
- `GET /api/hotel-partnerships` - Get operator partnerships

### Tours & Events
- `POST /api/tours/packages` - Create tour package
- `GET /api/tours/packages` - List packages
- `POST /api/events` - Create event
- `POST /api/events/{eventId}/logistics` - Plan logistics

### Payments
- `POST /api/payments/initiate` - Initiate payment
- `POST /api/payments/verify` - Verify payment status
- `GET /api/payments/history/{userId}` - Payment history

### Invoices
- `GET /api/invoices/{invoiceId}` - Get invoice
- `POST /api/invoices/generate` - Generate invoice
- `GET /api/invoices/user/{userId}` - User invoices

### Analytics (Operator/Agent)
- `GET /api/analytics/revenue/{period}` - Revenue data
- `GET /api/analytics/occupancy` - Occupancy rates
- `GET /api/analytics/bookings/summary` - Booking summary
- `GET /api/analytics/commission` - Commission tracking

---

## 5. DISCOUNT STRUCTURE

### Individual Tier Discounts
| Passengers | Discount |
|------------|----------|
| 1-5        | 0%       |
| 6-9        | 10%      |
| 10-24      | 15%      |
| 25-39      | 25%      |
| 40+        | 35%      |

### Dynamic Pricing Strategy
- Base fare varies by route, bus type, and demand
- Peak season (weekends, holidays): +10-20%
- Off-peak (weekdays): -5-10%
- Early bird (7+ days): Additional 5% discount
- Last-minute (24 hours): 0-10% variable

### GST Structure
- 5% on bus tickets for same-state travel
- 5% on hotel accommodations
- 18% on event management services
- Combined invoicing for multi-service packages

---

## 6. TECHNOLOGY STACK

### Frontend
- **Web:** React.js, Redux, TypeScript, Tailwind CSS
- **Mobile:** React Native (iOS/Android) or Flutter
- **Maps:** Google Maps API for route visualization
- **Payments:** Razorpay, PhonePe SDKs
- **PWA:** Service workers for offline booking confirmation

### Backend
- **Runtime:** Node.js with Express.js
- **Database:** MySQL 8.0 with Sequelize ORM
- **Cache:** Redis for seat availability, discount tiers
- **Message Queue:** RabbitMQ for booking confirmations
- **PDF:** PDFKit for e-ticket & invoice generation

### Infrastructure
- **Hosting:** AWS EC2, RDS, S3
- **CDN:** CloudFront for static assets
- **Logging:** ELK Stack (Elasticsearch, Logstash, Kibana)
- **Monitoring:** New Relic / Datadog
- **CI/CD:** GitHub Actions / Jenkins

### Third-party Integrations
- **SMS:** Twilio for OTP & confirmations
- **Email:** SendGrid for invoices
- **Maps:** Google Maps Platform
- **Payment Gateways:** Razorpay, PhonePe, Stripe
- **Analytics:** Google Analytics, Mixpanel

---

## 7. IMPLEMENTATION ROADMAP

### Phase 1: MVP (Weeks 1-8)
- User authentication (OTP-based)
- Route search & booking engine
- Seat selection with real-time availability
- Basic discount structure
- Payment integration (Razorpay)
- E-ticket generation
- Email confirmations

**Deliverables:**
- Landing page & customer app
- Booking flow (search → select → pay)
- Basic operator dashboard

---

### Phase 2: Agent & Operator (Weeks 9-14)
- Travel agent bulk booking system
- Commission tracking dashboard
- Operator route & discount management
- Dynamic pricing algorithm
- Revenue analytics

**Deliverables:**
- Travel agent portal
- Operator management dashboard
- Reporting & analytics

---

### Phase 3: Hotel & Tour Integration (Weeks 15-20)
- Hotel partnership module
- Tour package builder
- Multi-day package pricing
- Event management system
- Combined billing

**Deliverables:**
- Hotel inventory system
- Tour package marketplace
- Event logistics planner

---

### Phase 4: Mobile Apps & Optimization (Weeks 21-24)
- iOS app development
- Android app development
- Offline booking capability
- Push notifications
- Mobile payment optimizations

**Deliverables:**
- Native iOS app
- Native Android app
- Performance optimization

---

### Phase 5: Advanced Features (Weeks 25+)
- AI-based dynamic pricing
- Predictive analytics for demand
- Loyalty program integration
- Corporate account management
- Vehicle tracking (GPS integration)
- Driver app

---

## 8. SECURITY CONSIDERATIONS

### Authentication & Authorization
- JWT-based token management
- OTP verification for transactions
- Role-based access control (RBAC)
- Session timeout after 30 minutes

### Data Protection
- AES-256 encryption for sensitive data
- PCI-DSS compliance for payment data
- GDPR compliance for user data
- Regular security audits

### Payment Security
- PCI-DSS Level 1 compliance
- 3D Secure authentication
- Encrypted API communication (HTTPS/TLS 1.3)
- Rate limiting on payment endpoints

### Fraud Prevention
- Duplicate booking detection
- Unusual activity monitoring
- Chargeback protection
- Identity verification for agents

---

## 9. REVENUE MODEL

### Commission Structure
- **Individual bookings:** 5-8% commission from operator
- **Agent commissions:** 3-5% on bulk bookings
- **Hotel partnerships:** 10-15% commission per booking
- **Tour packages:** 10-12% platform fee
- **Event management:** 8-10% on total package value

### Pricing Strategy
- Operators set base fares
- Platform suggests dynamic pricing
- Peak hour surcharges captured by operator
- Platform takes percentage-based fees

---

## 10. COMPLIANCE & REGULATIONS

### India-Specific Requirements
- NHAI Motor Vehicle Rules 2017 for road safety
- GST registration and compliance
- DGFT approvals for tour operators
- Hotel star ratings (Ministry of Tourism)
- Consumer Protection Act 2019
- RTO registration for buses

### Data Privacy
- Compliance with DISHA (Digital Information Security in Higher Education)
- Privacy policy & terms of service
- User consent for SMS/email marketing
- Right to deletion compliance

---

## 11. SCALABILITY METRICS

### Expected Load
- **Day 1:** 100 concurrent users
- **Month 6:** 10,000 concurrent users
- **Year 1:** 50,000 concurrent users

### Performance Targets
- Page load time: < 2 seconds
- Search results: < 500ms
- Booking confirmation: < 1 second
- 99.9% uptime SLA

### Database Optimization
- Connection pooling (50-200 connections)
- Query caching for popular routes
- Seat availability caching (Redis)
- Archival of old bookings quarterly

---

## 12. TESTING STRATEGY

### Unit Tests
- Coverage: > 80%
- Framework: Jest/Mocha
- Scope: All business logic functions

### Integration Tests
- API endpoint testing
- Database transaction testing
- Payment gateway testing (sandbox)
- Email/SMS delivery verification

### End-to-End Tests
- Complete booking flow
- Multi-role scenarios
- Payment flow with refunds
- Invoice generation

### Load Testing
- Apache JMeter for stress testing
- Target: 10,000 concurrent bookings/hour
- Database connection pool limits

---

## 13. CUSTOMER SUPPORT

### Support Channels
- In-app chat support
- WhatsApp support
- Email: support@bushubpro.com
- Phone: 1800-BUSHUB-1 (toll-free)

### Ticket Cancellation Policy
- Free cancellation up to 24 hours before departure
- 50% refund between 12-24 hours
- No refund within 12 hours
- Full refund for cancelled trips

### Refund Processing
- Instant for wallets (< 1 minute)
- 3-5 working days for cards
- Agent settlements: Weekly

---

## 14. DEPLOYMENT INSTRUCTIONS

### Database Setup
```bash
mysql -u root -p < database_schema.sql
npm run db:migrate
npm run db:seed:initial
```

### Environment Configuration
```bash
cp .env.example .env
# Edit .env with:
# DATABASE_URL
# REDIS_URL
# RAZORPAY_KEY_ID
# RAZORPAY_KEY_SECRET
# JWT_SECRET
# AWS_ACCESS_KEY_ID
# AWS_SECRET_ACCESS_KEY
```

### Installation & Launch
```bash
npm install
npm run build
npm start
```

### Docker Deployment
```bash
docker build -t bushub-pro:latest .
docker run -d -p 3000:3000 --env-file .env bushub-pro:latest
```

---

## 15. KEY METRICS & KPIs

### Business Metrics
- Monthly active users (MAU)
- Average revenue per user (ARPU)
- Booking conversion rate
- Customer lifetime value (CLV)
- Churn rate

### Operational Metrics
- Average bus utilization: Target 75%
- On-time performance: Target 98%
- Customer satisfaction score: Target 4.5/5
- Support ticket resolution time: < 24 hours

---

## 16. CONTACT & SUPPORT

**For Technical Issues:**
- Tech Support: tech@bushubpro.com
- API Documentation: api-docs.bushubpro.com
- Status Page: status.bushubpro.com

**For Business Inquiries:**
- Partnership: partners@bushubpro.com
- Enterprise Sales: enterprise@bushubpro.com
- Investor Relations: investors@bushubpro.com

---

**Document Version:** 1.0  
**Last Updated:** July 2026  
**Next Review:** August 2026
