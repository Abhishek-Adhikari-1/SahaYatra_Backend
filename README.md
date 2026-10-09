# SahaYatra Backend (GPS Tracking API)

SahaYatra (सहयात्रा - "Journey Together") Backend is a production-oriented, high-performance GPS tracking server built on the **Bun** runtime. It acts as the critical bridge between physical GPS tracking hardware and client applications (Web/Mobile).

The system is designed around a strict asynchronous pipeline: 
**GPS Gateway → Redis Stream → Worker → MongoDB → Pub/Sub → SSE**

## 🧠 Core Architectural Principle

> **The GPS hardware device does not directly communicate with the frontend.** 
> The Gateway is responsible for communicating with the hardware. The frontend communicates with the application backend.

This strict separation ensures that hardware-specific logic (packet framing, binary decoding, proprietary protocols) never spreads throughout the application. Everything downstream of the Gateway works with a single, standardized internal representation.

## 🏗️ High-Level Architecture

```text
[GPS Hardware Device] 
       │ (TCP/UDP Raw Packets)
       ▼
[GPS Gateway] ────► Protocol Handling, IMEI Auth, Packet Decoding, Telemetry Extraction
       │ (Normalized JSON)
       ▼
[Redis Stream] ───► Async Queue (Decouples Gateway from DB)
       │ (XREAD)
       ▼
[Worker] ─────────► Validates, Transforms, Calculates Geofence Events, Persists Data
       │
       ├──────────► [MongoDB] (Persistent Storage + GIS Geofencing)
       │
       └──────────► [Redis Pub/Sub] ──► [SSE Server] ──► [Web/Mobile Frontends]
```

---

## 🗺️ MongoDB GIS (Geospatial Strategy)

Since SahaYatra relies heavily on maps, routes, and geofencing, we leverage **MongoDB's native Geospatial (GIS)** capabilities using **GeoJSON** and **2dsphere indexes**.

### 1. Data Representation (GeoJSON)
- **Stops & Bus Locations:** Stored as `Point` coordinates `[longitude, latitude]`.
- **Routes:** Stored as `LineString` coordinates (the path drawn on MapLibre).

### 2. Geofencing Logic (`$geoWithin`)
Instead of calculating distance manually for every ping, the Worker uses MongoDB's `$centerSphere` operator to instantly determine if a bus has entered a stop's geofence.

```javascript
// Example: Check if Bus is inside Stop's 150m Arrival Zone
db.stops.find({
  location: {
    $geoWithin: {
      $centerSphere: [ [busLng, busLat], radiusInRadians ]
    }
  }
})
```

### 3. Nearest Stop Detection (`$near`)
Used to calculate ETAs and determine which stop the bus is approaching next.

---

## ⚙️ Core Components

### 1. GPS Gateway
The gateway is strictly responsible for hardware communication.
- **Connection Management:** TCP/UDP handling, timeouts, and disconnects.
- **Protocol Handling:** Packet framing, binary/text decoding, and validation.
- **Authentication:** Verifying device identity via **IMEI** upon connection.
- **Telemetry Extraction:** Converting hardware-specific packets into normalized JSON.

**Normalization Example:**
```text
// Raw Hardware Packet (Device A)
$GPS,860123,27.7172,85.3240,42,180

// Normalized Internal Event (Output of Gateway)
{
  "deviceId": "device-db-id",
  "vehicleId": "vehicle-db-id",
  "latitude": 27.7172,
  "longitude": 85.3240,
  "speed": 42,
  "heading": 180,
  "timestamp": "2026-09-16T05:30:00Z"
}
```

### 2. Redis (The Real-Time Engine)
Redis decouples the high-speed hardware ingestion from database writes.
- **Redis Streams:** The Gateway writes telemetry here immediately. The Gateway never waits for MongoDB.
- **Device Cache:** Stores real-time connection status and last known location.
- **Pub/Sub:** Broadcasts live location updates to the SSE server.

### 3. Worker
The worker consumes events from the Redis Stream.
- **Pipeline:** `Redis Stream → Worker → Validate → Transform → Calculate Events → Publish`
- **Geofence Calculation:** Checks bus coordinates against MongoDB GIS stop geofences.
- **Persistence:** Saves historical telemetry to MongoDB.

### 4. Dual-Timestamp Tracking
To handle delayed or out-of-order packets from GPS hardware, we store two timestamps:
- `recorded_at`: When the GPS device *says* the location was recorded (Device Clock).
- `received_at`: When our server *actually received* the packet (Server Clock).

### 5. Real-Time Distribution (SSE)
- **Redis Pub/Sub** distributes live telemetry internally.
- **SSE (Server-Sent Events)** pushes one-way continuous location updates to the Admin Dashboard and Parent Apps.

---

## 🛠️ Tech Stack
- **Runtime:** Bun (for ultra-fast I/O and native TCP/UDP socket support)
- **API Framework:** Hono / ElysiaJS
- **Database:** MongoDB (with Mongoose / Native Driver)
- **Geospatial:** MongoDB GIS (GeoJSON, 2dsphere Indexes)
- **Queue & Cache:** Redis (Streams, Pub/Sub)
- **Real-time:** Server-Sent Events (SSE)

---

### 🔗 Part of the SahaYatra Ecosystem
- 📱 [Mobile App (Flutter)](https://github.com/Abhishek-Adhikari-1/SahaYatra_Flutter.git)
- 💻 [Admin Dashboard (Next.js)](https://github.com/Abhishek-Adhikari-1/SahaYatra_Web.git)
