# Complete System Flow - Microsoft Graph Webhooks

This document explains the complete end-to-end flow from Azure AD authentication to email processing.

---

## Table of Contents

1. [Azure AD Setup & Authentication](#azure-ad-setup--authentication)
2. [Subscription Creation Flow](#subscription-creation-flow)
3. [Notification Delivery Flow](#notification-delivery-flow)
4. [Complete Email Processing Flow](#complete-email-processing-flow)
5. [Security & Secrets](#security--secrets)

---

## Azure AD Setup & Authentication

### What is Azure AD App Registration?

Think of it as an **identity card** for your application that allows it to access Microsoft services.

### Setup (One-time)

```
You (Admin) → Azure Portal → App Registrations → New Registration
     ↓
Creates: Application with CLIENT_ID
     ↓
Generate: CLIENT_SECRET (like a password for the app)
     ↓
Grant Permissions: Mail.Read (Application level)
     ↓
Admin Consent: Approve these permissions
```

**What you get:**
- `CLIENT_ID`: Public identifier (like username)
- `CLIENT_SECRET`: Private key (like password)
- `TENANT_ID`: Your organization's ID

### Client Credentials Flow (How Authentication Works)

Every time your app needs to access Microsoft Graph:

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Request Access Token                                │
└─────────────────────────────────────────────────────────────┘

Your Server                           Microsoft Identity Platform
    │                                           │
    │  POST /oauth2/v2.0/token                 │
    │  {                                        │
    │    client_id: "abc123...",               │
    │    client_secret: "secret456...",        │
    │    grant_type: "client_credentials",     │
    │    scope: "https://graph.microsoft.com/.default"
    │  }                                        │
    │──────────────────────────────────────────>│
    │                                           │
    │  Validates:                               │
    │  - Is CLIENT_ID valid?                    │
    │  - Is CLIENT_SECRET correct?              │
    │  - Does app have permissions?             │
    │                                           │
    │  <- Returns Access Token                  │
    │  {                                        │
    │    access_token: "eyJ0eXAi...",          │
    │    expires_in: 3600 (1 hour)             │
    │  }                                        │
    │<──────────────────────────────────────────│
    │                                           │
    │  Token cached for 1 hour                  │
    │                                           │
```

**Code Implementation:**

```python
def get_access_token():
    # Step 1: Prepare request
    token_url = f'https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token'
    
    data = {
        'client_id': CLIENT_ID,         # Who you are
        'client_secret': CLIENT_SECRET,  # Prove it's you
        'scope': 'https://graph.microsoft.com/.default',  # What you want
        'grant_type': 'client_credentials'  # How you authenticate
    }
    
    # Step 2: Request token
    response = requests.post(token_url, data=data)
    token_data = response.json()
    
    # Step 3: Extract and cache token
    access_token = token_data['access_token']  # Valid for 1 hour
    
    return access_token
```

**Why Client Secret?**

- Proves your application is legitimate
- Like a password, but for applications
- Never expires (unless you regenerate it)
- Must be kept secret (hence the name)

---

## Subscription Creation Flow

### What is a Subscription?

A **subscription** tells Microsoft: *"When an email arrives at this mailbox, notify my webhook URL"*

### Complete Subscription Creation Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Your Server Requests Subscription                   │
└─────────────────────────────────────────────────────────────┘

Your Server                           Microsoft Graph API
    │                                           │
    │  POST /v1.0/subscriptions                │
    │  Authorization: Bearer {access_token}     │
    │  {                                        │
    │    "changeType": "created",               │  <- What events to notify
    │    "notificationUrl": "https://your-webhook.com/webhook",
    │    "resource": "users/email@company.com/messages",  <- What to monitor
    │    "expirationDateTime": "2025-12-25T...", │  <- When subscription expires (3 days max)
    │    "clientState": "babajishivram@1706"    │  <- Your secret for validation
    │  }                                        │
    │──────────────────────────────────────────>│
    │                                           │

┌─────────────────────────────────────────────────────────────┐
│ Step 2: Microsoft Validates Your Webhook                    │
└─────────────────────────────────────────────────────────────┘

Microsoft Graph                       Your Webhook Server
    │                                           │
    │  POST /webhook?validationToken=abc123...  │
    │──────────────────────────────────────────>│
    │                                           │
    │                              Receives validation token
    │                              Must respond within 5 seconds
    │                                           │
    │  <- 200 OK                                │
    │  Plain text: "abc123..."                  │
    │<──────────────────────────────────────────│
    │                                           │

┌─────────────────────────────────────────────────────────────┐
│ Step 3: Subscription Created                                │
└─────────────────────────────────────────────────────────────┘

Microsoft Graph                       Your Server
    │                                           │
    │  <- 201 Created                           │
    │  {                                        │
    │    "id": "fe1dca07-6321...",             │  <- Subscription ID
    │    "resource": "users/email@company.com/messages",
    │    "notificationUrl": "https://your-webhook.com/webhook",
    │    "expirationDateTime": "2025-12-25T23:59:59Z",
    │    "clientState": "babajishivram@1706"   │
    │  }                                        │
    │<──────────────────────────────────────────│
    │                                           │
```

**Key Points:**

1. **Validation is required** - Microsoft tests your webhook before creating subscription
2. **Must respond fast** - You have 5 seconds to respond with validation token
3. **clientState** - Your secret that Microsoft includes in every notification
4. **Expires in 3 days** - That's why we have auto-renewal

---

## Notification Delivery Flow

### When Email Arrives

```
┌─────────────────────────────────────────────────────────────┐
│ Email Arrives at Mailbox                                    │
└─────────────────────────────────────────────────────────────┘

Someone sends email
       ↓
Microsoft 365 Mailbox (it.ops@babajishivram.com)
       ↓
Email created in Inbox
       ↓
Microsoft Graph Event System detects change
       ↓
Checks active subscriptions
       ↓
Finds your subscription
       ↓
Triggers notification

┌─────────────────────────────────────────────────────────────┐
│ Microsoft Sends Notification (within seconds)               │
└─────────────────────────────────────────────────────────────┘

Microsoft Graph                       Your Webhook
    │                                           │
    │  POST /webhook                            │
    │  {                                        │
    │    "value": [                             │
    │      {                                    │
    │        "subscriptionId": "fe1dca07...",   │
    │        "clientState": "babajishivram@1706",  <- Your secret
    │        "changeType": "created",           │
    │        "resource": "users/it.ops@babajishivram.com/messages/AAMkADU...",
    │        "resourceData": {                  │
    │          "@odata.type": "#Microsoft.Graph.Message",
    │          "id": "AAMkADU..."               │  <- Message ID
    │        }                                  │
    │      }                                    │
    │    ]                                      │
    │  }                                        │
    │──────────────────────────────────────────>│
    │                                           │
    │                              Validates clientState
    │                              Must respond within 5 seconds
    │                                           │
    │  <- 200 OK {"status": "accepted"}        │
    │<──────────────────────────────────────────│
    │                                           │
```

**Important:**
- Notification contains **only the message ID**, not the full email
- You must fetch full email using Graph API
- Response must be within 5 seconds (that's why we use fire-and-forget)

---

## Complete Email Processing Flow

### End-to-End Flow with All Components

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Email Arrives                                            │
└─────────────────────────────────────────────────────────────┘

vendor@example.com sends email with attachment
       ↓
Microsoft 365 Mailbox: it.ops@babajishivram.com/Inbox
       ↓
Email ID: AAMkADUwZTYy...

┌─────────────────────────────────────────────────────────────┐
│ 2. Notification Sent (< 1 second)                           │
└─────────────────────────────────────────────────────────────┘

Microsoft Graph → POST /webhook
{
  "value": [{
    "subscriptionId": "fe1dca07-6321-4349-894c-23733f128f10",
    "clientState": "babajishivram@1706",
    "resource": "users/it.ops@babajishivram.com/messages/AAMkADUwZTYy...",
    "resourceData": {"id": "AAMkADUwZTYy..."}
  }]
}

┌─────────────────────────────────────────────────────────────┐
│ 3. Webhook Endpoint (Your Server)                           │
└─────────────────────────────────────────────────────────────┘

@router.post("/webhook")
async def webhook_notification(request: Request):
    data = await request.json()
    
    # Validate client state
    await webhook_validator.validate_notification(data)  # Checks "babajishivram@1706"
    
    # Log
    processing_logger.log_notification_received(len(data['value']))
    
    # Fire and forget - respond immediately
    asyncio.create_task(process_notifications(data['value']))
    
    return {"status": "accepted"}  # < 5 seconds

┌─────────────────────────────────────────────────────────────┐
│ 4. Background Processing Starts                             │
└─────────────────────────────────────────────────────────────┘

async def process_notifications(notifications):
    # Deduplicate
    unique = deduplicator.deduplicate(notifications)  # 1 notification
    
    # Load utilities
    utilities = await config_service.get_all_utilities()
    
    # Process
    for notification in unique:
        await process_single_email(notification, utilities)

┌─────────────────────────────────────────────────────────────┐
│ 5. Fetch Full Email from Graph API                          │
└─────────────────────────────────────────────────────────────┘

async def process_single_email(notification, utilities):
    # Extract message ID from notification
    resource = notification['resource']
    # "users/it.ops@babajishivram.com/messages/AAMkADUwZTYy..."
    
    message_id = "AAMkADUwZTYy..."
    mailbox = "it.ops@babajishivram.com"
    
    # Fetch full email
    email = await email_fetcher.fetch_email(notification)

┌─────────────────────────────────────────────────────────────┐
│ 6. Email Fetcher Gets Full Data                             │
└─────────────────────────────────────────────────────────────┘

async def fetch_email(notification):
    # Get access token
    token = self.graph.get_access_token()  # Uses CLIENT_ID + CLIENT_SECRET
    
    # Fetch email metadata
    url = f'https://graph.microsoft.com/v1.0/users/{mailbox}/messages/{message_id}'
    headers = {'Authorization': f'Bearer {token}'}
    
    response = requests.get(url, headers=headers)
    email_data = response.json()
    
    # Download attachments if present
    if email_data['hasAttachments']:
        attachments = await attachment_downloader.download_attachments(mailbox, message_id)
    
    # Create EmailMetadata object
    return EmailMetadata(...)

┌─────────────────────────────────────────────────────────────┐
│ 7. Download Attachments                                     │
└─────────────────────────────────────────────────────────────┘

async def download_attachments(mailbox, message_id):
    token = graph_service.get_access_token()  # Same token
    
    url = f'https://graph.microsoft.com/v1.0/users/{mailbox}/messages/{message_id}/attachments'
    
    response = await aiohttp.get(url, headers={'Authorization': f'Bearer {token}'})
    attachments_data = await response.json()
    
    attachments = []
    for att in attachments_data['value']:
        # Content is base64 encoded
        content_bytes = base64.b64decode(att['contentBytes'])
        
        attachments.append({
            'name': att['name'],
            'content': content_bytes,  # Actual file bytes
            'size': att['size']
        })
    
    return attachments

┌─────────────────────────────────────────────────────────────┐
│ 8. Rule Matching                                            │
└─────────────────────────────────────────────────────────────┘

matched_utilities = await RuleMatcher.find_matching_utilities(email, utilities)

# Checks:
# - Email from: vendor@example.com ✓
# - Has attachments: True ✓
# - Subject contains keywords ✓
# → Matches "Attachment Processor" utility

┌─────────────────────────────────────────────────────────────┐
│ 9. Dispatch to Utility APIs                                 │
└─────────────────────────────────────────────────────────────┘

await Dispatcher.dispatch_to_utilities(email, matched_utilities)

For each utility:
    # Serialize email to JSON
    json_data = email.to_dict()  # Converts bytes to base64
    
    # POST to utility API
    async with retry_handler:
        response = await aiohttp.post(
            utility.endpoint['url'],  # https://attachment-processor.onrender.com/process
            json=json_data
        )

┌─────────────────────────────────────────────────────────────┐
│ 10. Utility API Receives & Processes                        │
└─────────────────────────────────────────────────────────────┘

@app.post("/process")
async def process_attachment_email(email_data: dict):
    # Extract ID from filename
    filename = email_data['attachments'][0]['name']  # "INVOICE_ID_12345.pdf"
    match = re.search(r'ID_(\d+)', filename)
    extracted_id = match.group(1)  # "12345"
    
    # Decode attachment
    attachment_bytes = base64.b64decode(email_data['attachments'][0]['content'])
    
    # Check ID exists
    job_data = await check_id_exists(extracted_id)
    
    # Upload attachment
    await upload_attachment(job_data['job_id'], filename, attachment_bytes)
    
    return {"status": "success"}

┌─────────────────────────────────────────────────────────────┐
│ 11. Complete - Logs Recorded                                │
└─────────────────────────────────────────────────────────────┘

Processing logs saved to: logs/processing/processing_20251223.jsonl
{
  "timestamp": "2025-12-23T15:15:49Z",
  "event": "processing_complete",
  "internet_message_id": "<unique-id@example.com>",
  "total_time_ms": 2340,
  "success_count": 1,
  "failure_count": 0
}
```

---

## Security & Secrets

### What Secrets Are Used?

**1. CLIENT_SECRET (Azure AD App Secret)**
- **Purpose**: Authenticates your application with Microsoft
- **Used in**: Token requests (every Graph API call)
- **Lifetime**: Never expires (unless regenerated)
- **Security**: Must be kept secret, stored in environment variables

**2. CLIENT_STATE (Webhook Validation Secret)**
- **Purpose**: Validates notifications are from Microsoft
- **Used in**: Subscription creation and notification validation
- **Lifetime**: Per subscription (attached to subscription)
- **Security**: Your secret string, can be anything

### How Client Secret is Used

```python
# Every time you need to call Graph API:

1. Get token using CLIENT_SECRET
   POST to Microsoft Identity Platform
   Body: {client_id, client_secret, ...}
   ↓
   Receive: {access_token: "eyJ0eXAi..."}

2. Use token for Graph API calls
   GET https://graph.microsoft.com/v1.0/users/{email}/messages
   Headers: {Authorization: "Bearer eyJ0eXAi..."}
   ↓
   Receive: Email data

Token is cached for 1 hour to avoid repeated authentication
```

### How Client State is Used

```python
# Creating subscription:
subscription_data = {
    'clientState': 'babajishivram@1706'  # Your secret
}

# Microsoft stores this with subscription

# When notification arrives:
notification = {
    'clientState': 'babajishivram@1706'  # Microsoft sends it back
}

# You validate:
if notification['clientState'] != 'babajishivram@1706':
    raise Unauthorized  # Not from Microsoft or wrong subscription
```

---

## Summary Flow Diagram

```
Email Arrives
    ↓
Microsoft 365 Mailbox
    ↓
Graph Event System (monitors subscriptions)
    ↓
Sends Notification (includes message ID only)
    ↓
Your Webhook (/webhook endpoint)
    ↓
Validate clientState ← Security check
    ↓
Respond 200 OK (< 5 seconds) ← Critical timing
    ↓
Background: Fetch full email using Graph API
    ↓
Authenticate with CLIENT_SECRET → Get access_token
    ↓
Download email + attachments
    ↓
Match against utility filters
    ↓
Dispatch to matched utility APIs (with retry)
    ↓
Utility processes email
    ↓
Complete - Logged
```

---

## Key Takeaways

1. **Client credentials** = App identity (like username/password for your app)
2. **Access token** = Temporary pass (1 hour) to call Graph API
3. **Subscription** = "Watch this mailbox, notify this URL"
4. **Client state** = Your secret to validate notifications
5. **Notification** = Lightweight alert (only message ID)
6. **Full email** = Fetched separately using Graph API
7. **5 second limit** = Why we use fire-and-forget pattern

**The entire flow happens in real-time (< 10 seconds from email arrival to utility processing)!**
