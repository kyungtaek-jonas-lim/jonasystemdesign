# FastPass
FastPass is a large-scale virtual queue management system used in theme parks like Disney World to optimize ride wait times and enhance visitor experience. It allows guests to reserve access to popular attractions at specific times, enabling them to enjoy other park activities while waiting in a virtual line. The system must handle millions of users, synchronize real-time reservations across distributed services, prioritize fairness and availability, and integrate with ticketing, mobile apps, and physical entry scanners in a scalable and resilient way.


---

## Index

1. [Requirements](#1-requirements)
2. [Core Entities](#2-core-entities)
3. [APIs](#3-apis)
4. [High-level Design](#system-design-diagram-high-level-design)
5. [Data Flow](#4-data-flow)
6. [Deep Dive](#5-deep-dive)
<!-- 7. [Additional Info](#6-additional-info) -->
<!-- 8. [Reference](#7-reference) -->

---

&nbsp;

## System Design Diagram (High Level Design)
![System Design Diagram](https://raw.githubusercontent.com/kyungtaek-jonas-lim/jonasystemdesign/main/fastpass/fastpass.png)


&nbsp;
---
&nbsp;

## 1. Requirements
### Functional Requirements:
1. Users should be able to book a ride for a specific time slot.
2. Users should be able to check their booking history.
3. Users should be able to cancel the booking.
4. Users should be able to modify the time slot.
5. Users should receive a notification when their reserved time is approaching.
6. Users should be able to check in during their reserved time window.

### Non-functional Requirements:
1. No overbooking for each time.
2. High consistency
3. No-show handling


&nbsp;
---
&nbsp;


## 2. Core Entities
1. Users
2. Rides
3. Booking
4. Amusement Park Branches
5. Time Slot

&nbsp;
---
&nbsp;


## 3. APIs
#### 1. [POST] `/rides/book`
- **Description:** Booking Rides
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
    ```json
    {
        "rideId": 123,
        "cnt": 5,
        "startTime": "2025-06-23T07:30:41Z",
        "endTime": "2025-06-23T08:00:41Z"
    }
    ```
- **Response:**
    ```json
    {
        "bookingId": 12345, // All-or-Nothing
        "bookingTime": "2025-06-23T05:30:41Z"
    }
    ```

#### 2. [GET] `/rides/book`
- **Description:** Get Ride Booking History
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
    ```
    ?size=5
    &lastReceivedBookingId= // Regarding paging
    &sort=bookingTime
    &order=desc
    ```
- **Response:**
    ```json
    {
        "bookings": [
            {
                "bookingId": 12345,
                "bookingTime": "2025-06-23T05:30:41Z"
                "rideId": 123,
                "cnt": 5,
                "startTime": "2025-06-23T07:30:41Z",
                "endTime": "2025-06-23T08:00:41Z"
            }
        ]
    }
    ```

#### 3. [PUT] `/rides/book/{:bookingId}`
- **Description:** Update Ride Booking Info
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
    ```json
    {
        "cnt": 3,
        "startTime": "2025-06-23T07:30:41Z",
        "endTime": "2025-06-23T08:00:41Z"
    }
    ```
- **Response:**
    ```json
    {
        "success": true // All-or-Nothing
    }
    ```

#### 4. [DELETE] `/rides/book/{:bookingId}`
- **Description:** Delete Ride Booking
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
- **Response:**
    ```json
    {
        "success": true // All-or-Nothing
    }
    ```

#### 5. [GET] `/rides/code/{:bookingId}`
- **Description:** Get Ride Entrance Code
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
- **Response:**
    ```json
    {
        "s3PresignedUrl": "https://fastpass.s3.amazonaws.com/..."
    }
    ```

#### 6. [PATCH] `/rides/book/{:bookingId}/checkin`
- **Description:** Checkin Rides
- **Headers:** Authorization: Bearer <accessToken> (Case member)
- **Request:**
    ```json
    {
        "status": "checked-in",
        "checkInTime": "2025-06-23T07:35:41Z"
    }
    ```
- **Response:**
    ```json
    {
        "success": true // All-or-Nothing
    }
    ```
    

&nbsp;
---
&nbsp;

## 4. Data Flow
1. Booking Service
    - Handles CRUD operations for booking information. Before booking, it checks availability count from Redis, and after booking, it updates the availability count in Redis.
2. QR Code Service
    - Triggered by a cron job at the start of each time slot. It generates QR code images, uploads them to S3, and saves the S3 presigned URLs in Redis. It also activates the Notification Service by sending messages to the message broker.
3. Notification Service
    - Sends push notifications through APNS and FCM to notify users that their reserved time is approaching.
4. S3 Presigned URL
    - When the client requests the presigned URL to get the QR code, the QR Code Service retrieves it from Redis and returns it. The client then uses the presigned URL to download the QR image directly from S3.
5. Booking Service
    - When a Cast Member scans the QR code, the Booking Service updates the booking status in both Redis and the database.
    

&nbsp;
---
&nbsp;

## 5. Deep Dive

## Database (RDBMS)
#### 1. Users
- user_id (PK)
- password
- phone_number
- created_at

#### 2. Rides
- ride_id (PK)
- branch_id (FK)
- ride_name
- open_at
- close_at

#### 3. Booking
- booking_id (PK)
- ride_id (FK)
- time_slot_id (FK)
- booking_time
- user_id (FK)
- count
- status // ['BOOKING', 'USED', 'CANCELED', 'NO-SHOW']

#### 4. Amusement Park Branches
- branch_id (PK)
- branch_name
- region
- open_at
- close_at

#### 5. Time Slot
- time_slot_id (PK)
- start_time
- end_time

---

## Redis
#### 1. Ride Availability Count
- key) ride:availability:{rideId}:{timeSlotId}
- value) {Availability Count}

#### 2. S3PresignedUrl
- key) code:presignedurl:{bookingId}
- value) {S3 Presigned Url}
- ttl) 10 mins

#### 3. Ride Checkin Status (used to handle no-shows; saved when the reserved time is approaching)
- key) ride:checkin:status:{bookingId}
- value) {status} // ['READY'] (when used, canceled, or no-show, the key is deleted)
- ttl) {until the time slot ends}