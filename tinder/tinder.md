
# Tinder
Tinder is a large-scale, location-based social matching platform that connects users through profile swiping and mutual interest. The system must support millions of daily active users, real-time matching, geographically-aware recommendations, and secure, scalable media storage.


---

## Index

1. [Requirements](#1-requirements)
2. [Core Entities](#2-core-entities)
3. [APIs](#3-apis)
4. [High-level Design](#system-design-diagram-high-level-design)
5. [Data Flow](#4-data-flow)
6. [Deep Dive](#5-deep-dive)
7. [Additional Info](#6-additional-info)
<!-- 8. [Reference](#7-reference) -->

---

&nbsp;

## System Design Diagram (High Level Design)
![System Design Diagram](https://raw.githubusercontent.com/kyungtaek-jonas-lim/jonasystemdesign/main/tinder/tinder.png)


&nbsp;
---
&nbsp;

## 1. Requirements
### Functional Requirements:
1. Users should be able to log in using their phone number via OTP (One-Time Password).
2. Users should be able to log in with social accounts (e.g., Google, Apple).
3. Users should be able to create and update their profile, including name, gender, bio, interests, etc.
4. Each user should have exactly one profile linked to their account.
5. Users should be able to receive a stack of nearby profiles based on location, preferences, and swipe history.
6. Users should be able to swipe left (dislike) or right (like) on other user profiles.
7. Users should be notified of new matches via push notifications.
8. Users should be able to share and update their current location to enable proximity-based recommendations.

### Non-functional Requirements:
1. The system should be able to serve millions of daily active users and scale horizontally under high load.
2. The system should target API response times under 300ms (95th percentile) for core operations.
3. Real-time interactions (e.g., swipes, match notifications) should be processed with low latency.
4. The system should scale to handle a high write throughput, e.g., 10 million users × 100 swipes per day = 1B swipe events/day.

### Out of Scope:
1. Users should be able to chat in real time with matched users.
2. Users should be notified of messages via push notifications.
3. Services should be loosely coupled using event-driven architecture (e.g., Kafka) and independently deployable.


&nbsp;
---
&nbsp;


## 2. Core Entities
1. Profile
2. Swipe
3. Match
4. User
5. Message

&nbsp;
---
&nbsp;


## 3. APIs
#### 1. [POST] `/auth/request-otp`
- **Description:** Phone number verification request
- **Request:**
    ```json
    {
        "phone": "+1-1012345678"
    }
    ```
- **Response:**
    ```json
    {
        "success": true
    }
    ```

#### 2. [POST] `/auth/login/phone`
- **Description:** Phone Login(/Sign-Up) (Phone number verification)
- **Request:**
    ```json
    {
        "phone": "+1-1012345678",
        "otp": "123456"
    }
    ```
- **Response:**
    ```json
    {
        "accessToken": "JWT...",
        "refreshToken": "JWT...",
        "userId": "4f91b58a-3ae3-432c-b0c6-c1f179f2d273"
    }
    ```

#### 3. [POST] `/auth/login/social`
- **Description:** Social Login
- **Request:**
    ```json
    {
    "provider": "GOOGLE",
    "id_token": "eyJhbGciOi..." // Servers extract `provider_user_id` from it, specifically `sub` claim
    }
    ```
- **Response:**
    ```json
    {
        "accessToken": "JWT...",
        "refreshToken": "JWT...",
        "userId": "4f91b58a-3ae3-432c-b0c6-c1f179f2d273"
    }
    ```

#### 4. [POST] `/profiles/me/info`
- **Description:** Profile Setup
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
    ```json
    {
        "name": "Jonas",
        "birthdate": "2025-01-01",
        "gender": "male",
        "interests": ["exercising", "food"],
        "bio": "Just a software engineer in LA.",
        "minAge": 25,
        "maxAge": 35,
        "radius": 20, // How far it can get from the user
        "showMe": "", // ['male', 'female', 'everyone']
        "isDiscoverable": false // 	Whether to show this user in others' swipe stacks
    }
    ```
- **Response:**
    ```json
    {
        "success": true
    }
    ```

#### 5. [POST] `/profiles/me/photo`
- **Description:** Upload profile photos (When the profile photo upload view appears, or user clicks "Upload photo")
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
- **Response:**
    ```json
    {
        "uploadUrl": "https://s3.amazonaws.com/..." // Client uses uploadUrl to upload the image via `PUT`.
    }
    ```

#### 6. [PUT] `/profiles/me/photo`
- **Description:** Notify the completion of the profile photo upload
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
- **Response:**
    ```json
    {
        "success": true
    }
    ```

#### 7. [GET] `/profiles/me/info`
- **Description:** Fetch the user's profile information
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
- **Response:**
    ```json
    {
        "name": "Jonas",
        "birthdate": "2025-01-01",
        "gender": "male",
        "interests": ["exercising", "food"],
        "bio": "Just a software engineer in LA."
    }
    ```

#### 8. [GET] `/profiles/me/photos`
- **Description:** Fetch the user's profile photos
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
- **Response:**
    ```json
    {
        "photoUrls": [
            {   
                "index": 1,
                "photoUrl": "https://s3.amazonaws.com/..." // s3 presigned url
            },
            {   
                "index": 2,
                "photoUrl": "https://s3.amazonaws.com/..." // s3 presigned url
            }
        ]
    }
    ```

#### 9. [GET] `/profiles`
- **Description:** Fetch profile stacks to swipe (Profiles photos, )
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
    ```
    ?lat=33.333
    &lng=33.333
    &limit=10
    &cursor=4f91b58a-3ae3-432c-b0c6-c1f179f2d333 // last item key (userId) in the previous page
    ```
- **Response:**
    ```json
    {
        "profiles": [
            {
                "userId": "4f91b58a-3ae3-432c-b0c6-c1f179f2d273",
                "name": "Jonas",
                "birthdate": "2025-01-01",
                "gender": "male",
                "interests": ["exercising", "food"],
                "bio": "Just a software engineer in LA."
            }
        ]
    }
    ```

#### 10. [GET] `/profiles/{userId}/photos`
- **Description:** Fetch other users' profile photos
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
- **Response:**
    ```json
    {
        "photoUrls": [
            {   
                "index": 1,
                "photoUrl": "https://s3.amazonaws.com/..." // s3 presigned url
            },
            {   
                "index": 2,
                "photoUrl": "https://s3.amazonaws.com/..." // s3 presigned url
            }
        ]
    }
    ```

#### 11. [POST] `/swipe/{userId}`
- **Description:** Fetch other users' profile photos
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
    ```json
    {
        "like": true
    }
    ```
- **Response:**
    ```json
    {
        "success": true,
        "matched": true,
        "chatId": "9e91b58a-4cc4-432c-d0e2-d9f179f2d273"
    }
    ```
    

&nbsp;
---
&nbsp;

## 4. Data Flow
    

&nbsp;
---
&nbsp;

## 5. Deep Dive

## Database (RDBMS)
#### users
- user_id (PK)
- auth_provider // ['PHONE', 'GOOGLE', ...]
- provider_user_id // Phone number or social account ID if it's social login
- created_at


#### profiles
- *// profile_id // put this as a PK if each user can have more than one profile.*
- user_id (PK) // Each user has exactly one profile.
- photo_url // File Storage Path
- username
- gender
- bio
- minAge
- maxAge
- radius
- showMe // ['male', 'female', 'everyone']
- is_discoverable // Whether to show this user in others' swipe stacks


#### matches
- user_id1 (PK)
- user_id2 (PK)
- created_at
cf>
Store user_id1 and user_id2 in a sorted (ascending) order.
Always use user_id1 as the query base.
This way, a single-direction table is sufficient to support bidirectional lookup.



## Cassandra (NoSQL)
### swipes
- user_id1 (PK: Partition Key)
- user_id2 (PK: Clustering Key)
- like
- created_at

---

## Redis
#### OTP
- key: `auth::phone::otp`
- value: (string) otp
- ttl: 3 mins

#### Profile Photo upload S3 path
- key: `upload_temp::user_id`
- value: (string) {s3_object_key} (e.g., users/4f91b58a-3ae3-432c-b0c6-c1f179f2d273/profile_tmp_94234.jpg)
- ttl: 10 mins

#### Swiped Users
- key: `user1:user2`
- value: (string) like / dislike


&nbsp;
---
&nbsp;

## 6. Additional Info
1. PostgreSQL with PostGIS is also a good alternative to using Elasticsearch (for geo-spatial queries) in Stack Profiles.
2. Using Redis for swipes is unnecessary if you're using an RDBMS that supports row-level locking (e.g., PostgreSQL), unless you're handling very high traffic where in-memory performance is required.