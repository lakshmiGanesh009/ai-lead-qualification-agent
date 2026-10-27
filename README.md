# AI Agency Communication & Lead Intelligence Platform

> **A cost-aware communication management system for AI/software
> agencies.**\
> Capture messages from multiple channels, classify the intent before
> invoking an LLM, route existing-client support and enquiries to the
> right queues, and use LLMs selectively for genuine sales-lead
> qualification.

```{=html}
<p align="center">
```
`<strong>`{=html}Capture → Persist → Classify → Route → Qualify (when
eligible) → Track`</strong>`{=html}
```{=html}
</p>
```

------------------------------------------------------------------------

## Table of Contents

-   [Overview](#overview)
-   [Problem Statement](#problem-statement)
-   [Current Situation](#current-situation)
-   [Project Goals](#project-goals)
-   [Core Design Principles](#core-design-principles)
-   [Message Categories and Routing](#message-categories-and-routing)
-   [System Architecture](#system-architecture)
-   [End-to-End Workflow](#end-to-end-workflow)
-   [Technology Stack](#technology-stack)
-   [Data Model](#data-model)
-   [LLM Cost-Control Strategy](#llm-cost-control-strategy)
-   [Reliability, Security, and Human
    Review](#reliability-security-and-human-review)
-   [Metrics and Evaluation](#metrics-and-evaluation)
-   [Implementation Roadmap](#implementation-roadmap)
-   [Proposed Repository Structure](#proposed-repository-structure)
-   [Local Development](#local-development)
-   [Environment Configuration](#environment-configuration)
-   [API Design](#api-design)
-   [Testing Strategy](#testing-strategy)
-   [Deployment Strategy](#deployment-strategy)
-   [Limitations and Future
    Enhancements](#limitations-and-future-enhancements)
-   [Project Status](#project-status)

------------------------------------------------------------------------

## Overview

AI and software agencies receive communications from prospective
customers and existing clients through email, WhatsApp, website forms,
Google Forms, and other channels. These messages may describe a new
project, request pricing, ask a general question, report a production
issue, request a code change, or ask for help with an already-delivered
solution.

Treating every message as a sales lead creates poor routing, unnecessary
LLM usage, delayed support, and incomplete records. This project
proposes a unified platform that stores every incoming message,
identifies its likely intent using low-cost checks first, and routes it
to the appropriate workflow.

**The LLM is not the universal message classifier.** It is reserved
primarily for qualifying messages that have been identified as potential
sales opportunities. Support issues, routine doubts, and general
enquiries follow their own workflows and do not enter the
lead-qualification LLM pipeline by default.

### Product vision

Build a reliable, auditable communication hub that helps an AI agency:

-   Keep a central record of incoming communications.
-   Distinguish new sales opportunities from existing-client support and
    general enquiries.
-   Avoid losing messages when an integration or downstream process
    fails.
-   Reduce unnecessary model calls and control AI operating costs.
-   Give sales and delivery teams clear ownership, status, and next
    actions.
-   Measure routing quality, lead conversion, support handling, and
    system reliability.

## Problem Statement

Agencies often manage communications across disconnected inboxes and
forms. The same customer may contact the team on WhatsApp, follow up by
email, and later report an issue with a delivered project. Without a
shared context, the team may create duplicate records, mistake support
requests for sales leads, miss a potential expansion opportunity, or
fail to respond on time.

A single message may also contain more than one intent. For example:

> "The chatbot is failing after the latest deployment. Once that is
> fixed, we also want to discuss adding a voice agent."

This contains a support issue and a potential new sales opportunity. A
reliable system must preserve both intents rather than forcing the
message into one oversimplified category.

The proposed platform uses a **persist-first, classify-second,
route-third** architecture. Deterministic rules and existing
customer/project context handle straightforward cases. Only eligible
sales messages proceed to LLM-based lead qualification. Uncertain cases
are reviewed or clarified rather than silently discarded.

## Current Situation

### Typical manual process

1.  A message arrives in an email inbox, WhatsApp account, website form,
    or spreadsheet.
2.  A team member reads it and decides whether it is a lead, a support
    request, or a general question.
3.  The message is copied into a CRM, spreadsheet, project tracker, or
    chat.
4.  Sales enquiries are manually assessed for service fit, budget,
    urgency, and timeline.
5.  Support requests are forwarded to the developer or delivery team.
6.  Follow-ups and status updates depend on individual reminders.
7.  Managers manually compile reports on response time, conversions, and
    unresolved issues.

### Key inefficiencies

-   Messages and customer history are scattered across tools.
-   Existing-client issues can be misclassified as sales opportunities.
-   Potential new projects from existing clients can be missed.
-   Duplicate messages can create duplicate leads or tickets.
-   Applying an LLM to every incoming message increases cost without
    adding value to routine routing.
-   Failures in integrations may cause silent message loss.
-   Unclear ownership and follow-up deadlines make it difficult to
    measure performance.

## Project Goals

1.  **Unified ingestion:** receive communications through pluggable
    channel adapters.
2.  **Durable records:** persist the original message and source
    metadata before downstream processing.
3.  **Cost-aware routing:** use sender history, project context,
    deterministic rules, and lightweight checks before any LLM call.
4.  **Separate workflows:** route sales leads, support issues, change
    requests, general enquiries, and ambiguous messages independently.
5.  **Selective LLM qualification:** extract structured business
    requirements only for messages eligible for lead qualification.
6.  **Traceability:** retain classification reasons, processing events,
    status changes, and relevant evidence.
7.  **Human control:** support review and approval for uncertain
    classifications and consequential outbound actions.
8.  **Operational visibility:** track message processing, unresolved
    work, model usage, response times, and outcomes.

## Core Design Principles

### 1. Persist before processing

Store the incoming event and its source identifier before attempting
classification. If classification fails, the message remains available
for retry or review.

### 2. Rules first, LLM selectively

Known customer/project context and deterministic rules should handle
obvious cases. Model calls are allowed only after the application marks
a message as eligible for lead qualification.

### 3. Classification is not qualification

-   **Classification** answers: "Which workflow should handle this
    message?"
-   **Lead qualification** answers: "How suitable and actionable is this
    sales opportunity?"

These are separate operations with separate responsibilities.

### 4. Preserve multiple intents

A message can create more than one work item. For example, a bug report
and a request for a new paid feature may need separate support and sales
workflows linked to the same original message.

### 5. Uncertainty must be explicit

Do not invent a budget, infer a confirmed buying decision, or silently
force an unclear message into a category. Use `needs_review`, an
explicit confidence/reason field, or a clarification workflow.

### 6. Idempotent processing

Reprocessing the same provider event must not create duplicate records
or repeat external side effects.

### 7. Human oversight for consequential actions

Drafting a response can be automated, but sending messages, committing
to timelines/prices, changing client data, or closing important work
must follow configured authorization and approval rules.

------------------------------------------------------------------------

## Message Categories and Routing

The initial taxonomy is designed for an AI/software agency. Categories
and rules should be configurable rather than permanently hard-coded.

  ---------------------------------------------------------------------------------------
  Primary category       Example           Destination               Lead-qualification
                                                                     LLM by default?
  ---------------------- ----------------- ------------------------- --------------------
  `sales_lead`           "We need an AI    Lead qualification and    **Yes, if eligible**
                         agent connected   sales queue               
                         to our CRM. Can                             
                         you share a                                 
                         proposal?"                                  

  `support_issue`        "The chatbot      Support/issue queue       No
                         returns a 500                               
                         error after                                 
                         deployment."                                

  `change_request`       "Can you add CSV  Delivery/change-request   No
                         export to our     queue; optionally sales   
                         existing          review if commercial      
                         dashboard?"       scope is unclear          

  `general_enquiry`      "What services    Enquiry queue or approved No by default
                         does your agency  FAQ workflow              
                         offer?"                                     

  `project_question`     "How does the     Project/client enquiry    No
                         document          queue                     
                         retrieval feature                           
                         work?"                                      

  `mixed_intent`         "Fix the chatbot  Create linked support and Only the sales work
                         and quote for a   sales work items          item, if eligible
                         new voice agent."                           

  `unclear`              "Hi, I need       Clarification or human    No by default
                         help."            review                    

  `spam_or_irrelevant`   Unrelated         Spam/review queue         No
                         solicitation or   according to policy       
                         obvious spam                                
  ---------------------------------------------------------------------------------------

### Important routing rules

-   A known client is not automatically a support case. Existing
    customers can request new paid services.
-   A new sender is not automatically a sales lead. New contacts may ask
    general questions or request support.
-   Keywords are signals, not proof of intent. Negation, context, sender
    history, and the complete message must be considered.
-   A message with multiple intents should create multiple linked work
    items when appropriate.
-   Uncertain messages must remain visible in a review queue.
-   `llm_eligible` is an application-controlled permission flag. The LLM
    client must enforce it; it must not trust a value supplied by an
    external sender.

## System Architecture

``` mermaid
flowchart TD
    subgraph Sources["1. Communication Sources"]
        EMAIL["Email"]
        WA["WhatsApp Business"]
        WEB["Website Forms"]
        GOOGLE["Google Forms"]
        OTHER["CRM / Manual Entry / Other Adapters"]
    end

    subgraph Ingestion["2. Ingestion and Durable Storage"]
        ADAPTER["Channel Adapters and Webhook/API Validation"]
        NORMALIZE["Normalize Message and Source Metadata"]
        STORE["Persist Raw Message + Provider Event ID"]
        DB[("PostgreSQL: Messages, Contacts, Projects, Events")]
        ADAPTER --> NORMALIZE --> STORE --> DB
    end

    EMAIL --> ADAPTER
    WA --> ADAPTER
    WEB --> ADAPTER
    GOOGLE --> ADAPTER
    OTHER --> ADAPTER

    subgraph Routing["3. Cost-Aware Classification"]
        CONTEXT["Resolve Contact and Project Context"]
        RULES["Deterministic Rules and Lightweight Checks"]
        DECISION{"Classification Decision"}
        REVIEW["Unclear / Low Confidence Review"]
        CONTEXT --> RULES --> DECISION
    end

    DB --> CONTEXT

    subgraph Workflows["4. Specialized Workflows"]
        LEAD["Potential Sales Lead"]
        SUPPORT["Support Issue / Bug"]
        CHANGE["Change Request"]
        ENQUIRY["General Enquiry / Project Question"]
        MIXED["Mixed Intent: Split into Linked Work Items"]
        SPAM["Spam / Irrelevant"]
    end

    DECISION -->|Sales eligible| LEAD
    DECISION -->|Support| SUPPORT
    DECISION -->|Change request| CHANGE
    DECISION -->|General question| ENQUIRY
    DECISION -->|Multiple intents| MIXED
    DECISION -->|Spam| SPAM
    DECISION -->|Uncertain| REVIEW

    subgraph LeadAI["5. Selective Lead Qualification"]
        GATE["Application-Enforced LLM Eligibility Gate"]
        LLM["LLM: Structured Requirement Extraction and Summary"]
        VALIDATE["Validate Output and Evidence"]
        SCORE["Python Qualification Rules and Score"]
        HANDOFF["Sales Queue / Follow-up Task"]
        GATE --> LLM --> VALIDATE --> SCORE --> HANDOFF
    end

    LEAD --> GATE
    MIXED --> GATE

    subgraph Operations["6. Operations and Observability"]
        HUMAN["Human Review / Approval"]
        TASKS["Queues, Owners, Status and Due Dates"]
        AUDIT["Audit Events, Retry Queue and Failure Alerts"]
        DASH["Dashboard and Metrics"]
    end

    SUPPORT --> TASKS
    CHANGE --> TASKS
    ENQUIRY --> TASKS
    REVIEW --> HUMAN
    SPAM --> HUMAN
    HANDOFF --> TASKS
    DB --> AUDIT
    TASKS --> AUDIT
    AUDIT --> DASH
    HUMAN --> TASKS
```

### Architecture walkthrough

1.  **Communication sources** deliver messages through channel-specific
    adapters.
2.  **Ingestion** validates the event, normalizes its fields, and
    persists the original message and provider event ID.
3.  **Context resolution** looks up the sender, known customer, project,
    and prior interactions where available.
4.  **Low-cost classification** applies deterministic rules and
    lightweight checks. It does not call the lead-qualification LLM.
5.  **Workflow routing** sends each intent to the correct queue.
    Mixed-intent messages can generate multiple linked work items.
6.  **Selective lead qualification** runs only for messages that pass
    the application's eligibility gate.
7.  **Operational controls** record status changes, retries, approvals,
    and outcomes so the team can investigate failures and measure
    performance.

The diagram describes the intended architecture. External channel
integrations and deployment components should be marked as implemented
only after they have been built and tested.

## End-to-End Workflow

### Step 1 --- Receive and persist

-   Accept a message from a configured source.
-   Validate the payload and authenticate provider webhooks where
    supported.
-   Store the original text, sender reference, channel, provider event
    ID, and timestamps.
-   Acknowledge the source only after the event has been durably
    accepted, following the provider's delivery contract.

### Step 2 --- Normalize and deduplicate

-   Normalize whitespace and metadata without destroying the original
    message.
-   Use provider event IDs or idempotency keys to identify repeated
    deliveries.
-   Resolve contacts using verified identifiers and explicit matching
    rules.
-   Do not merge people based on name similarity alone.

### Step 3 --- Resolve context

-   Check whether the sender is associated with a known contact, client,
    or project.
-   Retrieve relevant project status and recent interaction metadata.
-   Treat missing history as "unknown," not as proof that the sender is
    a new prospect.
-   Apply access controls before exposing project or customer
    information.

### Step 4 --- Classify intent

-   Evaluate explicit patterns, configured rules, sender context, and
    message signals.
-   Record the selected category, subcategory, reason codes, rule
    version, and classification method.
-   Support mixed intent and an explicit review state.
-   Keep the original message available for staff verification.

### Step 5 --- Route to the correct workflow

-   Sales candidates go to the lead qualification eligibility gate.
-   Bugs and operational failures go to the support queue.
-   Code changes and enhancements go to the change-request queue.
-   General doubts and project questions go to the enquiry queue.
-   Unclear messages go to review or clarification.
-   Spam is handled according to a configurable policy.

### Step 6 --- Qualify eligible leads

The LLM may extract a structured record containing:

-   Requested service
-   Business problem
-   Company and industry, when stated or verified
-   Budget, if explicitly provided
-   Timeline and urgency
-   Required integrations
-   Missing information
-   Concise conversation summary

The application validates the result. A separate Python rules engine
calculates the qualification score using configured criteria. Missing
information remains unknown rather than being fabricated.

### Step 7 --- Assign ownership and next actions

-   Assign a responsible team or owner.
-   Set the status, priority, and due date.
-   Draft follow-up questions only when relevant.
-   Require approval when the action policy demands it.
-   Track delivery, replies, opt-outs, and failed actions.

### Step 8 --- Learn from reviewed outcomes

-   Let staff correct incorrect categories.
-   Record confirmed labels and final outcomes.
-   Evaluate rule performance on reviewed examples.
-   Update rules only through tested, versioned changes.
-   Consider a lightweight trained classifier only when enough
    representative, labeled data is available.

------------------------------------------------------------------------

## Technology Stack

The following is the proposed stack for the initial production-oriented
implementation. These are architectural choices, not claims that every
component is already implemented.

  -----------------------------------------------------------------------
  Layer                   Technology              Why it is used
  ----------------------- ----------------------- -----------------------
  Backend API             **Python + FastAPI**    Type hints, request
                                                  validation,
                                                  asynchronous endpoints,
                                                  automatic OpenAPI
                                                  documentation, and
                                                  straightforward
                                                  integration with Python
                                                  AI libraries.

  Data validation         **Pydantic**            Defines validated
                                                  message, contact,
                                                  classification, and
                                                  qualification schemas.
                                                  Helps reject malformed
                                                  data and validate model
                                                  output.

  Primary database        **PostgreSQL**          Stores messages,
                                                  contacts, projects,
                                                  classifications, work
                                                  items, and audit events
                                                  with relational
                                                  integrity and
                                                  transactions.

  Database access and     **SQLAlchemy +          Provides structured
  migrations              Alembic**               database access and
                                                  version-controlled
                                                  schema migrations.

  First-stage classifier  **Python rules engine** Cheap, deterministic
                                                  routing for obvious
                                                  cases. Easy to test,
                                                  explain, and version
                                                  without a model call.

  Lead qualification      **LLM provider behind   Extracts structured
                          an adapter**            business requirements
                                                  and summarizes eligible
                                                  sales opportunities.
                                                  The provider can be
                                                  changed without
                                                  rewriting routing
                                                  logic.

  Workflow orchestration  **LangGraph, when       Useful for branching,
                          stateful workflows      persisted workflow
                          justify it**            state, retries, and
                                                  human approval in
                                                  multi-step lead
                                                  qualification. It is
                                                  not required for the
                                                  initial deterministic
                                                  router.

  Background jobs         **Celery + Redis, or a  Runs slower
                          managed queue**         integrations, retries,
                                                  and scheduled
                                                  follow-ups outside
                                                  request-response
                                                  handling. Start with
                                                  one queue approach;
                                                  avoid adding
                                                  infrastructure before
                                                  it is needed.

  Frontend                **React + TypeScript**  Builds the
                                                  communication inbox,
                                                  lead/support queues,
                                                  contact timeline,
                                                  filters, and
                                                  operational dashboard.

  Channel adapters        **Provider APIs and     Connects email,
                          webhooks**              WhatsApp Business,
                                                  website forms, and
                                                  Google Forms through
                                                  supported integration
                                                  mechanisms.

  Authentication          **OAuth2/JWT with       Protects API routes and
                          role-based access       restricts access to
                          control**               sales, support, and
                                                  administrative actions.
                                                  Use provider OAuth for
                                                  integrations that
                                                  require it.

  Testing                 **pytest**              Tests classification
                                                  rules, edge cases, API
                                                  behavior, persistence,
                                                  retries, and LLM
                                                  eligibility
                                                  enforcement.

  LLM observability       **Langfuse or           Tracks eligible model
                          LangSmith**             calls, latency, token
                                                  usage, errors, and
                                                  evaluation results.
                                                  Choose one initially
                                                  and configure
                                                  privacy-safe tracing.

  Application             **Structured logging +  Helps detect processing
  observability           metrics and error       failures, queue delays,
                          tracking**              integration outages,
                                                  and unusual retry
                                                  rates.

  Packaging               **Docker + Docker       Provides reproducible
                          Compose**               local development and
                                                  consistent packaging of
                                                  the API, worker, and
                                                  supporting services.

  CI/CD                   **GitHub Actions**      Runs linting, tests,
                                                  security checks, and
                                                  build validation on
                                                  pull requests; deploys
                                                  through controlled
                                                  environments.

  Cloud deployment        **AWS**                 Provides a path to
                                                  managed compute,
                                                  PostgreSQL, secrets,
                                                  logs, object storage,
                                                  and queue services as
                                                  deployment needs grow.

  Secrets management      **Environment variables Keeps credentials out
                          locally; AWS Secrets    of source control and
                          Manager or equivalent   supports controlled
                          in production**         secret rotation.
  -----------------------------------------------------------------------

### Why PostgreSQL first?

PostgreSQL is a strong starting point because the main data is
structured and relational: messages belong to contacts, projects belong
to clients, and work items reference their source messages. Transactions
and unique constraints help prevent inconsistent records and duplicate
ingestion.

`pgvector` is optional. It should be added only if semantic retrieval
over historical conversations or approved knowledge becomes a concrete
requirement. It is not necessary for the first-stage rule-based
classifier.

### Why not use an LLM for every message?

The initial routing problem is often a small set of operational
categories with useful context already available in the database.
Calling a large model for every message can increase latency and cost
without improving obvious decisions.

A rules-first architecture also makes routing explainable and testable.
Its limitation is that rules can miss nuanced language, so we need a
review path, a labeled evaluation set, and a plan for improving the
classifier when real data becomes available.

### Why use LangGraph selectively?

LangGraph can be valuable when lead qualification requires several
stateful steps, such as gathering missing fields, waiting for human
approval, and resuming after a prospect replies. A simple classification
function does not need an agent framework. Keep deterministic routing
independent so it remains fast and easy to test.

------------------------------------------------------------------------

## Data Model

The first iteration can use three core entities and expand as workflows
mature.

### 1. `contacts`

Stores known people or organizations that interact with the agency.

  Field                        Purpose
  ---------------------------- ---------------------------------------------------
  `id`                         Internal primary key
  `name`                       Display name, when known
  `email`                      Verified or supplied email, subject to validation
  `phone`                      Normalized phone number when available
  `company_name`               Associated organization
  `relationship_type`          `prospect`, `client`, `partner`, `unknown`
  `created_at`, `updated_at`   Audit-friendly timestamps

Avoid storing more personal information than the application needs.
Identity matching must be cautious, particularly when messages arrive
from multiple channels.

### 2. `projects`

Represents active or delivered work for a customer.

  Field                 Purpose
  --------------------- ------------------------------------------------
  `id`                  Internal project identifier
  `client_contact_id`   Reference to the client/organization record
  `name`                Project name
  `service_type`        RAG, agent, chatbot, integration, and so on
  `status`              Proposed, active, delivered, paused, or closed
  `delivered_at`        Delivery timestamp, when applicable
  `support_status`      Indicates whether support is active

Project records provide context for routing a message about an existing
delivery. They must not automatically cause every message from that
client to be categorized as support.

### 3. `messages`

The durable record of each received message or communication event.

  -----------------------------------------------------------------------
  Field                               Purpose
  ----------------------------------- -----------------------------------
  `id`                                Internal message ID

  `source`                            Email, WhatsApp, website form,
                                      Google Form, etc.

  `external_message_id`               Provider event/message identifier
                                      for deduplication

  `contact_id`                        Resolved contact, if known

  `project_id`                        Related project, if known

  `received_at`                       Source receipt time

  `raw_text`                          Original message content, protected
                                      by access controls

  `category`                          Primary intent category

  `subcategory`                       More specific intent

  `classification_method`             Rule, lightweight model, or human
                                      review

  `classification_reason`             Explainable reason codes

  `confidence`                        Optional calibrated confidence
                                      value

  `llm_eligible`                      Application-controlled eligibility
                                      decision

  `processing_status`                 Received, classified, routed,
                                      failed, review required, etc.

  `created_at`, `updated_at`          Internal timestamps
  -----------------------------------------------------------------------

### Recommended next tables

As the project grows, add:

-   `work_items`: lead, support ticket, enquiry, or change request
    linked to one or more messages.
-   `lead_qualifications`: validated extraction output, scoring details,
    model/version metadata, and qualification status.
-   `classification_events`: append-only history of decisions,
    corrections, and rule versions.
-   `follow_up_tasks`: owner, due time, state, attempts, and stop
    reason.
-   `integration_events`: provider delivery metadata, retries, and
    processing errors.

This separation makes it possible to represent mixed-intent messages
without duplicating the original message or losing its provenance.

------------------------------------------------------------------------

## LLM Cost-Control Strategy

Cost management is a core architectural requirement, not a later
optimization.

### Eligibility gate

The application should enforce this sequence:

1.  Persist the incoming message.
2.  Resolve contact/project context.
3.  Run deterministic classification and configured lightweight checks.
4.  Route clear support issues, routine enquiries, and change requests
    without a lead-qualification LLM call.
5.  Mark a sales work item as eligible only when the evidence and policy
    support it.
6.  Invoke the LLM through a service that independently checks
    eligibility.
7.  Validate structured output and record token usage, latency, model
    version, and errors.

A value such as `llm_eligible=true` must be set by trusted server-side
code, never accepted directly from a public request as authorization to
use the model.

### Cost-control techniques

-   Use deterministic rules for high-confidence, common cases.
-   Use a small, separately evaluated classifier only if rules and human
    review are insufficient.
-   Keep prompts focused and send only the context needed for lead
    qualification.
-   Use structured outputs and validate them against Pydantic schemas.
-   Avoid repeated calls by caching or reusing results only when the
    message and relevant context have not changed.
-   Apply timeouts, bounded retries, rate limits, and monthly budgets.
-   Track cost by channel, workflow, model, and qualified lead.
-   Do not send support messages to the lead-qualification model merely
    to make routing easier.
-   Treat provider errors and malformed output as recoverable processing
    states, not as valid classifications.

### Illustrative example

Suppose an agency receives 10,000 messages in one month and 2,000 are
eligible for lead qualification after first-stage routing. That would
mean 2,000 rather than 10,000 lead-qualification calls, an 80% reduction
in calls under those assumptions.

This is an example for capacity and cost modeling, not a measured
project result. Real savings depend on message distribution, tokens per
call, retries, model pricing, and classifier quality. We must also
monitor missed leads so cost reductions do not come at the expense of
sales opportunities.

------------------------------------------------------------------------

## Reliability, Security, and Human Review

### Reliability

-   Use a unique constraint on provider event IDs or an equivalent
    idempotency key.
-   Persist events before asynchronous processing.
-   Track state transitions and retry recoverable failures with bounded
    backoff.
-   Route repeated failures to a dead-letter or manual-review queue.
-   Prevent duplicate external actions by storing action IDs and
    outcomes.
-   Reconcile provider delivery states where supported.
-   Monitor ingestion lag, queue depth, failed jobs, and unprocessed
    messages.

### Security and privacy

-   Authenticate provider webhooks and validate their signatures where
    supported.
-   Use TLS for network communication and encryption at rest through the
    selected infrastructure.
-   Apply role-based access control to customer data and administrative
    functions.
-   Store credentials in a secrets manager, not in Git or logs.
-   Redact sensitive message content from routine logs and LLM traces.
-   Define data-retention and deletion policies appropriate to the
    agency and its customers.
-   Follow applicable privacy requirements and each messaging provider's
    policies.
-   Treat message content as untrusted input. Instructions embedded in a
    message must not override system policies or authorize
    tools/actions.

### Human review

Human review should be available when:

-   A message has ambiguous or mixed intent.
-   Contact or project identity is uncertain.
-   The classifier's decision is low-confidence or conflicts with
    context.
-   A sales response includes unverified pricing, delivery commitments,
    or contractual statements.
-   An action changes important customer records or triggers a
    consequential external communication.
-   An integration repeatedly fails or a client reports a high-priority
    issue.

------------------------------------------------------------------------

## Metrics and Evaluation

A system that reduces model calls but misses sales leads is not
successful. Evaluate routing quality and business outcomes together.

  -----------------------------------------------------------------------
  Metric                              What it tells us
  ----------------------------------- -----------------------------------
  Message ingestion success rate      Whether incoming events are durably
                                      accepted

  Unprocessed message count and age   Whether work is getting stuck

  Duplicate ingestion rate            Whether repeated events create
                                      duplicate records

  Per-category precision and recall   Whether messages are routed to the
                                      correct workflow

  Sales-lead recall                   Whether potential sales
                                      opportunities are being missed

  Mixed-intent detection rate         Whether multi-purpose messages are
                                      handled correctly

  Human correction rate               How often staff overturn
                                      classifications

  LLM eligibility rate                What proportion of messages enter
                                      the qualification workflow

  LLM calls per 1,000 messages        Model usage intensity

  Cost per qualified lead             AI cost relative to a meaningful
                                      business outcome

  First-response time                 How quickly a customer receives an
                                      appropriate response

  Support resolution time             How long issues remain unresolved

  Lead-to-meeting and lead-to-sale    Whether qualification supports
  conversion                          business outcomes

  Follow-up compliance                Whether permitted follow-ups occur
                                      on time and stop when required
  -----------------------------------------------------------------------

### Evaluation plan

1.  Create a synthetic dataset covering sales, support, questions,
    change requests, spam, mixed intent, negation, and ambiguous
    messages.
2.  Establish expected labels and routing decisions before testing.
3.  Test deterministic rules with unit tests and edge cases.
4.  Measure per-category precision, recall, and confusion matrices on a
    held-out labeled set.
5.  Review false negatives, especially sales leads incorrectly routed to
    support or enquiries.
6.  Record human corrections and use them to improve versioned rules.
7.  Evaluate LLM extraction separately from first-stage classification.
8.  Run a pilot in review-only mode before allowing automated external
    actions.

Synthetic examples are useful for development but do not establish
real-world accuracy. Production performance must be measured against
representative, human-reviewed messages.

------------------------------------------------------------------------

## Implementation Roadmap

### Phase 1 --- Data foundation and synthetic dataset

-   Create the project structure and local development environment.
-   Define PostgreSQL schemas and migrations.
-   Seed synthetic contacts, projects, and messages.
-   Establish message and classification status models.

**Deliverable:** A reproducible database with sample agency
communications.

### Phase 2 --- Rules-first classification

-   Implement normalization and configurable routing rules.
-   Add contact/project context resolution.
-   Support mixed intent, explicit review states, and reason codes.
-   Write pytest coverage for edge cases.

**Deliverable:** A tested classifier that routes obvious messages
without an LLM.

### Phase 3 --- API and persistence

-   Build FastAPI endpoints for message ingestion and review.
-   Enforce idempotency and server-side eligibility.
-   Persist classification events and processing errors.
-   Add structured logging and health checks.

**Deliverable:** A durable ingestion and routing backend.

### Phase 4 --- Lead qualification

-   Add an LLM provider adapter.
-   Define structured qualification schemas.
-   Validate output and keep scoring logic in Python.
-   Track token usage, model versions, latency, and failures.

**Deliverable:** LLM qualification only for eligible sales work items.

### Phase 5 --- Queues and dashboard

-   Build a unified inbox and separate sales, support, change-request,
    and enquiry queues.
-   Show message history, classification reasons, owner, and next
    action.
-   Add manual correction and approval controls.

**Deliverable:** An operational interface for the agency team.

### Phase 6 --- Integrations and background processing

-   Add channel adapters incrementally, beginning with the sources
    required by the pilot.
-   Add a worker/queue for slower tasks and scheduled follow-ups.
-   Test webhook retries, duplicate deliveries, and provider outages.

**Deliverable:** Connected sources with recoverable processing.

### Phase 7 --- Production hardening and deployment

-   Add OAuth2/JWT, role-based permissions, and secrets management.
-   Add CI checks, test coverage, Docker packaging, and deployment
    automation.
-   Deploy to AWS using appropriately managed services.
-   Configure alerts, backups, retention, and operational runbooks.

**Deliverable:** A monitored deployment ready for a controlled pilot.

------------------------------------------------------------------------

## Proposed Repository Structure

``` text
ai-agency-communication-platform/
├── app/
│   ├── main.py
│   ├── api/
│   │   ├── routes/
│   │   │   ├── messages.py
│   │   │   ├── contacts.py
│   │   │   ├── projects.py
│   │   │   ├── work_items.py
│   │   │   └── health.py
│   │   └── dependencies.py
│   ├── core/
│   │   ├── config.py
│   │   ├── logging.py
│   │   ├── security.py
│   │   └── policies.py
│   ├── db/
│   │   ├── session.py
│   │   ├── models/
│   │   └── migrations/
│   ├── schemas/
│   │   ├── message.py
│   │   ├── classification.py
│   │   ├── contact.py
│   │   └── qualification.py
│   ├── services/
│   │   ├── ingestion.py
│   │   ├── identity_resolution.py
│   │   ├── classification/
│   │   │   ├── rules.py
│   │   │   ├── router.py
│   │   │   └── taxonomy.py
│   │   ├── lead_qualification/
│   │   │   ├── eligibility.py
│   │   │   ├── llm_client.py
│   │   │   ├── extraction.py
│   │   │   └── scoring.py
│   │   ├── work_item_service.py
│   │   └── audit_service.py
│   ├── integrations/
│   │   ├── email/
│   │   ├── whatsapp/
│   │   ├── website_forms/
│   │   └── google_forms/
│   ├── workers/
│   │   ├── tasks.py
│   │   └── scheduler.py
│   └── observability/
│       ├── metrics.py
│       └── tracing.py
├── frontend/
│   └── README.md
├── data/
│   ├── synthetic_messages.jsonl
│   └── seed_data.py
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── api/
│   └── evaluation/
├── scripts/
├── docker/
├── .env.example
├── .gitignore
├── Dockerfile
├── compose.yaml
├── pyproject.toml
├── alembic.ini
└── README.md
```

This structure is a proposed organization. Create directories when their
corresponding functionality is implemented; do not treat the presence of
a path in this diagram as evidence that code already exists.

------------------------------------------------------------------------

## Local Development

### Prerequisites

-   Python 3.11 or a compatible project-supported version
-   PostgreSQL
-   Git
-   Docker and Docker Compose (recommended)
-   Node.js LTS if developing the React frontend
-   An LLM API key only when testing the lead-qualification workflow

### 1. Clone the repository

``` bash
git clone <your-repository-url>
cd ai-agency-communication-platform
```

### 2. Create a virtual environment

``` bash
python -m venv .venv
```

Activate it:

**Windows PowerShell**

``` powershell
.venv\Scripts\Activate.ps1
```

**macOS/Linux**

``` bash
source .venv/bin/activate
```

### 3. Install project dependencies

Once `pyproject.toml` is configured:

``` bash
pip install -e ".[dev]"
```

### 4. Configure environment variables

Create a local `.env` file based on `.env.example`. Never commit the
real `.env` file.

### 5. Start the database

If a Compose configuration has been implemented:

``` bash
docker compose up -d postgres
```

### 6. Apply migrations and seed synthetic data

After the database models and migration scripts are implemented:

``` bash
alembic upgrade head
python -m data.seed_data
```

### 7. Start the API

``` bash
uvicorn app.main:app --reload
```

Open the FastAPI documentation at:

``` text
http://127.0.0.1:8000/docs
```

Commands that depend on application modules or Compose services will
work only after those components have been created and configured.

------------------------------------------------------------------------

## Environment Configuration

Example variable names for `.env.example`:

``` dotenv
APP_ENV=local
APP_NAME=ai-agency-communication-platform
LOG_LEVEL=INFO

DATABASE_URL=postgresql+psycopg://app_user:change_me@localhost:5432/agency_comms

# Configure only for the selected LLM provider.
LLM_PROVIDER=
LLM_API_KEY=
LLM_MODEL=

# Optional background processing.
REDIS_URL=redis://localhost:6379/0

# Authentication settings should use secure generated values.
JWT_SECRET_KEY=replace_with_a_secure_random_secret
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=30

# Provider credentials and webhook secrets should be added only
# when the corresponding integration is implemented.
EMAIL_PROVIDER=
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_WEBHOOK_VERIFY_TOKEN=
GOOGLE_FORMS_INTEGRATION_MODE=

# Operational limits.
MAX_LLM_RETRIES=2
LLM_MONTHLY_BUDGET_LIMIT=
```

Do not use the example secrets in any environment. Validate required
settings at startup, keep provider credentials in a secrets manager in
production, and avoid logging secret values.

------------------------------------------------------------------------

## API Design

The following endpoints are a proposed initial contract.

  ---------------------------------------------------------------------------------------------
  Method                  Endpoint                                      Purpose
  ----------------------- --------------------------------------------- -----------------------
  `POST`                  `/api/v1/messages`                            Accept and persist an
                                                                        incoming message

  `GET`                   `/api/v1/messages`                            Search and filter
                                                                        message records

  `GET`                   `/api/v1/messages/{message_id}`               View a message and its
                                                                        classification history

  `POST`                  `/api/v1/messages/{message_id}/reclassify`    Request a controlled
                                                                        reclassification

  `GET`                   `/api/v1/contacts`                            Search known contacts

  `GET`                   `/api/v1/projects`                            Retrieve authorized
                                                                        project context

  `GET`                   `/api/v1/work-items`                          View sales, support,
                                                                        and enquiry queues

  `PATCH`                 `/api/v1/work-items/{work_item_id}`           Update owner, status,
                                                                        or priority with
                                                                        authorization

  `POST`                  `/api/v1/work-items/{work_item_id}/approve`   Approve an eligible
                                                                        proposed action

  `GET`                   `/health/live`                                Check whether the
                                                                        process is alive

  `GET`                   `/health/ready`                               Check readiness and
                                                                        required dependencies
  ---------------------------------------------------------------------------------------------

For real provider integrations, use dedicated webhook endpoints and
validate provider signatures. Public endpoints must not permit clients
to set arbitrary categories, qualification outcomes, or LLM permissions
without server-side validation.

------------------------------------------------------------------------

## Testing Strategy

### Unit tests

-   Sales keyword matches and counterexamples
-   Support issue and change-request routing
-   Existing client requesting a new service
-   Mixed-intent messages
-   Negation, ambiguous language, and missing context
-   Identity matching and duplicate-event handling
-   LLM eligibility enforcement
-   Qualification schema validation and scoring rules

### Integration tests

-   Database persistence and transaction rollback
-   Unique constraints on provider event IDs
-   Retry behavior and duplicate webhook delivery
-   Message routing into the correct work-item queue
-   Model-provider timeout and malformed output handling
-   Authentication and authorization boundaries

### Evaluation tests

Maintain a versioned, human-reviewed dataset with representative
messages. Track precision and recall per category, especially sales-lead
recall and support-routing accuracy. Test on examples that were not used
to tune the rules.

### Operational tests

-   Database or queue unavailable
-   Provider sends the same event more than once
-   LLM provider is unavailable
-   Worker restarts during processing
-   Message is received but downstream routing fails
-   A prospect opts out while a follow-up is pending

The expected behavior for failures should be documented and tested
before production automation is enabled.

------------------------------------------------------------------------

## Deployment Strategy

A practical AWS deployment can evolve in stages:

  -----------------------------------------------------------------------
  Component               Possible AWS service    Purpose
  ----------------------- ----------------------- -----------------------
  API and worker          ECS with Fargate        Run backend and
  containers                                      background processing
                                                  without managing
                                                  servers

  Relational database     Amazon RDS for          Managed relational
                          PostgreSQL              storage and backups

  Queue                   Amazon SQS, or Redis    Buffer asynchronous
                          where required by the   work and support
                          chosen worker design    retries

  Secrets                 AWS Secrets Manager     Protect and rotate
                                                  integration/API
                                                  credentials

  Logs and metrics        Amazon CloudWatch       Centralize application
                                                  logs, metrics, and
                                                  alarms

  Container registry      Amazon ECR              Store versioned
                                                  container images

  Static frontend         Amazon S3 + CloudFront, Deliver the web
                          or an appropriate       dashboard
                          managed frontend host   

  CI/CD                   GitHub Actions with AWS Run checks and deploy
                          OIDC where configured   without long-lived AWS
                                                  credentials
  -----------------------------------------------------------------------

Choose one queue and deployment model based on the implementation. This
table presents options, not an assertion that the infrastructure is
provisioned.

### Production rollout

1.  Deploy a development environment with synthetic data.
2.  Validate authentication, database migrations, backups, and alerting.
3.  Connect one real communication source.
4.  Run in review-only mode and compare automated routing with human
    labels.
5.  Enable selected low-risk routing actions after the evaluation
    criteria are met.
6.  Enable lead qualification only behind the server-side eligibility
    gate.
7.  Expand integrations gradually and monitor errors, cost, and routing
    quality.

------------------------------------------------------------------------

## Limitations and Future Enhancements

-   **Rules are not perfect:** keyword rules can miss nuanced intent.
    Use human review and measured evaluation before considering a
    lightweight classifier.
-   **Synthetic data is not production evidence:** fabricated messages
    help build tests but cannot establish real-world accuracy.
-   **Identity resolution is difficult:** the same person may use
    different addresses or phone numbers. Merge records only with
    sufficient evidence.
-   **Channel integrations vary:** each provider has its own
    authentication, webhook, rate-limit, retention, and messaging
    requirements.
-   **LLMs can produce incorrect extraction:** validate outputs,
    preserve original messages, and avoid treating generated values as
    verified facts.
-   **Automation must respect consent:** honor opt-outs, communication
    preferences, provider policies, and approved follow-up limits.
-   **Analytics need outcome labels:** conversion and resolution metrics
    become meaningful only when sales and support outcomes are recorded
    consistently.

Potential future enhancements:

-   A lightweight intent classifier trained on reviewed examples
-   Semantic retrieval over approved project documentation
-   CRM synchronization with conflict handling
-   SLA alerts and support escalation policies
-   Sales pipeline forecasting based on validated historical outcomes
-   Human-in-the-loop review queues and active-learning workflows
-   MCP-based, permission-scoped tools for authorized internal systems
-   Multi-tenant support for offering the platform as a SaaS product
-   Advanced analytics for channel performance and cost per qualified
    opportunity

## Project Status

**Status: Architecture and implementation plan defined.**

The architecture, proposed schemas, workflow boundaries, technology
choices, and evaluation strategy are specified in this README. Actual
implementation status should be updated as code, tests, integrations,
and deployment are completed.

### Initial acceptance criteria

-   [ ] Synthetic contacts, projects, and messages can be seeded
    locally.
-   [ ] Incoming messages are persisted before classification.
-   [ ] Duplicate provider events do not create duplicate message
    records.
-   [ ] Obvious sales, support, change-request, and enquiry cases route
    correctly in tests.
-   [ ] Mixed-intent messages can produce separate linked work items.
-   [ ] Unclear messages enter a review or clarification path.
-   [ ] Support and general enquiry paths do not call the
    lead-qualification LLM by default.
-   [ ] The LLM eligibility gate is enforced server-side.
-   [ ] Qualification outputs are schema-validated and scoring is
    deterministic.
-   [ ] Failures are observable and recoverable.
-   [ ] A human can correct classifications and record outcomes.
-   [ ] Security, integration, and deployment checks pass before
    production rollout.

------------------------------------------------------------------------

## Summary

The **AI Agency Communication & Lead Intelligence Platform** is designed
to give an AI/software agency one reliable place to manage new
opportunities and existing-client communications.

Its defining architectural decision is to **classify and route messages
before invoking an LLM**. Support issues, change requests, and routine
questions go to their own workflows; eligible sales opportunities
receive structured AI-assisted qualification; uncertain and mixed-intent
messages remain traceable and reviewable.

The result should be a cost-aware, testable, and extensible foundation
for agency operations---not merely a chatbot or a lead-scoring script.
