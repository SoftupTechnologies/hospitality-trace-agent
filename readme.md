# Apaleo AI Trace Management System

An intelligent automation system that processes Apaleo booking and reservation data to automatically generate actionable tasks (traces) using AI. The system consists of two complementary n8n workflows:

1. **Trace Generation Flow**: Processes Apaleo webhooks in real-time to create intelligent traces from booking comments and reservation data
2. **Daily Traces Email Flow**: Sends a daily summary report of all generated traces as an Excel file at 8:00 AM of the next day.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [1. Apaleo Configuration](#1-apaleo-configuration)
  - [2. OpenAI Configuration](#2-openai-configuration)
  - [3. Supabase Database Setup](#3-supabase-database-setup)
  - [4. Sweeply Integration Setup](#4-sweeply-integration-setup)
  - [5. Daily Email Report Configuration](#5-daily-email-report-configuration)
- [Flow Architecture](#flow-architecture)
- [Usage](#usage)
  - [Importing Workflows](#importing-workflows)
  - [Workflow Configuration Placeholders](#workflow-configuration-placeholders)
  - [Testing the Trace Generation Flow](#testing-the-trace-generation-flow)

## Prerequisites

Before setting up this system, ensure you have:

- An active [Apaleo](https://apaleo.com/) account with access to create custom apps
- An [OpenAI](https://platform.openai.com/) account with API access
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
    "booking/changed"
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

### 3. Supabase Database Setup

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

#### Create the trace_logs Table

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
) tablespace pg_default;
```

Click **Run** to create the table. This table will store all generated traces and their associated metadata.

### 4. Sweeply Integration Setup

The system integrates with Sweeply for task management. You need to configure Basic Authentication credentials.

#### Configure Sweeply Basic Auth in n8n

1. In the **Traces Agent** workflow, locate the **Sweeply Basic Auth Token** node
2. Update the value of the **authorization** field (Basic {{ "username:password".base64Encode() }}). Replace username and password with the real values from Sweeply
3. Click **Save**

**Note:** The workflow currently uses the Sweeply development API endpoint (`https://app.getsweeply.dev`). The production URL is not yet available. Update the API URLs in the **Post to Sweeply** and **Update to Sweeply** nodes when the production endpoint becomes available.

### 5. Daily Email Report Configuration

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

### Flow Phases

#### Phase 1: Webhook Reception

##### Receive Reservation Webhook
- **Node**: `Receive reservation`
- **Purpose**: Receives webhook events from Apaleo when a reservation is created (`reservation/*` events)
- **Output**: Webhook payload containing reservation entity ID and account information
- **Next Step**: Triggers the "Get reservation" node

##### Receive Booking Changes Webhook
- **Node**: `Receive booking changes`
- **Purpose**: Receives webhook events from Apaleo when a booking is modified (`booking/changed` events)
- **Output**: Webhook payload containing booking entity ID and account information
- **Next Step**: Triggers the "Map Booking Id / Account Id" node

#### Phase 2: Data Extraction and Mapping

##### Map Booking Id / Account Id
- **Node**: `Map Booking Id / Account Id`
- **Purpose**: Extracts the booking ID and account ID from the booking changes webhook payload
- **Input**: Webhook payload from "Receive booking changes"
- **Output**: Structured data with `bookingId` and `accountId` fields
- **Next Step**: Triggers "Get booking details"

##### Get Reservation
- **Node**: `Get reservation`
- **Purpose**: Fetches detailed reservation information from Apaleo API using the reservation ID from the webhook
- **API Call**: `BookingReservationsByIdGet` operation
- **Input**: Reservation ID from webhook (`$json.body.data.entityId`)
- **Output**: Complete reservation object with unit, unit group, property, and comment details
- **Next Step**: Triggers "Get booking details"

##### Get Booking Details
- **Node**: `Get booking details`
- **Purpose**: Fetches complete booking information from Apaleo API, including all reservations
- **API Call**: `BookingBookingsByIdGet` operation with `expand=reservations`
- **Input**: Booking ID (from webhook or previous node)
- **Output**: Complete booking object with:
  - Booking-level comments (`bookerComment`, `comment`)
  - Booker information (first name, last name)
  - All associated reservations
  - Property information
- **Next Step**: Triggers "Extract booking details"

#### Phase 3: Data Processing

##### Extract Booking Details
- **Node**: `Extract booking details`
- **Purpose**: Combines and normalizes data from booking and reservation sources into a unified structure
- **Logic**:
  - Reads booking data from "Get booking details"
  - Optionally reads reservation data from "Get reservation" (if available)
  - Matches the reservation from the booking's reservations array
  - Extracts and prioritizes comments:
    - Booking-level: `bookerComment`, `extraBookingComment` (from `comment` field)
    - Reservation-level: `reservationComment`, `guestComment`
  - Extracts unit, unit group, and property details (preferring reservation-level data when available)
  - Handles fallback scenarios when reservation data is missing (booking-only flow)
- **Output**: Normalized JSON object containing:
  - Identifiers: `propertyId`, `propertyName`, `reservationId`, `bookingId`
  - Booker info: `bookerFirstName`, `bookerLastName`
  - Comments: `bookerComment`, `reservationComment`, `guestComment`, `extraBookingComment`
  - Unit details: `unitId`, `unitName`, `unitDescription`, `unitGroupIdFromUnit`
  - Unit group: `unitGroupId`, `unitGroupCode`, `unitGroupName`, `unitGroupType`
  - Dates: `arrival`, `departure`
  - Other: `adults`, `channelCode`, `ratePlanCode`
- **Next Step**: Triggers "Task agent"

#### Phase 4: AI Task Generation

##### Task Agent
- **Node**: `Task agent`
- **Type**: LangChain Agent with OpenAI model
- **Model**: GPT-5 (configured for JSON output)
- **Purpose**: Analyzes booking/reservation data and generates actionable tasks using AI
- **System Role**: Hotel operations router that converts booking comments into Sweeply-ready tasks
- **Input**: Normalized booking details from "Extract booking details"
- **Process**:
  1. Receives structured input with all booking/reservation data and comments
  2. Uses the "Get trace logs" tool to check for existing tasks and prevent duplicates
  3. Analyzes comments to identify actionable items
  4. Applies business rules:
     - Splits multiple requests into separate tasks
     - Assigns appropriate roles based on task type
     - Determines priority levels
     - Sets due dates for time-sensitive tasks
     - Checks for extra bed requirements based on room type and number of adults
  5. Generates structured JSON output matching the defined schema
- **Role Assignment Rules**:
  - **Property Admin**: Maintenance, equipment, infrastructure, property-wide operations
  - **Reservation Manager**: Bookings, cancellations, changes, rate issues, payments, invoices
  - **Rezeptionist/in**: Guest communications, check-in/out notes, key handover, generic front-desk actions
  - **Senior Rezeptionist/in**: Escalations, VIP/edge cases, unresolved issues
  - **Housekeeping**: Cleaning, room prep, amenities (towels, linens, pillows), turn-down service
- **Output**: JSON object with `tasks` array, where each task contains:
  - `title`: Action-oriented task summary
  - `description`: Optional detailed description
  - `assigned_to`: Role/department responsible
  - `priority`: "low", "normal", or "high"
  - `due`: ISO 8601 datetime (if applicable)
- **Next Step**: Triggers "Map booking details to tasks"

##### Get Trace Logs (Tool)
- **Node**: `Get trace logs`
- **Type**: Supabase Tool (used by Task Agent)
- **Purpose**: Retrieves existing trace logs for a booking to prevent duplicate task creation
- **Operation**: Queries Supabase `trace_logs` table filtered by `booking_id`
- **Usage**: The Task Agent calls this tool before generating tasks to:
  - Check if similar tasks already exist
  - Compare task titles and descriptions
  - Avoid creating repetitive traces for the same booking/reservation
- **Output**: Array of existing trace log records

##### Structured Output Parser
- **Node**: `Structured Output Parser1`
- **Purpose**: Ensures the AI agent outputs valid JSON conforming to the defined schema
- **Schema**: Defines the structure for task objects with required and optional fields
- **Validation**: Enforces that output matches the expected format exactly

#### Phase 5: Task Enrichment

##### Map Booking Details to Tasks
- **Node**: `Map booking details to tasks`
- **Purpose**: Enriches AI-generated tasks with booking/reservation metadata
- **Input**: 
  - Tasks array from "Task agent"
  - Booking details from "Extract booking details"
- **Process**:
  - Takes each generated task from the AI agent
  - Merges it with booking/reservation identifiers and details
  - Converts field names to snake_case for database storage
- **Output**: Array of enriched task objects, each containing:
  - AI-generated fields: `title`, `description`, `assigned_to`, `priority`, `due`
  - Booking metadata: `reservation_id`, `booking_id`, `booker_first_name`, `booker_last_name`
  - Unit metadata: `unit_id`, `unit_name`, `unit_description`, `unit_group_id_from_unit`
  - Unit group metadata: `unit_group_id`, `unit_group_code`, `unit_group_name`, `unit_group_type`
- **Next Step**: Triggers "Source webhook"

#### Phase 6: Trace Logging

##### Source Webhook
- **Node**: `Source webhook`
- **Purpose**: Determines which webhook triggered the flow (reservation or booking changes)
- **Logic**: Attempts to read from both webhook nodes and returns whichever one executed
- **Output**: Original webhook payload
- **Next Step**: Triggers "Webhook details"

##### Webhook Details
- **Node**: `Webhook details`
- **Purpose**: Extracts metadata from the webhook payload for trace logging
- **Extracted Fields**:
  - `webhook_entity_id`: Entity ID from webhook (`$json.body.data.entityId`)
  - `webhook_topic`: Webhook topic/event type (`$json.body.topic`)
  - `apaleo_account_id`: Account ID from webhook (`$json.body.accountId`)
- **Output**: Structured webhook metadata
- **Next Step**: Triggers "Create trace logs"

##### Create Trace Logs
- **Node**: `Create trace logs`
- **Type**: Supabase Insert Operation
- **Purpose**: Saves all generated tasks and webhook metadata to the database
- **Table**: `trace_logs`
- **Data Stored**:
  - `apaleo_property_id`: Property ID from booking
  - `webhook_topic`: Event type that triggered the flow
  - `apaleo_account_id`: Apaleo account identifier
  - `webhook_entity_id`: Entity ID from webhook
  - `traces`: Array of all generated tasks (from "Map booking details to tasks")
  - `booking_id`: Booking identifier
- **Output**: Confirmation of successful database insertion

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
3. Select the workflow JSON files:
   - `Traces Agent.json` - Main trace generation flow
   - `Daily Traces Email.json` - Daily email report flow
4. **Do not activate yet** - you must replace the placeholders first (see table below)

### Workflow Configuration Placeholders

After importing the workflows into n8n, you need to configure the following values. Most credential connections will be set up through n8n's credential manager, but you'll need to manually update these specific placeholders:

| Placeholder | Workflow File | Node/Location | What to Replace With |
|------------|---------------|---------------|---------------------|
| `Bearer xxxxxxxx` | Daily Traces Email.json | Send email node → Headers | Your Resend API key (format: `Bearer re_xxxxx`) |
| `username:password` | Traces Agent.json | Sweeply Basic Auth Token node | Your Sweeply credentials (will be base64 encoded automatically) |
| `Your Company Name <no-reply@yourdomain.com>` | Daily Traces Email.json | Send email node → Body Parameters | Your verified sender email address |
| `ACCOUNT_ID` | Daily Traces Email.json | Account details node | Your Apaleo account ID |
| `PROPERTY_ID` | Daily Traces Email.json | Account details node | Your Apaleo property ID |
| `EMAIL_ADDRESS` | Daily Traces Email.json | Account details node | Recipient email address(es) for daily reports |
| `YOUR_RESERVATION_WEBHOOK_PATH` | Traces Agent.json | Receive reservation webhook node | Custom webhook path (e.g., `apaleo/reservations`) |
| `YOUR_BOOKING_WEBHOOK_PATH` | Traces Agent.json | Receive booking changes webhook node | Custom webhook path (e.g., `apaleo/bookings`) |

**Important Notes:**
- **Credential IDs** (`YOUR_APALEO_CREDENTIAL_ID`, `YOUR_OPENAI_CREDENTIAL_ID`, etc.) will be automatically set when you connect credentials through n8n's credential selector in each node
- **Webhook IDs** will be auto-generated by n8n when you save the webhook nodes
- **Workflow and Instance IDs** are auto-generated during import
- After configuration, save and activate both workflows
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
