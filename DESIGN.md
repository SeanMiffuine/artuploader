# ArtUploader — Design Document

## Problem

Artists create work and want to share it across multiple social media platforms (Instagram, Twitter/X, ArtStation, Pixiv). Manually uploading the same piece to each platform — with different caption styles, hashtags, and formats — is tedious and time-consuming. This friction causes artists to either skip platforms or post inconsistently.

## Goal

A lightweight CLI tool that watches a Google Drive folder for new artwork, auto-generates platform-specific captions, and uploads to all configured social media platforms on a schedule — with zero daemons or background processes. It runs via system cron and exits after each cycle.

## Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Runtime | **Node.js** | Core application runtime |
| Browser Automation | **Playwright** | Uploads to platforms using saved browser sessions |
| Scheduling | **System cron** | Triggers posting cycles — nothing stays running |
| Database | **SQLite** | Tracks queue state, post history, platform status |
| Caption Generation | **Claude API** | Generates platform-tailored captions from image + metadata |
| File Storage | **Google Drive (local sync)** | Artist drops files into a synced folder on disk |
| CLI Framework | **inquirer** or **prompts** | Interactive terminal UI for configuration and queue management |

## Architecture Overview

```
Google Drive Folder (local sync)
        │
        ▼
  ┌─────────────┐
  │ Folder Scan  │  Detects new images by modified time
  └──────┬──────┘
         ▼
  ┌─────────────┐
  │   SQLite     │  Checks if image already queued/posted
  └──────┬──────┘
         ▼
  ┌─────────────┐
  │ Claude API   │  Generates captions per platform
  └──────┬──────┘
         ▼
  ┌─────────────┐
  │ Playwright   │  Opens saved browser sessions, uploads to each platform
  └──────┬──────┘
         ▼
  ┌─────────────┐
  │   SQLite     │  Marks image as posted, logs results
  └─────────────┘
```

The cron job triggers this pipeline once per cycle. Each run processes **one image** (the next in the queue), posts it to all configured platforms, then exits.

## Components

### 1. Folder Scanner
- Watches a configurable local directory (Google Drive sync folder)
- Detects new images by comparing folder contents against SQLite records
- Queues new images ordered by file creation/modified time (oldest first)
- Supports common formats: PNG, JPG, JPEG, WEBP, PSD (converted), TIFF

### 2. Queue Manager (SQLite)
- **images** table: file path, hash, queue position, status (queued/posted/failed/skipped), added timestamp
- **posts** table: image ID, platform, caption used, posted timestamp, success/failure, error message
- **config** table: schedule, platforms enabled, caption preferences
- Handles crash recovery — if a run fails mid-upload, the image is marked per-platform so it retries only the failed ones

### 3. Caption Generator
- Sends the image to Claude API (vision) along with any user-provided tags/title
- Generates a tailored caption for each target platform:
  - **Instagram**: longer description, heavy on hashtags, emoji-friendly
  - **Twitter/X**: concise, 280-char limit, 2-3 hashtags max
  - **ArtStation**: professional/technical description, tools/software used
  - **Pixiv**: tag-focused, supports Japanese tags if configured
- User can append or prepend custom text per-post or globally
- Captions are stored in SQLite and can be previewed/edited before posting

### 4. Platform Uploader (Playwright)
- Each platform has its own uploader module with the browser automation flow
- Uses persistent browser contexts (saved login sessions) — no re-auth each run
- Includes human-like delays between actions to avoid bot detection
- Handles platform-specific requirements:
  - Image resizing/cropping if needed
  - Character limits
  - Tag/hashtag formatting
- Reports success/failure per platform back to the queue manager

### 5. CLI Interface
- Not a persistent UI — just commands you run when you need them
- Built with an interactive terminal prompt library

## CLI Commands

| Command | Description |
|---------|-------------|
| `artuploader run` | Process next image in queue — generate captions, upload to all platforms. This is what cron calls. |
| `artuploader force` | Immediately post the next image in the queue, regardless of schedule. |
| `artuploader status` | Show current queue: upcoming, posted, failed. |
| `artuploader preview` | Show the next image and its generated captions for each platform before posting. |
| `artuploader edit <id>` | Edit/override the generated caption for a specific queued image. |
| `artuploader skip <id>` | Skip an image — move it out of the queue without posting. |
| `artuploader retry <id>` | Retry a failed post for a specific image (all failed platforms or a specific one). |
| `artuploader login <platform>` | Interactive login — opens a browser so you can log into a platform. Saves the session for future runs. |
| `artuploader config` | Interactive setup — set folder path, schedule, enabled platforms, caption preferences. |
| `artuploader history` | Show full post history with timestamps and platform results. |

## User Stories

### Initial Setup
- **As an artist**, I want to run `artuploader config` and be guided through setup (folder path, which platforms, posting schedule, Claude API key) so I can get started quickly.
- **As an artist**, I want to run `artuploader login instagram` and have a browser window open where I log in once, so the tool can reuse my session for future uploads.
- **As an artist**, I want to add a single cron line and never think about scheduling again.

### Day-to-Day Posting
- **As an artist**, I want to drop finished artwork into my Google Drive folder and have it automatically detected and added to the posting queue, ordered by when I added it.
- **As an artist**, I want the tool to auto-generate captions tailored to each platform (hashtag-heavy for Instagram, concise for Twitter, professional for ArtStation) so I don't have to write four different captions.
- **As an artist**, I want to append or prepend my own text to the auto-generated caption (e.g., "Commissions open! DM me") either per-post or as a global default.
- **As an artist**, I want the cron job to post one piece at a time on my configured schedule so my feed has a consistent posting cadence rather than dumping everything at once.

### Queue Management
- **As an artist**, I want to run `artuploader status` to see what's coming up next, what's already been posted, and what failed so I have full visibility into my queue.
- **As an artist**, I want to run `artuploader preview` to see the next image and its generated captions before it goes live, so I can catch anything weird.
- **As an artist**, I want to edit a generated caption if the AI got something wrong or I want to add context the AI couldn't know.
- **As an artist**, I want to skip an image in the queue (e.g., a WIP I accidentally saved to the folder) without deleting the file.
- **As an artist**, I want to reorder or prioritize a specific image in the queue if I want something posted sooner.

### Force Posting
- **As an artist**, I want to run `artuploader force` to immediately post the next image in the queue right now, bypassing the schedule, for time-sensitive posts.
- **As an artist**, I want to run `artuploader force <id>` to immediately post a specific image from anywhere in the queue, not just the next one.

### Failure Handling
- **As an artist**, I want the tool to retry only the platforms that failed for a given image, not re-post to platforms that succeeded, so I don't get duplicate posts.
- **As an artist**, I want to see clear error messages when a post fails (e.g., "Instagram: session expired, run `artuploader login instagram`") so I know how to fix it.
- **As an artist**, I want to run `artuploader retry <id>` to retry a failed post manually after fixing the issue.

### History and Tracking
- **As an artist**, I want to see a full history of what was posted, when, and to which platforms so I can track my posting consistency.
- **As an artist**, I want to know if a platform login session has expired before the cron job runs and fails silently.

### Configuration
- **As an artist**, I want to configure my posting schedule (e.g., "daily at 10am", "every 12 hours", "Mon/Wed/Fri at 6pm") through an interactive prompt.
- **As an artist**, I want to enable or disable specific platforms without losing my login sessions, so I can temporarily stop posting to one platform.
- **As an artist**, I want to set global caption preferences (tone, default hashtags, appended text) that apply to all generated captions.
- **As an artist**, I want to configure per-platform settings (e.g., always add "Commission info in bio" to Instagram posts).

## Cron Setup

The user adds one line to their crontab:

```bash
# Post one artwork daily at 10:00 AM
0 10 * * * cd ~/artuploader && node index.js run >> ~/.artuploader/cron.log 2>&1
```

The `run` command:
1. Scans folder for new images → adds to queue
2. Picks the next queued image
3. Generates captions (or uses pre-generated/edited ones)
4. Uploads to each enabled platform via Playwright
5. Records results in SQLite
6. Exits

If there's nothing to post, it exits silently. If a post partially fails, it logs the error and marks which platforms failed so the next `retry` or `run` can handle it.

## File Structure

```
artuploader/
├── index.js              # CLI entry point, command routing
├── config.js             # Config loading/saving, interactive setup
├── scanner.js            # Folder scanning, new file detection
├── queue.js              # SQLite queue operations
├── caption.js            # Claude API caption generation
├── uploader/
│   ├── index.js          # Uploader orchestrator
│   ├── instagram.js      # Instagram automation
│   ├── twitter.js        # Twitter/X automation
│   ├── artstation.js     # ArtStation automation
│   └── pixiv.js          # Pixiv automation
├── browser.js            # Playwright session management (login, saved contexts)
├── db.js                 # SQLite setup, migrations, queries
├── package.json
├── .artuploader/         # Created at runtime in user home dir
│   ├── config.json       # User configuration
│   ├── artuploader.db    # SQLite database
│   ├── sessions/         # Playwright saved browser sessions
│   └── cron.log          # Cron output log
└── DESIGN.md
```

## Open Questions

1. **Image format handling** — Should we auto-convert formats like PSD/TIFF, or require the artist to export as PNG/JPG before dropping in the folder?
2. **Multi-image posts** — Some platforms support carousel/multi-image posts. Should we support grouping images (e.g., by subfolder or naming convention)?
3. **Video support** — Some artists post timelapses or animations. Worth supporting in v1, or defer?
4. **Notifications** — Should the tool notify the artist on success/failure (e.g., desktop notification, email)? Or is checking `artuploader status` enough?
5. **Rate limiting** — How many platforms can we realistically post to in one cron cycle before hitting timing/detection issues?
