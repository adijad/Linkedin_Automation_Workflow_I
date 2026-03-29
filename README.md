# LinkedIn Automation Workflow - Phase 1

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




