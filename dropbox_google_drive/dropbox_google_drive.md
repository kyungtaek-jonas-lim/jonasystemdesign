# Dropbox (/Google Drive)
Dropbox is a cloud-based file storage and synchronization service. It allows users to upload, store, and manage files in the cloud, and access them from multiple devices such as laptops, smartphones, or tablets.


---

## Index

1. [Requirements](#1-requirements)
2. [Core Entities](#2-core-entities)
3. [APIs](#3-apis)
4. [High-level Design](#system-design-diagram-high-level-design)
5. [Data Flow](#4-data-flow)
6. [Deep Dive](#5-deep-dive)
7. [Additional Info](#6-additional-info)

---

&nbsp;

## System Design Diagram (High Level Design)
![System Design Diagram](https://raw.githubusercontent.com/kyungtaek-jonas-lim/jonasystemdesign/main/dropbox_google_drive/dropbox_google_drive.png)

&nbsp;
---
&nbsp;

## 1. Requirements
### Functional Requirement:
1. Users should be able to upload files
2. Users should be able to download files
3. Files should be synchronous automatically across devices (Eventually)

### Non-functional Requirements:
1. Availabilty >> Consistency
2. High data integrity
3. Low latency upload & download
4. Support large files (50GB)
    - resumable uploads (sync accuracy)

### Out of Scope:
- roll own blob storage
: "Roll your own blob storage" means building and managing your own system to store large binary files (like videos, images, or documents), instead of using a cloud service like Amazon S3.


&nbsp;
---
&nbsp;


## 2. Core Entities
1. Files
2. File Metadata
3. Users


&nbsp;
---
&nbsp;

## 3. APIs
- `user_id` is extracted from the JWT in the request header. 

1. [POST] files/upload
- Request:
```json
{
    "files": [
        {
            "filepath": "/jonas/jonas.md",
            "description": "About Jonas",
            "chunkCount": 3 // Specify the chunk counts on the client side depending on the file size
        }
    ]
}
```
- Response:
```json
{
    "files": [
        {
            "file_id": "01FZHB6X9YV1XW42JZ5K9RZN7V", // ULID
            "filepath": "/jonas/jonas.md",
            "uploadId": "b3f4a2d8f2e649b9bc7d5b82f1d71d1a",
            "partUrls": [
                {
                    "partNumber": 1,
                    "url": "https://s3.aws.com/...&partNumber=1&uploadId=..."
                },
                {
                    "partNumber": 2,
                    "url": "https://s3.aws.com/...&partNumber=2&uploadId=..."
                },
                {
                    "partNumber": 3,
                    "url": "https://s3.aws.com/...&partNumber=3&uploadId=..."
                }
            ],
            "completeUrl": "https://s3.aws.com/.../filepath?uploadId=..."
        }
    ]
}
```

2. [POST] files/download
- Request:
```json
{
    "file_ids": [
        "01FZHB6X9YV1XW42JZ5K9RZN7V"
    ]
}
```
- Response:
```json
{
    "files": [
        {
            "file_id": "01FZHB6X9YV1XW42JZ5K9RZN7V",
            "presignedUrl": "https://s3.aws.com/upload/jonaslim"
        }
    ]
}
```

3. [GET] files/updates
- Request:
```
?path=/jonas
&since=2025-04-21T12:50:00Z
// optional: &status=moved
// optional: &page=1
// optional: &page_size=20
```
- Response:
```json
{
    "files": [
        {
            "file_id": "01FZHB6X9YV1XW42JZ5K9RZN7V",
            "status": "modified",
            "presignedUrl": "https://s3.aws.com/download/jonas.md"
            "updated": "2025-04-18T12:50:00Z"
        },
        {
            "file_id": "01FZHB6X9YV1XW42JZ5K9RD4S",
            "status": "deleted", // No need for a presignedUrl
            "updated": "2025-04-18T12:50:00Z" // Deletion timestamp in 'updated'
        },
        {
            "file_id": "01FZHB6X9YV1XW42JZ5K9RZS4W",
            "status": "moved",
            "filepath": {
                "old": "/jonas/before.pdf",
                "new": "/jonas/after.pdf"
            },
            "updated": "2025-04-18T12:50:00Z"
        },
        {
            "file_id": "01FZHB6X9YV1XW42JZ5K9RZQ2Q",
            "status": "created",
            "presignedUrl": "https://s3.aws.com/download/created.docx"
            "updated": "2025-04-18T10:50:00Z"
        }
    ]
}
```

&nbsp;
---
&nbsp;

## 4. Data Flow

### 1. Upload

1. **Get Presigned URLs for Multipart Upload**
    - Client → API Gateway → File Service → Blob Storage (S3)
    - File Service calls `createMultipartUpload` on S3 and stores basic file metadata in the database (e.g., `filepath` is `null` for now)
    - File Service returns the `uploadId`, presigned part URLs, and a presigned URL for completing the upload

2. **Upload File Parts**
    - Client → Blob Storage (S3) using presigned part URLs
    - File is uploaded in multiple parts (multipart upload)

3. **Complete Multipart Upload**
    - Client → Blob Storage (S3) using the presigned `completeMultipartUpload` URL
    - Client provides the `uploadId`, `partNumbers`, and `ETags` to finalize the upload

4. **Update File Metadata**
    - S3 Event Notification → File Metadata Service
    - Metadata service updates `created_datetime`, `file_path`, and `size` in the database

---

### 2. Download

1. **Get Presigned Download URL**
    - Client → API Gateway → File Service
    - File Service fetches file metadata from the database and returns a presigned download URL to the client

2. **Download File**
    - Client → Blob Storage (S3) using the presigned URL
    - File is downloaded in chunks (client may control chunk size using the Range header)

---

### 3. Sync

1. **Get Changes from Server**
    - Triggered:
        - When the app starts
        - When the user opens a file
        - Every 10 minutes
    - Client → API Gateway → Sync Service → Database → File Service → Blob Storage
    - Sync Service checks `File History` table for modified/deleted/moved files
    - File Service returns presigned URLs for updated files
    - To support partial file updates efficiently, block-level synchronization (e.g., chunk-based diffing) may be applied.

2. **Update Files**
    - Client uploads changed files to Blob Storage using new presigned URLs
    - After uploading all parts, client calls the `completeMultipartUpload` presigned URL
    - If files are marked as deleted or moved, the client applies these changes to local storage (e.g., file explorer)

&nbsp;
---
&nbsp;

## 5. Deep Dive
### Database
#### Files
- file_id // ULID
- filepath
- mimeType // pdf, png, ...
- size
- description
- owner_id (FK)
- s3_path // nullable (null til you get the s3 notification after uploading the file // the bucket and object key (e.g., 'my-bucket/user-uploads/jonas/profile-picture.jpg')
- created_datetime
- updated_datetime
- is_deleted

---

#### File History
- history_id // ULID
- file_id (FK)
- status // modified, deleted, created, moved
- user_id (FK)
- content // JSON (e.g., {"version": {"old": <s3_old_version_id>, "new": <s3_new_version_id>}, "path": {"old": "/jonas_old/jonas.md", "new": "/jonas/jonas.md"}}
- created_datetime


&nbsp;
---
&nbsp;

## 6. Additional Info

### ✅ 1.How Google Drive & Dropbox Handle Files
**"Do Google Drive and Dropbox download all files to the user's device, or do they just show file metadata and only download the file when the user opens it? And what happens after the user finishes editing?"**

> **They usually show only metadata first** and **download the full file only when the user opens or requests it.**

---

#### 🧾 1. **File Metadata First (Lazy Downloading)**

- When you open the app or web interface, it shows:
  - File name
  - Size
  - Modified time
  - Thumbnail/preview
- But the **actual file content is not downloaded yet**.
- This saves bandwidth and speeds up loading.

---

#### ⬇️ 2. **Download On-Demand**

- When the user clicks or opens a file:
  - The app **downloads it in real-time**.
  - It might **cache** it locally depending on the platform (desktop vs mobile vs web).
- Some desktop apps (like Dropbox with Smart Sync or Google Drive File Stream):
  - Show files as if they exist locally.
  - Only download them **when you open or edit them.**

---

#### ✍️ 3. **What Happens After Editing?**

After the user finishes editing:

- The client app will:
  1. **Detect file changes** (using file system events or file hashes).
  2. **Upload only the changed parts** (if supported — called *delta sync*).
  3. **Update the metadata** (modified time, version, etc.).
- In Google Drive (Docs, Sheets, etc.), changes are often saved **in real-time**.

---

#### 🧠 Summary:

| Step | Dropbox / Google Drive Behavior |
|------|---------------------------------|
| Show files | ✅ Load only metadata |
| Open file | ✅ Download on demand |
| After editing | ✅ Upload changes back to the cloud |

---

### ✅ 2. Behavior by Platform after editing

**So after editing, does the client app delete the downloaded file that was used for editing?**

> **It depends on the platform and settings**, but usually:  
> **No — the client app does not immediately delete the file.**  
> It often **keeps it temporarily cached**, and may delete it **later** based on space, user preferences, or sync rules.

---

#### 🖥️ **Desktop apps (Dropbox, Google Drive File Stream, OneDrive, etc.):**
- Files are downloaded to a **local cache or virtual drive**.
- After editing:
  - ✅ Changes are synced to the cloud.
  - 🕒 The file **stays on the device** unless:
    - You manually set it to "Online-only".
    - The system needs to free up space (e.g., Smart Sync in Dropbox).
- This makes opening the file again much faster.

#### 🌐 **Web apps (Google Drive in browser):**
- The file is usually edited **in the browser or via temporary download**.
- If downloaded manually:
  - It stays in your browser’s Downloads folder unless you delete it.
- If edited via Google Docs:
  - No local copy is saved — it's all in the cloud.

#### 📱 **Mobile apps (Drive, Dropbox, etc.):**
- Files are downloaded only when opened.
- They may be cached temporarily.
- The app may clear them automatically if storage is low.

---

#### 🧠 Summary:

| Platform | Keeps File After Edit? | Deletes It Automatically? |
|----------|------------------------|----------------------------|
| Desktop | ✅ Usually keeps it | ❌ Not immediately |
| Web     | ❌ Usually not saved | ✅ Unless downloaded manually |
| Mobile  | 🕒 Cached temporarily | ✅ Often cleared automatically |



---

### ✅ 3. Should You Include File Size or Mimetype in Presigned URLs?

You can include things like file size limits, mimetype, and expiration when generating a presigned URL, but it’s optional. Most services don’t include them because S3 already handles basic limits, and validations are typically done after upload using S3 event notifications or `headObject`. So yes, it’s possible—but not commonly used.


---
### ✅ 4. Block-Level Synchronization (/Delta Sync or Differential Synchronization)

To update only a specific part of a modified file, you need to use **block-level synchronization**.

`Block-level synchronization` (also known as chunk-based sync or delta sync) is a method that divides a file into smaller, fixed-size blocks (e.g., 4MB each).  
Instead of syncing the entire file when a change occurs, only the modified blocks are uploaded or downloaded.
This technique is also commonly referred to as `delta sync` or `differential synchronization`.

---

#### ⚙️ How does it work?

1. The file is split into multiple blocks on the client side.
2. Each block is hashed (e.g., using SHA-256).
3. The client compares the current block hashes with the previous state or server version.
4. Only blocks with different hashes are uploaded or downloaded.
5. The file is reassembled on the server or client using the updated blocks.

---

#### 🚀 Benefits

- Efficient sync for large files (e.g., videos, design files, documents)
- Reduces upload/download bandwidth
- Improves sync speed and user experience

---

#### 🧠 Use Cases

- Cloud storage services (e.g., Dropbox, Google Drive)
- Document collaboration platforms
- Versioned binary file management



---



### ✅ 5. Is Kafka + Cursor More Effective for Sync?

Using **Kafka with cursors** can be powerful for large-scale, real-time file sync systems — but it's not always necessary.

---

#### When Kafka is Effective

- When you need fast and accurate sync across multiple devices
- When you're handling **hundreds of thousands of users or files**
- When **real-time streaming** is required

Kafka can stream file change events, and **cursors** (or offsets) help each client track where they left off.

A common pattern is to **partition Kafka topics by `userId`**, so that each user's file changes are isolated and scalable.

---

#### But... it Adds Complexity

- Kafka requires managing brokers, partitions, replication, and failover
- You need to store and manage offsets (e.g., per user or device)
- It can be overkill for small or mid-size systems

---

#### When You Might Not Need Kafka

- If sync can happen **every few minutes**
- If your DB tracks changes with `updated_at` timestamps
- If **polling from a REST API** is fast enough for your use case

---

### Architecture Overview

#### Kafka-based Sync (Large Scale)

```
Client ─▶ API Gateway ─▶ Sync Service ─▶ Kafka (partitioned by userId)
                                              │
                                    File History Service
                                              │
                                         S3 / DB
```

- Kafka holds all file change events
- Each user/device reads from their own partition using a cursor (offset)
- Highly scalable & low-latency, but complex

---

#### Polling-based Sync (Simple System)

```
Client ─▶ API Gateway ─▶ Sync API ─▶ DB (e.g., FileHistory with updated_at)
                                    │
                                  S3
```

- Client calls `/files/updates?since=...`
- Backend queries updated files from DB
- Easier to build and maintain

---

#### 🧭 Conclusion

Kafka + Cursor works great for large, high-scale sync platforms,  
but for simpler systems, using a database with polling may be much easier and totally sufficient.