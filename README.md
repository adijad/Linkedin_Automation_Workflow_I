# LinkedIn Automation Workflow - Phase 1

This project represents the first phase of a LinkedIn automation system designed to discover companies, clean and deduplicate them, and enrich them with relevant people data. The workflow is built using n8n, with Google Sheets acting as the central storage and orchestration layer.

---

# Overview
This repository contains the first phase of a larger LinkedIn automation system built to support structured company discovery and downstream people enrichment for targeted outreach.

At a high level, this phase does four things:
- Scrapes startup/company information from a source endpoint.
- Normalizes and structures the company data into a consistent schema.
- Checks for duplicates against existing records in Google Sheets.
- Triggers controlled employee/person enrichment for each company and stores the results in a separate sheet.

The workflow is implemented in n8n and uses Google Sheets as the system of record for orchestration, intermediate persistence, and downstream automation.

---

# Architecture Diagram

The core Phase 1 workflow is shown below:
![Linkedin_Automation_Workflow_I](./Linkedin%20automation%20I.png)

---

# High-Level Flow

The workflow follows this sequence:

## 1. Manual execution trigger
The flow begins with a manual n8n trigger:
- When clicking Execute workflow
This is useful during development because it allows controlled testing of the pipeline without depending on a schedule or webhook.

## 2. Scrape company information
The first HTTP node calls a scraping/API layer that returns startup/company information.
- HTTP Request to scrape the company information
This step is responsible for retrieving raw company records from the upstream source.
The code for this step is available at: https://github.com/adijad/Fastapi_Startups_Workflow

## 3. Structure and normalize company data
The raw company payload is then passed through a code node:
- Code to structure the company information

This step typically:
- extracts only the fields needed downstream
- standardizes the schema
- normalizes text values
- prepares company records for sheet insertion and duplicate comparison

## 4. Read existing company rows
Before inserting anything new, the workflow reads the current contents of the company sheet:
- Get the current rows in the sheet
This acts as the reference state for deduplication.

## 5. Deduplication logic
The structured company records and the existing sheet rows are compared in a code node:
- Code to check for duplicates
This is one of the most important steps in the architecture.

The deduplication layer prevents:
- re-appending companies that already exist in the sheet
- adding duplicate records within the same incoming batch
- accidental reprocessing of identical or near-identical companies

This step usually outputs one of two outcomes:
- new unique companies found
- all incoming companies are duplicates

## 6. Branch on uniqueness
The workflow then uses an If node to decide what happens next.

Branch A - unique companies found
If new companies are detected:
- Uniques found, get the rows to append
- Append company data in the sheet
This path prepares only the truly new rows and writes them into the company sheet.

After appending, the workflow reads the sheet again:
- Get all the rows in the sheet
This refresh is important because the downstream loop should operate on the latest persisted state rather than on stale pre-insert data.

Branch B - duplicates found
If the incoming records already exist:
- Duplicates found, don't append, just get the existing rows. 
In this branch, the workflow skips insertion and simply continues using the already available sheet records.

Architecturally, this is a strong design choice because it prevents the pipeline from failing or going empty just because nothing new was inserted. Instead, it preserves continuity and allows downstream stages to continue operating from the sheet state.

## 7.Controlled per-company processing loop
Once the workflow has a stable set of company rows, it enters the looping stage:
- Loop Over Items, one at a time
This is where the architecture becomes especially important.

Rather than sending bulk employee-enrichment requests in parallel, the flow processes companies one at a time. That gives several advantages:
- easier debugging
- better control over request ordering
- lower risk of rate limits or anti-bot triggers
- simpler retry behavior
- clearer association between a company row and the employee records generated from it

This loop effectively turns the sheet into a queue of companies to enrich.

## 8.Delay and throttling layer

Before making the people/employee lookup request, the workflow inserts a delay stage:
- Code to generate a random delay
- Wait

This is a reliability feature, not just a convenience.

The delay layer helps by:
- reducing back-to-back request bursts
- introducing more human-like request spacing
- lowering the chance of rate limiting from downstream services
- making long-running automations more stable

Using a randomized delay instead of a fixed wait is usually better for external scraping/enrichment systems because it avoids highly regular traffic patterns.

## 9.Employee Enrichment Stage
For each company in the loop, the workflow calls an employee/people lookup service.
- Triggered via an HTTP request from n8n
- Connected to a backend service running locally

This stage is responsible for retrieving people associated with the company, typically leadership or hiring-relevant contacts.

### Playwright-Based LinkedIn Search
The enrichment service uses Playwright to automate LinkedIn search:
- Launches a controlled Chromium browser session
- Navigates LinkedIn using search queries derived from company data
- Extracts relevant people cards (e.g., founders, engineers, recruiters)
- Simulates real user behavior instead of direct API calls

This approach enables more flexible and realistic data extraction compared to static APIs.

### Rate Limiting and Anti-Bot Design
A delay is intentionally introduced before each enrichment request:
- Prevents near-synchronous requests to LinkedIn
- Reduces the risk of triggering anti-bot detection
- Mimics natural human browsing patterns

This is critical for maintaining reliability when running repeated searches.

### Local <-> Cloud Connectivity
Since n8n runs in the cloud and the Playwright service runs locally:
- Cloudflare Tunnel is used to expose the local backend
- Allows secure communication between n8n and the local Playwright service
- Eliminates the need for direct public deployment
- Roles Extracted

Depending on the search results, this stage retrieves roles such as:
- Founder / Co-founder
- CEO / CTO
- VP Engineering / Head of Engineering
- Engineering Manager
- Recruiter / Talent Acquisition

The code for this step is available at: https://github.com/adijad/Fastapi_Startups_Resolver

---

# Design principles behind the architecture

1. Google Sheets as the operational database
For this phase, Google Sheets acts as a lightweight control plane.
That is a practical choice because it gives:
- easy inspection of records
- fast debugging during iteration
- low setup overhead
- clean handoff into non-engineering workflows if needed
- a visible source of truth for company and people records

While not a replacement for a production database, Sheets is very effective for an early-stage automation pipeline.

2. Separation between company ingestion and people enrichment
The workflow does not mix raw scraping logic, dedupe logic, and people extraction into one opaque block.
Instead, it separates them into clear phases:
- fetch company data
- normalize it
- persist only unique records
- loop through persisted companies
- enrich people data
- persist employee records

This separation makes the system easier to maintain and extend.

3. Persist-first, then enrich
One subtle but strong architectural decision is that company data is persisted before large-scale downstream processing begins.
That means:
- the system has a durable checkpoint
- retries do not depend entirely on in-memory execution
- downstream failures do not erase upstream progress
- the company sheet becomes a reusable source for future workflows

4. One-at-a-time looping for reliability
Serial processing is slower than full parallelization, but in this context it is often the correct tradeoff.
For enrichment pipelines that depend on external APIs, browser automation, or scraping infrastructure, controlled sequential execution is usually more reliable than aggressive fan-out.

5. Built-in protection against empty-flow failures
The duplicate branch ensures the workflow can continue even when no new companies are appended.
That is important because many automation failures happen when a branch produces zero items and downstream nodes unexpectedly stop. This architecture explicitly avoids that failure mode.

---

# Future Work
The second phase of this system is currently in progress and builds directly on the outputs of this workflow.
- Use the Company and Employee sheets as input for downstream automation
- Generate personalized LinkedIn connection notes for identified people
- Automate job discovery within target companies
- Combine company context + role + person data for better personalization
- Track outreach status and responses in a structured manner

This phase aims to transition the system from data collection → actionable outreach, completing the pipeline from discovery to engagement.

---

