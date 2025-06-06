
# Tinder (**🛠️ In Progress**)
Tinder is a large-scale, location-based social matching platform that connects users through profile swiping and mutual interest. The system must support millions of daily active users, real-time matching, geographically-aware recommendations, and secure, scalable media storage.


---

## Index

1. [Requirements](#1-requirements)
2. [Core Entities](#2-core-entities)
3. [APIs](#3-apis)
<!-- 4. [High-level Design](#system-design-diagram-high-level-design) -->
<!-- 5. [Data Flow](#4-data-flow) -->
<!-- 6. [Deep Dive](#5-deep-dive) -->
<!-- 7. [Additional Info](#6-additional-info) -->
<!-- 8. [Reference](#7-reference) -->

<!-- ---

&nbsp;

## System Design Diagram (High Level Design)
![System Design Diagram](https://raw.githubusercontent.com/kyungtaek-jonas-lim/jonasystemdesign/main/tinder/tinder.png) -->


&nbsp;
---
&nbsp;

## 1. Requirements

&nbsp;
---
&nbsp;


## 2. Core Entities
1. Profile
2. Swipe
3. Match

&nbsp;
---
&nbsp;


## 3. APIs
### 1. [POST] `/auth/request-otp`
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

### 2. [POST] `/auth/login/phone`
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
        "userId": "abc123"
    }
    ```

3. [POST] `/auth/login/social`
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
        "userId": "abc123"
    }
    ```

4. [POST] `/profiles/me/info`
- **Description:** Profile Setup
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
    ```json
    {
        "name": "Jonas",
        "birthdate": "2025-01-01",
        "gender": "male",
        "interests": ["exercising", "food"],
        "bio": "Just a software engineer in LA."
    }
    ```
- **Response:**
    ```json
    {
        "success": true
    }
    ```

5. [POST] `/profiles/me/photo`
- **Description:** Upload profile photos (When the profile photo upload view appears, or user clicks "Upload photo")
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
- **Response:**
    ```json
    {
        "uploadUrl": "https://s3.amazonaws.com/..." // Client uses uploadUrl to upload the image via `PUT`.
    }
    ```

6. [PUT] `/profiles/me/photo`
- **Description:** Notify the completion of the profile photo upload
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
- **Response:**
    ```json
    {
        "success": true
    }
    ```

7. [GET] `/profiles/me/info`
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

8. [GET] `/profiles/me/photo`
- **Description:** Fetch the user's profile photo
- **Headers:** Authorization: Bearer <accessToken>
- **Request:**
- **Response:**
    ```json
    {
        "photoUrl": "https://s3.amazonaws.com/..." // s3 presigned url
    }
    ```


&nbsp;
---
&nbsp;

## Database
### users
- user_id (PK)
- auth_provider // ['PHONE', 'GOOGLE', ...]
- provider_user_id // Phone number or social account ID if it's social login
- created_at


## profiles
- *// profile_id // put this as a PK if each user can have more than one profile.*
- user_id (PK) // Each user has exactly one profile.
- photo_url // File Storage Path
- username
- gender
- bio

## Redis
### OTP
- key: `auth::phone::otp`
- value: (string) otp
- ttl: 3 mins

### Profile Photo upload S3 path
- key: `upload_temp::user_id`
- value: (string) {s3_object_key} (e.g., users/abc123/profile_tmp_94234.jpg)
- ttl: 10 mins