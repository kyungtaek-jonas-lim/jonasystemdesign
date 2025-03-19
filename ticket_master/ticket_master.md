# Ticketmaster

You are tasked with designing a large-scale online ticketing system similar to Ticketmaster, which sells tickets for concerts, sports events, theaters, and more. The system must handle millions of concurrent users, especially during high-demand events where tickets can sell out in seconds.

---

# Orders

1. [Requirements](#requirements)
2. [Core Entities](#core-entities)
3. [APIs](#apis)
4. [High-level Design](#high-level-design)
5. Data Flow
6. [Deep Dive](#deep-dive)

---

# Requirements

## Core Requirements:
1. High Traffic Handling:
    - The system should support millions of users searching, reserving, and purchasing tickets simultaneously.

2. Prevent Overselling:
    - Ensure the same ticket is never sold to more than one user.

3. Seat Selection:
    - For reserved seating events, users must be able to choose specific seats from a seating chart.

4. Reservation & Timeout:
    - When a user adds tickets to their cart, the tickets should be temporarily reserved for a set period. If the user doesn't complete the purchase, the tickets should automatically be released.

5. Payment Processing:
    - Integrate with reliable payment gateways to handle transactions securely.

6. Scalability & High Availablity:
    - The system should scale seamlessly with growing demand and remain available with minimal downtime, even during failures.

7. Event Notifications & Queueing:
    - Implement a queueing system for sold-out events or notify users if tickets become available again.


## Additional Considerations:
1. Database Design:
    - Describe how you would model key entities such as events, tickets, seats, users, reservations, and payments.
    - Discuss how you will ensure transaction consistency (e.g., ACID vs. BASE principles)

2. Caching & Concurrency Control:
    - Explain how in-memory caches (like Redis) will be used to optimize performance.
    - Discuss how to handle concurrency issues using distributed locks (e.g., Redis Redlock).

3. Traffic Management:
    - How would you prevent server overload during high-demand ticket releases? Consider implementing queue systems or rate limiting.

4. Payment Failure & Rollbacks:
    - Describe how to handle situations where a payment fails after a ticket has been reserved, ensuring the ticket is properly released.

5. Notification System:
    - Outline how you would implement notification features via email, SMS, or push notifications.
    - Describe how to send real-time updates to users waiting in line (e.g., Websocket, Server-Sent Events).


---


# Core Entities
- Event
- Venue
- Ticket
- User
- Payment
- Reservation


---

# APIs

- Get user information from JWTs in cookies or headers.
- The tickets are assigned by seat.
- In the frontend, the payment service is integrated to complete the payment, and the backend further verifies the payment information with the payment service.

---

### 1. [GET] /events/types
- **Request:** 
- **Response:**

```json
{
    "event_types": [
        {
            "type": "concert", 
            "description": "Music concerts"
        },
        {
            "type": "sports",
            "description": "Sporting events"
        }
    ]
}
```

---

### 2. [GET] /events
- **Request:** 
    ```
    ?type={event_type}
    &location={location}
    &date_from={date_from}
    &date_to={date_to}
    &sort_by=date  // date, event_name, or location
    &order=asc  // asc or desc
    &page=1
    &size=20
    ```
- **Response:**

```json
{
    "events": [
        {
            "venue": "",
            "performer": "",
            "detail": "",
            "datetime": "2025-05-18T12:16:20.333Z",
            "total_ticket_cnt": 50000,
            "available_ticket_cnt": 400
        }
    ]
}
```

---

### 3. [GET] /events/:event_id
- **Request:**
- **Response:**

```json
{
    "venue": "",
    "performer": "",
    "detail": "",
    "datetime": "2025-05-18T12:16:20.333Z",
    "total_ticket_cnt": 50000,
    "available_ticket_cnt": 400,
    "tickets": [
        {
            "id": "45601",
            "seat": "",
            "class": "",
            "available": false,
            "price": "3000",
            "unit": "USD"
        }
    ]
}
```

---

### 4. [POST] /events/ticket/reserve
- **Request:**

```json
{
    "ticket_ids": [
        "12345",
        "23456"
    ]
}
```

- **Response:**

```json
{
    "expire_datetime": "2025-03-19T16:00:00Z",
    "expire_in_seconds": 1800 // The number set in Redis
}
```

---

### 5. [POST] /events/ticket/confirm
- **Request:**

```json
{
    "ticket_ids": [
        "12345",
        "23456"
    ],
    "payment_info": {  // Information from the frontend payment service, to be verified on the backend by calling the payment service API
        "payment_id": "txn_12345",
        "payment_method": "credit_card",
        "amount_paid": 100,
        "currency": "USD",
        "transaction_id": "abc12345",
        "payment_status": "approved"
    }
}
```

- **Response:**

```json
{
    "status": "success", // "success", "failure", or "error"
    "message": "Payment confirmed successfully",
    "confirmed_tickets": [
        {
            "ticket_id": "12345",
            "seat": "A1",
            "class": "VIP",
            "price": "100",
            "currency": "USD",
            "status": "confirmed"
        },
        {
            "ticket_id": "23456",
            "seat": "A2",
            "class": "VIP",
            "price": "100",
            "currency": "USD",
            "status": "confirmed"
        }
    ],
    "payment_details": {
        "payment_id": "txn_12345",
        "payment_method": "credit_card",
        "amount_paid": 100,
        "currency": "USD",
        "transaction_id": "abc12345",
        "payment_status": "approved"
    },
    "confirmation_datetime": "2025-03-19T16:30:00Z"
}
```

---

### 6. [POST] /events/ticket/cancel
- **Request:**

```json
{
    "ticket_ids": [
        "12345",
        "23456"
    ],
    "status": "reserved", // "reserved" or "confirmed"
    "cancel_reason": ""
}
```

- **Response:**

```json
{
    "status": "success", // "success", "failure", or "error"
    "message": "Tickets successfully cancelled",
    "cancelled_tickets": [
        {
            "ticket_id": "12345",
            "seat": "A1",
            "class": "VIP",
            "status": "cancelled",
            "reason": "User requested cancellation"
        },
        {
            "ticket_id": "23456",
            "seat": "A2",
            "class": "VIP",
            "status": "cancelled",
            "reason": "User requested cancellation"
        }
    ],
    "cancel_datetime": "2025-03-19T16:30:00Z"
}
```


---

# High-level Design

---

# Deep Dive

## Database

### Event
- event_id (PK)
- title
- datetime
- type_id (FK)
- venue_id (FK)
- location (Not necessary but for better performance)
- description
- total_ticket_count
- image_path (From file storage like S3)
- create_datetime

### Event Type
- type_id (PK)
- type
- create_datetime

### Venue
- venue_id (PK)
- name
- location
- capacity
- address
- phone
- create_datetime

### Ticket
- ticket_id (PK)
- event_id (FK)
- seat
- class_id (FK)
- currency
- price
- status ("available", "reserved", "confirmed", "expired")
- status_update_datetime
- user_id (FK, nullable)
- payment_id (FK, nullable)
- create_datetime

### User
- user_id (PK)
- username
- email
- phone
- address
- create_datetime

### Payment
- payment_id (PK)
- user_id (FK)
- method
- currency
- price
- status
- payment_datetime
- transaction_id (From the payment service)
- create_datetime

### Performer
- performer_id (PK)
- name (Unique)
- create_datetime

### Event Performer
- event_id (PK#1, FK)
- performer_id (PK#2, FK)
- create_datetime

### Class
- class_id (PK)
- name
- description
- create_datetime

---

## Redis

### Reserved Tickets with TTL
- **key:** `reserved_ticket:ticket_{ticket_id}`
- **value:**
    - `user_id`
    - `expire_datetime`
    - `status` ("reserved", "expired")

### Available Ticket Count (Not necessary but better performance)
- **key:** `ticket_count:event_{event_id}`
- **value:** `{available count}`