# Youtube
Youtube is a video-sharing platform that allows users to upload, view, and interact with video content.
This system design only focused on the two most essential features: `uploading` and `watching(viewing)`.


---

## Index

1. [Requirements](#1-requirements)
2. [Core Entities](#2-core-entities)
3. [APIs](#3-apis)
4. [High-level Design](#system-design-diagram-high-level-design)
5. [Data Flow](#4-data-flow)
6. [Deep Dive](#5-deep-dive)
7. [Additional Info](#6-additional-info)
8. [Reference](#7-reference)

---

&nbsp;

## System Design Diagram (High Level Design)
![System Design Diagram](https://raw.githubusercontent.com/kyungtaek-jonas-lim/jonasystemdesign/main/youtube/youtube.png)

&nbsp;
---
&nbsp;

## 1. Requirements
### Functional Requirements:
1. Users should be able to upload videos
2. Users should be able to watch/stream videos

Scale
~1M uploads/day
100M DAU
Max video size of 256GB

### Non-functional Requirements:
1. Availability >> consistency for video uploads
2. Support uploading/streaming for large videos (256GB)
3. Low latency streaming (<500ms), true in low bandwidth
4. Scalability to scale to 1M uploads/day and 100M views

### Out of Scope
- Comments
- Likes
- Search


&nbsp;
---
&nbsp;


## 2. Core Entities
1. Video
2. Video Metadata
3. User


&nbsp;
---
&nbsp;

## 3. APIs
- `user_id` is extracted from the JWT in the request header.  

---

#### 1. [POST] /video
- **Description:** Request to upload a video (returns presigned URL)
- **Request:**  
```json
{
  "title": "My Vacation Vlog",
  "description": "Trip to Jeju Island"
}
```
- **Response:**  
```json
{
  "videoId": "abc123",
  "uploadUrl": "https://s3.amazonaws.com/bucket-name/abc123.mp4?X-Amz-Security-Token=..."
}
```

#### 2. [GET] /video/:videoId
- **Description:** Fetch metadata and streaming info for a video
- **Request:**  
- **Response:**
```json
{
  "title": "My Vacation Vlog",
  "description": "Trip to Jeju Island",
  "thumbnailUrl": "https://cdn.yourdomain.com/thumbnails/abc123.jpg",
  "streamUrl": "https://cdn.yourdomain.com/videos/abc123/master.m3u8"
}
```

&nbsp;
---
&nbsp;

## 4. Data Flow

### 1. **Upload Flow (Presigned URL Approach)**
1. Client sends upload request to `VideoService` via `API Gateway`.
2. `VideoService` generates a presigned URL and sends it back.
3. Client uploads video file directly to S3 using that URL.
4. S3 triggers `Chunker (Lambda)` → then `Transcoder`.

**Pros:**
- Offloads large video traffic away from your service (no bandwidth bottleneck on your backend).
- Cheap and fast; S3 handles direct upload/download efficiently.
- Common practice in production systems.

**Cons:**
- Security risks if presigned URLs are leaked (though they're short-lived).
- Harder to enforce fine-grained access control (e.g. quota, user type).
- Client needs more logic to handle upload/download behavior.

**View Flow (Adaptive Playback):**
1. Client → API Gateway → VideoService
- Fetches metadata (title, thumbnail, master.m3u8 URL)
2. Client → CDN
- Requests master.m3u8 → finds available resolutions
3. Client (Player)
- Chooses the best resolution based on network speed
- Downloads only one segment at a time (adaptive)
4. Client → CDN
- Fetches segment0.ts, segment1.ts, ...
- May switch to different resolution segments (e.g., 360p → 720p)


&nbsp;
---
&nbsp;

## 5. Deep Dive

### Database
#### Video
- id (PK)
- name
- description
- status // [pending | uploaded | chunking | chunks_complete]
- full_s3_url
- chunks_by_resolution[]
    {
        "240p": [], // chunks for each resolution
        "720p": [], ...
    }


&nbsp;
---
&nbsp;

## 6. Additional Info

### ✅ 1. Is it okay to call S3 directly from the client using pre-signed URLs?

**Yes, absolutely — it’s okay and widely used in production**, **as long as**:

- You **generate the pre-signed URL** from your backend after authenticating the user.
- The URL is **short-lived** (e.g., expires in a few minutes).
- You apply **permissions properly** (e.g., only allow `putObject` or `getObject`, and only on the specific key the user is allowed to access).

**Why this is okay:**
- S3 is built to handle large file transfers efficiently.
- Your backend stays lightweight and scalable.
- You still control the process securely through the pre-signing step.

So yes — **it's safe and recommended** as long as you're controlling the URL generation securely.

---

### ✅ 2. What does the "Chunker" do? Is it splitting physical video data?

Excellent observation. Let’s clarify.

**There are two meanings of “chunk”** in the context of video systems:

---

#### ⚙️ (A) System-Level Chunking (physical chunking) (e.g., multipart upload to S3)
- When uploading big files, AWS S3 lets you **split files into binary chunks** (e.g., 5 MB each) for performance and reliability.
- This is just raw data chunks — not related to actual video frames or time segments.
- Example: `ABC...Z` is split into parts: `ABC`, `DEF`, ..., `XYZ` → stitched into one file in S3.

📌 **Your Chunker Lambda does *not* need to do this**, because S3 already supports multipart upload from the client.

---

#### 🎥 (B) Video-Level Chunking (logical chunking) (e.g., for adaptive streaming)
- This is what you mentioned: breaking a video into **2~10 second segments**.
- It’s done **after** transcoding, when preparing for streaming via **[HLS](https://github.com/kyungtaek-jonas-lim/jonastudy/blob/main/concept/http/hls_dash_en.md) (HTTP Live Streaming)** or **[DASH](https://github.com/kyungtaek-jonas-lim/jonastudy/blob/main/concept/http/hls_dash_en.md)**.
- These are **real video segments** — not just data slices — and they include timestamps, keyframes, etc.
- So the 10-second clip starting at time 0 contains actual playable frames (audio + video) from `00:00` to `00:10`.

Example:

```plaintext
video.mp4 → transcoded → segment_0.ts, segment_1.ts, ...
```

Each segment:
- Is a playable video file.
- Contains all necessary encoding headers.
- Can be independently loaded by the video player.

This is how platforms like YouTube or Netflix let you **seek, buffer, and switch resolutions smoothly**.

---

### 🧠 Summary

| Term | What it does | Physical or Logical? | Example Use |
|------|--------------|----------------------|-------------|
| S3 Multipart Upload | Upload big file in binary chunks | Physical | Raw upload from client |
| Video Chunking (Streaming) | Cut video into 2~10s segments for playback | Logical, frame-based | HLS, DASH |

So yes — video-level chunking **is based on real video playback time**, and the segments are playable. It's much more than just raw file splitting.




---

### ✅ 3. How is a video file structured?

A video file (e.g., `.mp4`, `.webm`, `.mov`) contains **multiple data streams**, all packed into a single file format called a **container**.

#### 🧱 Components of a video file:

| Component | Description |
|-----------|-------------|
| **Container** | The file format that holds everything together. Examples: `.mp4`, `.mkv`, `.webm`. |
| **Video Stream** | Encoded frames using a codec like H.264, VP9, AV1, etc. |
| **Audio Stream** | Sound data, often AAC or Opus codec. |
| **Subtitles/Metadata** | Optional: captions, thumbnails, chapters, etc. |
| **Index / Moov Box** | Tells the player where everything is located inside the file (needed for seeking). |

---

#### 📦 Example (MP4 File):

```
[MP4 Container]
 ├── Video Track (H.264)
 ├── Audio Track (AAC)
 ├── Metadata (duration, title, etc.)
 └── Index (for seeking/playback info)
```

This is why `.mp4` is called a **container format** — it "contains" video + audio + metadata, just like a ZIP file holds many types of data.

- **Video files** = structured containers that store compressed video, audio, and metadata in a unified format like `.mp4`.


---
Excellent question — pushing video content to a **CDN (Content Delivery Network)** involves not just the raw video segments, but also supporting metadata (also called **manifests**) that the player needs to stream the video correctly. Let’s go over what you should include, how it works, and how it’s typically done.

---

### 4. ✅ When putting videos on a CDN, what files and metadata should you include?

#### 📦 Files to include on the CDN

At a minimum, you should store the following:

##### 1. **Video Segments (Chunks)**
- All resolutions (e.g., 240p, 480p, 720p, 1080p, etc.)
- Format depends on protocol:
  - **[HLS](https://github.com/kyungtaek-jonas-lim/jonastudy/blob/main/concept/http/hls_dash_en.md)**: `.ts` files
  - **[DASH](https://github.com/kyungtaek-jonas-lim/jonastudy/blob/main/concept/http/hls_dash_en.md)**: `.mp4` (fragmented MP4)
- Organized by resolution or variant.

##### 2. **Manifest / Playlist File (Required)**
- Tells the player how to stream the video.
- HLS: `.m3u8` file
- DASH: `.mpd` file
- Includes:
  - Duration
  - Segment file paths
  - Bitrate
  - Resolution
  - Codec

> ✅ This file is **critical** — the player reads it first before fetching any video data.

##### 3. **Thumbnails (Optional but recommended)**
- Static thumbnails (e.g., cover image): `.jpg`, `.webp`
- Preview thumbnails (for hover/seek): a grid sprite or VTT file
- Usually stored separately, referenced by the app logic, not by the playlist.

##### 4. **Subtitles or Captions (Optional)**
- Formats: `.vtt`, `.srt`
- Referenced in the playlist or by the player separately

##### 5. **Metadata / Description / Title (Optional)**
- These are **not** stored in the CDN directly as files.
- Usually managed by your **backend/database** (e.g., in RDS, DynamoDB)
- Your application fetches them via API when rendering the video page

---

### 5. ✅ What does the manifest contain?

Let’s take **HLS (.m3u8)** as an example:

#### 🎬 Master Playlist (e.g., `master.m3u8`)
```m3u8
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=1280x720
720p/playlist.m3u8
```

#### 📺 Media Playlist (e.g., `720p/playlist.m3u8`)
```m3u8
#EXTM3U
#EXT-X-TARGETDURATION:10
#EXTINF:10.0,
segment0.ts
#EXTINF:10.0,
segment1.ts
...
```

The player:
- Reads the **master playlist** first → finds available qualities
- Then loads the **media playlist** → downloads segments one by one


&nbsp;
---
&nbsp;

## 7. Reference
- [Youtube - Hello Interview](https://www.youtube.com/watch?v=IUrQ5_g3XKs)