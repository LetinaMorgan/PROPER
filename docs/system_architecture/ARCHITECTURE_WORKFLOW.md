# System Architecture & Core Workflows

## 1. High-Level Architecture Overview
The system is built as a lightweight, API-first web application. Since the primary users (the service experts) and clients operate on mobile devices, the entire architecture is built **mobile-first** with highly optimized responsive web clients.

```
       [ Client App (Responsive) ]       [ Pro Dashboard (Responsive Mobile/Web) ]
                   │                                         │
                   ▼                                         ▼
         ┌─────────────────────────────────────────────────────────────┐
         │                    DNS / API Gateway / CDN                  │
         └──────────────────────────────┬──────────────────────────────┘
                                        │
                                        ▼
                   ┌─────────────────────────────────────────┐
                   │        Backend API Server (Node/Go)     │
                   └───────┬────────────┬────────────┬───────┘
                           │            │            │
                           ▼            ▼            ▼
         ┌───────────────────┐    ┌───────────┐    ┌───────────────────┐
         │ Database (Postgr) │    │ Redis/Job │    │  3rd Party APIs   │
         │ - User Profiles   │    │  Queue    │    │ - Stripe Connect  │
         │ - Bookings/Slots  │    │ - SMS Qs  │    │ - Twilio (SMS)    │
         │ - Service Menus   │    │ - Buffers │    │ - Google Maps     │
         └───────────────────┘    └───────────┘    └───────────────────┘
```

---

## 2. Technical Stack Recommendation
* **Frontend Clients:** React (TypeScript) or Next.js with Tailwind CSS or Vanilla CSS, packaged as a Progressive Web App (PWA) so experts can install it directly on their home screens without App Store overhead.
* **Backend Engine:** Node.js (Express/Fastify) or Python (FastAPI) for swift development and straightforward async queue handling.
* **Database:** PostgreSQL (highly reliable for transactional integrity, scheduling, and complex queries). PostGIS extension can be easily enabled later for geographic radius checks.
* **Caching & Queues:** Redis for managing scheduled SMS reminder jobs and rapid session lookups.

---

## 3. Core Operational Workflows

### A. Provider Onboarding & Setup Workflow
Before a provider can accept bookings, they must configure their storefront.

```
[ Step 1: Sign Up ] ➔ [ Step 2: Service Menu ] ➔ [ Step 3: Calendar Setup ] ➔ [ Step 4: Stripe Connect ]
  Register email &      Add services, prices,      Define operating hours        Complete Stripe KYB
  business name.        durations, and images.     and service radius.           to accept card payouts.
```

1. **Identity & Setup:** Pro registers via phone/email and creates their business subdomain (`expertly.app/joes-detailing`).
2. **Menu Creation:** Pro enters service menu items. 
   * *Example:* "Exterior Wax & Wash | $120 | 90 minutes."
3. **Availability & Boundaries:** Pro inputs working hours (e.g., Monday-Friday 9 AM - 5 PM) and service parameters (e.g., "Within 15 miles of zip code 90210").
4. **Payout Onboarding:** Pro completes Stripe Connect standard onboarding. This links their bank account or debit card safely to the platform.

---

### B. Client Booking & Escrow Payment Flow
This flow ensures the client secures a spot while protecting the expert from no-shows.

```
[ Client Visits Link ] ➔ [ Configures Location ] ➔ [ Chooses Slot & Service ]
                                                          │
  [ Client Receives SMS ] ◄─── [ Booking Confirmed ] ◄──── [ Stripe Authorizes Deposit ]
```

1. **Radius Check:** Client lands on the Pro’s booking page and enters their service address. The backend checks if the address falls inside the provider's defined polygon/radius.
2. **Dynamic Slot Calculation:** 
   * System fetches Pro's working hours and merges them with existing bookings.
   * **Routing Engine check:** Automatically injects buffer zones. If Booking A ends at 11:30 AM in North Side, and the client wants to book at 12:00 PM in South Side (25 mins away), the system calculates transit and blocks that slot, dynamically showing only feasible times.
3. **Checkout:** Client selects an available slot and enters billing details.
4. **Escrow Hold / Payment Split:**
   * Stripe charges the client's card for the deposit amount (e.g., 20%).
   * Payment sits safely in Stripe's platform account.
   * A draft booking record is written to the Database with a `Pending` state.
5. **Real-time SMS Alert:** A webhook triggers an SMS to the expert: *"New Booking from Sarah! Tap here to review."*

---

### C. Execution & Dispatch Workflow
The lifecycle of an active appointment.

#### 24 Hours Prior: The SMS Shield
1. A cron job reads tomorrow's pending bookings.
2. The system triggers Twilio to send an SMS to the client: 
   * *"Hi Sarah, Joe's Detailing is scheduled for tomorrow at 10:00 AM at [Address]. Please reply with '1' to confirm or click here to reschedule. Thanks!"*
3. If the client replies `1`, the DB record status updates to `Confirmed`. If they cancel last minute or do not confirm, the provider is alerted so they can re-allocate the slot.

#### Day of Appointment:
1. **Transit Alert:** Pro taps "I'm on my way" in their Pro Dashboard. 
2. **Client Notification:** Twilio sends SMS: *"Joe is on his way to your location! Estimated arrival: 10:15 AM."*
3. **Service Delivery:** Pro completes the work.
4. **Completion & Charge Release:**
   * Pro taps "Mark as Complete" in the app.
   * System initiates the Stripe charge capture for the remaining 80% of the balance.
   * Stripe Connect splits the payout: **95% directly to the Pro's linked account**, **5% to the platform's account** as our booking fee.
   * An receipt is emailed to the client alongside a 1-click review link.

---

## 4. Key Database Schema Concepts

### Table: `users` (Providers and Clients)
* `id` (Primary Key)
* `email` / `phone_number`
* `role` (enum: 'provider', 'client')
* `stripe_connect_id` (null for clients)

### Table: `services`
* `id` (Primary Key)
* `provider_id` (Foreign Key -> `users.id`)
* `title` (e.g., "Standard Wash")
* `price` (Decimal)
* `duration_minutes` (Integer)

### Table: `bookings`
* `id` (Primary Key)
* `client_id` (Foreign Key -> `users.id`)
* `provider_id` (Foreign Key -> `users.id`)
* `service_id` (Foreign Key -> `services.id`)
* `scheduled_start_time` (Timestamp)
* `scheduled_end_time` (Timestamp)
* `status` (enum: 'pending', 'confirmed', 'completed', 'cancelled')
* `payment_intent_id` (Stripe ID)
* `service_address` (Text/Geom)
