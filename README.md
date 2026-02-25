<div align="center">

<img width="100%" src="https://capsule-render.vercel.app/api?type=waving&color=0:0f0c29,40:302b63,100:1a0533&height=180&section=header&text=Mails%20to%20Leads&fontSize=52&fontColor=ffffff&fontAlignY=38&desc=n8n%20%E2%80%A2%20Google%20Sheets%20%E2%80%A2%20Cloudinary%20%E2%80%A2%20Gmail%20%E2%80%A2%20100%25%20Autonomous&descAlignY=60&descSize=15&animation=fadeIn" />

<br/>

<img src="https://img.shields.io/badge/Status-Live%20%26%20Active-22c55e?style=for-the-badge&logo=circle&logoColor=white" />
&nbsp;
<img src="https://img.shields.io/badge/Trigger-Daily%203PM-7c3aed?style=for-the-badge&logo=clockify&logoColor=white" />
&nbsp;
<img src="https://img.shields.io/badge/Leads%20Per%20Run-10%20max-EA4B71?style=for-the-badge&logo=target&logoColor=white" />
&nbsp;
<img src="https://img.shields.io/badge/Built%20With-n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white" />

</div>

---

## 📌 Overview

**Mails to Leads** is a fully autonomous cold-outreach automation built in [n8n](https://n8n.io). It runs every day at **3PM**, reads qualified leads from a Google Sheet, takes a **live screenshot of each lead's website**, uploads it to **Cloudinary**, and sends a personalised **HTML email** — all without a single human click.

After sending, it marks the row as *"email done"* in the sheet so no lead is ever contacted twice.

> **Live at:** [Slora AI](https://www.sloraai.com/) — serving real outreach for real clients daily.

---

## ⚡ Workflow Architecture

```mermaid
flowchart LR
    A(["⏰ Schedule\nTrigger\n3PM Daily"]) --> C
    B(["🖱️ Manual\nTrigger"]) --> C

    C["📊 Google Sheets\nGet All Rows"] --> D{"🔀 IF Filter\n4 Conditions"}

    D -- "✅ PASS" --> E["🔢 Limit\n10 leads/run"]
    D -- "❌ FAIL" --> Z(["🚫 Stop"])

    E --> F["📸 ScreenshotOne API\nCapture Website"]
    F --> G["☁️ Cloudinary\nUpload & Get CDN URL"]
    G --> H["📧 Gmail\nSend HTML Email"]
    H --> I["📊 Google Sheets\nMark as Emailed"]

    style A fill:#302b63,color:#fff,stroke:#7c3aed
    style B fill:#302b63,color:#fff,stroke:#7c3aed
    style C fill:#34A853,color:#fff,stroke:#2d8f47
    style D fill:#EA4B71,color:#fff,stroke:#c73a5d
    style E fill:#6D00CC,color:#fff,stroke:#5800aa
    style F fill:#1a1a2e,color:#fff,stroke:#7c3aed
    style G fill:#3448C5,color:#fff,stroke:#2a3aa0
    style H fill:#D14836,color:#fff,stroke:#b03a2d
    style I fill:#34A853,color:#fff,stroke:#2d8f47
    style Z fill:#555,color:#fff,stroke:#333
```

---

## 🔗 Node-by-Node Breakdown

### 1 · Triggers

Two ways to start the workflow:

| Trigger | Type | When |
|:---|:---|:---|
| ⏰ **Schedule Trigger** | Automated | Every day at **15:00 (3PM)** |
| 🖱️ **Manual Trigger** | On-demand | When you click *"Execute workflow"* in n8n |

Both triggers feed into the same pipeline. The schedule trigger has **no connection** in the JSON (it's wired but currently routes to manual for testing), while the manual trigger kicks off the full chain.

---

### 2 · Google Sheets — Read Leads

```
Node   : Get row(s) in sheet
Sheet  : Tatto shops leads  →  tab: mail saved
Action : Reads ALL rows from the sheet
```

**Sheet columns used downstream:**

| Column | Purpose |
|:---|:---|
| `Mail ids` | Recipient email address |
| `Website` | URL to screenshot |
| `CheckBox` | Manual approval flag |
| `Emailed?` | Deduplication guard |

---

### 3 · IF Filter — Smart Qualification Gate

All **4 conditions must pass** (AND logic) for a lead to proceed:

```
╔══════╦═══════════════════╦════════════════════════════════════════╗
║  #   ║  Field            ║  Condition                             ║
╠══════╬═══════════════════╬════════════════════════════════════════╣
║  1   ║  Mail ids         ║  NOT empty  →  valid email exists      ║
║  2   ║  Website          ║  NOT empty  →  has a site to capture   ║
║  3   ║  CheckBox         ║  ≠ false    →  manually pre-approved   ║
║  4   ║  Emailed?         ║  IS empty   →  not yet contacted       ║
╚══════╩═══════════════════╩════════════════════════════════════════╝
```

> Leads failing any condition are silently dropped — no errors, no duplicates.

---

### 4 · Limit — Rate Control

```
Max items : 10 per execution
```

Prevents over-sending and keeps the workflow within API rate limits. Even if 100 leads qualify, only 10 get processed per run.

---

### 5 · HTTP Request — Website Screenshot

```
API     : ScreenshotOne  (screenshotone.com)
Method  : GET
Output  : JPEG binary image data
```

**Parameters sent:**

| Param | Value |
|:---|:---|
| `url` | Lead's website from sheet |
| `format` | `jpg` |
| `block_cookie_banners` | `true` — clean screenshots |

The raw binary JPG flows directly into Cloudinary.

---

### 6 · Cloudinary — CDN Upload

```
Operation : Upload File
Output    : secure_url  (public HTTPS image link)
```

Converts the raw screenshot binary into a permanent, publicly accessible CDN URL used in the email body as `{{ $json.secure_url }}`.

---

### 7 · Gmail — Send HTML Email

```
To      : {{ $('HTTP Request').item.json['Mail ids'] }}
Subject : personalised outreach
Body    : Full HTML email with embedded screenshot
```

**Email structure:**

```
┌─────────────────────────────────────┐
│           Slora AI Logo             │
├─────────────────────────────────────┤
│   [LIVE Screenshot of their site]   │
│        [ Visit Your New Site ]      │
├─────────────────────────────────────┤
│  CONSULTATION OFFER headline        │
│  "Find your dream Website Demo..."  │
│        [ Book a consultation ]      │
├─────────────────────────────────────┤
│  🔥 35% OFF special offer box       │
│  Why choose us? (3-column icons)    │
│    • Accelerate Growth              │
│    • Smarter Automation             │
│    • Enterprise Tech, Small Price   │
│        [ CLAIM MY 35% DISCOUNT ]    │
└─────────────────────────────────────┘
```

The personalisation hook: each lead sees a screenshot of **their own website** inside the email — making it feel handcrafted.

---

### 8 · Google Sheets — Log & Deduplication

```
Operation : Update row (matched by Mail ids)
```

After the email is sent, the sheet row is updated:

| Column | Written Value |
|:---|:---|
| `Emailed?` | `"email done"` |
| `Date` | Today's date (`YYYY-MM-DD`) |

This ensures the IF filter (Step 3) will skip this lead on all future runs.

---

## 📊 Data Flow Summary

```mermaid
sequenceDiagram
    participant S as ⏰ Scheduler
    participant G1 as 📊 Google Sheets (Read)
    participant IF as 🔀 IF Filter
    participant L as 🔢 Limit
    participant SS as 📸 ScreenshotOne
    participant C as ☁️ Cloudinary
    participant GM as 📧 Gmail
    participant G2 as 📊 Google Sheets (Write)

    S->>G1: Trigger at 3PM
    G1->>IF: All rows
    IF->>L: Qualified leads only
    L->>SS: Website URL (max 10)
    SS->>C: Screenshot binary
    C->>GM: CDN image URL
    GM->>G2: Email sent ✅
    G2-->>G1: Row marked "email done"
```

---

## 🛠️ Tech Stack

<div align="center">

| Tool | Role |
|:---|:---|
| ![n8n](https://img.shields.io/badge/n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white) | Workflow orchestration |
| ![Google Sheets](https://img.shields.io/badge/Google_Sheets-34A853?style=for-the-badge&logo=google-sheets&logoColor=white) | Lead database + logging |
| ![ScreenshotOne](https://img.shields.io/badge/ScreenshotOne_API-1a1a2e?style=for-the-badge&logo=camera&logoColor=white) | Website capture |
| ![Cloudinary](https://img.shields.io/badge/Cloudinary-3448C5?style=for-the-badge&logo=cloudinary&logoColor=white) | Image CDN hosting |
| ![Gmail](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white) | HTML email delivery |

</div>

---

## 🚀 Setup Guide

### Prerequisites

- [ ] n8n instance (self-hosted or cloud)
- [ ] Google Sheets OAuth2 credentials configured in n8n
- [ ] Gmail OAuth2 credentials configured in n8n
- [ ] Cloudinary API credentials configured in n8n
- [ ] [ScreenshotOne](https://screenshotone.com) API key

### Google Sheet Structure

Your sheet must have these exact column headers:

```
Name | Phone | Mail ids | Country | Website | Address | Email Template | Image Url | New Site Url | CheckBox | Emailed? | Date
```

### To Activate

1. Import the workflow JSON into n8n
2. Connect all credential nodes (Google Sheets, Gmail, Cloudinary)
3. Replace the ScreenshotOne `access_key` with your own key
4. Update the Google Sheet URL in both Sheets nodes to your sheet
5. Toggle the workflow to **Active** ✅

---

## ⚙️ Configuration Reference

| Setting | Value |
|:---|:---|
| Execution mode | `v1` |
| Binary mode | `separate` |
| Max leads/run | `10` |
| Schedule | `15:00` daily |
| Workflow ID | `yphrR9L7urLmPTDv` |

---

## 📈 Why This Works

The personalisation loop is the secret:

```
Lead sees their OWN website in the email
        ↓
Instant credibility — "they did their research"
        ↓
Higher open rate → Higher reply rate → More consultations booked
```

Combined with the 4-point filter, only **genuinely qualified, approved** leads receive emails — keeping deliverability high and spam complaints near zero.

---

<div align="center">

**Built by [Abdul Rehman](https://github.com/ar-rehman786)**

[![Gmail](https://img.shields.io/badge/Email-abdulrehmanhameed4321%40gmail.com-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:abdulrehmanhameed4321@gmail.com)
&nbsp;
[![Slora AI](https://img.shields.io/badge/🚀_Slora_AI-sloraai.com-5D3EFF?style=for-the-badge)](https://www.sloraai.com/)

</div>

<img width="100%" src="https://capsule-render.vercel.app/api?type=waving&color=0:1a0533,50:302b63,100:0f0c29&height=100&section=footer&animation=fadeIn" />
