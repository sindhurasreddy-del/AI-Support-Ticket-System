# AI-Support-Ticket-System
AI-powered support ticket processing pipeline using n8n, OpenAI GPT-4o-mini, and Qdrant — featuring RAG-based response generation, automated ticket classification, urgency routing, and bilingual (German/English) support for B2B clients.
NOAVIA AI-Powered Support Ticket Processing System
Overview
This project implements an AI-powered support ticket intake and processing pipeline using n8n, OpenAI, and Qdrant vector database. The system receives customer support tickets via webhook, classifies them using AI, retrieves relevant knowledge base articles via RAG (Retrieval-Augmented Generation), generates draft responses, and routes tickets based on urgency.
Built for NOAVIA — a German AI solutions company serving B2B clients across industries including Maschinenbau, Logistik, Chemie, Elektrotechnik, and IT.
---
Architecture
```
Webhook (POST) → Validation → PDF Check → AI Classification → RAG Draft Response → Urgency Routing → Google Sheets + Email + Webhook Response
```
System Components
Webhook Intake & Validation — Receives support tickets via POST request with fields: name, email, subject, message, and optional PDF attachment. Validates all required fields are present.
PDF Attachment Handling — If a PDF is attached, extracts text content and appends it to the message body before AI processing. Handles extraction failures gracefully.
AI Classification (Step 1) — Uses OpenAI GPT-4o-mini to classify each ticket, returning structured JSON with:
Category (billing, technical, account, product, contract, privacy, general)
Urgency (critical, high, medium, low)
Sentiment (angry, frustrated, neutral, satisfied)
Confidence score (0.0–1.0)
Brief English summary
RAG Component (Step 2) — Retrieves the top 3 most relevant knowledge base chunks from Qdrant vector database. An AI Agent uses the classification results and retrieved context to generate a professional draft response in the customer's language.
Urgency-Based Routing — Routes tickets based on urgency level:
Critical/High → Full email notification + Google Sheets logging
Medium → Brief email notification + Google Sheets logging
Low → Google Sheets logging only
Low confidence (<0.6) → Flagged as "needs-manual-review" regardless of urgency
Google Sheets Storage — Logs every ticket with columns: ticketId, timestamp, name, email, subject, category, urgency, sentiment, confidence, summary, draftResponse, status, output, message.
Error Handling — Key nodes (OpenAI, AI Agent, Gmail, Google Sheets) are configured with "Continue On Fail" to ensure the workflow doesn't stop on individual node failures.
---
Tech Stack
Component	Technology
Workflow Engine	n8n (self-hosted)
LLM	OpenAI GPT-4o-mini
Embeddings	OpenAI text-embedding-3-small
Vector Database	Qdrant (Docker)
Storage	Google Sheets
Email	Gmail API
Language	German / English (bilingual support)
---
Prerequisites
Docker Desktop installed and running
n8n installed locally
OpenAI API key with access to GPT-4o-mini and text-embedding-3-small
Google Cloud project with Gmail API, Google Sheets API, and Google Drive API enabled
Google OAuth2 credentials (Client ID + Client Secret)
---
Setup Instructions
Step 1 — Start Qdrant
```bash
docker run -p 6333:6333 qdrant/qdrant
```
Verify at: http://localhost:6333/dashboard
Step 2 — Prepare Knowledge Base Files
Place the 7 NOAVIA knowledge base markdown files in a folder accessible to n8n (e.g., `C:\Users\<user>\.n8n-files\knowledgebase\`):
`01-faq.md` — General company information
`02-contract-cancellation-policy.md` — Contract terms and cancellation
`03-billing-payments.md` — Payment methods, invoicing, Mahnung process
`04-products-services.md` — AI Agents, Workflow Automation, Consulting
`05-technical-troubleshooting.md` — Agent issues, webhook debugging, RAG quality
`06-pricing-plans.md` — Pilot/Standard/Enterprise tiers
`07-data-privacy-gdpr.md` — DSGVO, German hosting, AI data handling
Step 3 — Run the Ingestion Workflow
Open the "Knowledge Base Ingestion" workflow in n8n and execute it. This reads the markdown files, chunks them, generates embeddings via OpenAI, and stores them in the `noavia_knowledge_base` Qdrant collection.
Step 4 — Configure Credentials in n8n
Set up the following credentials in n8n:
OpenAI account — API key
Google Sheets account — OAuth2 (Client ID + Client Secret)
Gmail account — OAuth2 (Client ID + Client Secret)
Qdrant — URL: http://localhost:6333 (no API key needed for local)
Step 5 — Activate the Main Workflow
Open the "Support Ticket Processing" workflow and activate it. The webhook endpoint will be available at:
```
POST http://localhost:5678/webhook/support-ticket
```
---
Usage
Sending a Test Ticket (PowerShell)
```powershell
Invoke-WebRequest -Uri "http://localhost:5678/webhook/support-ticket" -Method POST -ContentType "application/json" -Body '{"name":"Thomas Mueller","email":"mueller@maschinenbau-mueller.de","subject":"Login Problem","message":"Ich kann mich nicht einloggen seit heute morgen. Bitte um dringende Hilfe."}'
```
Sending a Test Ticket (curl)
```bash
curl -X POST http://localhost:5678/webhook/support-ticket \
  -H "Content-Type: application/json" \
  -d '{"name":"Thomas Mueller","email":"mueller@maschinenbau-mueller.de","subject":"Login Problem","message":"Ich kann mich nicht einloggen seit heute morgen."}'
```
Request Body
Field	Type	Required	Description
name	string	Yes	Customer name
email	string	Yes	Customer email
subject	string	Yes	Ticket subject
message	string	Yes	Ticket message body
file	binary	No	Optional PDF attachment
Response (Success)
```json
{
  "status": "success",
  "ticketId": "TKT-1775668520413",
  "urgency": "critical",
  "category": "account",
  "message": "Your support ticket has been received and is being processed."
}
```
Response (Validation Error)
```json
{
  "error": "Missing required fields. Please provide name, email, subject, and message.",
  "status": "validation_failed"
}
```
---
Workflow Nodes
Main Workflow: Support Ticket Processing
#	Node	Purpose
1	Webhook	Receives POST requests with ticket data
2	If	Validates required fields (name, email, subject, message)
3	Respond to Webhook (error)	Returns 400 error on validation failure
4	Edit Fields	Generates ticketId, timestamp, organizes ticket data
5	If1	Checks if PDF attachment exists
6	Extract from File	Extracts text from PDF (TRUE branch)
7	Edit Fields3	Merges PDF text with original message
8	Message a Model	OpenAI GPT-4o-mini classifies ticket (category, urgency, sentiment, confidence, summary)
9	Edit Fields1	Parses JSON classification response
10	AI Agent	Generates draft response using RAG (Qdrant retrieval + OpenAI)
11	Edit Fields2	Captures draft response, merges all fields, sets status
12	Switch	Routes by urgency (critical/high/medium/low)
13	Send a message (Gmail)	Full email for critical/high tickets
14	Send a message1 (Gmail)	Brief email for medium tickets
15	Append row in sheet	Logs all tickets to Google Sheets
16	Respond to Webhook1	Returns success response with ticket details
Ingestion Workflow: Knowledge Base Ingestion
#	Node	Purpose
1	Manual Trigger	Manual execution trigger
2	Read/Write Files from Disk	Reads all .md files from knowledge base folder
3	Extract from File	Extracts text content from files
4	Qdrant Vector Store	Stores document chunks with embeddings in Qdrant
---
Knowledge Base
The RAG system uses 7 NOAVIA-specific knowledge base documents covering:
Company FAQ and contact information
Contract terms and cancellation policies
Billing, payments, SEPA, and Zahlungsziel handling
Product and service descriptions (AI Agents, Workflow Automation, Consulting)
Technical troubleshooting guides
Pricing plans (Pilot/Standard/Enterprise)
Data privacy, DSGVO compliance, and security measures
Documents reference actual customer data including industries (Maschinenbau, Logistik, Chemie), customer types (Stammkunden, Neukunden, Großkunden), and payment terms (14/21/30/45/60 Tage).
---
Dataset
The `customer.csv` file contains 25 German B2B customer records with fields:
Kundennr, Firma, Ansprechpartner, E-Mail, Telefon
Straße, PLZ, Ort, Branche
Kundenseit, Umsatz_2025, Zahlungsziel, Bemerkung
Includes intentional data quality issues for testing: duplicate record (10001), incomplete record (10017), and special characters (Özkan, Weiß, Nüßler, Großmann).
---
Error Handling
All critical nodes configured with "Continue On Fail" error handling
Validation errors return structured JSON with 400 status code
PDF extraction failures are handled gracefully (workflow continues without PDF text)
Low confidence scores (<0.6) automatically flag tickets as "needs-manual-review"
---
Project Structure
```
├── README.md                          # This file
├── customer.csv                       # Customer dataset
├── knowledge-base/                    # Knowledge base documents
│   ├── 01-faq.md
│   ├── 02-contract-cancellation-policy.md
│   ├── 03-billing-payments.md
│   ├── 04-products-services.md
│   ├── 05-technical-troubleshooting.md
│   ├── 06-pricing-plans.md
│   └── 07-data-privacy-gdpr.md
├── workflows/
│   ├── support-ticket-processing.json # Main workflow (export from n8n)
│   └── knowledge-base-ingestion.json  # Ingestion workflow (export from n8n)
└── docker-compose.yml                 # (Optional) Docker setup
```
---
Key Architecture Decisions
Two-step AI pipeline: Classification and response generation are separated into distinct nodes. This allows independent validation of the classification output before it feeds into the response generation step, and makes debugging easier.
AI Agent with Tool for RAG: Instead of a standalone Qdrant retrieval node, the RAG component is implemented as a Tool within n8n's AI Agent node. This lets the LLM decide the optimal search query for knowledge retrieval rather than using a fixed query, resulting in more relevant context retrieval.
GPT-4o-mini for all LLM calls: Chosen for its balance of cost efficiency, speed, and quality. For a support ticket system processing many tickets, keeping costs low while maintaining good classification accuracy is essential.
Single Google Sheets node: All urgency branches merge into one Google Sheets node to avoid data inconsistency and simplify maintenance. Routing logic handles email notifications before the shared storage step.
German-language knowledge base: Since NOAVIA serves German B2B clients, the knowledge base documents are written in German with German business terminology (Zahlungsziel, Mahnung, DSGVO, etc.) to ensure RAG retrieval matches real customer queries.
---
AI Output Validation Approach
The classification step (Step 1) uses a strict system prompt that instructs the LLM to return ONLY a valid JSON object with predefined categories and value ranges. The JSON response is then parsed in an Edit Fields node with explicit field extraction. Key validation measures:
Structured JSON output: The system prompt enforces a strict schema with enumerated values for category, urgency, and sentiment.
Markdown stripping: The parsing expressions strip markdown code block fences (`json ... `) before JSON.parse, handling cases where the LLM wraps output in code blocks.
Confidence threshold: Tickets with confidence < 0.6 are automatically flagged as "needs-manual-review" regardless of the urgency classification.
Continue On Fail: All critical nodes use error continuation so a parsing failure in one step doesn't crash the entire pipeline.
---
RAG Implementation Details
Chunking Strategy: Documents are split using n8n's Default Data Loader with simple text splitting (1000 characters per chunk, 200 character overlap). This provides sufficient context per chunk while ensuring overlap preserves information at chunk boundaries. For the knowledge base size (7 documents, ~1-2 pages each), this produces approximately 22 chunks — manageable and effective.
Embedding Model: OpenAI text-embedding-3-small (1536 dimensions) was chosen for its strong multilingual performance (critical for German-language documents), low cost per embedding, and high retrieval quality. It handles German compound words and technical terminology well.
Retrieval Approach: The Qdrant Vector Store is configured as a Tool for the AI Agent, retrieving the top 3 most relevant chunks per query. The AI Agent's system prompt instructs it to always use the knowledge base tool before responding, and to note when no specific policy is found. Include Metadata is enabled so the agent can reference source documents.
Vector Database: Qdrant was chosen per the project requirements, running locally via Docker on port 6333. The collection `noavia_knowledge_base` stores all document chunks with their embeddings.
---
What I Would Improve With More Time
Explicit knowledge source tracking: Add a dedicated field in Google Sheets that logs which specific knowledge base documents were retrieved and their similarity scores, rather than relying on the AI to mention them in the draft response.
Processing log column: Implement a structured processing log that records timestamps and outcomes for each step (validation passed, classification result, RAG chunks retrieved, email sent, etc.).
Better chunking strategy: Implement semantic chunking that respects document structure (headings, sections) rather than fixed-size splitting, which would improve retrieval quality for structured documents like policies and FAQs.
Input sanitization: Add more robust input validation — HTML/script injection prevention, message length limits, and email domain validation against the customer database.
Customer lookup: Cross-reference incoming ticket emails against the customer.csv database to enrich tickets with customer information (company, industry, revenue tier, payment terms) and prioritize accordingly.
Retry logic: Add retry mechanisms for API calls (OpenAI, Gmail) with exponential backoff instead of just "Continue On Fail".
Monitoring dashboard: Build a real-time dashboard showing ticket volume, urgency distribution, average confidence scores, and RAG retrieval quality metrics.
Multi-language support: Detect the customer's language automatically and ensure both classification and response generation match the detected language.
---
Author
Built as part of the NOAVIA AI Agent Engineer interview assessment.
