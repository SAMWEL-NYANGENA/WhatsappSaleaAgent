# WhatsApp AI Sales & Order-Taking Workflow (n8n)

## Overview

This workflow is a **production-grade WhatsApp AI sales assistant** built in **n8n**, designed to:

- Respond to customer messages on WhatsApp
- Show available products and prices from Google Sheets
- Take, confirm, update, and cancel orders
- Answer business-related questions via a RAG (Retrieval-Augmented Generation) knowledge base
- Detect low-confidence or human-requested scenarios
- Automatically **escalate conversations to a human agent** when needed

It combines **LLM reasoning**, **strict tool usage**, **confidence scoring**, **chat memory**, and **human-handoff logic** into a single clean automation.

This is not a demo toy. It is structured for real businesses.

---

## High-Level Architecture

**Flow summary:**

1. WhatsApp message arrives
2. AI Agent processes intent using tools only
3. AI response is analyzed for confidence or escalation
4. Either:
   - AI replies to the customer, or
   - Conversation is escalated to a human agent

A visual overview of the workflow is provided in `n8n.png`.

---

## Core Components

### 1. WhatsApp Trigger

**Node:** `WhatsApp Trigger1`

- Entry point of the workflow
- Listens for incoming WhatsApp messages via Meta Cloud API
- Extracts:
  - Customer phone number
  - Message text
  - Metadata (phone_number_id, contact name)

This node is pinned for testing but should be unpinned in production.

---

### 2. AI Sales Agent (LangChain)

**Node:** `supabase bot1`

This is the brain of the system.

#### Responsibilities
- Understand customer intent
- Use tools deterministically
- Never hallucinate products, prices, or orders
- Follow a strict order-taking protocol
- Escalate only when rules demand it

#### Tools Available to the Agent

| Tool | Purpose |
|----|----|
| `search goods` | Read in-stock products from Google Sheets |
| `record order` | Create a new order (after confirmation only) |
| `update orders` | Modify an existing order |
| `delete order` | Cancel an order (with confirmation) |
| `RAG tool` | Retrieve business info (policies, hours, support, etc.) |

The agent **cannot** respond freely. It must act through tools or output plain text.

---

### 3. Language Model

**Node:** `OpenAI Chat Model1`

- Uses `gpt-4o-mini`
- Provides reasoning and response generation
- No direct access to WhatsApp or data sources

---

### 4. Chat Memory (PostgreSQL)

**Node:** `Postgres Chat Memory1`

- Stores the last 10 messages per customer
- Uses the WhatsApp phone number as session key
- Enables multi-turn conversations:
  - Order follow-ups
  - Corrections
  - Clarifications

---

### 5. Product & Order Management (Google Sheets)

**Tools:**

- `search goods`
- `record order`
- `update orders`
- `delete order`

#### Sheets Used

- **Products Sheet**
  - product_id
  - product_name
  - category
  - description
  - price
  - currency
  - in_stock
  - stock_qty
  - unit

- **Orders Sheet**
  - order_id
  - customer_name
  - email
  - product_id
  - product_name
  - qty
  - price
  - total
  - status

The AI only reads products marked `in_stock = YES`.

---

### 6. RAG Knowledge Base (Supabase)

**Node:** `RAG tool`

- Vector search over business documents
- Used only for:
  - Policies
  - Customer support info
  - Operating hours
  - Delivery & return rules

If the RAG tool returns nothing, the AI **must escalate**.

---

### 7. Confidence & Escalation Analysis

**Node:** `Analyze Confidence`

This node is critical.

#### What it does
- Reads the AI’s raw output
- Detects if the AI output is exactly:
- - Assigns a confidence score
- Cleans responses so `ESCALATE` never reaches the user
- Outputs routing flags

#### Escalation Conditions
- AI explicitly outputs `ESCALATE`
- Confidence score drops below threshold
- User asks for:
- Human agent
- Customer support
- Manager
- Representative

---

### 8. Escalation Decision

**Node:** `Should Escalate?`

- Simple IF node
- Checks `shouldEscalate = true`

#### TRUE path
1. Notify customer they’re being transferred
2. Alert human agent via WhatsApp with:
 - Customer number
 - Original message

#### FALSE path
- Send AI response directly to the customer

---

### 9. WhatsApp Messaging

**Nodes:**
- `Send AI Response`
- `Send Escalation Message`
- `Notify Human Agent`

These nodes handle all outbound WhatsApp communication.

---

## How to Use This Workflow

### 1. Import the Workflow

1. Open n8n
2. Click **Import from File**
3. Select `workflow.json`
4. Save

---

### 2. Configure Credentials

You must configure the following credentials:

- WhatsApp Cloud API (Trigger + Send)
- OpenAI API
- Google Sheets OAuth
- PostgreSQL
- Supabase

Ensure all credential names match the workflow nodes or update them accordingly.

---

### 3. Prepare Google Sheets

Ensure your spreadsheet has:

- A **Products** sheet with correct columns
- An **Orders** sheet for order storage

Only products with `in_stock = YES` will be shown.

---

### 4. Populate the RAG Knowledge Base

- Insert business documents into the Supabase `documents` table
- Generate embeddings using the provided embeddings node
- Store:
- Policies
- FAQs
- Support information

---

### 5. Test the Flow

Recommended test messages:

- “Hi”
- “What do you sell?”
- “How much is item X?”
- “I want to order 2 of item Y”
- “Yes” (to confirm order)
- “Cancel my order”
- “I want to talk to customer support”

---

### 6. Activate the Workflow

Once verified:
- Remove pinned trigger data
- Activate the workflow
- Connect your WhatsApp number

---

## Design Principles

- Deterministic over creative
- Tools over hallucination
- Escalate early rather than guess
- One order at a time
- No silent failures

---

## What This Workflow Is Good For

- WhatsApp commerce
- Small business sales bots
- MVPs for customer support automation
- AI-assisted order processing
- Human-in-the-loop AI systems

---

## What It Intentionally Does NOT Do

- No fake products
- No guessed prices
- No auto-orders without confirmation
- No pretending to be human
- No uncontrolled LLM replies

---

## Files Included

- `workflow.json` – n8n workflow export
- `n8n.png` – Visual overview of the workflow canvas
- `README.md` – This document

---

## Final Note

This workflow is built to be **auditable, extensible, and safe**.
You can plug in different models, replace Sheets with a database, or extend the escalation logic — without breaking the core guarantees.

If you’re building serious WhatsApp automation, this is a solid foundation.

