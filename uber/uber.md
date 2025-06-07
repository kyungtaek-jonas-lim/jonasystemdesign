# Uber
Uber is a ride-hailing service that connects passengers (riders) with drivers through a mobile app or website. Users can request a ride, and nearby available drivers are matched to pick them up and drop them off at their destination.


---

## Index

1. [Requirements](#1-requirements)
2. [Core Entities](#2-core-entities)
3. [APIs](#3-apis)
4. [High-level Design](#system-design-diagram-high-level-design)
5. [Data Flow](#4-data-flow)
6. [Deep Dive](#5-deep-dive)

---

&nbsp;

## System Design Diagram (High Level Design)
![System Design Diagram](https://raw.githubusercontent.com/kyungtaek-jonas-lim/jonasystemdesign/main/uber/uber.png)

&nbsp;

---

## High-Level Goals
- Connect riders with nearby drivers
- Minimize wait time for riders
- Efficiently assign drivers
- Provide real-time updates (map, ETA(Estimated Time of Arrival))
- Handle dynamic pricing (surge pricing)
- Track trips and store ride history
- Handle payments securely

&nbsp;

---
&nbsp;

## 1. Requirements
### Functional Requirements:
These are the must-have features:

#### For Riders:
1. User registration/login
2. Request a ride by entering pickup and drop-off locations
3. Get matched with a nearby driver
4. Real-time ride tracking on the map
5. Cancel a ride
6. Rate drivers
7. View ride history
8. Secure payments
9. Estimate fare before confirming a ride
10. Support surge pricing during high demand periods
    - cf. Surge pricing is a form of dynamic pricing that adjusts fares based on real-time demand and supply.
        - Dynamic Pricing = adjusting prices based on real-time factors
            - demand/supply
            - traffic
            - weather
            - time of day, etc.
        - Surge Pricing = Uber’s specific dynamic pricing model where the fare increases when demand exceeds supply in an area.


#### For Drivers:
1. Driver registration/login
2. Set availability (go online/offline)
3. Get ride requests and accept/decline
4. Real-time navigation to pickup/drop-off
5. Earnings tracking
6. Rate riders


### Non-functional Requirements:
These are qualities the system should have:
1. High availability
    - Downtime should be minimal.
2. Low latency
    - Fast response when matching rider and driver.
    - `less than 1 min to match` or `failure`
3. Scalability
    - Must handle millions of users.
4. Real-time updates
    - Location tracking, ride status, etc.
5. Data consistency
    - Important for payments, trip info.
6. Security
    - Protect user data and transactions.
7. High throughput
    - Should handle a high volume of ride requests and driver matches per second.
8. Consistency in matching
    - Ensure one-to-one matching between rider and driver — no duplicate assignments.


&nbsp;

---
&nbsp;


## 2. Core Entities
1. Ride
2. Driver
3. Rider
4. Location
5. Ride History
6. Payment

&nbsp;

---
&nbsp;


## 3. APIs

- `user_id` is extracted from the JWT in the request header.  
- Authentication and payment services are not exposed via APIs.  
- Surge pricing is retrieved from a third-party estimation service.

---

#### 1. [POST] /ride/request  
- **Description:** Insert ride request into the database and push it to Kafka.
- **Request:**  
  ```json
  {
    "pick_up_location": { "lat": 12.212, "lng": 12.312 },
    "drop_off_location": { "lat": 12.212, "lng": 13.312 }
  }
  ```
- **Response:**  
  ```json
  {
    "ride_id": "01HV7Y8TP1WAWM0BFGPV5RWFXG", // Generated using ULID with additional entropy
    "estimation": {
      "fare": 12000,
      "currency": "USD"
    }
  }
  ```

---

#### 2. [PATCH] /ride/driver/status  
- **Description:** Set driver availability in Redis (`driver:status:{user_id}`).
- **Request:**  
  ```json
  {
    "status": "available"
  }
  ```
- **Response:**  
  ```json
  {
    "status": "available"
  }
  ```

---

#### 3. [PATCH] /ride/request/confirm  
- **Description:** If the driver approves the request, update driver availability and ride info in DB.
- **Request:**  
  ```json
  {
    "ride_id": "01HV7Y8TP1WAWM0BFGPV5RWFXG",
    "approval": true
  }
  ```
- **Response:**  
  ```json
  {
    "confirmation_datetime": "2025-03-31T13:49:00.001Z"
  }
  ```

---

#### 4. [PATCH] /ride/arrival/pick-up/confirm  
- **Description:** Driver notifies rider that they have arrived at pickup location. *(Not shown in the diagram.)*
- **Request:**  
  ```json
  {
    "ride_id": "01HV7Y8TP1WAWM0BFGPV5RWFXG"
  }
  ```
- **Response:**  
  ```json
  {
    "success": true
  }
  ```

---

#### 5. [PATCH] /ride/on-trip/confirm  
- **Description:** Driver confirms the rider has boarded and starts the trip. *(Not shown in the diagram.)*
- **Request:**  
  ```json
  {
    "ride_id": "01HV7Y8TP1WAWM0BFGPV5RWFXG"
  }
  ```
- **Response:**  
  ```json
  {
    "departure_datetime": "2025-03-31T13:51:00.001Z"
  }
  ```

---

#### 6. [PATCH] /ride/arrival/drop-off/confirm  
- **Description:** Driver confirms arrival at drop-off. Updates departure and arrival times.
- **Request:**  
  ```json
  {
    "ride_id": "01HV7Y8TP1WAWM0BFGPV5RWFXG"
  }
  ```
- **Response:**  
  ```json
  {
    "departure_datetime": "2025-03-31T13:51:00.001Z",
    "arrival_datetime": "2025-03-31T14:01:00.001Z"
  }
  ```

---

#### 7. [GET] /ride/price/surge-price  
- **Description:** Retrieve surge multiplier based on real-time demand. Used for client-side caching.
- **Request:**  
  ```
  ?ride_id=01HV7Y8TP1WAWM0BFGPV5RWFXG
  ```
- **Response:**  
  ```json
  {
    "surge_multiplier": 1.5, // multiplier applied to base fare
    "valid_until": "2025-03-31T13:52:00Z" // client should cache until this time
  }
  ```

---

#### 8. [GET] /location/driver/near-by  
- **Description:** Get nearby available drivers from Redis using geospatial queries.
- **Request:**  
  ```
  ?lat=12.212
  &lng=12.812
  ```
- **Response:**  
  ```json
  {
    "near_by_drivers": [
      {
        "id": 4234,
        "name": "Jonas Lim",
        "rate": 8.5,
        "distance": 4,
        "unit": "miles",
        "location": { "lat": 12.212, "lng": 12.312 }
      },
      {
        "id": 423332,
        "name": "Kyungtaek Lim",
        "rate": 10,
        "distance": 6,
        "unit": "miles",
        "location": { "lat": 12.212, "lng": 12.912 }
      }
    ]
  }
  ```

---

#### 9. [PATCH] /location/driver  
- **Description:** Update driver's current location in Redis.
- **Request:**  
  ```json
  {
    "lat": 12.212,
    "lng": 12.812
  }
  ```
- **Response:**  
  ```json
  {
    "success": true
  }
  ```

---

#### 10. [GET] /location/ride  
- **Description:** Retrieve current location of the ride and estimated arrival time.
- **Request:**  
  ```
  ?ride_id=01HV7Y8TP1WAWM0BFGPV5RWFXG
  ```
- **Response:**  
  ```json
  {
    "location": { "lat": 12.212, "lng": 12.812 },
    "estimated_arrival_datetime": "2025-03-31T14:01:00.001Z"
  }
  ```

---

#### 11. [PATCH] /profile/rate  
- **Description:** Submit a rating and optional comment for a user.
- **Request:**  
  ```json
  {
    "user_id": "423332",
    "rate": 10,
    "comment": "So kind"
  }
  ```
- **Response:**  
  ```json
  {
    "success": true
  }
  ```


&nbsp;

---
&nbsp;

## 4. Data Flow

Riders and drivers use mobile applications that integrate the Google Maps SDK for real-time location tracking.  
The backend system consists of several services, including the **API Gateway, Auth Service, Ride Service, Location Service, Profile Service, and Payment Service**.

The **current locations of riders and drivers** are updated directly from the mobile app using the SDK.  
However, **the current ride location** (for active trips) is managed by the backend, which handles the overall ride state and route updates.

> *Note: Profile data management is out of scope for this design.*

---

### 1. API Gateway
- Handles authorization in coordination with the Auth Service
- Routes requests to appropriate backend services
- Performs load balancing across service instances


### 2. Auth Service
- Authenticates users (interacting with the database)
- Manages authorization rules
- Handles account creation and deletion

### 3. Ride Service
- Updates driver availability (stores available drivers in Redis)
- Handles incoming ride requests from riders  
  (via Kafka → finds nearby drivers via Location Service → checks available/unreserved drivers in Redis → creates ride entry in DB)
- Integrates with a third-party fare estimation service to support surge pricing
- Receives driver arrival confirmations  
  (updates driver availability in Redis and ride status in DB)
- Sends push notifications (via APNs or GCM), including:
  - Ride request prompts to drivers
  - Match confirmations to riders
  - “No available drivers” error notifications
  - Invalidations if no driver responds within timeout
  - Arrival alerts

### 4. Location Service
- Updates live driver locations (stored in Redis)
- Updates current ride locations for active trips (also stored in Redis)

### 5. Profile Service
- Manages and updates driver and rider ratings (stored in the database)

### 6. Payment Service
- Initiates payment requests (integrated with third-party payment providers)
- Provides access to user payment history (stored in the database)


&nbsp;
---
&nbsp;

## 5. Deep Dive

### Database

#### User
- id (PK)
- username
- role ['rider', 'driver']
- payment_id (FK)
- create_datetime
- update_datetime

#### Ride
- id (PK)
- rider_id (FK)
- pick_up_location
- drop_off_location
- status ['matching', 'matched', 'on-trip', 'completed', 'cancelled']
- driver_id (FK, nullable)
- driver_approved_datetime (nullable)
- departure_datetime (nullable)
- arrival_datetime (nullable)
- payment_id
- create_datetime

#### Payment
- payment_id (PK)
- user_id (FK)
- ride_id (FK)
- method
- currency
- price
- status
- payment_datetime
- transaction_id (From the payment service)
- create_datetime

#### Rate
- id (PK)
- commenter_user_id (FK)
- target_user_id (FK)
- rate
- comment
- create_datetime
- update_datetime


### Redis
- Geo Hashing
    - Redis uses sorted sets (ZSETs) to support geospatial indexing.
    - GeoHashing is a geospatial encoding system, and Redis uses this concept behind the scenes within its sorted sets.
    - Examples
        ```bash
        GEOADD drivers:locations 13.361389 38.115556 "driver1"
        ```
        ```bash
        GEORADIUS drivers:locations 15 37 100 km
        ```

#### Active Ride for Riders (For unique ride per rider)
- **key:** `rider:{user_id}:active_ride`
- **value:** `ride_id`

#### driver:{user_id}:active_ride (For unique ride per driver)
- **key:** `driver:{user_id}:active_ride`
- **value:** `ride_id`

#### Ride status
- **key:** `ride:{ride_id}`
- **value:**
    - `{current_status}` [`matching`, `matched`, `on-trip`, `completed`, `cancelled`]
    - `current_location` (long, lat) (Using GeoHashing)

#### Drivers' status (Only for the active drivers)
- **key:** `driver:status:{user_id}`
- **value:** `{current_status}` [`available`, `driving`] (Using GeoHashing)

#### Drivers' reservation(lock) status (With TTL: 5 sec)
- **key:** `driver:reservation:{user_id}`



&nbsp;
---
&nbsp;

## 6. Q & A
Below are answers to commonly anticipated system design questions, to clarify design decisions and assumptions.

### 1. How do you find “nearby drivers” efficiently in Redis?
> I use Redis' geospatial indexing capabilities (via **GEOADD**, **GEORADIUS**) to efficiently track driver locations. Drivers update their GPS coordinates every 5 seconds. When a rider requests a ride, the backend queries Redis using the rider’s location to find nearby available drivers within a specified radius, sorted by proximity.

---

### 2. What if multiple riders request at once?
> To handle high concurrency and ensure fairness, I place a **Kafka topic** in front of the Ride Service. Incoming rider requests are pushed to the topic, and the Ride Service consumes these requests sequentially or in batches, enabling scalable and orderly processing under high load.

---

### 3. What happens if no driver accepts?
> If no available driver accepts the ride within a defined timeout (e.g., 15–30 seconds), the system returns an error indicating **“no drivers available.”** The rider is then prompted to either retry the request or cancel. This ensures a responsive user experience while avoiding indefinite waiting.

---

### 4. Do you retry? Timeout?
> The system doesn’t automatically retry the request to avoid unintended bookings. Instead, it provides a **clear fallback mechanism**: if a match fails, the rider can manually retry. This gives users control and helps avoid double-booking or redundant notifications to drivers.

---

### 5. Do you use rate limiting or circuit breakers?
> Yes, if necessary. Rate limiting is applied at the API Gateway level to prevent users from overloading the system with excessive requests. Redis is used to track request counts with TTL-based keys.
For circuit breaking, I apply it on a per-service basis — especially for critical services like Ride or Payment. If a downstream service is failing or experiencing high latency, the circuit breaker temporarily stops further requests, avoiding cascading failures and improving overall system resilience.