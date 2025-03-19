# Short URL System Design

&nbsp;

## Short URL System Design Diagram
![Short URL System Design Diagram](https://raw.githubusercontent.com/kyungtaek-jonas-lim/jonasystemdesign/main//short_url/short_url_diagram.png)

&nbsp;

## Methods of Shortening URLs
- [Methods of Shortening URLs](https://github.com/kyungtaek-jonas-lim/jonasystemdesign/blob/main//short_url/shorten_url_methods_en.md) ([en](https://github.com/kyungtaek-jonas-lim/jonasystemdesign/blob/main//short_url/shorten_url_methods_en.md) / [ko](https://github.com/kyungtaek-jonas-lim/jonasystemdesign/blob/main//short_url/shorten_url_methods_ko.md))

&nbsp;

## Orders
1. [Requirements](#requirements)
2. [Core Entities](#core-entities)
3. [API](#apis)
4. [High-level Design](#short-url-system-design-diagram)
5. Data Flow
6. Deep Dive

&nbsp;

## Requirements

### Functional Requirements
1. Generate a short URL from a long URL
    - Optionally, use alias
    - Optionally, use expiration time
2. Redirect to the original (long) URL

### Nonfunctional Requirements
1. Low latency on redirects (~200ms)
2. Consistency & High Availability
3. Uniqueness of short URLs
4. Scale to support 100M DAU and 1B URLs

&nbsp;

## Core Entities
- User ID
- Original (Long) URLs
- Short URLs
- Click Counts (Optional)
- Expiration Date (Optional)
- Creation Time (Optional)

&nbsp;

## APIs

1. **[POST] /urls/shorten**
    - **Request:**
      ```json
      {
          "originalUrl": "",
          // "alias": "",
          // "expirationDate": "2025-02-04T21:29:30.333Z"
      }
      ```
    - **Response:**
      ```json
      {
          "shortUrl": ""
      }
      ```
    - **Explanation:**
      - JWT in Cookies or Headers contains user info (UserID)

2. **[GET] {shortUrl}**
    - **Redirect:** 302
    - **Explanation:**
      - Short URLs with expiration dates result in a Temporary Redirect (302)

&nbsp;

## Data Flow

### Short URL Creation
1. Client requests '/urls/shorten'
2. Generate short URLs
    - **Method 1**
        1. Generate sequential ID + random suffix
        2. Base62 Encoding
            - Short URLs will be shorter than 12 characters
    - **Method 2**
        1. Generate ULID
        2. Base62 Encoding
            - Short URLs will be around 22 characters
3. Base62 encoding
4. Write to DB
5. If there's a conflict, regenerate short URLs (409 error for alias)
6. Write sequential IDs to Redis (Method 1)
7. Respond with shortened URLs

### Redirect URLs
1. Client requests with short URLs
2. Query original URLs from Redis
3. Query original URLs from DB if cache miss
4. Return 404 error if the URL is expired or doesn't exist
5. If the userId does not match, return 403 error
6. Cache the URL if it was a cache miss
7. Optionally cache click counts (click counts will be updated to DB regularly)
8. Redirect to the original URL (302 for URLs with expiration date)

&nbsp;

## Deep Dive

### Read/Write Services
- Read will be much more than Write.
- If it's a small URL-shortening service, no need to divide them.

### Security
- Get user information from JWT in cookies or headers. Compare the userId with the userId in database or Redis. For invalid JWT, return 401 error.
- If short URLs are private, return 403 error (for unauthorized users for the URL).
- If short URLs are public, redirect to the original URL.

### Caching
- For short URLs, use lazy loading (TTL shouldn't be later than expiration dates).
- For sequential IDs used for short URLs, use Redis to maintain consistency.
- For click counts, update in Redis first and synchronize with the database regularly (scheduling).

### API Gateway
- Routing (Round Robin)
- Rate Limiting
    - **Java (Spring Cloud)**: Resilience4j
    - **Node.js (Express)**: express-rate-limit
    - **Python (FastAPI)**: slowapi
    - **Nginx**: limit_req_zone
- Client IPs
    - Forward client IPs using headers.
        - **Nginx**: `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`
        - **Applications**: `x-forwarded-for`

### Service Scaling
- Horizontal Scaling
- Auto Scaling

### Redis Scaling
- No need to scale but better to use for efficiency to support 100M DAU and 1B URLs.
- Read/Write Replication

### Database Scaling
- No need to scale but better to use for efficiency to support 100M DAU and 1B URLs.
- Read/Write Replication
- Use Database Proxy

&nbsp;

## Database Tables

### urls
- **Columns:**
    - short_url (PK) (could be alias)
    - original_url
    - user_id
    - expiration_date
    - creation_date (optional)
    - click_cnt (optional)
- **Indexes:**
    - user_id, short_url
    - expiration_date
    - creation_date (optional)

### users
- **Columns:**
    - id (PK)
    - role
    - create_date
    - user_email (unique)
    - ...

&nbsp;

## Redis Keys & Values

### shorturl:url:{Short Urls}
- **Map:**
    - originalUrl
    - userId

### shorturl:sequentialID
- **String:**
    - {sequential ID}

### shorturl:clicks:{Short Urls}
- **String:**
    - {counts}