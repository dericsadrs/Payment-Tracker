# Payment Tracker

Automated payment notification extractor that monitors Gmail for payment emails from various services (Wise, PayPal, Remitly, Bill.com) and logs them to a Notion database.

## Features

- **Email Monitoring**: Connects to Gmail via IMAP to scan for payment notifications
- **Multi-Service Support**: Extracts payments from Wise, PayPal, Remitly, and Bill.com
- **Smart Extraction**: Uses regex patterns to extract sender, amount, currency, and dates
- **Duplicate Prevention**: Prevents duplicate entries using message IDs
- **Automated Scheduling**: Runs twice monthly (1st and 15th) via Cloud Scheduler
- **Notion Integration**: Automatically creates and manages Notion database schema

## Architecture

- **Runtime**: Google Cloud Functions (Python 3.9)
- **Scheduler**: Google Cloud Scheduler 
- **Database**: Notion API
- **Email**: Gmail IMAP
- **Secrets**: Google Secret Manager

## Setup

### Prerequisites

1. Google Cloud Project with billing enabled
2. Gmail account with App Password configured
3. Notion workspace with API integration

### Required Secrets (Google Secret Manager)

```bash
# Gmail credentials
gmail-app-password: your-gmail-app-password

# Notion API
notion-api-token: your-notion-integration-token
notion-database-id: your-notion-database-id  # Optional, creates if not provided
```

### Environment Variables

- `GCP_PROJECT_ID`: Your Google Cloud project ID
- `GMAIL_APP_PASSWORD`: Gmail app password (from Secret Manager)
- `NOTION_TOKEN`: Notion API token (from Secret Manager)
- `NOTION_DATABASE_ID`: Notion database ID (from Secret Manager)

### Deploy

1. Clone repository
2. Set up GitHub secrets:
   - `GCP_PROJECT_ID`
   - `GCP_SA_KEY` (Service Account JSON)
   - Secret Manager secrets (above)
3. Push to `main` branch - GitHub Actions will deploy automatically

Or deploy manually:
```bash
gcloud functions deploy payment-extractor \
  --runtime python39 \
  --trigger-http \
  --allow-unauthenticated \
  --region us-central1 \
  --source . \
  --entry-point payment_extractor \
  --set-secrets "GMAIL_APP_PASSWORD=gmail-app-password:latest,NOTION_TOKEN=notion-api-token:latest,NOTION_DATABASE_ID=notion-database-id:latest"
```

## Configuration

### Gmail Setup

1. Enable 2-factor authentication
2. Generate App Password: Google Account → Security → App passwords
3. Store in Secret Manager as `gmail-app-password`

### Notion Setup

1. Create integration: https://www.notion.so/my-integrations
2. Get API token and store as `notion-api-token`
3. (Optional) Create database and share with integration, store ID as `notion-database-id`

### Supported Email Services

| Service | Email Pattern |
|---------|---------------|
| Wise | `noreply@wise.com` |
| PayPal | `service@paypal.com` |
| Remitly | `no-reply@remitly.com` |
| Bill.com | `account-services@hq.bill.com` |

## Usage

### Automatic Execution
- Runs automatically on 1st and 15th of each month at 9 AM EST
- Scans last 600 days of emails for payments

### Manual Execution
```bash
# Test endpoint
curl -X POST "https://us-central1-PROJECT_ID.cloudfunctions.net/payment-extractor" \
  -H "Content-Type: application/json" \
  -d '{"test": true}'

# Full extraction
curl -X POST "https://us-central1-PROJECT_ID.cloudfunctions.net/payment-extractor" \
  -H "Content-Type: application/json"
```

## Notion Database Schema

The system automatically creates/updates the database with these fields:

| Field | Type | Description |
|-------|------|-------------|
| Sender | Title | Payment sender name |
| Service | Select | Payment service (Wise, PayPal, etc.) |
| Amount | Number | Payment amount |
| Currency | Select | Currency code (USD, PHP, EUR, etc.) |
| Date | Date | Email date |
| Subject | Rich Text | Email subject |
| Days Ago | Rich Text | Human-readable time difference |
| Message ID | Rich Text | Unique email identifier |
| From Email | Rich Text | Sender email address |
| To Email | Rich Text | Recipient email address |
| Extraction Timestamp | Rich Text | When data was extracted |
| Created | Created Time | Record creation time |

## Monitoring

### Logs
- Cloud Functions → payment-extractor → Logs tab
- Cloud Logging: Filter by `resource.type="cloud_function"`

### Scheduler Status
- Cloud Scheduler → payment-tracker-monthly → Execution history

### Health Check
```bash
curl "https://us-central1-PROJECT_ID.cloudfunctions.net/payment-extractor/health"
```

## Project Structure

```
├── src/
│   ├── __init__.py
│   ├── logger.py              # Centralized logging
│   ├── paytment_extractor.py  # Email processing logic
│   └── notion_client.py       # Notion API integration
├── main.py                    # Cloud Function entry point
├── requirements.txt           # Python dependencies
├── .github/workflows/
│   └── deploy.yml            # GitHub Actions deployment
└── README.md
```

## Troubleshooting

### Common Issues

**Authentication Error**
- Verify Gmail App Password is correct
- Check Secret Manager permissions

**No Payments Found**
- Check email search patterns in logs
- Verify payment keywords detection
- Ensure emails are in inbox (not spam/other folders)

**Notion API Errors**
- Verify integration has database access
- Check database ID is correct
- Ensure API token has write permissions

### Debug Mode

Add detailed logging by checking Cloud Function logs for:
- `RAW_EMAIL_JSON`: Full email content
- `Payment keywords found`: Keyword detection
- `Amount extracted`: Currency/amount parsing
- `Sender extracted`: Sender name parsing

## Security

- All credentials stored in Google Secret Manager
- No sensitive data in code repository
- HTTPS-only endpoints
- Service account with minimal required permissions