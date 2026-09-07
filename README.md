# Automated Multi-Platform Content Publishing Engine

A spreadsheet-driven social media publishing system built with **n8n, Google Sheets, Make.com, Buffer, JavaScript, and webhooks**.

The system automatically selects scheduled content, validates when it should be published, routes it to the correct social platform, handles image and text-only posts separately, publishes the content, and updates the source record after successful execution.

> This project intentionally does not use an LLM.  
> The workflow is built around deterministic business logic, scheduling, routing, state tracking, and API/webhook integrations.

---

## Overview

Managing scheduled content across multiple social platforms manually can become repetitive and difficult to track.

I built this automation to use **Google Sheets as a central content queue** and **n8n as the main orchestration layer**.

Each content record contains information such as:

- Unique Post ID
- Publishing date
- Scheduled time
- Platform
- Post copy
- Optional image URL
- Publishing status

The workflow automatically determines which content is due, routes it to the appropriate publishing system, and marks it as completed after successful execution.

---

## Architecture

```mermaid
flowchart TD

    A[Schedule Trigger] --> B[Google Sheets Content Queue]

    B --> C[Time & Catch-Up Validation]

    C --> D{Route by Platform}

    D -->|LinkedIn| E[Make.com Webhook]

    E --> F{Image Available?}
    F -->|Yes| G[LinkedIn Image Post]
    F -->|No| H[LinkedIn Text Post]

    D -->|X| I{Image Available?}

    I -->|Yes| J[Buffer Image Post]
    I -->|No| K[Buffer Text Post]

    G --> L[Success Checkpoint]
    H --> L
    J --> L
    K --> L

    L --> M[Update Google Sheets Status to Posted]
```

---

## How It Works

### 1. Scheduled Execution

The n8n workflow runs at predefined publishing times.

The current implementation is configured around the publishing schedule stored in the content plan.

---

### 2. Fetch Pending Content

n8n retrieves content records from Google Sheets.

Google Sheets acts as the central source for:

```text
Post_ID
Date
Peak Viral Time
Platform
Image URL
Post Copy
Status
```

The `Status` field is initially empty.

After a post is successfully processed, the workflow updates it to:

```text
Posted
```

This allows the spreadsheet to act as both a **content queue** and a simple **publishing-state tracker**.

---

## Timezone-Aware Catch-Up Logic

The workflow uses **Africa/Lagos** as its scheduling timezone.

Instead of requiring the workflow to execute at the exact scheduled second, the automation checks whether:

1. The content is scheduled for today.
2. The current time is greater than or equal to the scheduled publishing time.

Example:

```text
Scheduled time: 12:15 PM
Current time:   12:18 PM

Result: Eligible for publishing
```

This provides catch-up behavior if execution happens slightly later than expected.

The logic is implemented in JavaScript inside an n8n Code node.

```javascript
const now = new Date(
  new Date().toLocaleString("en-US", {
    timeZone: "Africa/Lagos"
  })
);

const todayStr =
  now.getFullYear() +
  "-" +
  String(now.getMonth() + 1).padStart(2, "0") +
  "-" +
  String(now.getDate()).padStart(2, "0");

return items.filter(item => {
  const postDate = item.json["Date"];
  const timeStr = item.json["Peak Viral Time"];

  if (postDate !== todayStr) return false;

  const [time, modifier] = timeStr.split(" ");

  let [hours, minutes] = time.split(":");

  hours = parseInt(hours, 10);

  if (hours === 12 && modifier === "AM") hours = 0;
  if (hours < 12 && modifier === "PM") hours += 12;

  const scheduledTime = new Date(
    now.getFullYear(),
    now.getMonth(),
    now.getDate(),
    hours,
    parseInt(minutes, 10)
  );

  return now >= scheduledTime;
});
```

---

# Platform Routing

After the correct content record is selected, n8n routes it according to the `Platform` value stored in Google Sheets.

Currently supported:

- LinkedIn
- X

Each platform has its own publishing implementation.

---

## LinkedIn Publishing Flow

LinkedIn publishing is handled through a **Make.com webhook integration**.

```text
n8n
 ↓
HTTP Webhook
 ↓
Make.com
 ↓
Router
 ├── Image Post
 └── Text-Only Post
 ↓
LinkedIn
```

n8n remains responsible for:

- Scheduling
- Selecting the correct content
- Preparing the data
- Formatting the post
- Routing by platform
- Tracking completion

Make.com acts as the LinkedIn publishing layer.

Inside Make.com, another router determines whether the content contains an image.

### LinkedIn with Image

```text
n8n
 ↓
Make Webhook
 ↓
Image detected
 ↓
LinkedIn image post
```

### LinkedIn without Image

```text
n8n
 ↓
Make Webhook
 ↓
No image
 ↓
LinkedIn text post
```

Using a webhook between n8n and Make.com also demonstrates how separate automation platforms can be connected when a service is easier to integrate through one platform than another.

---

## X Publishing Flow

X publishing is handled using **Buffer** from inside n8n.

Before publishing, the workflow checks whether the Google Sheets row contains an image URL.

```text
Route to X
      ↓
Check Image Status
     / \
    /   \
Image   No Image
  ↓        ↓
Buffer   Buffer
Image    Text
Post     Post
```

### X with Image

The workflow sends:

- Post copy
- Image URL

to the Buffer publishing node.

### X without Image

The workflow sends only:

- Post copy

to the text-only Buffer publishing path.

---

# Custom Content Formatting System

One issue encountered while storing a large amount of content inside Google Sheets was preserving readable paragraph formatting.

Using HTML-style line breaks such as:

```html
<br>
```

caused formatting problems across parts of the publishing pipeline.

Instead, I implemented a lightweight custom formatting convention.

### Formatting Tokens

```text
~   = single line break

~~  = paragraph break
```

Example content stored inside Google Sheets:

```text
Automation can save teams hours every week.~~Here are three examples:~1. Lead routing~2. Reporting~3. Content publishing
```

Before publishing, n8n converts the formatting tokens into proper new lines:

```javascript
$json["Post Copy"]
  .replace(/~~/g, "\n\n")
  .replace(/~/g, "\n");
```

Final output:

```text
Automation can save teams hours every week.

Here are three examples:
1. Lead routing
2. Reporting
3. Content publishing
```

This keeps the spreadsheet content compact while still producing properly formatted social media posts.

---

# Success Checkpoint

Different publishing services can return different response structures.

Instead of allowing those external responses to determine what is written back into Google Sheets, the workflow reconstructs a clean result containing only the values needed for the final update.

Example:

```javascript
const cleanData =
  $items("Filter Time (Catch-Up Logic)")[0].json;

return [
  {
    json: {
      Post_ID: cleanData.Post_ID,
      Status: "Posted"
    }
  }
];
```

The workflow then uses `Post_ID` to locate the correct spreadsheet record and updates:

```text
Status → Posted
```

This keeps the final Google Sheets update independent of the response format returned by the publishing platform.

---

# Google Sheets Content Model

Example structure:

| Post_ID | Date | Peak Viral Time | Platform | Image URL | Post Copy | Status |
|---|---|---|---|---|---|---|
| POST001 | 2026-09-07 | 8:30 AM | LinkedIn | image-url | Content... | Posted |
| POST002 | 2026-09-07 | 12:15 PM | X | | Content... | Posted |
| POST003 | 2026-09-08 | 8:30 AM | LinkedIn | image-url | Content... | |

A screenshot of the actual content-planning sheet is included in the repository with sensitive or unnecessary information hidden.

---

# Workflow Reliability

Several reliability measures are included in the workflow.

### Google Sheets retries

The workflow retries failed Google Sheets operations before stopping.

```text
Retry on failure: Enabled
Maximum attempts: 5
Delay between attempts: 5 seconds
```

### Catch-up scheduling

Posts remain eligible when the current time has passed their scheduled time, preventing small execution delays from automatically skipping content.

### Publishing state

Successfully processed content is marked:

```text
Posted
```

in Google Sheets.

### Unique Post IDs

Each content record has a unique `Post_ID`, which is used to identify the correct row during the final update.

---

# Technology Stack

| Technology | Purpose |
|---|---|
| **n8n** | Main workflow orchestration |
| **Google Sheets** | Content queue, scheduling data and publishing state |
| **Make.com** | LinkedIn publishing integration |
| **Buffer** | X publishing integration |
| **JavaScript** | Time logic, filtering and data transformation |
| **HTTP Webhooks** | Communication between n8n and Make.com |
| **OAuth / API Integrations** | External service authentication |
| **Schedule Trigger** | Automated execution |

---

# Repository Structure

```text
automated-social-content-publishing-engine/
│
├── README.md
│
├── .gitignore
│
├── workflows/
│   │
│   ├── n8n/
│   │   └── content-publishing-engine.json
│   │
│   └── make/
│       └── linkedin-publisher-blueprint.json
│
├── docs/
│   ├── architecture.md
│   ├── workflow-logic.md
│   ├── linkedin-make-integration.md
│   └── formatting-system.md
│
└── screenshots/
    ├── n8n-full-workflow.png
    ├── n8n-platform-routing.png
    ├── make-linkedin-routing.png
    └── google-sheets-content-queue.png
```

---

# Screenshots

## n8n Workflow

_Add full workflow screenshot here._

```markdown
![n8n Workflow](screenshots/n8n-full-workflow.png)
```

---

## Platform Routing

_Add a screenshot showing the LinkedIn/X routing logic._

```markdown
![Platform Routing](screenshots/n8n-platform-routing.png)
```

---

## LinkedIn Make.com Scenario

_Add the Make.com scenario showing the image/text router._

```markdown
![LinkedIn Make Workflow](screenshots/make-linkedin-routing.png)
```

---

## Google Sheets Content Queue

_Add a cropped screenshot of the Google Sheet showing fields such as Post ID, Date, Time, Platform, Image URL, Post Copy and Status._

```markdown
![Google Sheets Content Queue](screenshots/google-sheets-content-queue.png)
```

---

# Why This Project Matters

This project demonstrates automation engineering without depending on generative AI.

The main problems are solved through:

- Deterministic workflow logic
- Scheduling
- Timezone handling
- Conditional routing
- Webhooks
- Cross-platform integration
- Data transformation
- Publishing-state management
- JavaScript
- External API/service integration

The goal was not to add AI where it was unnecessary, but to design a predictable system that could automatically execute a clearly defined business process.

---

# Current Status

The workflow has been running in my environment for approximately **two weeks**, automatically processing scheduled content for LinkedIn and X.

The repository contains a sanitized version of the workflow. Credentials, private webhook URLs, account identifiers, and other sensitive configuration values are replaced with placeholders.

---

# Author

**Peter Ayoola**

AI Automation Specialist & System Architect

GitHub: [@Oppy001](https://github.com/Oppy001)
