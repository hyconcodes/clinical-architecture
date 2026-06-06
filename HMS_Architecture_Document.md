# Hospital Management System (HMS)
## Business Analysis & Software Architecture Document

**Prepared by:** Senior Business Analyst & Software Architect  
**Stack:** Laravel 13 (REST API) · React (Vite) · WebRTC · MySQL · Redis · Laravel Reverb  
**Timeline:** 4 Weeks (Parallel Frontend + Backend Sprints)

---

## SECTION 1 — USER ROLES & ACCESS MATRIX

| Role | Primary Concern | Key Permissions |
|---|---|---|
| **Admin** | System oversight & configuration | Full CRUD on all entities, reports, user management |
| **Doctor** | Patient care & consultations | EMR read/write (own patients), prescriptions, lab orders, video calls |
| **Nurse** | Patient support & vitals | EMR read, vitals write, triage, lab order view |
| **Patient** | Self-service health access | Own records read, booking, payments, video calls |
| **Lab Technician** | Lab orders & results | Lab queue view, results upload |
| **Pharmacist** | Drug dispensing | Prescription queue view, inventory management |

---

## SECTION 2 — END-TO-END USER WORKFLOWS & SYSTEM SEQUENCES

---

### WORKFLOW 1 — Patient Registration & Onboarding

```
PATIENT                     SYSTEM                        ADMIN/BACKEND
   |                           |                               |
   |-- Sign Up (name,          |                               |
   |   email, phone, DOB) ---> |                               |
   |                           |-- Validate uniqueness ------> |
   |                           |-- Send OTP (SMS via Termii) ->|
   |<-- OTP screen ------------|                               |
   |-- Enter OTP ------------> |-- Verify OTP --------------> |
   |                           |-- Create User (role=patient)->|
   |                           |-- Create PatientProfile ------>|
   |                           |-- Send welcome email -------> |
   |<-- Dashboard (Patient) ---|                               |
   |-- Complete profile ------> |                               |
   |   (blood group, allergies,|                               |
   |    emergency contact,     |                               |
   |    HMO/insurance info)    |                               |
   |                           |-- Store PatientProfile ------> |
   |<-- Profile complete -------|                               |
```

---

### WORKFLOW 2 — Appointment Booking with Upfront Payment

> **Business Rule:** Payment is mandatory at the point of booking. Slot is only confirmed after successful payment. Unpaid bookings auto-expire after 15 minutes.

```
PATIENT                     SYSTEM                        PAYMENT GATEWAY
   |                           |                               |
   |-- Search doctor ---------->|                               |
   |   (specialty, location,   |                               |
   |    consultation type:     |                               |
   |    in-person/video/home)  |                               |
   |                           |-- Query available doctors --> |
   |<-- Doctor listings --------|                               |
   |-- Select doctor ---------->|                               |
   |                           |-- Fetch available slots ----> |
   |<-- Calendar with slots ----|   (SlotGenerationAlgorithm)  |
   |-- Pick slot + type ------> |                               |
   |                           |-- LOCK slot (15min TTL,       |
   |                           |   Redis key) ---------------> |
   |                           |-- Calculate fee               |
   |                           |   (consultation type matrix)->|
   |<-- Booking summary +       |                               |
   |    payment form -----------|                               |
   |-- Enter card details ----> |                               |
   |                           |-- Initialize Paystack txn --> |--- POST /transaction/initialize
   |<-- Redirect to Paystack ---|                               |<-- return auth_url
   |-- Complete payment ------> |                               |---> Paystack processes
   |                           |<-- Webhook: charge.success ---|
   |                           |-- Verify webhook signature    |
   |                           |-- CONFIRM appointment ------> |
   |                           |-- RELEASE slot lock           |
   |                           |-- Send confirmation SMS+email>|
   |<-- Booking confirmed ------|                               |
   |   (with video room link   |                               |
   |    if telemedicine)        |                               |
```

**Slot Locking Logic (Redis):**
```
Key:   slot_lock:{doctor_id}:{date}:{time}
Value: {patient_id, expires_at}
TTL:   900 seconds (15 minutes)
On payment success: DELETE key, INSERT appointment (status=confirmed)
On TTL expiry:      Slot automatically becomes available again
```

---

### WORKFLOW 3 — Home Visit Booking

```
PATIENT                     SYSTEM                        DOCTOR/NURSE
   |                           |                               |
   |-- Book home visit -------> |                               |
   |   (address, symptoms,     |                               |
   |    preferred date/time)   |                               |
   |                           |-- Validate service area ----> |
   |                           |-- Calculate home visit fee    |
   |                           |   (base fee + distance levy)->|
   |<-- Fee summary + form -----|                               |
   |-- Pay upfront -----------> |-- Paystack flow (same        |
   |                           |   as Workflow 2) -----------> |
   |<-- Booking confirmed ------|                               |
   |                           |-- Notify assigned doctor ----> |
   |                           |   (push + SMS) --------------> |
   |                           |                               |-- Accept/Decline visit
   |                           |<-- Doctor accepts -------------|
   |<-- Doctor confirmed, ETA --|                               |
   |   and contact shared ------|                               |
```

---

### WORKFLOW 4 — Pre-Consultation Triage (Nurse)

```
NURSE                       SYSTEM                        DOCTOR
   |                           |                               |
   |-- Open appointment queue ->|                               |
   |<-- Today's appointments ---|                               |
   |-- Select patient ---------->|                              |
   |-- Record vitals ----------->|                              |
   |   (BP, temperature,       |                               |
   |    pulse, weight, SpO2,   |                               |
   |    chief complaint)        |                               |
   |                           |-- Save to VitalsRecord -----> |
   |                           |-- Update appt status:         |
   |                           |   Scheduled → Triaged ------> |
   |                           |-- Notify doctor (WebSocket) ->|
   |                           |                               |-- Doctor sees patient
   |                           |                               |   is ready in dashboard
```

---

### WORKFLOW 5 — Video Consultation (WebRTC Signaling Flow)

```
PATIENT (Browser A)       LARAVEL REVERB (WS)         DOCTOR (Browser B)
   |                           |                               |
   |-- GET /rooms/{token} ---> |                               |
   |<-- Room data, TURN config--|                               |
   |-- WS: join_room ---------->|                               |
   |                           |-- WS: join_room ------------->|
   |                           |<-- WS: peer_joined -----------|
   |<-- WS: peer_joined --------|                               |
   |                           |                               |
   |-- Create RTCPeerConn      |                               |
   |-- getUserMedia() -------->| (camera+mic permission)       |
   |-- createOffer() --------> |                               |
   |-- WS: sdp_offer --------->|-- WS: sdp_offer ------------->|
   |                           |                               |-- setRemoteDescription()
   |                           |                               |-- createAnswer()
   |                           |<-- WS: sdp_answer ------------|
   |<-- WS: sdp_answer ---------|                               |
   |-- setRemoteDescription()  |                               |
   |                           |                               |
   |-- WS: ice_candidate ------>|-- WS: ice_candidate -------->|
   |<-- WS: ice_candidate ------|<-- WS: ice_candidate --------|
   |                           |                               |
   |<======= P2P WebRTC media stream (audio + video) ========>|
   |         (Laravel sees NONE of this traffic)               |
   |                           |                               |
   |-- Call ends: ------------>|                               |
   |   WS: call_ended -------->|-- WS: call_ended ------------>|
   |                           |-- POST /appointments/{id}     |
   |                           |   status: completed           |
   |                           |-- Log session duration        |
```

**TURN Server Flow (fallback when P2P fails):**
```
PATIENT -----> COTURN Server <------ DOCTOR
               (relays media only when
                direct P2P is blocked by NAT/firewall)
```

---

### WORKFLOW 6 — Post-Consultation Clinical Workflow

```
DOCTOR                      SYSTEM                    LAB TECH / PHARMACIST
   |                           |                               |
   |-- Open consultation notes->|                              |
   |-- Write diagnosis -------->|                              |
   |-- Add prescription ------->|                              |
   |   (drug, dosage, freq,    |                              |
   |    duration, notes)        |                              |
   |-- Request lab tests ------>|                              |
   |   (test type, urgency,    |                              |
   |    notes)                  |                              |
   |-- Submit/Sign ------------>|                              |
   |                           |-- Save EMR record ----------> |
   |                           |-- Route to Pharmacy queue --> |--- Pharmacist sees Rx
   |                           |-- Route to Lab queue -------> |--- Lab Tech sees order
   |                           |-- Update appt: Completed ---> |
   |                           |-- Generate invoice ---------> |
   |                           |-- Notify patient (receipt, -> |
   |                           |   prescription PDF)           |
```

---

### WORKFLOW 7 — Laboratory Workflow

```
LAB TECH                    SYSTEM                        PATIENT / DOCTOR
   |                           |                               |
   |-- View lab queue --------->|                              |
   |<-- Pending orders ---------|                              |
   |-- Select order ----------->|                              |
   |-- Mark: In-Progress ------>|-- Update LabOrder status --> |
   |-- Upload results ---------->|                             |
   |   (file + structured data)|                              |
   |-- Mark: Completed -------->|-- Notify doctor (WS+email)->|
   |                           |                               |-- Doctor reviews results
   |                           |                               |-- Doctor adds notes to EMR
```

---

### WORKFLOW 8 — Pharmacy Workflow

```
PHARMACIST                  SYSTEM                        PATIENT
   |                           |                               |
   |-- View Rx queue ---------->|                              |
   |<-- Pending prescriptions --|                              |
   |-- Select prescription ---->|                              |
   |-- Check inventory -------->|-- Query drug stock -------> |
   |-- Dispense drugs ---------->|                             |
   |   (confirm qty dispensed) |                              |
   |-- Mark: Dispensed -------->|-- Update Prescription -----> |
   |                           |-- Deduct from inventory ----> |
   |                           |-- Notify patient (pickup) --> |--- Patient notified
   |                           |-- Generate pharmacy receipt ->|
```

---

### WORKFLOW 9 — Telemetry Monitoring (Real-Time)

```
IoT DEVICE / WEARABLE        SYSTEM                        NURSE / DOCTOR
   |                           |                               |
   |-- POST /telemetry -------->|                              |
   |   {patient_id, pulse,     |                              |
   |    spo2, bp, timestamp}   |                              |
   |                           |-- Validate token -----------> |
   |                           |-- Save TelemetryRecord -----> |
   |                           |-- Run threshold checks:       |
   |                           |   pulse < 50 OR > 120?        |
   |                           |   SpO2 < 90%?                 |
   |                           |   if YES → ALERT event -----> |
   |                           |-- Broadcast via WebSocket --> |--- Dashboard updates live
   |                           |                               |--- Alert banner fires
   |                           |-- Log to AuditLog ----------> |
```

---

## SECTION 3 — FRONTEND DELIVERABLES

---

### 3.1 — Global / Shared Components

| Component | Description |
|---|---|
| `AuthGuard` | Route wrapper enforcing role-based access |
| `RoleLayout` | Shell layout that adapts sidebar/nav per role |
| `ToastSystem` | Global alert banners (calls, telemetry alerts, errors) |
| `WebSocketProvider` | Laravel Echo context, auto-reconnects |
| `NotificationBell` | Live notification dropdown |
| `ConfirmModal` | Reusable confirmation dialog |
| `PageLoader` | Skeleton screens for async data |
| `ErrorBoundary` | Graceful error fallback UI |
| `FileUpload` | Drag-and-drop + preview component |
| `DataTable` | Sortable, filterable, paginated table |

---

### 3.2 — Authentication Module

| Screen / Component | Interactions |
|---|---|
| Login screen | Email/password form, show/hide password |
| MFA entry screen | 6-digit OTP input, resend OTP countdown timer |
| Password reset (request) | Email entry, success feedback |
| Password reset (confirm) | New password + confirm, strength indicator |
| Session expiry modal | Auto-logout warning with countdown |

**State:** `authStore` (Zustand) — `user`, `token`, `role`, `isAuthenticated`

---

### 3.3 — Role Dashboards

| Dashboard | Key Widgets |
|---|---|
| **Admin** | System stats cards, recent activity feed, pending user approvals, quick links |
| **Doctor** | Today's appointments, pending lab results, active prescriptions, unread messages |
| **Nurse** | Triage queue, vitals entry shortcut, assigned patients, alerts panel |
| **Patient** | Upcoming appointments, recent prescriptions, outstanding invoices, telemetry summary |
| **Lab Tech** | Pending orders queue, urgent orders flagged, completed today count |
| **Pharmacist** | Rx queue, low-stock alerts, dispense history |

---

### 3.4 — Profile Management

| Component | Fields |
|---|---|
| Patient profile form | Name, DOB, gender, phone, address, blood group, allergies, HMO/insurance, emergency contact |
| Doctor profile form | Name, specialization, qualifications, bio, consultation fee, availability toggle |
| Nurse profile form | Name, department, shift info |
| Avatar upload | Crop + upload, preview |
| Change password form | Current + new + confirm |

---

### 3.5 — Appointment Scheduling Module

| Component | Description |
|---|---|
| Doctor search/filter | Filter by specialty, availability, consultation type, rating |
| Doctor listing cards | Avatar, name, specialty, fee, next available slot, book CTA |
| Doctor profile modal | Full bio, qualifications, reviews |
| Availability calendar | Week view, color-coded slot states (available / booked / locked) |
| Slot picker | Time slot grid, 15-min lock countdown once selected |
| Booking summary card | Doctor info, date/time, type, fee breakdown |
| Consultation type selector | In-person / Video / Home visit tabs with fee display |
| Home visit form | Address input (with map picker), symptoms, preferred time |
| Paystack checkout embed | Inline payment form with card / bank transfer / USSD options |
| Payment success screen | Confirmation, receipt download, add to calendar link |
| Appointment list (patient) | Upcoming / past tabs, status badges, cancel / reschedule actions |
| Appointment list (doctor) | Today's view, weekly view, patient quick-profile on hover |

**State:** `appointmentStore` — `selectedDoctor`, `selectedSlot`, `lockExpiry`, `bookingStep`

---

### 3.6 — Video Consultation Room

| Component | Description |
|---|---|
| Pre-call lobby | Camera/mic test, device selector, join button |
| Video grid | Local + remote video tiles, speaking indicator |
| Control bar | Mute, camera toggle, screen share, end call, settings |
| Clinical notes panel | Doctor-only side panel: diagnosis input, prescription pad, inline during call |
| Chat panel | Text chat alongside video (via WebSocket) |
| Connection status banner | Reconnecting / poor connection warnings |
| Call ended screen | Duration summary, rate experience, view notes |
| Waiting room | Patient waits here until doctor joins |

**WebRTC State (React refs + local state):**
- `localStream`, `remoteStream`
- `peerConnection` (RTCPeerConnection instance)
- `callStatus`: idle / ringing / connecting / connected / ended
- `isMuted`, `isCameraOff`, `isScreenSharing`

---

### 3.7 — Electronic Medical Records (EMR)

| Component | Description |
|---|---|
| Patient EMR viewer | Tabbed: Overview, Vitals, History, Prescriptions, Labs, Files |
| Vitals chart | Line/area chart (Chart.js) — pulse, BP, SpO2 over time |
| Clinical history timeline | Chronological consultation entries |
| Diagnosis form | ICD-10 code search + description, severity, notes |
| Vital signs entry form | Nurse-facing: BP, pulse, temp, weight, SpO2, RR |
| Allergy & medication list | Add/flag allergies, current medications |
| File attachments panel | Upload + view scans, reports, referral letters |

---

### 3.8 — Prescription Management

| Component | Description |
|---|---|
| Prescription pad (doctor) | Drug search (autocomplete from formulary), dosage, frequency, duration, special instructions |
| Prescription viewer (patient) | Formatted prescription card, download PDF |
| Prescription queue (pharmacist) | List with patient name, drugs, urgency, status badges |
| Dispense confirmation modal | Confirm qty, add batch/expiry notes |

---

### 3.9 — Laboratory Management

| Component | Description |
|---|---|
| Lab order form (doctor) | Test type selector, urgency, clinical notes, submit |
| Lab queue (lab tech) | Sortable by urgency, patient, test type, status filter |
| Results upload form | Structured input + file attachment (PDF/image) |
| Results viewer (doctor/patient) | Formatted result card with reference ranges, flag indicators |

---

### 3.10 — Pharmacy Management

| Component | Description |
|---|---|
| Drug inventory table | Drug name, category, stock qty, reorder level, expiry, actions |
| Add/edit drug form | Name, category, unit, stock qty, reorder threshold, supplier |
| Low stock alert list | Drugs below reorder level, order prompt |
| Dispense history table | Date, patient, drug, qty, dispensed by |

---

### 3.11 — Billing & Payments

| Component | Description |
|---|---|
| Invoice viewer | Itemized: consultation fee, lab fees, drug costs, totals |
| Payment history table | Date, description, amount, status, receipt link |
| Receipt modal / PDF | Downloadable formatted receipt |
| Outstanding invoices list | Admin view of all unpaid invoices |
| Refund request form | Patient submits refund reason, admin approves |

---

### 3.12 — Telemetry Monitoring

| Component | Description |
|---|---|
| Live vitals dashboard | Real-time updating cards: pulse, SpO2, BP (WebSocket driven) |
| Telemetry chart | Rolling time-series chart (last 60 minutes), ApexCharts |
| Alert threshold settings (admin) | Set per-vital alert thresholds per patient |
| Critical alert banner | Full-width dismissable banner when threshold breached |
| Telemetry history table | Raw data log, exportable |

---

### 3.13 — Admin Module

| Component | Description |
|---|---|
| User management table | List all users, filter by role, activate/deactivate, reset password |
| Create staff form | Name, email, role, department, send invite |
| Role & permissions matrix | Visual grid of role permissions, toggle to adjust |
| Audit log viewer | Searchable, filterable log of all system events |
| System settings form | Hospital name, logo, working hours, consultation fee matrix, service area config |

---

### 3.14 — Reports & Analytics

| Chart / Widget | Data |
|---|---|
| Monthly revenue bar chart | Revenue by month, segmented by consultation type |
| Appointment volume line chart | Daily/weekly appointments, completed vs cancelled |
| Patient retention metric | New vs returning patients |
| Doctor performance table | Appointments completed, avg rating, consultation count |
| Department workload chart | Lab orders, Rx count by department |
| Bed/slot utilization gauge | Real-time slot fill rate |
| Export buttons | CSV / PDF export for all report tables |

---

## SECTION 4 — BACKEND DELIVERABLES

---

### 4.1 — Database Schema

#### Core Tables

```sql
users
  id, name, email, phone, password, role_id, is_active,
  email_verified_at, phone_verified_at, avatar, created_at, updated_at

roles
  id, name (admin|doctor|nurse|patient|lab_tech|pharmacist), guard_name

permissions
  id, name, guard_name

role_has_permissions
  permission_id, role_id

patient_profiles
  id, user_id, date_of_birth, gender, blood_group, allergies (json),
  emergency_contact_name, emergency_contact_phone,
  hmo_provider, hmo_number, address, created_at, updated_at

doctor_profiles
  id, user_id, specialization, qualifications (json), bio,
  consultation_fee, home_visit_fee, video_consultation_fee,
  department_id, is_available, license_number, created_at, updated_at

nurse_profiles
  id, user_id, department_id, shift (morning|afternoon|night),
  created_at, updated_at

departments
  id, name, head_doctor_id, created_at, updated_at
```

#### Scheduling Tables

```sql
doctor_schedules
  id, doctor_id, day_of_week (0-6), start_time, end_time,
  slot_duration_minutes, is_active

doctor_schedule_exceptions
  id, doctor_id, exception_date, reason, is_off_day

appointments
  id, patient_id, doctor_id, appointment_date, start_time, end_time,
  type (in_person|video|home_visit), status (pending_payment|confirmed|
  triaged|in_progress|completed|cancelled|no_show),
  chief_complaint, home_address (nullable), notes,
  cancellation_reason, cancelled_by, created_at, updated_at

appointment_slot_locks
  id, doctor_id, appointment_date, start_time, patient_id,
  locked_at, expires_at
  (managed in Redis, table for audit only)
```

#### EMR Tables

```sql
emr_records
  id, patient_id, appointment_id, doctor_id,
  chief_complaint, clinical_notes, assessment, plan,
  signed_at, created_at, updated_at

diagnoses
  id, emr_record_id, icd10_code, description,
  severity (mild|moderate|severe), is_primary, notes

vitals
  id, patient_id, appointment_id, recorded_by (user_id),
  blood_pressure_systolic, blood_pressure_diastolic,
  pulse_rate, temperature, weight, height, spo2,
  respiratory_rate, notes, recorded_at

allergies
  id, patient_id, allergen, reaction, severity, recorded_by

patient_documents
  id, patient_id, uploader_id, document_type,
  file_path, file_name, file_size, mime_type,
  description, created_at
```

#### Prescription & Pharmacy Tables

```sql
prescriptions
  id, emr_record_id, patient_id, doctor_id,
  status (pending|dispensed|cancelled), notes,
  dispensed_at, dispensed_by, created_at

prescription_items
  id, prescription_id, drug_id, dosage, frequency,
  duration_days, route (oral|iv|topical|inhaled),
  special_instructions, qty_prescribed, qty_dispensed

drugs
  id, name, generic_name, category, unit,
  stock_qty, reorder_level, supplier, unit_price,
  created_at, updated_at

drug_stock_movements
  id, drug_id, type (in|out|adjustment), qty,
  reference_id, reference_type, note, performed_by, created_at
```

#### Laboratory Tables

```sql
lab_orders
  id, emr_record_id, patient_id, requested_by,
  status (pending|in_progress|completed|cancelled),
  urgency (routine|urgent|critical), notes, created_at

lab_order_items
  id, lab_order_id, test_id, status, notes

lab_tests
  id, name, category, turnaround_hours, reference_ranges (json),
  unit, cost

lab_results
  id, lab_order_item_id, result_value, result_file_path,
  is_flagged, flag_type (high|low|critical),
  processed_by, notes, resulted_at
```

#### Billing Tables

```sql
invoices
  id, patient_id, appointment_id, invoice_number,
  status (pending|paid|partially_paid|refunded|cancelled),
  subtotal, discount, tax, total, paid_amount,
  due_date, notes, created_at

invoice_items
  id, invoice_id, description, category
  (consultation|lab|pharmacy|home_visit), qty, unit_price, total

payments
  id, invoice_id, patient_id, amount, payment_method,
  paystack_reference, paystack_status, gateway_response (json),
  paid_at, created_at

refunds
  id, payment_id, amount, reason, status (pending|approved|rejected|processed),
  requested_by, processed_by, processed_at, notes
```

#### Telemedicine Tables

```sql
consultation_rooms
  id, appointment_id, room_token (unique, signed), status
  (waiting|active|ended), started_at, ended_at,
  duration_seconds, created_at

room_events
  id, room_id, user_id, event_type
  (joined|left|muted|camera_off|screen_share_start|call_ended),
  payload (json), created_at
```

#### Telemetry Tables

```sql
telemetry_records
  id, patient_id, device_id, pulse_rate, spo2,
  blood_pressure_systolic, blood_pressure_diastolic,
  temperature, recorded_at, created_at

telemetry_alert_thresholds
  id, patient_id, metric, min_value, max_value,
  alert_level (warning|critical), is_active

telemetry_alerts
  id, patient_id, threshold_id, metric, recorded_value,
  alert_level, is_acknowledged, acknowledged_by, acknowledged_at,
  created_at
```

#### Audit & System Tables

```sql
audit_logs
  id, user_id, action, model_type, model_id,
  old_values (json), new_values (json),
  ip_address, user_agent, created_at
  (INSERT ONLY — no updates or deletes ever)

notifications
  id, user_id, type, title, body, data (json),
  read_at, created_at

system_settings
  id, key, value, description, updated_by, updated_at
```

---

### 4.2 — API Endpoints

#### Authentication

```
POST   /api/auth/register              Patient self-registration
POST   /api/auth/login                 Login (returns token)
POST   /api/auth/verify-otp            Verify OTP (email or SMS)
POST   /api/auth/resend-otp            Resend OTP
POST   /api/auth/logout                Revoke token
POST   /api/auth/forgot-password       Send reset link/OTP
POST   /api/auth/reset-password        Set new password
POST   /api/auth/refresh               Refresh token
GET    /api/auth/me                    Get authenticated user profile
```

#### Users & Profiles

```
GET    /api/users                      [Admin] List all users (paginated, filterable)
POST   /api/users                      [Admin] Create staff account
GET    /api/users/{id}                 Get user + profile
PUT    /api/users/{id}                 Update user
DELETE /api/users/{id}                 [Admin] Deactivate user
PUT    /api/users/{id}/activate        [Admin] Reactivate user
PUT    /api/profile/patient            Update patient profile
PUT    /api/profile/doctor             Update doctor profile
PUT    /api/profile/nurse              Update nurse profile
POST   /api/profile/avatar             Upload avatar
PUT    /api/profile/change-password    Change password
```

#### Doctors & Availability

```
GET    /api/doctors                    List doctors (filterable: specialty, availability, type)
GET    /api/doctors/{id}               Doctor public profile
GET    /api/doctors/{id}/availability  Available slots for a doctor (date range)
POST   /api/doctors/{id}/schedule      [Doctor/Admin] Set weekly schedule
PUT    /api/doctors/{id}/schedule      Update weekly schedule
POST   /api/doctors/{id}/exceptions    Add schedule exception (day off, etc.)
PUT    /api/doctors/{id}/availability  Toggle doctor available/unavailable
```

#### Appointments

```
GET    /api/appointments               List appointments (role-scoped)
POST   /api/appointments               Create appointment (locks slot, returns lock token)
GET    /api/appointments/{id}          Get appointment detail
PUT    /api/appointments/{id}          Update (status, notes)
DELETE /api/appointments/{id}          Cancel appointment
POST   /api/appointments/{id}/reschedule   Reschedule (payment handling)
PUT    /api/appointments/{id}/status   Update status (triaged, in_progress, completed, no_show)
GET    /api/appointments/today         [Doctor/Nurse] Today's queue
```

#### Home Visit

```
POST   /api/home-visits                Book home visit (address, date, symptoms)
GET    /api/home-visits                List home visits (role-scoped)
GET    /api/home-visits/{id}           Get home visit detail
PUT    /api/home-visits/{id}/accept    [Doctor] Accept home visit
PUT    /api/home-visits/{id}/decline   [Doctor] Decline home visit
PUT    /api/home-visits/{id}/complete  [Doctor] Mark completed
```

#### Payments & Billing

```
POST   /api/payments/initialize        Initialize Paystack transaction for appointment
GET    /api/payments/verify/{ref}      Verify Paystack transaction
POST   /api/payments/webhook           Paystack webhook (public, HMAC validated)
GET    /api/invoices                   List invoices (role-scoped)
GET    /api/invoices/{id}              Get invoice with items
GET    /api/invoices/{id}/pdf          Download invoice PDF
POST   /api/refunds                    [Patient] Request refund
PUT    /api/refunds/{id}               [Admin] Approve or reject refund
GET    /api/payments/history           Patient payment history
```

#### Video Consultation (Signaling)

```
POST   /api/consultation-rooms         Create room (on appointment confirmation)
GET    /api/consultation-rooms/{token} Validate token, get room + TURN config
POST   /api/consultation-rooms/{token}/end   End consultation session
GET    /api/consultation-rooms/{token}/events  Room event log
```

**WebSocket Events (Laravel Reverb — Broadcasting)**

```
presence-room.{roomToken}
  → client-sdp-offer          Relay SDP offer
  → client-sdp-answer         Relay SDP answer
  → client-ice-candidate      Relay ICE candidate
  → client-call-ringing       Notify callee
  → client-call-accepted      Notify caller
  → client-call-ended         Both parties
  → client-peer-joined        Someone entered room
  → client-peer-left          Someone left room
  → client-chat-message       In-call text chat

private-user.{userId}
  → AppointmentConfirmed
  → AppointmentReminder
  → LabResultReady
  → PrescriptionDispensed
  → TelemetryAlert
  → NewIncomingCall
  → InvoiceGenerated
```

#### EMR

```
GET    /api/patients/{id}/emr          Get full EMR (paginated history)
POST   /api/emr                        Create EMR record (doctor, post-consultation)
GET    /api/emr/{id}                   Get single EMR record
PUT    /api/emr/{id}                   Update EMR (only before signed)
POST   /api/emr/{id}/sign              Doctor digitally signs EMR (locks record)
POST   /api/patients/{id}/vitals       [Nurse] Record vitals
GET    /api/patients/{id}/vitals       Get vitals history
POST   /api/patients/{id}/documents    Upload patient document
GET    /api/patients/{id}/documents    List patient documents
DELETE /api/documents/{id}             Delete document
```

#### Diagnoses

```
GET    /api/icd10/search               Search ICD-10 codes (autocomplete)
POST   /api/diagnoses                  Add diagnosis to EMR
PUT    /api/diagnoses/{id}             Update diagnosis
DELETE /api/diagnoses/{id}             Remove diagnosis
```

#### Prescriptions

```
POST   /api/prescriptions              [Doctor] Create prescription
GET    /api/prescriptions/{id}         Get prescription with items
GET    /api/prescriptions/queue        [Pharmacist] Pending dispense queue
PUT    /api/prescriptions/{id}/dispense  [Pharmacist] Mark dispensed
GET    /api/prescriptions/{id}/pdf     Download prescription PDF
GET    /api/patients/{id}/prescriptions  Patient prescription history
```

#### Laboratory

```
POST   /api/lab-orders                 [Doctor] Create lab order
GET    /api/lab-orders                 [Lab Tech/Doctor] List orders
GET    /api/lab-orders/{id}            Get order with items
PUT    /api/lab-orders/{id}/status     Update status (in_progress)
POST   /api/lab-orders/{id}/results    [Lab Tech] Upload results
GET    /api/lab-tests                  List available lab tests
GET    /api/patients/{id}/lab-results  Patient's lab result history
```

#### Pharmacy / Drug Inventory

```
GET    /api/drugs                      List drugs (filterable, searchable)
POST   /api/drugs                      [Admin/Pharmacist] Add drug
PUT    /api/drugs/{id}                 Update drug
POST   /api/drugs/{id}/stock           Record stock movement (in/out/adjustment)
GET    /api/drugs/low-stock            Drugs below reorder level
GET    /api/drugs/{id}/movements       Stock movement history
```

#### Telemetry

```
POST   /api/telemetry                  [Device] Submit telemetry data (high-throughput)
GET    /api/patients/{id}/telemetry    Get telemetry history (paginated)
GET    /api/patients/{id}/telemetry/live  SSE or WS endpoint for live feed
POST   /api/telemetry/thresholds       [Doctor/Admin] Set alert thresholds
GET    /api/telemetry/alerts           List alerts (role-scoped)
PUT    /api/telemetry/alerts/{id}/acknowledge  Acknowledge alert
```

#### Admin & Reports

```
GET    /api/admin/stats                System-wide stats (users, appointments, revenue)
GET    /api/reports/revenue            Revenue report (filterable by date range)
GET    /api/reports/appointments       Appointment volume report
GET    /api/reports/doctors            Doctor performance report
GET    /api/reports/patients           Patient retention report
GET    /api/reports/pharmacy           Pharmacy sales and stock report
GET    /api/audit-logs                 [Admin] Audit log (searchable, filterable)
GET    /api/system/settings            Get system settings
PUT    /api/system/settings            [Admin] Update system settings
```

#### Notifications

```
GET    /api/notifications              Get user notifications (paginated)
PUT    /api/notifications/{id}/read    Mark one as read
POST   /api/notifications/read-all    Mark all as read
DELETE /api/notifications/{id}         Delete notification
```

---

### 4.3 — Background Jobs (Laravel Queues)

| Job | Trigger | Action |
|---|---|---|
| `SendOtpSms` | Registration, login, password reset | Dispatch SMS via Termii |
| `SendAppointmentConfirmation` | Payment webhook success | Send email + SMS to patient and doctor |
| `SendAppointmentReminder` | Scheduled: 24hr & 1hr before | Notify both patient and doctor |
| `ReleaseExpiredSlotLock` | Scheduled: every minute | Delete expired Redis slot locks |
| `GenerateInvoice` | Appointment completed | Build and store invoice PDF |
| `SendInvoiceEmail` | Invoice generated | Email invoice to patient |
| `ProcessTelemetryAlert` | Telemetry POST exceeds threshold | Broadcast alert event, notify assigned doctor |
| `SendLabResultNotification` | Lab result uploaded | Email + in-app notify doctor and patient |
| `SendPrescriptionReadyNotification` | Rx dispensed | Notify patient |
| `CheckLowDrugStock` | Scheduled: daily | Alert pharmacist/admin if stock below reorder level |
| `GenerateMonthlyReport` | Scheduled: 1st of each month | Aggregate and cache report data |
| `DatabaseBackup` | Scheduled: daily 2AM | Encrypted DB dump to S3/remote storage |
| `PruneOldNotifications` | Scheduled: weekly | Delete read notifications older than 90 days |

---

### 4.4 — Security Requirements

| Requirement | Implementation |
|---|---|
| Authentication | Laravel Sanctum (Bearer tokens) |
| Role-based access | Spatie Laravel Permission middleware on all routes |
| OTP verification | 6-digit, 10-minute TTL, max 3 attempts then cooldown |
| Paystack webhook security | HMAC-SHA512 signature validation on every webhook |
| Audit logging | Model observer on all sensitive models (EMR, User, Payment) |
| Rate limiting | `throttle:60,1` globally, `throttle:5,1` on auth routes |
| Input sanitization | Laravel FormRequest validators on all endpoints |
| File upload security | MIME type whitelist, virus scan hook, 10MB size limit |
| EMR immutability | EMR signed records locked — no update/delete permitted |
| Data encryption | Sensitive fields (HMO number, allergies) encrypted at rest |
| CORS | Strict origin whitelist (only React frontend domain) |
| HTTPS | SSL enforced via Nginx, HTTP redirected to HTTPS |
| Database backups | Daily encrypted backup, 30-day retention |

---

### 4.5 — Laravel Project Folder Structure

```
app/
├── Http/
│   ├── Controllers/Api/
│   │   ├── AuthController.php
│   │   ├── AppointmentController.php
│   │   ├── ConsultationRoomController.php
│   │   ├── EmrController.php
│   │   ├── LabOrderController.php
│   │   ├── PharmacyController.php
│   │   ├── BillingController.php
│   │   ├── TelemetryController.php
│   │   ├── ReportController.php
│   │   └── AdminController.php
│   ├── Middleware/
│   │   ├── RoleMiddleware.php
│   │   ├── AuditMiddleware.php
│   │   └── ValidatePaystackWebhook.php
│   └── Requests/        (FormRequest per endpoint)
├── Models/
│   ├── User.php
│   ├── Appointment.php
│   ├── ConsultationRoom.php
│   ├── EmrRecord.php
│   ├── Prescription.php
│   ├── LabOrder.php
│   ├── Drug.php
│   ├── Invoice.php
│   ├── TelemetryRecord.php
│   └── AuditLog.php
├── Events/              (Broadcast events)
│   ├── SdpOfferReceived.php
│   ├── IceCandidateReceived.php
│   ├── CallEnded.php
│   ├── TelemetryAlert.php
│   └── AppointmentStatusChanged.php
├── Jobs/
│   ├── SendOtpSms.php
│   ├── GenerateInvoice.php
│   ├── ProcessTelemetryAlert.php
│   └── SendAppointmentReminder.php
├── Observers/
│   ├── EmrRecordObserver.php
│   ├── AppointmentObserver.php
│   └── PaymentObserver.php
├── Services/
│   ├── SlotGenerationService.php
│   ├── PaystackService.php
│   ├── TelemetryThresholdService.php
│   ├── PdfGeneratorService.php
│   └── OtpService.php
└── Policies/            (Laravel Policies per model)
```

---

<!-- ## SECTION 5 — API CONTRACT (Day 1 Sync — Frontend & Backend)

The following payloads must be agreed before development starts.

### Appointment Booking Flow

**POST /api/appointments** (lock slot)
```json
Request:
{
  "doctor_id": 5,
  "appointment_date": "2025-08-15",
  "start_time": "10:00",
  "type": "video",
  "chief_complaint": "Persistent headache for 3 days"
}

Response 200:
{
  "appointment_id": 112,
  "lock_token": "lock_abc123",
  "lock_expires_at": "2025-08-10T14:30:00Z",
  "fee": 5000,
  "currency": "NGN"
}
```

**POST /api/payments/initialize**
```json
Request:
{
  "appointment_id": 112,
  "lock_token": "lock_abc123"
}

Response 200:
{
  "payment_reference": "HMS-PAY-20250810-112",
  "authorization_url": "https://checkout.paystack.com/xyz",
  "access_code": "xyz"
}
```

### WebSocket Room Join

**GET /api/consultation-rooms/{token}**
```json
Response 200:
{
  "room_token": "room_tok_abc",
  "appointment_id": 112,
  "status": "waiting",
  "turn_servers": [
    {
      "urls": "turn:your-turn-server.com:3478",
      "username": "user123",
      "credential": "pass123"
    }
  ],
  "participants": {
    "doctor": { "id": 5, "name": "Dr. Adeyemi", "avatar": "..." },
    "patient": { "id": 22, "name": "John Doe", "avatar": "..." }
  }
} -->


---

*End of Document*
*Version 1.0 — Review with both frontend and backend teams before sprint kickoff*
