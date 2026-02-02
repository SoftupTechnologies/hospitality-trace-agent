# Apaleo AI Trace Management System

An intelligent automation system that processes Apaleo booking and reservation data to automatically generate actionable tasks (traces) using AI. The system consists of a modular set of n8n workflows designed for scalability and concurrency management.

## Workflows Overview

The system is composed of the following workflows:

| Workflow | Type | Description |
|----------|------|-------------|
| **Booking Traces** | Main | Entry point for `booking/changed, booking/created` webhooks |
| **Reservation Traces** | Main | Entry point for `reservation/changed, reservation/created` webhooks |
| **Manage Lock - Create and Wait** | Sub-workflow | Handles concurrency to prevent race conditions |
| **Sub - Trace AI Agent** | Sub-workflow | Core AI logic that generates tasks |
| **Release Lock** | Sub-workflow | Cleans up locks after processing |
| **Daily Traces Email** | Standalone | Sends daily summary reports |

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [1. Apaleo Configuration](#1-apaleo-configuration)
  - [2. OpenAI Configuration](#2-openai-configuration)
  - [3. Rule Book Configuration (Google Sheets)](#3-rule-book-configuration-google-sheets)
  - [4. Supabase Database Setup](#4-supabase-database-setup)
  - [5. Sweeply Integration Setup](#5-sweeply-integration-setup)
  - [6. Daily Email Report Configuration](#6-daily-email-report-configuration)
- [Flow Architecture](#flow-architecture)
  - [Concurrency & Locking Mechanism](#concurrency--locking-mechanism)
- [Usage](#usage)
  - [Importing Workflows](#importing-workflows)
  - [Workflow Configuration Placeholders](#workflow-configuration-placeholders)
  - [Testing the Trace Generation Flow](#testing-the-trace-generation-flow)

## Prerequisites

Before setting up this system, ensure you have:

- An active [Apaleo](https://apaleo.com/) account with access to create custom apps
- An [OpenAI](https://platform.openai.com/) account with API access
- A [Google](https://google.com/) account with access to Google Sheets (for rule book storage)
- A [Supabase](https://supabase.com/) account for database storage
- A [Sweeply](https://sweeply.com/) account for task management
- A [Resend](https://resend.com/) account for email delivery
- An n8n instance (cloud)

## Setup Instructions

### 1. Apaleo Configuration

#### Create an App in Apaleo

1. Navigate to **APPS > Connected apps** in your Apaleo account
2. Click **Add a new app**
3. Select **Add custom app**
4. In the scopes section, select **reservations.read**
5. Fill in the following details:
   - **Client code**: Enter your client code
   - **Client name**: Enter a descriptive name for your app (e.g., "n8n Trace Automation")
   - **Description**: Add a description explaining the purpose of the app
6. **Important**: Securely store the **Client Secret** and **Client ID** - you'll need these for authentication in n8n

#### Get Access Token

To create webhooks, you first need to obtain an access token using your client credentials:

**Request:**
```http
POST https://identity.apaleo.com/connect/token
```

Include your Client ID and Client Secret in the request to receive an access token.

#### Create Webhook Subscriptions

After obtaining the access token, create webhook subscriptions using the Apaleo Webhook API.

**Request:**
```http
POST https://webhook.apaleo.com/v1/subscriptions
```

Include the access token in the Authorization header.

##### Booking Changes Webhook

**Body:**
```json
{
  "endpointUrl": "url from the webhook node in n8n",
  "events": [
    "booking/changed",
    "booking/created"
  ],
  "propertyIds": [
    // Leave empty - booking is on account level
  ]
}
```

Replace `"url from the webhook node in n8n"` with the actual webhook URL from your n8n workflow (found in the "Receive booking changes" webhook node).

##### Reservations Webhook

**Body:**
```json
{
  "endpointUrl": "url from the webhook node in n8n",
  "events": [
    "reservation/created",
    "reservation/changed"
  ],
  "propertyIds": [
    "[Property id of the hotel]"
  ]
}
```

Replace `"url from the webhook node in n8n"` with the actual webhook URL from your n8n workflow (found in the "Receive reservation" webhook node).

**Note:** The `propertyIds` array can be left empty to subscribe to all properties, or you can specify specific property IDs if you only want to receive webhooks for certain properties.

### 2. OpenAI Configuration

The system uses OpenAI's GPT-5 model for intelligent trace generation.

#### Create OpenAI API Key

1. Sign up or log in to [OpenAI Platform](https://platform.openai.com/)
2. Navigate to **API Keys** section
3. Click **Create new secret key**
4. Give your key a descriptive name (e.g., "n8n Trace Agent")
5. Copy and securely store the API key (it will only be shown once)

#### Add OpenAI Credentials to n8n

1. In your n8n workflow, locate the **Task agent** node
2. Click on **OpenAi Model1** node
3. Click on the **Credential to connect with** dropdown
4. Select **Create New Credential** or **OpenAi Account**
5. Enter the following details:
   - **API Key**: Paste your OpenAI API key
   - **Organization ID**: (Optional) Enter if applicable to your account
6. Click **Save**
7. The credential is now available for use in the Task Agent node

### 3. Rule Book Configuration (Google Sheets)

The AI agent uses rule books to determine which guest and booking inputs should create operational traces. These rule books are stored in Google Sheets and are retrieved dynamically during workflow execution.

#### Understanding the Rule Books

The system uses two types of rule books:

1. **System Rule Book** (`System_Trace_Rulebook` sheet): Defines system-wide rules for trace creation, including:
   - Which guest/booking inputs trigger traces
   - Hard-disabled domains (e.g., credit card intents, invoice sending)
   - Canonical task titles and routing rules
   - Priority and due-date logic

2. **Property Rule Book** (`Property_Trace_Rulebook` sheet): Contains property-specific rules that can override or extend system defaults.

#### Import the Flag Rulebook to Google Sheets

1. Open your Google Drive account
2. Navigate to **Google Sheets** and create a new spreadsheet named **"Flag Rulebook"**
3. Download and open the `Flag Rulebook.xlsx` file from this repository
4. Import each sheet from the Excel file into the Google Sheet:
   - **System_Trace_Rulebook**: Contains the master rules for trace generation
   - **Property_Trace_Rulebook**: Contains property-specific configurations
5. Note the **Google Sheets document ID** from the URL:
   - The URL format is: `https://docs.google.com/spreadsheets/d/DOCUMENT_ID/edit`
   - Copy the `DOCUMENT_ID` part - you will need it for the workflow configuration

#### Configure Google Sheets Credentials in n8n

1. In the **Sub Trace Ai Agent** workflow, locate the **Get System Rule Book** node
2. Click on the **Credential to connect with** dropdown
3. Select **Create New Credential** or an existing Google account
4. Click **Sign in with Google** and authenticate with your Google account
5. Grant n8n permission to access your Google Sheets
6. The credential is now available for use in both rule book nodes

#### Update Workflow Nodes

After creating the Google Sheets credential:

1. In the **Get System Rule Book** node:
   - Select your Google account from the credential dropdown
   - In the **Document** field, select your **Flag Rulebook** spreadsheet
   - In the **Sheet** field, select **System_Trace_Rulebook**
2. Repeat for the **Get Property Rule Book** node, selecting the **Property_Trace_Rulebook** sheet

### 5. Supabase Database Setup

#### Create Supabase Account and Project

1. Sign up or log in to [Supabase](https://supabase.com/)
2. Click **New Project**
3. Fill in your project details:
   - **Name**: Choose a descriptive name (e.g., "apaleo-trace-management")
   - **Database Password**: Generate a strong password and store it securely
   - **Region**: Select a region closest to your users
4. Wait for your project to be provisioned

#### Configure n8n Connection to Supabase

For detailed instructions on setting up Supabase credentials in n8n, refer to the [official n8n Supabase documentation](https://docs.n8n.io/integrations/builtin/credentials/supabase/#related-resources).

**Quick Setup:**
1. In your Supabase project, go to **Project Settings > API**
2. Copy the **Project URL** (this is your **Host**)
3. Reveal and copy the **service_role** key from **Project API keys** (this is your **Service Role Secret**)
4. In n8n, create a new Supabase credential with these values

#### Create the tables

In your Supabase project, navigate to the **SQL Editor** and execute the following schema:

```sql
create table public.trace_logs (
  id bigint generated by default as identity not null,
  created_at timestamp with time zone not null default now(),
  apaleo_property_id character varying not null,
  webhook_topic character varying not null,
  webhook_entity_id character varying not null,
  trace jsonb null,
  sweeply_status character varying not null default 'pending'::character varying,
  notes text null,
  apaleo_account_id character varying not null,
  booking_id text not null,
  sweeply_trace_id text null,
  source_comments text null,
  constraint trace_logs_pkey primary key (id)
) TABLESPACE pg_default;

create table public.webhook_event_queue (
  id bigint generated by default as identity not null,
  created_at timestamp with time zone not null default now(),
  event_identifier text not null,
  constraint webhook_event_queue_pkey primary key (id)
) TABLESPACE pg_default;
```

Click **Run** to create the table. This table will store all generated traces and their associated metadata.

### 4. Sweeply Integration Setup

The system integrates with Sweeply for task management. You need to configure Basic Authentication credentials.

#### Configure Sweeply Basic Auth in n8n

1. In the **Sub Trace Ai Agent** workflow, locate the **Sweeply Basic Auth Token** node
2. Update the value of the **authorization** field: replace `REDACTED:REDACTED` with your actual Sweeply credentials in format `username:password` (the workflow will base64 encode them automatically)
3. Click **Save**

**Note:** The workflow currently uses the Sweeply production API endpoint (`https://app.getsweeply.com`). Update the API URLs in the **Post to Sweeply** and **Update to Sweeply** nodes if needed.

The workflow sends a `pms: "apaleo"` parameter with each request to identify the source PMS system.

### 6. Daily Email Report Configuration

The **Daily Traces Email** workflow sends a daily summary report at 23:59 containing all traces generated during the day.

#### Configure Account Details

1. Open the **Daily Traces Email** workflow
2. Locate the **Account details** node
3. Update the following fields:
   - **apaleo_account_id**: Your Apaleo account ID
   - **apaleo_property_id**: Your property ID from Apaleo
   - **email_addresses**: A comma-separated list of email addresses that should receive the daily report (e.g., `"manager@hotel.com,admin@hotel.com"`)

#### Configure Resend Email Service

The workflow uses [Resend](https://resend.com/) to send the daily email reports.

##### Create Resend Account and API Key

1. Sign up or log in to [Resend](https://resend.com/)
2. Navigate to **API Keys** in the dashboard
3. Click **Create API Key**
4. Give your key a descriptive name (e.g., "n8n Daily Traces Email")
5. Select the appropriate permissions (sending access is required)
6. Copy and securely store the API key (it will only be shown once)

##### Configure Resend in n8n

1. In the **Daily Traces Email** workflow, locate the **Send email** node (HTTP Request node)
2. Find the **Header Parameters** section
3. Update the **Authorization** header value:
   - Replace `Bearer xxxxxxxx` with `Bearer YOUR_RESEND_API_KEY`
   - Example: `Bearer re_123456789abcdef`
4. Click **Save**

**Note:** The workflow is pre-configured to send emails from `Your Company Name <no-reply@yourdomain.com>`. You should update the sender address in the `from` field in the **Send email** node body parameters to match your verified domain in Resend.

#### Configure PostgreSQL Connection

The workflow needs to connect to your Supabase database using the PostgreSQL node.

1. In the **Daily Traces Email** workflow, locate the **Execute a SQL query** node (PostgreSQL node)
2. Click on the **Credential to connect with** dropdown
3. Select **Create New Credential** or **Postgres Account**
4. Get your connection details from Supabase:
   - Go to your Supabase project
   - Navigate to **Project Settings > Database**
   - Click on **Connection string** tab
   - Select **Connection pooler** (recommended for n8n workflows)
   - Copy the connection string which will look like:
     ```
     postgresql://postgres.[project-ref]:[password]@aws-0-[region].pooler.supabase.com:6543/postgres
     ```
5. Fill in the n8n PostgreSQL credential fields:
   - **Host**: Extract from connection string (e.g., `aws-0-us-east-1.pooler.supabase.com`)
   - **Database**: `postgres`
   - **User**: Extract from connection string (e.g., `postgres.abcdefgh`)
   - **Password**: Your Supabase database password
   - **Port**: `6543` (connection pooler port)
   - **SSL**: Enable (recommended)
6. Click **Save**

## Flow Architecture

### Trace Generation Flow Overview

The workflow processes webhooks from Apaleo through two main paths:

1. **Reservation Creation Path**: Triggered when a new reservation is created
2. **Booking Changes Path**: Triggered when an existing booking is modified

Both paths converge at the data processing stage and follow the same AI task generation workflow.

### Concurrency & Locking Mechanism

#### The Problem

When a booking or reservation is updated in Apaleo, multiple webhooks can fire in rapid succession for the same entity. For example:
- A reservation update may trigger both a `reservation/changed` and a `booking/changed` event
- Rapid user edits can send multiple webhooks within milliseconds
- Without coordination, parallel executions for the same entity can cause **duplicate tasks** or **race conditions**

#### The Solution: Database-Based Locking

The system implements a locking mechanism using the `webhook_event_queue` table in Supabase. This ensures that only one workflow processes a given entity at a time.

#### How It Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LOCKING MECHANISM FLOW                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Webhook Arrives                                                            │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────┐                                    │
│  │  Generate webhook_event_key         │                                    │
│  │  Format: {topic}:{propertyId}:{entityId}                                 │
│  │  Example: reservation:MUN:RES456                                         │
│  └─────────────────────────────────────┘                                    │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────┐                                    │
│  │  Check webhook_event_queue table    │                                    │
│  │  for existing lock with this key    │                                    │
│  └─────────────────────────────────────┘                                    │
│       │                                                                     │
│       ├──── Lock EXISTS ────────────────────┐                               │
│       │                                     ▼                               │
│       │                          ┌──────────────────────┐                   │
│       │                          │  Wait (10 seconds)   │                   │
│       │                          │  Then retry check    │◄─────┐            │
│       │                          └──────────────────────┘      │            │
│       │                                     │                  │            │
│       │                                     └──── Still locked ┘            │
│       │                                     │                               │
│       │                                     └──── Lock released             │
│       │                                              │                      │
│       ▼                                              ▼                      │
│  ┌─────────────────────────────────────┐                                    │
│  │  No lock exists                     │                                    │
│  │  CREATE lock in webhook_event_queue │                                    │
│  └─────────────────────────────────────┘                                    │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────┐                                    │
│  │  Process webhook normally           │                                    │
│  │  (Fetch data, AI agent, Sweeply)    │                                    │
│  └─────────────────────────────────────┘                                    │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────┐                                    │
│  │  RELEASE lock                       │                                    │
│  │  Delete from webhook_event_queue    │                                    │
│  └─────────────────────────────────────┘                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Lock Key Format

The lock key (`webhook_event_key`) is constructed from webhook data:
```
{webhook_topic}:{property_id}:{entity_id}
```

**Note:** The exact topic format may vary. The system uses the full topic from the webhook.

Examples:
- `reservation:MUN:ABCD1234-1` (reservation event with property ID)
- `booking::ABCD1234` (booking event - property ID may be empty as bookings are account-level)

This ensures that:
- Multiple events for the **same** entity are serialized (processed one at a time)
- Events for **different** entities can run in parallel
- Coordination between booking-level and reservation-level events for the same booking

#### Sub-Workflows

##### Manage Lock - Create and Wait
- **Purpose**: Called at the start of every main workflow
- **Process**:
  1. Receives webhook data: `body_data_entityId`, `body_topic`, `body_accountId`, `body_propertyId`
  2. **Webhook details** node: Generates a unique `webhook_event_key` in format `{topic}:{propertyId}:{entityId}`
  3. **Get webhook event** node: Queries `webhook_event_queue` to check if a lock exists
  4. **If locked**: Enters a wait loop (10 seconds per iteration) until the lock is released
  5. **If not locked**: Creates a new lock entry and proceeds
- **Output**: Returns structured data containing:
  - `webhook_event_key`: For lock release
  - `webhook_entity_id`: Entity ID from webhook
  - `webhook_topic`: Event type
  - `apaleo_account_id`: Account ID
  - `apaleo_property_id`: Property ID

##### Release Lock
- **Purpose**: Called at the end of every workflow path (success or failure)
- **Process**: Deletes the lock entry from `webhook_event_queue` using the `webhook_event_key`
- **Critical**: Must be called on ALL exit paths to prevent deadlocks

### Flow Phases

#### Phase 1: Webhook Reception & Lock Acquisition

##### Receive Reservation Webhook
- **Node**: `Receive reservation`
- **Purpose**: Receives webhook events from Apaleo when a reservation is created or changed (`reservation/created`, `reservation/changed` events)
- **Output**: Webhook payload containing reservation entity ID and account information
- **Next Step**: Triggers "Call Manage lock - create and wait"

##### Receive Booking Changes Webhook
- **Node**: `Receive booking changes`
- **Purpose**: Receives webhook events from Apaleo when a booking is created or modified (`booking/created`, `booking/changed` events)
- **Output**: Webhook payload containing booking entity ID and account information
- **Next Step**: Triggers "Call Manage lock - create and wait"

##### Call Manage Lock - Create and Wait
- **Node**: `Call Manage lock - create and wait`
- **Type**: Execute Workflow (sub-workflow call)
- **Purpose**: Acquires a lock for this entity before proceeding
- **Input**: `body_data_entityId`, `body_topic`, `body_accountId`, `body_propertyId`
- **Output**: `webhook_event_key`, `webhook_entity_id`, `apaleo_account_id`, `apaleo_property_id`
- **Next Step**: Triggers data fetching nodes

#### Phase 2: Data Extraction and Mapping

##### Get Reservation
- **Node**: `Get reservation` (Reservation Traces workflow only)
- **Purpose**: Fetches detailed reservation information from Apaleo API using the reservation ID from the webhook
- **API Call**: `BookingReservationsByIdGet` operation
- **Input**: Reservation ID from lock manager (`$json.webhook_entity_id`)
- **Output**: Complete reservation object with unit, unit group, property, and comment details
- **Error Handling**: On error, releases lock and exits
- **Next Step**: Triggers "Get booking details"

##### Get Booking Details
- **Node**: `Get booking details`
- **Purpose**: Fetches complete booking information from Apaleo API, including all reservations
- **API Call**: `BookingBookingsByIdGet` operation
- **Booking workflow**: Uses `expand=reservations,propertyValues`
- **Reservation workflow**: Uses `expand=reservations`
- **Input**: 
  - **Booking workflow**: Booking ID from lock manager (`$json.webhook_entity_id`)
  - **Reservation workflow**: Booking ID from reservation object (`$json.bookingId`)
- **Output**: Complete booking object with:
  - Booking-level comments (`bookerComment`)
  - Booker information (first name, last name)
  - All associated reservations
  - Property information
- **Error Handling**: On error, releases lock and exits
- **Next Step**: Triggers "Extract booking details"

#### Phase 3: Data Processing

##### Extract Booking Details
- **Node**: `Extract booking details`
- **Purpose**: Combines and normalizes data from booking and reservation sources into a unified structure
- **Context Setting**:
  - **Booking workflow**: Sets `context = 'booking'`
  - **Reservation workflow**: Sets `context = 'reservation'`
- **Logic**:
  - Reads booking data from "Get booking details"
  - **Reservation workflow only**: Also reads reservation data from "Get reservation"
  - Extracts comments based on workflow type:
    - **Booking workflow**: Only `bookerComment` (guest-level comments are null)
    - **Reservation workflow**: Only `guestComment` (booking-level comments are null)
  - Extracts unit, unit group, and property details:
    - **Booking workflow**: Property from `booking.propertyValues[0].property`
    - **Reservation workflow**: Unit/unitGroup from reservation, property from reservation or booking
  - Calculates `isUserBirthday` (reservation workflow only)
- **Output**: Normalized JSON object containing:
  - `context`: 'booking' or 'reservation'
  - `now`: Current timestamp
  - Identifiers: `propertyId`, `propertyName`, `reservationId`, `bookingId`
  - Guest info: `bookerFirstName`, `bookerLastName`, `primaryGuestFirstName`, `primaryGuestLastName`, `primaryGuestBirthday`
  - Comments: `bookerComment`, `extraBookingComment`, `reservationComment`, `guestComment`
  - Unit details: `unitId`, `unitName`, `unitDescription`, `unitGroupIdFromUnit`
  - Unit group: `unitGroupId`, `unitGroupCode`, `unitGroupName`, `unitGroupType`
  - Stay details: `arrival`, `departure`, `adults`, `childrenAges`, `channelCode`, `ratePlanCode`, `ratePlanName`, `ratePlanId`
  - Debug: `sourceComments`
- **Next Step**: Triggers "If is the valid property"

##### If is the valid property
- **Node**: `If is the valid property`
- **Purpose**: Filters incoming data to only process bookings from a specific property
- **Logic**: Compares the `propertyId` from extracted booking details against the configured `PROPERTY_ID`
- **Configuration**: Replace `PROPERTY_ID` placeholder with your actual Apaleo property ID
- **Output**: Only passes through data matching the configured property
- **Next Step**: Triggers "If Agent params do not exist" (on true branch) or "Release lock" (on false branch)

##### If Agent params do not exist
- **Node**: `If Agent params do not exist`
- **Purpose**: Determines if there are actionable comments that should trigger AI processing
- **Logic**: 
  - **Booking workflow**: Checks if `bookerComment` is empty/missing
  - **Reservation workflow**: Checks if `guestComment` is empty, not a birthday, occupancy is ≤2 adults with no children, and not a dayuse booking
- **Output**: Routes to either "Call Sub - Trace Ai Agent" or "Release lock"
- **Next Step**: 
  - If params exist (false branch): Triggers "Call Sub - Trace Ai Agent"
  - If params do not exist (true branch): Triggers "Release lock"

#### Phase 4: AI Agent Invocation

##### Call Sub - Trace Ai Agent
- **Node**: `Call Sub - Trace Ai Agent`
- **Type**: Execute Workflow (sub-workflow call)
- **Purpose**: Invokes the AI agent sub-workflow to analyze booking/reservation data and generate actionable tasks
- **Input**: 
  - `payload`: Normalized booking details from "Extract booking details"
  - `webhook_details`: Lock and webhook metadata from "Call Manage lock - create and wait"
- **Process**: The sub-workflow (detailed below) handles:
  - AI-powered task generation using GPT-5
  - Duplicate prevention by checking existing traces
  - Posting/updating tasks in Sweeply
  - Logging results to Supabase
  - Releasing the lock upon completion
- **Output**: None (the sub-workflow handles all downstream operations including lock release)
- **Next Step**: None (workflow ends here)

#### Phase 5: Lock Release

##### Release lock
- **Node**: `Release lock`
- **Type**: Execute Workflow (sub-workflow call)
- **Purpose**: Releases the lock acquired at the start, allowing other queued webhooks to proceed
- **Input**: `webhook_event_key` from the lock acquisition step
- **Called on exit paths**:
  - After "If is the valid property" fails (wrong property)
  - After "If Agent params do not exist" succeeds (no actionable comments)
  - After errors in "Get reservation" or "Get booking details"
- **Ensures**: No deadlocks occur; queued webhooks can proceed

---

### Sub-Workflow: Trace AI Agent

The **Sub - Trace Ai Agent** sub-workflow is called by both main workflows and contains the core AI logic. It processes the booking/reservation data, generates tasks, posts them to Sweeply, and releases the lock.

#### Flow Overview

1. **Task agent**: LangChain AI agent powered by GPT-5
   - Analyzes booking/reservation comments and data
   - Uses **Get System Rule Book** and **Get Property Rule Book** tools to retrieve trace generation rules
   - Uses **Get trace logs** tool to retrieve existing tasks for the booking
   - Applies business rules (occupancy logic, disabled domains, canonical mappings)
   - Outputs JSON array of tasks
   - **Department Rules**:
     - **Reception**: Guest communications, arrival times, room requests, dog amenities, parking, breakfast
     - **Housekeeping**: Cleaning, room prep, amenities, beds, pet in room
     - **Technik**: Maintenance, broken items

2. **Rule Book Tools**: Google Sheets integration for dynamic rule management
   - **Get System Rule Book**: Retrieves system-wide rules from the `System_Trace_Rulebook` sheet
   - **Get Property Rule Book**: Retrieves property-specific rules from the `Property_Trace_Rulebook` sheet
   - Rules are loaded dynamically during each execution, allowing updates without redeploying workflows

3. **Structured Output Parser**: Validates AI output matches the defined Sweeply schema

4. **If no traces**: Checks if the AI generated any tasks
   - If empty: Releases lock and exits
   - If tasks exist: Proceeds to enrichment

5. **Map booking details to tasks**: Enriches AI-generated tasks with booking/reservation metadata (IDs, guest names, unit details, property info)

5. **Sweeply Basic Auth Token**: Generates authentication header for Sweeply API

6. **If is create**: Routes tasks based on action type
   - `action = "create"` → **Create trace logs** → **Post to Sweeply**
   - `action = "update"` → **Create trace logs - update** → **Update to Sweeply**

7. **Trace Status Updates**: Updates the trace log record with Sweeply response
   - Success: Sets `sweeply_status = "success"` or `"updated"`
   - Failure: Sets `sweeply_status = "failed"` with error notes

8. **Release Lock**: Deletes the lock from `webhook_event_queue` after all operations complete

### Daily Traces Email Flow

The Daily Traces Email workflow is scheduled to run every day at 23:59 (configurable via the Schedule Trigger node). It queries all traces created during the day and sends a summary report as an Excel file.

#### Key Components

1. **Schedule Trigger**: Runs daily at 23:59
2. **Account details**: Holds configuration for account ID, property ID, and recipient email addresses
3. **Execute a SQL query**: Queries the `trace_logs` table for today's traces
4. **Data formatting**: Transforms trace data into Excel-compatible format
5. **Email delivery**: Sends the Excel file to configured recipients

## Usage

### Importing Workflows

1. In n8n, navigate to **Workflows**
2. Click **Import from File**
3. Import the workflow JSON files in this order (sub-workflows first):
   - `Manage Lock Create and Wait.json` - Lock management sub-workflow
   - `Release Lock n8n.json` - Lock release sub-workflow
   - `Sub Trace Ai Agent.json` - AI processing sub-workflow
   - `Booking Traces.json` - Main booking webhook flow
   - `Reservation Traces.json` - Main reservation webhook flow
   - `Daily Traces Email.json` - Daily email report flow
4. **Do not activate yet** - you must replace the placeholders first (see table below)
5. After importing, verify that the "Execute Workflow" nodes in the main workflows correctly reference the imported sub-workflows

### Workflow Configuration Placeholders

After importing the workflows into n8n, you need to configure the following values. Most credential connections will be set up through n8n's credential manager, but you'll need to manually update these specific placeholders:

#### Main Workflows (Booking & Reservation Traces)

| Placeholder | Workflow File | Node/Location | What to Replace With |
|------------|---------------|---------------|---------------------|
| `PROPERTY_ID` | Booking Traces.json | If is the valid property node | Your Apaleo property ID to filter incoming webhooks |
| `PROPERTY_ID` | Reservation Traces.json | If is the valid property node | Your Apaleo property ID to filter incoming webhooks |

**Note:** The webhook paths are auto-generated by n8n. After saving, copy the full webhook URLs from the "Receive booking changes" and "Receive reservation" nodes to configure Apaleo webhook subscriptions.

#### Sub - Trace AI Agent Workflow

| Placeholder | Workflow File | Node/Location | What to Replace With |
|------------|---------------|---------------|---------------------|
| `REDACTED:REDACTED` | Sub Trace Ai Agent.json | Sweeply Basic Auth Token node | Your Sweeply credentials in format `username:password` (will be base64 encoded automatically) |
| Google account | Sub Trace Ai Agent.json | Get System Rule Book & Get Property Rule Book nodes | Your Google account credential (sign in via Google from n8n) |

#### Daily Traces Email Workflow

| Placeholder | Workflow File | Node/Location | What to Replace With |
|------------|---------------|---------------|---------------------|
| `ACCOUNT_ID` | Daily Traces Email.json | Account details node (JavaScript) | Your Apaleo account ID |
| `PROPERTY_ID` | Daily Traces Email.json | Account details node (JavaScript) | Your Apaleo property ID |
| `EMAIL_ADDRESS` | Daily Traces Email.json | Account details node (JavaScript) | Recipient email address(es) for daily reports |
| `Bearer xxxxxxxx` | Daily Traces Email.json | Send email node → Headers | Your Resend API key (format: `Bearer re_xxxxx`) |
| `Your Company Name <no-reply@yourdomain.com>` | Daily Traces Email.json | Send email node → Body Parameters | Your verified sender email address |

#### Credential Configuration (All Workflows)

The following credentials need to be configured through n8n's credential manager:

| Credential Type | Used In | Purpose |
|----------------|---------|---------|
| Apaleo OAuth2 | Booking Traces, Reservation Traces | API access to fetch booking/reservation data |
| OpenAI API | Sub Trace Ai Agent | AI model for task generation |
| Google account | Sub Trace Ai Agent | Access to rule books stored in Google Sheets |
| Supabase API | Sub Trace Ai Agent, Manage Lock, Release Lock | Database access for trace logs and locking |
| PostgreSQL | Daily Traces Email | Database queries for daily reports |

**Important Notes:**
- **Credential IDs** will be automatically set when you connect credentials through n8n's credential selector in each node
- **Webhook IDs** will be auto-generated by n8n when you save the webhook nodes
- **Workflow and Instance IDs** are auto-generated during import
- **Sub-workflow references** in "Execute Workflow" nodes may need to be re-linked after import if they don't auto-connect
- After configuration, save and activate all workflows (activate main workflows last)
- Copy the full webhook URLs from n8n and use them to configure Apaleo webhook subscriptions (see Apaleo Configuration section)

### Testing the Trace Generation Flow

1. Create a test reservation or booking in Apaleo with comments
2. The webhook should trigger the workflow automatically
3. Monitor the workflow execution in n8n
4. Check the `trace_logs` table in Supabase to verify the traces were created
5. Verify traces appear in Sweeply (if integration is active)

### Monitoring

- **n8n Executions**: View all workflow runs in the n8n execution history
- **Supabase Dashboard**: Query the `trace_logs` table to see all generated traces
- **Daily Email**: Receive automatic summaries every day at 23:59

## Troubleshooting

### Webhooks Not Triggering

- Verify webhook subscriptions are active in Apaleo
- Check webhook URLs are correctly configured
- Ensure n8n instance is publicly accessible

### AI Agent Issues

- Verify OpenAI API key is valid and has sufficient credits
- Check the Task Agent node configuration
- Review execution logs for API errors

### Database Connection Issues

- Verify Supabase credentials in n8n
- Ensure the `trace_logs` table exists and has the correct schema
- Check PostgreSQL connection pooler settings for the Daily Email flow

### Lock-Related Issues

#### Workflows Stuck in Wait Loop
- Check the `webhook_event_queue` table for stale locks
- If a workflow crashed without releasing its lock, manually delete the stale entry:
  ```sql
  DELETE FROM webhook_event_queue WHERE event_identifier = 'stuck_key_here';
  ```

#### Duplicate Tasks Being Created
- Verify the lock sub-workflows are correctly linked in the main workflows
- Check that "Release lock" is called on ALL exit paths (including error branches)
- Ensure the `webhook_event_queue` table exists with the correct schema

### No Email Reports

- Verify the Schedule Trigger is active
- Check email credentials are configured correctly
- Ensure Account details node has correct email addresses
- Verify PostgreSQL connection to Supabase database

## Summary

The Apaleo AI Trace Management System provides an intelligent, automated solution for hotel operations management. By leveraging AI to analyze booking comments and generate actionable tasks, the system reduces manual work and ensures important guest requests are properly captured and assigned. The daily email reports provide visibility into all generated traces, helping management track operational efficiency.

### Key Features

- **Intelligent Task Generation**: AI-powered analysis of booking and reservation comments
- **Duplicate Prevention**: Checks existing traces to avoid redundant tasks
- **Concurrency Control**: Database-based locking mechanism prevents race conditions when multiple webhooks arrive simultaneously
- **Modular Architecture**: Separated sub-workflows for maintainability and reusability
- **Smart Role Assignment**: Automatically assigns tasks to appropriate departments
- **Priority Management**: Sets task priorities based on urgency and type
- **Comprehensive Logging**: Stores all traces with full metadata in Supabase
- **Daily Reporting**: Automated Excel reports sent via email
- **Integration Ready**: Works seamlessly with Apaleo, Sweeply, and other hotel management systems

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [MIT License](MIT%20License) file for details.

## Support

For issues, questions, or contributions, please open an issue in the GitHub repository or contact us at info@softup.co.
