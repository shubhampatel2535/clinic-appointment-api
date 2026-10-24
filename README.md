# Clinic Appointment Booking API

A REST API built with Node.js, Express, and MongoDB. Clinics publish appointment slots; patients can book and cancel appointments. The implementation focuses on input validation, useful HTTP errors, automated tests, and concurrency-safe booking.

## Features

- Create, list, retrieve, and cancel appointment slots.
- Book, retrieve, and cancel bookings.
- Prevent double booking even when requests arrive at nearly the same time.
- Validate date/time values, required fields, and email addresses.
- Filter slots by status and clinic name.
- Automated integration tests using Jest, Supertest, and an ephemeral MongoDB instance.

## Requirements

- Node.js 20 or newer
- npm
- MongoDB locally, or a MongoDB connection URI

## Setup

```bash
git clone <your-github-repository-url>
cd clinic-appointment-api
npm install
```

Copy `.env.example` to `.env`, then set `MONGODB_URI`:

```env
PORT=3000
MONGODB_URI=mongodb://127.0.0.1:27017/clinic_appointments
NODE_ENV=development
```

Start MongoDB locally, then run:

```bash
npm run dev
```

The API will be available at `http://localhost:3000`. Check `GET /health` for a health response.

## API endpoints

All request and response bodies use JSON.

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health` | Health check |
| `POST` | `/api/slots` | Create a slot |
| `GET` | `/api/slots` | List slots; optional `status` and `clinicName` filters |
| `GET` | `/api/slots/:slotId` | Get one slot |
| `DELETE` | `/api/slots/:slotId` | Cancel an available slot |
| `POST` | `/api/bookings` | Book an available slot |
| `GET` | `/api/bookings/:bookingId` | Retrieve an active booking |
| `DELETE` | `/api/bookings/:bookingId` | Cancel a booking |

### Create a slot

```bash
curl -X POST http://localhost:3000/api/slots \
  -H "Content-Type: application/json" \
  -d '{
    "clinicName": "Northside Clinic",
    "doctorName": "Dr. Rao",
    "startAt": "2030-05-10T09:00:00.000Z",
    "endAt": "2030-05-10T09:30:00.000Z"
  }'
```

`startAt` must be in the future and `endAt` must be later than `startAt`. Use a future date when running the example.

### List available slots

```bash
curl "http://localhost:3000/api/slots?status=available&clinicName=Northside"
```

Allowed `status` values: `available`, `booked`, `cancelled`.

### Book a slot

Replace `SLOT_ID` with the `_id` returned when creating the slot.

```bash
curl -X POST http://localhost:3000/api/bookings \
  -H "Content-Type: application/json" \
  -d '{
    "slotId": "SLOT_ID",
    "patientName": "Asha Patel",
    "patientEmail": "asha@example.com"
  }'
```

A successful response has status `201 Created` and includes a `bookingId`. Keep that ID for lookup and cancellation.

### Retrieve or cancel a booking

```bash
curl http://localhost:3000/api/bookings/BOOKING_UUID
curl -X DELETE http://localhost:3000/api/bookings/BOOKING_UUID
```

Cancelling a booking makes its slot available again. Cancelling a slot directly is allowed only while it is available.

## HTTP status codes

- `200 OK`: successful read or cancellation
- `201 Created`: slot or booking created
- `400 Bad Request`: invalid JSON, identifier, or input
- `404 Not Found`: route, slot, or active booking does not exist
- `409 Conflict`: slot already booked/cancelled, or requested state transition is not allowed
- `500 Internal Server Error`: unexpected server-side error

Errors have a consistent shape, for example:

```json
{
  "error": {
    "code": "SLOT_UNAVAILABLE",
    "message": "This slot is already booked."
  }
}
```

Validation errors may also include a `details` array.

## Concurrency and double-booking prevention

A booking is performed with a single MongoDB `findOneAndUpdate` call whose filter requires both the slot ID and `status: "available"`. The same atomic operation changes the status to `booked` and writes booking details.

MongoDB atomically updates a single document. With concurrent requests, only one request can match the still-available state; subsequent requests receive `409 Conflict`. The availability check is not separated from the write, and correctness does not depend on a process-local lock. Cancellation uses a corresponding conditional atomic update.

Booking details are embedded in the slot document deliberately: slot status and booking data change together in one atomic document operation. This avoids a partial state between updating a slot and inserting a separate booking document.

## Run tests

```bash
npm test
```

Tests use `mongodb-memory-server`, which starts a temporary MongoDB server for integration tests. The first run may need internet access to download a compatible MongoDB binary. A local MongoDB service is not required for these tests.

The test suite covers:
- Creating/listing slots
- Invalid slot input
- Successful booking and duplicate booking rejection
- Two concurrent booking requests (exactly one succeeds)
- Booking cancellation and rebooking
- Preventing direct cancellation of booked slots
- Missing slots and invalid patient email

## Suggested GitHub submission steps

1. Review the code and run `npm test`.
2. Confirm `.env` and `node_modules` are not committed (`.gitignore` excludes them).
3. Create a new empty GitHub repository.
4. Commit and push this project:

```bash
git init
git add .
git commit -m "Build concurrency-safe clinic appointment API"
git branch -M main
git remote add origin <your-github-repository-url>
git push -u origin main
```

## Scope and production considerations

This assignment version does not implement authentication or clinic/patient authorization. In production, add authentication, authorization, rate limiting, audit logging, and privacy controls before exposing patient details. A production booking system should also define timezone and slot-overlap policies, cancellation rules, and idempotency behavior explicitly.
