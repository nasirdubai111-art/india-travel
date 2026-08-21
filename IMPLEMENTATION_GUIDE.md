# BusHub Pro - Implementation Guide & Code Snippets

---

## QUICK START

### Prerequisites
- Node.js 18+ LTS
- MySQL 8.0
- Redis 6.0+
- Docker (optional)
- npm or yarn

### Installation (Local Development)

```bash
# 1. Clone repository
git clone https://github.com/bushubpro/platform.git
cd bushubpro-platform

# 2. Install backend dependencies
cd backend
npm install

# 3. Install frontend dependencies
cd ../frontend
npm install

# 4. Create environment files
cd ..
cp .env.example .env
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

---

## ENVIRONMENT CONFIGURATION

### `.env` (Root)
```env
# Database
DATABASE_URL=mysql://root:password@localhost:3306/bushubpro
DATABASE_POOL_MIN=5
DATABASE_POOL_MAX=20

# Redis
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=your_super_secret_key_min_32_chars_long
JWT_EXPIRY=24h
JWT_REFRESH_EXPIRY=7d

# Mail Service
SENDGRID_API_KEY=sg_xxx_xxxx_xxxx
SENDGRID_FROM_EMAIL=noreply@bushubpro.com

# SMS Service
TWILIO_ACCOUNT_SID=AC_xxx_xxxx
TWILIO_AUTH_TOKEN=xxx_xxxx
TWILIO_PHONE_NUMBER=+1xxxxxxxxxx

# Payment Gateways
RAZORPAY_KEY_ID=rzp_xxx_xxxxx
RAZORPAY_KEY_SECRET=xxx_xxxx_xxxx_xxxx
STRIPE_SECRET_KEY=sk_live_xxxx_xxxx

# AWS S3
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_S3_BUCKET=bushubpro-documents
AWS_REGION=ap-south-1

# Google Maps
GOOGLE_MAPS_API_KEY=AIzaSyxxx_xxx_xxxx

# Environment
NODE_ENV=development
PORT=3000
API_URL=http://localhost:3000
FRONTEND_URL=http://localhost:3001
LOG_LEVEL=debug
```

### `backend/.env`
```env
# Express Server
EXPRESS_PORT=3000
CORS_ORIGIN=http://localhost:3001,https://bushubpro.com

# Database ORM
ORM_DIALECT=mysql
ORM_LOGGING=false

# Notifications
NOTIFICATION_QUEUE_NAME=bushub-notifications
ENABLE_SMS=true
ENABLE_EMAIL=true
ENABLE_PUSH=true

# Security
BCRYPT_ROUNDS=10
MAX_LOGIN_ATTEMPTS=5
LOCK_TIME=15m

# API Rate Limiting
RATE_LIMIT_WINDOW=15m
RATE_LIMIT_MAX_REQUESTS=100

# Feature Flags
FEATURE_BULK_BOOKING=true
FEATURE_HOTEL_INTEGRATION=true
FEATURE_TOUR_PACKAGES=true
FEATURE_DYNAMIC_PRICING=true
```

### `frontend/.env`
```env
# API Configuration
REACT_APP_API_URL=http://localhost:3000
REACT_APP_API_TIMEOUT=10000

# Google Maps
REACT_APP_GOOGLE_MAPS_KEY=AIzaSyxxx_xxx_xxxx

# Analytics
REACT_APP_GA_ID=G-XXXXXXXXXX
REACT_APP_MIXPANEL_TOKEN=xxx_xxxx

# Payment
REACT_APP_RAZORPAY_KEY=rzp_live_xxxx
REACT_APP_STRIPE_PUBLISHABLE_KEY=pk_live_xxxx

# Environment
REACT_APP_ENV=development
REACT_APP_LOG_LEVEL=debug
```

---

## DATABASE SETUP

### Initialize Database

```bash
# 1. Create database
mysql -u root -p -e "CREATE DATABASE bushubpro CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# 2. Run migrations
npm run db:migrate

# 3. Seed initial data
npm run db:seed:initial

# 4. Verify connection
npm run db:verify
```

### Database Backup & Restore

```bash
# Backup
mysqldump -u root -p bushubpro > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore
mysql -u root -p bushubpro < backup_20240722_143022.sql

# Automated backups (daily at 2 AM)
# Add to crontab: 0 2 * * * /path/to/backup_script.sh
```

---

## KEY CODE SNIPPETS

### 1. Authentication Service

```javascript
// backend/src/services/auth.service.js

class AuthService {
  
  // Send OTP
  async sendOTP(phone) {
    const otp = generateOTP(6);
    const expiresAt = new Date(Date.now() + 10 * 60000); // 10 mins
    
    await OTPModel.create({
      phone,
      otp,
      expiresAt,
      attempts: 0
    });
    
    // Send via Twilio
    await twilioClient.messages.create({
      body: `Your BusHub OTP is ${otp}. Valid for 10 minutes.`,
      from: process.env.TWILIO_PHONE_NUMBER,
      to: `+91${phone}`
    });
    
    return { success: true, message: 'OTP sent' };
  }

  // Verify OTP and create session
  async verifyOTP(phone, otp) {
    const otpRecord = await OTPModel.findOne({
      phone,
      otp,
      expiresAt: { $gt: new Date() }
    });

    if (!otpRecord) throw new Error('Invalid or expired OTP');
    if (otpRecord.attempts >= 3) throw new Error('Max attempts exceeded');

    // Check if user exists
    let user = await UserModel.findOne({ phone });
    
    if (!user) {
      user = await UserModel.create({
        phone,
        role: 'customer'
      });
    }

    // Generate JWT
    const token = jwt.sign(
      { userId: user.id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRY }
    );

    // Mark OTP as used
    await OTPModel.deleteOne({ _id: otpRecord._id });

    return {
      user: {
        id: user.id,
        phone: user.phone,
        role: user.role
      },
      token,
      refreshToken: generateRefreshToken(user.id)
    };
  }

  // Role-based access control
  async validateRole(userId, requiredRoles) {
    const user = await UserModel.findById(userId);
    return requiredRoles.includes(user.role);
  }
}

module.exports = new AuthService();
```

### 2. Booking Service

```javascript
// backend/src/services/booking.service.js

class BookingService {

  // Create booking
  async createBooking(bookingData) {
    const { scheduleId, seatNumbers, passengersData, customerId } = bookingData;
    
    // Validate schedule exists
    const schedule = await ScheduleModel.findById(scheduleId);
    if (!schedule) throw new Error('Invalid schedule');

    // Get seat details
    const seats = await SeatModel.find({
      scheduleId,
      seatNumber: { $in: seatNumbers }
    });

    if (seats.length !== seatNumbers.length) {
      throw new Error('Some seats are not available');
    }

    // Check seat status
    const bookedSeats = seats.filter(s => s.status !== 'available');
    if (bookedSeats.length > 0) {
      throw new Error(`Seats ${bookedSeats.map(s => s.seatNumber).join(', ')} are already booked`);
    }

    // Calculate pricing
    const route = await RouteModel.findById(schedule.routeId);
    const discount = await this.calculateDiscount(bookingData);
    
    const baseAmount = route.baseFare * seatNumbers.length;
    const discountAmount = baseAmount * (discount.percent / 100);
    const subtotal = baseAmount - discountAmount;
    const gst = subtotal * 0.05; // 5% GST for bus tickets
    const finalAmount = subtotal + gst;

    // Create booking record
    const booking = await BookingModel.create({
      bookingRef: generateBookingRef(),
      customerId,
      scheduleId,
      seatIds: seats.map(s => s.id),
      passengerDetails: passengersData,
      baseAmount,
      discountAmount,
      totalAmount: subtotal,
      gst,
      finalAmount,
      status: 'pending',
      paymentStatus: 'unpaid'
    });

    // Lock seats
    await SeatModel.updateMany(
      { _id: { $in: seats.map(s => s.id) } },
      { 
        status: 'reserved',
        bookedBy: customerId,
        reservedUntil: new Date(Date.now() + 10 * 60000) // 10 min hold
      }
    );

    return booking;
  }

  // Calculate discount based on tier
  async calculateDiscount(bookingData) {
    const { scheduleId, passengersCount } = bookingData;
    const schedule = await ScheduleModel.findById(scheduleId);

    // Check bulk booking discount
    let discount = { percent: 0, type: 'none' };

    if (passengersCount >= 40) {
      discount = { percent: 35, type: 'bulk_premium' };
    } else if (passengersCount >= 25) {
      discount = { percent: 25, type: 'bulk_standard' };
    } else if (passengersCount >= 10) {
      discount = { percent: 15, type: 'bulk_economy' };
    }

    // Apply dynamic discount
    if (schedule.dynamicDiscount > discount.percent) {
      discount = { 
        percent: schedule.dynamicDiscount, 
        type: 'dynamic' 
      };
    }

    return discount;
  }

  // Confirm payment and booking
  async confirmBooking(bookingId, paymentId) {
    const booking = await BookingModel.findById(bookingId);
    const payment = await PaymentModel.findById(paymentId);

    if (payment.status !== 'success') {
      throw new Error('Payment not successful');
    }

    // Confirm booking
    await BookingModel.updateOne(
      { _id: bookingId },
      { status: 'confirmed', paymentStatus: 'paid' }
    );

    // Mark seats as booked
    await SeatModel.updateMany(
      { _id: { $in: booking.seatIds } },
      { status: 'booked', reservedUntil: null }
    );

    // Send confirmation
    await this.sendConfirmationEmail(booking);
    await this.sendConfirmationSMS(booking);

    // Generate e-ticket
    const eticket = await this.generateEticket(booking);

    return {
      booking,
      eticket,
      message: 'Booking confirmed successfully'
    };
  }

  // Send confirmation email
  async sendConfirmationEmail(booking) {
    const passenger = booking.passengerDetails[0];
    
    const emailContent = `
      <h1>Booking Confirmed</h1>
      <p>Booking Reference: ${booking.bookingRef}</p>
      <p>Dear ${passenger.name},</p>
      <p>Your bus booking has been confirmed.</p>
      <p>Travel Date: ${booking.travelDate}</p>
      <p>Seats: ${booking.seats.join(', ')}</p>
      <p>Amount: ₹${booking.finalAmount}</p>
      <p><a href="${process.env.FRONTEND_URL}/bookings/${booking.id}">View Details</a></p>
    `;

    await sgMail.send({
      to: passenger.email,
      from: process.env.SENDGRID_FROM_EMAIL,
      subject: `Booking Confirmed - ${booking.bookingRef}`,
      html: emailContent
    });
  }

  // Send confirmation SMS
  async sendConfirmationSMS(booking) {
    const passenger = booking.passengerDetails[0];
    
    await twilioClient.messages.create({
      body: `BusHub: Your booking ${booking.bookingRef} is confirmed. Seats: ${booking.seats.join(', ')}. Download e-ticket from app.`,
      from: process.env.TWILIO_PHONE_NUMBER,
      to: `+91${passenger.phone}`
    });
  }

  // Generate e-ticket PDF
  async generateEticket(booking) {
    const doc = new PDFDocument();
    
    // Add QR code with booking ref
    const qrCode = await QRCode.toBuffer(booking.bookingRef);
    
    doc.fontSize(24).text('Bus E-Ticket', { align: 'center' });
    doc.image(qrCode, 250, 100, { width: 100 });
    doc.fontSize(12).text(`Reference: ${booking.bookingRef}`);
    
    // Add booking details
    doc.text(`Route: ${booking.route.from} → ${booking.route.to}`);
    doc.text(`Date: ${booking.travelDate}`);
    doc.text(`Seats: ${booking.seats.join(', ')}`);
    doc.text(`Amount: ₹${booking.finalAmount}`);

    // Upload to S3
    const pdfBuffer = doc;
    const s3Params = {
      Bucket: process.env.AWS_S3_BUCKET,
      Key: `etickets/${booking.bookingRef}.pdf`,
      Body: pdfBuffer,
      ContentType: 'application/pdf'
    };

    const result = await s3.upload(s3Params).promise();
    
    return {
      url: result.Location,
      bookingRef: booking.bookingRef
    };
  }

  // Cancel booking
  async cancelBooking(bookingId, reason) {
    const booking = await BookingModel.findById(bookingId);
    
    if (booking.status === 'cancelled') {
      throw new Error('Booking already cancelled');
    }

    const cancelFee = booking.finalAmount * 0.1; // 10% cancellation fee
    const refundAmount = booking.finalAmount - cancelFee;

    // Update booking
    await BookingModel.updateOne(
      { _id: bookingId },
      { 
        status: 'cancelled',
        cancelledAt: new Date(),
        cancellationReason: reason,
        refundAmount
      }
    );

    // Free up seats
    await SeatModel.updateMany(
      { _id: { $in: booking.seatIds } },
      { status: 'available', bookedBy: null }
    );

    // Process refund
    if (booking.paymentStatus === 'paid') {
      await this.processRefund(booking, refundAmount);
    }

    return {
      message: 'Booking cancelled',
      refundAmount,
      estimatedRefundTime: '3-5 working days'
    };
  }

  // Process refund
  async processRefund(booking, amount) {
    const payment = await PaymentModel.findOne({ bookingId: booking.id });
    
    if (payment.gateway === 'razorpay') {
      const refund = await razorpay.payments.refund(payment.transactionId, {
        amount: amount * 100 // Convert to paise
      });
      
      await PaymentModel.updateOne(
        { _id: payment.id },
        { refundStatus: 'processed', refundId: refund.id }
      );
    }
  }
}

module.exports = new BookingService();
```

### 3. Discount Engine

```javascript
// backend/src/services/discount.service.js

class DiscountService {

  // Calculate bulk discount
  calculateBulkDiscount(passengerCount) {
    const tiers = [
      { min: 40, max: Infinity, percent: 35 },
      { min: 25, max: 39, percent: 25 },
      { min: 10, max: 24, percent: 15 },
      { min: 6, max: 9, percent: 10 },
      { min: 1, max: 5, percent: 0 }
    ];

    const tier = tiers.find(t => passengerCount >= t.min && passengerCount <= t.max);
    return tier ? tier.percent : 0;
  }

  // Dynamic pricing based on occupancy
  calculateDynamicDiscount(occupancyPercent, hoursUntilDeparture) {
    let discount = 0;

    // High occupancy = less discount
    if (occupancyPercent >= 80) {
      discount = 0;
    } else if (occupancyPercent >= 60) {
      discount = 5;
    } else if (occupancyPercent >= 40) {
      discount = 10;
    } else {
      discount = 15;
    }

    // Early bird discount
    if (hoursUntilDeparture >= 168) { // 7 days
      discount += 5;
    } else if (hoursUntilDeparture >= 72) { // 3 days
      discount += 3;
    }

    return Math.min(discount, 25); // Cap at 25%
  }

  // Generate discount invoice for travel agent
  async generateAgentInvoice(bulkBookingId) {
    const bulkBooking = await BulkBookingModel.findById(bulkBookingId);
    
    const invoiceData = {
      invoiceNumber: `AGT-${Date.now()}`,
      agentId: bulkBooking.agentId,
      bookingRef: bulkBooking.bulkRef,
      totalPassengers: bulkBooking.totalPassengers,
      subtotal: bulkBooking.subtotal,
      discount: bulkBooking.discountAmount,
      gst: bulkBooking.gst,
      total: bulkBooking.totalAmount,
      commissionPercent: 5,
      commissionAmount: bulkBooking.totalAmount * 0.05,
      issueDate: new Date(),
      dueDate: new Date(Date.now() + 30 * 24 * 60 * 60000)
    };

    return invoiceData;
  }

  // Apply seasonal discount
  applySeasonalDiscount(travelDate, discount) {
    const month = new Date(travelDate).getMonth();
    
    // Peak seasons (May, June, Dec, Jan) - no additional discount
    if ([4, 5, 11, 0].includes(month)) {
      return discount;
    }
    
    // Off-season - add 5% more discount
    return Math.min(discount + 5, 25);
  }
}

module.exports = new DiscountService();
```

### 4. Payment Processing

```javascript
// backend/src/services/payment.service.js

class PaymentService {

  // Initiate Razorpay payment
  async initiateRazorpayPayment(bookingId) {
    const booking = await BookingModel.findById(bookingId);
    
    const options = {
      amount: Math.round(booking.finalAmount * 100), // Convert to paise
      currency: 'INR',
      receipt: booking.bookingRef,
      payment_capture: 1, // Auto-capture
      notes: {
        bookingId: booking.id,
        customerId: booking.customerId,
        route: `${booking.route.from} → ${booking.route.to}`
      }
    };

    const order = await razorpay.orders.create(options);

    // Save order details
    await PaymentModel.create({
      bookingId,
      orderId: order.id,
      amount: booking.finalAmount,
      currency: 'INR',
      gateway: 'razorpay',
      status: 'pending'
    });

    return {
      orderId: order.id,
      amount: order.amount,
      currency: order.currency,
      key: process.env.RAZORPAY_KEY_ID
    };
  }

  // Verify payment signature
  async verifyRazorpayPayment(paymentDetails) {
    const { orderId, paymentId, signature } = paymentDetails;

    // Verify signature
    const hmac = crypto.createHmac('sha256', process.env.RAZORPAY_KEY_SECRET);
    hmac.update(`${orderId}|${paymentId}`);
    const generatedSignature = hmac.digest('hex');

    if (generatedSignature !== signature) {
      throw new Error('Invalid payment signature');
    }

    // Fetch payment from Razorpay
    const payment = await razorpay.payments.fetch(paymentId);

    // Update payment record
    const paymentRecord = await PaymentModel.findOne({
      orderId,
      gateway: 'razorpay'
    });

    await PaymentModel.updateOne(
      { _id: paymentRecord.id },
      {
        status: 'success',
        transactionId: paymentId,
        gatewayResponse: payment,
        confirmedAt: new Date()
      }
    );

    // Confirm booking
    const booking = await BookingModel.findById(paymentRecord.bookingId);
    await BookingService.confirmBooking(booking.id, paymentRecord.id);

    return {
      success: true,
      bookingRef: booking.bookingRef,
      amount: payment.amount / 100 // Convert back to rupees
    };
  }

  // Generate invoice
  async generateInvoice(bookingId) {
    const booking = await BookingModel.findById(bookingId);
    const doc = new PDFDocument();

    // Invoice header
    doc.fontSize(18).text('INVOICE', { align: 'center' });
    doc.fontSize(10).text(`Invoice Date: ${new Date().toLocaleDateString()}`);
    doc.text(`Invoice Number: INV-${booking.bookingRef}`);
    doc.text(`GST Number: GSTIN XXXXXXXXXXXXX`);

    // Bill to
    doc.moveDown().fontSize(11).text('Bill To:');
    const passenger = booking.passengerDetails[0];
    doc.fontSize(10).text(passenger.name);
    doc.text(passenger.email);
    doc.text(`Phone: +91${passenger.phone}`);

    // Line items
    doc.moveDown().table([
      ['Description', 'Qty', 'Rate', 'Amount'],
      [
        `Bus Ticket: ${booking.route.from} → ${booking.route.to}`,
        booking.seats.length,
        `₹${booking.baseAmount / booking.seats.length}`,
        `₹${booking.baseAmount}`
      ],
      [
        'Discount',
        '',
        `${(booking.discountAmount / booking.baseAmount * 100).toFixed(1)}%`,
        `-₹${booking.discountAmount}`
      ],
      [
        'Subtotal',
        '',
        '',
        `₹${booking.totalAmount}`
      ],
      [
        'GST (5%)',
        '',
        '',
        `₹${booking.gst}`
      ],
      [
        'TOTAL',
        '',
        '',
        `₹${booking.finalAmount}`
      ]
    ]);

    // Terms
    doc.moveDown().fontSize(9).text('Terms: Valid for immediate travel. Non-transferable.');

    // Upload to S3
    const fileName = `invoices/INV-${booking.bookingRef}.pdf`;
    const s3Params = {
      Bucket: process.env.AWS_S3_BUCKET,
      Key: fileName,
      Body: doc,
      ContentType: 'application/pdf'
    };

    const result = await s3.upload(s3Params).promise();

    // Save invoice URL
    await BookingModel.updateOne(
      { _id: bookingId },
      { invoiceUrl: result.Location }
    );

    return result.Location;
  }
}

module.exports = new PaymentService();
```

---

## API USAGE EXAMPLES

### Search Routes

```bash
# Request
curl -X GET "http://localhost:3000/api/routes/search?from=BNG&to=HYD&date=2024-07-24" \
  -H "Content-Type: application/json"

# Response
{
  "success": true,
  "data": [
    {
      "id": "route_123",
      "operator": "SafariGold",
      "from": "Bengaluru",
      "to": "Hyderabad",
      "departure": "08:00 AM",
      "arrival": "02:00 PM",
      "baseFare": 1200,
      "availableSeats": 45,
      "discount": 15,
      "finalPrice": 1020,
      "ratings": 4.8
    }
  ]
}
```

### Create Booking

```bash
# Request
curl -X POST "http://localhost:3000/api/bookings" \
  -H "Authorization: Bearer JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "scheduleId": "sch_456",
    "seatNumbers": ["9", "16"],
    "passengersData": [
      {
        "name": "John Doe",
        "email": "john@email.com",
        "phone": "9876543210",
        "idType": "Aadhar",
        "idNumber": "1234567890123456"
      }
    ]
  }'

# Response
{
  "success": true,
  "booking": {
    "id": "book_789",
    "bookingRef": "BHP-2607-8947",
    "status": "pending",
    "totalAmount": 1320,
    "gst": 66,
    "finalAmount": 1386,
    "paymentUrl": "https://razorpay.com/..."
  }
}
```

### Verify Payment

```bash
# Request
curl -X POST "http://localhost:3000/api/payments/verify" \
  -H "Content-Type: application/json" \
  -d '{
    "orderId": "order_1234567890",
    "paymentId": "pay_1234567890",
    "signature": "signature_hash"
  }'

# Response
{
  "success": true,
  "bookingRef": "BHP-2607-8947",
  "message": "Payment verified successfully"
}
```

---

## DEPLOYMENT

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass123
      MYSQL_DATABASE: bushubpro
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  backend:
    build: ./backend
    depends_on:
      - mysql
      - redis
    environment:
      DATABASE_URL: mysql://root:rootpass123@mysql:3306/bushubpro
      REDIS_URL: redis://redis:6379
    ports:
      - "3000:3000"
    volumes:
      - ./backend:/app

  frontend:
    build: ./frontend
    depends_on:
      - backend
    environment:
      REACT_APP_API_URL: http://localhost:3000
    ports:
      - "3001:3001"
    volumes:
      - ./frontend:/app

volumes:
  mysql-data:
  redis-data:
```

### Deploy to Production

```bash
# Build Docker images
docker build -t bushubpro/backend:latest ./backend
docker build -t bushubpro/frontend:latest ./frontend

# Push to registry
docker push bushubpro/backend:latest
docker push bushubpro/frontend:latest

# Deploy to AWS ECS
aws ecs update-service \
  --cluster bushub-prod \
  --service bushubpro-api \
  --force-new-deployment
```

---

## MONITORING & LOGGING

### Health Check Endpoint

```javascript
// GET /api/health
{
  "status": "healthy",
  "uptime": 3600,
  "database": "connected",
  "redis": "connected",
  "timestamp": "2024-07-24T10:30:00Z"
}
```

### Error Tracking (Sentry)

```javascript
// Initialize Sentry
const Sentry = require("@sentry/node");

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 1.0,
  environment: process.env.NODE_ENV
});

app.use(Sentry.Handlers.errorHandler());
```

### Request Logging

```javascript
// Winston Logger
const logger = require('winston');

logger.info('Booking created', {
  bookingRef: 'BHP-2607-8947',
  customerId: 'cust_123',
  amount: 1386,
  timestamp: new Date()
});
```

---

## PERFORMANCE OPTIMIZATION CHECKLIST

- [ ] Enable database indexing on frequently queried fields
- [ ] Implement Redis caching for seat availability
- [ ] Use CDN for static assets (JS, CSS, images)
- [ ] Enable GZIP compression on API responses
- [ ] Implement pagination for list endpoints
- [ ] Use connection pooling for database
- [ ] Optimize database queries with EXPLAIN
- [ ] Enable browser caching headers
- [ ] Minify and bundle frontend code
- [ ] Use lazy loading for images
- [ ] Implement request debouncing on frontend
- [ ] Monitor API response times

---

## TESTING CHECKLIST

- [ ] Unit tests for business logic (80%+ coverage)
- [ ] Integration tests for API endpoints
- [ ] E2E tests for critical user flows
- [ ] Load testing with 10K concurrent users
- [ ] Security testing (OWASP Top 10)
- [ ] Payment gateway testing in sandbox
- [ ] SMS/Email delivery testing
- [ ] Mobile app testing (iOS & Android)
- [ ] Cross-browser testing
- [ ] Accessibility testing (WCAG 2.1)

---

## TROUBLESHOOTING

### Database Connection Issues
```bash
# Test connection
mysql -h localhost -u root -p bushubpro

# Check pool status
SELECT COUNT(*) FROM information_schema.processlist;
```

### Redis Cache Issues
```bash
# Connect to Redis CLI
redis-cli

# Flush cache
FLUSHALL

# Monitor commands
MONITOR
```

### Payment Gateway Issues
```bash
# Test Razorpay API
curl -X GET "https://api.razorpay.com/v1/payments/{PAYMENT_ID}" \
  -u "KEY_ID:KEY_SECRET"
```

---

## SUPPORT & DOCUMENTATION

- **API Docs:** https://api-docs.bushubpro.com
- **GitHub:** https://github.com/bushubpro/platform
- **Issues:** https://github.com/bushubpro/platform/issues
- **Wiki:** https://github.com/bushubpro/platform/wiki
- **Discord:** https://discord.gg/bushubpro

---

**Implementation Guide Version:** 1.0  
**Last Updated:** July 2026  
**Next Review:** August 2026
