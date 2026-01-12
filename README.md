# Support Ticket Classifier Evaluation

This evaluation workflow tests an LLM-based support ticket classification system. It fetches the support tickets in the Google Sheet "Customer support ticket eval 1", classifies them using a language model, and compares the predictions against expected ground truth values to calculate accuracy metrics.

## Overview

The eval workflow evaluates how well an LLM can classify customer support tickets across multiple dimensions:
- **Priority** (urgency level)
- **Queue** (routing to appropriate support team)
- **Type** (ticket category)
- **Tags** (relevant keywords)
- **Language** (detection)
- **Response** (generating a helpful customer response)

The workflow uses n8n to orchestrate the evaluation process, integrating with Hugging Face datasets, Ollama API for LLM inference, and Google Sheets for storing results and ground truth data.

## Workflow Architecture

The evaluation workflow consists of the following components:

```
Evaluation Trigger (Google Sheets)
    ↓
Manual Trigger / No Operation
    ↓
HTTP Request (Fetch Random Ticket from Hugging Face)
    ↓
Extract Ticket (Extract subject and body)
    ↓
LLM Classify Ticket (Ollama API)
    ↓
Parse Response (Extract classification fields)
    ↓
Set Outputs (Write predictions to Google Sheets)
    ↓
Evaluation (Calculate match metrics)
```

### Workflow Components

1. **Evaluation Trigger**: Reads test cases from Google Sheets with expected values
2. **HTTP Request**: Fetches a random support ticket from the Hugging Face dataset (`Tobi-Bueck/customer-support-tickets`)
3. **Extract Ticket**: Extracts `subject` and `body` fields from the dataset response
4. **LLM Classify Ticket**: Sends the ticket to Ollama API (nemotron-3-nano:30b-cloud model) for classification
5. **Parse Response**: Parses the JSON response from the LLM and extracts all classification fields
6. **Set Outputs**: Writes predicted values back to Google Sheets
7. **Evaluation**: Calculates binary match metrics comparing predictions to expected values

## Classification Schema

The LLM is prompted to classify tickets according to the following schema:

### Priority
One of: `low`, `medium`, `high`, `critical`

### Queue (Support Team)
One of:
- Technical Support
- Billing and Payments
- Customer Service
- IT Support
- Product Support
- Sales and Pre-Sales
- Returns and Exchanges
- Service Outages and Maintenance
- Human Resources
- General Inquiry

### Type
One of: `Incident`, `Request`, `Problem`, `Change`

### Tags
1-3 relevant tags (e.g., Network, Security, Billing, Hardware, Performance, etc.)

### Language
ISO 639-1 language code (e.g., `en`, `de`, `fr`, `es`, `ja`)

### Response
A professional, helpful response to the customer written in the detected language

### Reasoning
One sentence explaining the classification decision

## Metrics and Evaluation

The evaluation calculates binary match metrics (1 for match, 0 for mismatch) for the following fields:

- **type_match**: Whether predicted type matches expected type
- **queue_match**: Whether predicted queue matches expected queue
- **priority_match**: Whether predicted priority matches expected priority
- **language_match**: Whether predicted language matches expected language

These metrics are stored as custom metrics in the n8n evaluation system and can be aggregated across multiple test cases to calculate overall accuracy rates.

## Data Sources

### Input Dataset
- **Source**: Hugging Face Datasets Server
- **Dataset**: `Tobi-Bueck/customer-support-tickets`
- **Split**: `train`
- **Selection**: Random ticket (offset = random number between 0-1000)

### Ground Truth Storage
- **Platform**: Google Sheets
- **Document**: "Customer support ticket eval 1"
- **Sheet**: "Sheet2"
- **Columns**: Contains expected values (`expected_type`, `expected_queue`, `expected_priority`, `expected_language`, `expected_tag1-3`, `expected_answer`) and predicted values (`predicted_type`, `predicted_queue`, `predicted_priority`, `predicted_language`, `predicted_tag1-3`, `predicted_response`, `reasoning`)

## LLM Configuration

- **API**: Ollama Cloud API (`https://ollama.com/api/chat`)
- **Model**: `nemotron-3-nano:30b-cloud`
- **Authentication**: Bearer token (configured in workflow)
- **Prompt**: Structured prompt that instructs the LLM to:
  1. Analyze the ticket subject and body
  2. Classify across all required dimensions
  3. Generate a customer response
  4. Provide reasoning
  5. Return results as valid JSON

The prompt uses n8n expression syntax to inject ticket data: `{{ $json.subject }}` and `{{ $json.body }}`

## Setup and Configuration

### Prerequisites
- n8n instance with evaluation nodes enabled
- Google Sheets API credentials configured
- Ollama API access with valid bearer token

### Configuration Steps
1. Import the workflow JSON file (`Support ticket classifier - eval.json`) into n8n
2. Configure Google Sheets OAuth2 credentials
3. Update Ollama API bearer token in the "LLM Classify Ticket" node
4. Ensure the Google Sheet exists with the correct structure (see Data Sources section)
5. Activate the workflow

### Running the Evaluation

The workflow can be triggered in two ways:

1. **Manual Execution**: Click "Execute workflow" to run a single evaluation
2. **Evaluation Trigger**: Automatically runs when triggered by the evaluation system reading from Google Sheets

Each execution:
- Fetches one random ticket from the dataset
- Classifies it using the LLM
- Writes predictions to Google Sheets
- Calculates and stores match metrics

## Results and Outputs

### Google Sheets Output
All predictions are written to the Google Sheet with the following columns:
- `predicted_type`
- `predicted_queue`
- `predicted_priority`
- `predicted_language`
- `predicted_tag1`, `predicted_tag2`, `predicted_tag3`
- `predicted_response`
- `reasoning`

### Evaluation Metrics
Metrics are stored in n8n's evaluation system and can be viewed in the evaluation dashboard. Each metric is a binary value (0 or 1) indicating whether the prediction matched the expected value.

## Examples

### Example 1: German Login Issue
**Input:**
- Subject: "Probleme bei der Anmeldung Benutzerkonten"
- Body: "Ein großes Problem bei der Anmeldung wurde festgestellt..."

**Expected:**
- Type: `Incident`
- Queue: `Customer Service`
- Priority: `high`
- Language: `de`
- Tags: `login`, `bug`, `performance`

**Predicted:**
- Type: `Request` (mismatch)
- Queue: `Product Support` (mismatch)
- Priority: `medium` (mismatch)
- Language: `de` (match)

### Example 2: Medical Data Security Request
**Input:**
- Subject: "Guidance for Securing Medical Data with Action-Kamera on macOS Monterey"
- Body: "Dear Customer Support, I am reaching out for assistance..."

**Expected:**
- Type: `Request`
- Queue: `Technical Support`
- Priority: `medium`
- Language: `en`
- Tags: `security`, `documentation`, `IT`

**Predicted:**
- Type: `Problem` (mismatch)
- Queue: `IT Support` (mismatch)
- Priority: `high` (mismatch)
- Language: `en` (match)

These examples show that while language detection is often accurate, classification of type, queue, and priority can vary, highlighting areas for model improvement.

