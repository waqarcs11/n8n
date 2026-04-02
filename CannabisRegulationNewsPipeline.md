# Cannabis Regulation News Workflow Documentation

## Overview

This workflow automates the process of discovering, filtering, and publishing cannabis regulation news stories. It consists of two main parts:

1. **Daily News Discovery & Filtering** – Runs on schedule to find and evaluate news stories  
2. **Approval & Publishing** – Triggered by email replies to publish approved stories  

---

# Part 1: Daily News Discovery & Filtering

## Trigger
- **Schedule Trigger** – Runs daily to fetch new cannabis regulation news

---

## Step 1: Fetch News

### Fetch Cannabis News from GNews
- Fetches latest cannabis regulation news from GNews API  
- Search query: `"cannabis regulation"`  
- Returns:
  - title
  - description
  - url
  - source
  - publishedAt  

---

## Step 2: Process Articles

### Split Articles
- Splits the news feed into individual article items for processing  

### Prepare Article Data
Transforms each article into standardized format:

- `url` → Article URL  
- `title` → Article headline  
- `source` → News source name  
- `publishDate` → Publication date  
- `summary` → Article description/snippet  
- `dateFound` → Current date (when discovered)  
- `status` → Initial status: **"New"**

---

## Step 3: Duplicate Detection

### Get Existing Stories
- Fetches all existing stories from Google Sheets tracker  
- Sheet: **"Story Tracker"**

### Check for Duplicates
- Compares new articles against existing ones by URL  
- Identifies truly new stories  

### Keep Only New Stories
- Filters out duplicates  
- Only processes new stories  

---

## Step 4: Log New Stories

### Log All Stories
Adds new stories to Google Sheets:

| Field | Column |
|------|--------|
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
Uses OpenAI to evaluate if each story is relevant.

**Input:**
- Title  
- Summary  
- Source  

**Output:**
- `Candidate` or `Ignored`
- Reasoning  

### OpenAI Configuration
- Model: `gpt-4o-mini`

### Structured Output
- `status`
- `reasoning`

---

## Step 6: Update Tracker

### Update Status
- Matches by **URL**
- Updates **Status column**

Values:
- `Candidate`
- `Ignored`

### Preserve Data
Ensures all fields remain intact:
- url, title, source, publishDate, summary, status, reasoning

---

## Step 7: Send Summary Email

### Filter Candidates
- Only `Candidate` stories proceed  

### Aggregate
- Combine all candidates into one email  

### Email Content
- Title  
- Source  
- Summary  
- URL  
- AI reasoning  

---

# Part 2: Approval & Publishing Workflow

## Trigger

### Gmail Trigger
- Watches label: **cannabis-approvals**

### Setup
- Create Gmail label: `cannabis-approvals`
- Create filter:
  - Subject: *Cannabis Regulation News - Candidate Stories for Review*
  - Apply label

---

## Step 1: Extract Approval Data

Extract from email:
- `storyUrl`
- `approvalDecision`
- `reviewerNotes`

---

## Step 2: Find Story in Tracker

- Fetch all stories  
- Match by URL  

---

## Step 3: Update Status

- Update status → **Approved**

---

## Step 4: Prepare for WordPress

Prepare:
- title  
- source  
- summary  
- url  
- reviewerNotes  

---

## Step 5: Generate WordPress Draft

### AI Draft Generation
- Model: `gpt-4o-mini`

### Input:
- Title  
- Source  
- Summary  
- URL  
- Notes  

### Output:
- Full article draft  

---

## Step 6: Save to WordPress

- Create post as **Draft**
- Not published automatically  

---

## Step 7: Notify CEO

- Send email with draft link  
- CEO reviews and approves  

---

# Google Sheets Structure

## Sheet Name
**Story Tracker**

## Columns

- URL  
- Title  
- Source  
- PublishDate  
- Summary  
- DateFound  
- Status  

---

# Setup Requirements

## 1. Google Sheets
- Create sheet: **Story Tracker**
- Add columns
- Connect n8n credentials  

## 2. Gmail
- Create label: `cannabis-approvals`
- Setup filter  

## 3. GNews API
- Add API key in n8n  

## 4. OpenAI
- Configure API key  
- Used for:
  - relevance evaluation  
  - draft generation  

## 5. WordPress
- Enable API access  
- Configure credentials  
- Allow draft creation  

---

# Workflow Logic Summary

## Daily Flow