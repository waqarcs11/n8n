# Cannabis Regulation News Workflow Documentation

## Overview

This workflow automates the process of discovering, filtering, and publishing cannabis regulation news stories. It consists of two main parts:

1. **Daily News Discovery & Filtering** – Runs on schedule to find and evaluate news stories
2. **Approval & Publishing** – Triggered by email replies to publish approved stories

---

# Part 1: Daily News Discovery & Filtering

## Trigger

### Schedule Trigger: Daily News Search
- **Schedule:** Runs daily at 9:00 AM (your local timezone)
- **Cron Expression:** `0 9 * * *`
- **Purpose:** Automatically fetches new cannabis regulation news every morning

---

## Step 1: Fetch News

### Fetch Cannabis News from GNews
- **API Provider:** GNews.io
- **Endpoint:** `https://gnews.io/api/v4/search`
- **API Key Setup:**
  1. Visit https://gnews.io/
  2. Sign up for a free account
  3. Navigate to your dashboard
  4. Copy your API key
  5. In n8n, configure HTTP Request credentials with your API key
- **Search Query:** `"cannabis regulation"`
- **Parameters:**
  - `q`: cannabis regulation
  - `lang`: en (English)
  - `country`: us (United States)
  - `max`: 10 (returns up to 10 articles)
  - `apikey`: Your GNews API key
- **Returns:** title, description, url, source, publishedAt

---

## Step 2: Process Articles

### Split Articles
- Splits the news feed into individual article items for processing

### Prepare Article Data
Transforms each article into standardized format:

| Field | Description |
|-------|-------------|
| `url` | Article URL |
| `title` | Article headline |
| `source` | News source name |
| `publishDate` | Publication date |
| `summary` | Article description/snippet |
| `dateFound` | Current date (when discovered) |
| `status` | Initial status: **"New"** |

---

## Step 3: Duplicate Detection

### Get Existing Stories
- Fetches all existing stories from Google Sheets tracker
- Sheet: **"Story Tracker"**

### Check for Duplicates
- Compares new articles against existing ones by URL
- Identifies which stories are truly new

### Keep Only New Stories
- Filters out duplicates
- Only processes stories not already in the tracker

---

## Step 4: Log New Stories

### Log All Stories
Adds new stories to Google Sheets tracker:
- **Mapping Mode:** Manual (explicit field mapping)

| Field | Column |
|-------|--------|
| url | URL |
| title | Title |
| source | Source |
| publishDate | PublishDate |
| summary | Summary |
| dateFound | DateFound |
| status | Status |

---

## Step 5: AI Relevance Evaluation

### Evaluate Relevance
Uses OpenAI to evaluate if each story is relevant to cannabis regulation.

**Prompt includes:**
- Story title
- Story summary
- Source

**Returns:** `"Candidate"` or `"Ignored"` status with reasoning

### OpenAI Chat Model
- **Model:** `gpt-4o-mini`
- **API Key Setup:**
  1. Visit https://platform.openai.com/
  2. Sign up or log in
  3. Navigate to API Keys section
  4. Create a new API key
  5. In n8n, configure OpenAI credentials with your API key
- **Endpoint:** `https://api.openai.com/v1/chat/completions`
- Evaluates relevance based on cannabis regulation focus

### Structured Output Parser
Parses AI response into structured format:
- `status` – `"Candidate"` or `"Ignored"`
- `reasoning` – AI's explanation

### Update Story Status
- Preserves all original story fields
- Adds AI evaluation results (status, reasoning)

---

## Step 6: Update Tracker

### Update Tracker Status
- Updates Google Sheets with AI evaluation
- **Column to Match On:** URL (`={{ $json.url }}`)
- **Column to Update:** Status
- **Value:** AI-evaluated status (`"Candidate"` or `"Ignored"`)

### Preserve Story Data
Ensures all story fields flow through to email:
- url, title, source, publishDate, summary, status, reasoning

---

## Step 7: Send Summary Email

### Filter Candidates Only
- Filters stories where `status = "Candidate"`
- Only candidate stories proceed to email

### Aggregate Candidates
- Combines all candidate stories into single item for email

### Send Candidate Summary
- **To:** CEO email address
- **Subject:** `"Cannabis Regulation News - Candidate Stories for Review"`
- **Body includes for each story:**
  - Title
  - Source
  - Summary
  - URL
  - AI Reasoning
- **Email Service:** Gmail (configured via n8n Gmail credentials)

---

# Part 2: Approval & Publishing Workflow

## Trigger

### Approval Email Received (Gmail Trigger)
- **Watches for:** Gmail label `"cannabis-approvals"`
- **Triggers when:** You reply to the candidate summary email
- **Polling Interval:** Checks every 5 minutes for new labeled emails

### Setup Required

**Create Gmail label:**
1. In Gmail, go to Settings → Labels
2. Click "Create new label"
3. Name it: `cannabis-approvals`

**Create Gmail filter:**
1. In Gmail, go to Settings → Filters and Blocked Addresses
2. Click "Create a new filter"
3. Subject: `"Cannabis Regulation News - Candidate Stories for Review"`
4. Click "Create filter"
5. Check "Apply the label:" and select `cannabis-approvals`
6. Click "Create filter"

**Configure Gmail credentials in n8n:**
- Use OAuth2 authentication
- Grant necessary permissions for reading emails and labels

---

## Step 1: Extract Approval Data

### Extract Approval Data
Extracts from your reply email:
- `storyUrl` – URL of story to approve (from email body text)
- `approvalDecision` – Your decision (from subject/body)
- `reviewerNotes` – Any notes you included
- **Field Used:** `textPlain` (plain text version of email body)

---

## Step 2: Find Story in Tracker

### Find Story Record
- Fetches all stories from Google Sheets tracker
- Sheet: **Story Tracker**

### Match Story by URL
- Finds the specific story you're approving
- Matches by URL from your email

---

## Step 3: Update Status

### Update to Approved
- Updates Google Sheets tracker
- **Column to Match On:** URL (`={{ $json.url }}`)
- **Column to Update:** Status
- **Value:** `"Approved"`

---

## Step 4: Prepare for WordPress

### Prepare Article for Draft
> **Note:** This step uses existing summary data instead of fetching full articles to avoid site blocking (403 errors)

Gathers all data needed for WordPress draft:
- `title` – Story headline
- `source` – News source
- `summary` – Description/snippet from GNews API (already stored in Google Sheets)
- `url` – Original article URL
- `reviewerNotes` – Your notes from approval email

---

## Step 5: Generate WordPress Draft

### Generate Draft (AI Agent)
Uses OpenAI to create WordPress draft post:
- **Model:** `gpt-4o-mini`
- **API Endpoint:** `https://api.openai.com/v1/chat/completions`
- **Input data:** Title, Source, Summary (from GNews API), URL, Reviewer notes
- **Generates:** Professional article draft with proper formatting

### OpenAI Draft Model
- Configured to generate WordPress-ready content
- Maintains source attribution
- Includes original URL reference
- Uses same OpenAI credentials as relevance evaluation

---

## Step 6: Save to WordPress

### Prepare Draft Record
- Formats the generated draft for WordPress API

### Save Draft
- Creates draft post in WordPress
- **Status:** Draft (not published)
- **WordPress API Setup:**
  1. Ensure WordPress REST API is enabled
  2. Create application password in WordPress:
     - Go to Users → Your Profile
     - Scroll to "Application Passwords"
     - Create new application password
  3. Configure WordPress credentials in n8n with:
     - WordPress site URL
     - Username
     - Application password
- **Endpoint:** `https://your-wordpress-site.com/wp-json/wp/v2/posts`
- Ready for CEO review

---

## Step 7: Notify CEO

### Send to CEO for Approval
- Sends email notification via Gmail
- **Includes:** Link to WordPress draft for final review
- **Subject:** `"New Draft Ready for Review"`

---

# Google Sheets Structure

## Sheet Name: Story Tracker

### Columns

| Column | Description |
|--------|-------------|
| URL | Article URL (used for duplicate detection and matching) |
| Title | Article headline |
| Source | News source name |
| PublishDate | Original publication date |
| Summary | Article description/snippet from GNews |
| DateFound | Date discovered by workflow |
| Status | Current status: `"New"` → `"Candidate"`/`"Ignored"` → `"Approved"` |

### Setup
1. Create a new Google Spreadsheet
2. Name the first sheet `"Story Tracker"`
3. Add column headers exactly as listed above
4. Configure Google Sheets credentials in n8n:
   - Use OAuth2 authentication
   - Grant read/write permissions

---

# API Keys & Credentials Setup Summary

## 1. GNews API
- **Website:** https://gnews.io/
- **Free Tier:** 100 requests/day
- **Setup Steps:**
  1. Create account at gnews.io
  2. Copy API key from dashboard
  3. In n8n: Configure HTTP Request node with API key parameter

## 2. OpenAI API
- **Website:** https://platform.openai.com/
- **Pricing:** Pay-per-use (gpt-4o-mini is cost-effective)
- **Setup Steps:**
  1. Create account at platform.openai.com
  2. Add payment method
  3. Generate API key in API Keys section
  4. In n8n: Configure OpenAI credentials
- **Used For:**
  - Relevance evaluation (Evaluate Relevance node)
  - Draft generation (Generate Draft node)

## 3. Google Sheets
- **Setup Steps:**
  1. Create Google Cloud project
  2. Enable Google Sheets API
  3. Create OAuth2 credentials
  4. In n8n: Configure Google Sheets OAuth2 credentials
- **Permissions Needed:** Read and write access to spreadsheets

## 4. Gmail
- **Setup Steps:**
  1. Use same Google Cloud project as Sheets
  2. Enable Gmail API
  3. Create OAuth2 credentials
  4. In n8n: Configure Gmail OAuth2 credentials
- **Permissions Needed:**
  - Read emails
  - Send emails
  - Manage labels
- **Used For:**
  - Sending candidate summary emails
  - Receiving approval replies (trigger)
  - Sending draft notification emails

## 5. WordPress
- **Setup Steps:**
  1. Ensure WordPress site has REST API enabled (default in modern WordPress)
  2. Create application password in WordPress admin
  3. In n8n: Configure WordPress credentials with site URL and app password
- **Permissions Needed:** Create and edit posts
- **Endpoint:** `https://your-site.com/wp-json/wp/v2/posts`

---

# Workflow Timing & Schedule

## Daily News Discovery
- **Runs:** Every day at 9:00 AM
- **Cron Expression:** `0 9 * * *`
- **Duration:** Typically 2–5 minutes depending on number of articles
- **Email Sent:** Shortly after 9:00 AM with candidate stories

## Approval Processing
- **Trigger:** Email-based (when you reply)
- **Polling:** Checks Gmail every 5 minutes for new labeled emails
- **Processing Time:** 1–2 minutes per approved story
- **WordPress Draft:** Created immediately after approval

---

# Workflow Logic Summary

## Daily Flow (9:00 AM)
1. Fetch news from GNews API → Split articles → Format data
2. Check for duplicates against Google Sheets → Log new stories
3. AI evaluates relevance using OpenAI → Update tracker with status
4. Email candidate stories to CEO via Gmail

## Approval Flow (On-demand)
1. CEO replies to email → Gmail trigger fires (checks every 5 minutes)
2. Extract approval data from email → Find story in Google Sheets tracker
3. Update status to `"Approved"` in Google Sheets
4. Prepare article data (uses existing summary from GNews, not full article)
5. AI generates WordPress draft using OpenAI
6. Save draft to WordPress via REST API
7. Notify CEO via Gmail with link to draft

## Key Features
- Automatic duplicate detection via URL matching
- AI-powered relevance filtering (OpenAI gpt-4o-mini)
- Email-based approval workflow (Gmail)
- Uses existing summary data to avoid site blocking
- WordPress draft generation (not auto-published)
- Full audit trail in Google Sheets

---

# Important Notes

## Why We Use Summary Data Instead of Full Articles
- **Problem:** Many news sites block automated content fetching (403 Forbidden errors)
- **Solution:** Use the summary/snippet already provided by GNews API
- **Benefits:**
  - No site blocking issues
  - Faster processing
  - Still provides enough context for AI to generate quality drafts
  - Original URL is always included for reference

## Status Flow
| Status | Meaning |
|--------|---------|
| New | Story just discovered and logged |
| Candidate | AI determined it's relevant to cannabis regulation |
| Ignored | AI determined it's not relevant |
| Approved | CEO approved for WordPress draft creation |

## Email Reply Format
When replying to approve a story, your email should:
- Keep the original subject line (Gmail adds "Re:" automatically)
- Include the story URL you want to approve
- Optionally add reviewer notes for context
- The Gmail filter will automatically apply the `"cannabis-approvals"` label

## WordPress Drafts
- Drafts are never auto-published
- CEO has final review before making posts live
- Drafts include source attribution and original URL
- AI-generated content maintains professional tone

---

# Troubleshooting

## No Articles Found
- Check GNews API key is valid
- Verify API quota hasn't been exceeded (100/day on free tier)
- Check search query is returning results on gnews.io website

## Duplicate Stories Not Detected
- Verify Google Sheets "Story Tracker" sheet name is exact
- Check URL column exists and is spelled correctly
- Ensure Google Sheets credentials have read access

## Approval Email Not Triggering
- Verify Gmail label `"cannabis-approvals"` exists
- Check Gmail filter is correctly configured
- Ensure Gmail credentials have label access permissions
- Wait up to 5 minutes for polling interval

## WordPress Draft Not Created
- Verify WordPress REST API is accessible
- Check application password is correct
- Ensure user has permission to create posts
- Verify WordPress site URL is correct in credentials
