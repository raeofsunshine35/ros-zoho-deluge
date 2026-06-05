# API & Functions Reference — Zoho One

External API calls, webhook handling, custom functions setup, and scheduled tasks.

---

## invokeURL — Calling External APIs from Deluge

`invokeURL` is how Deluge communicates with anything outside Zoho.

### Basic GET Request

```deluge
url = "https://api.example.com/data";
headers = Map();
headers.put("Authorization", "Bearer YOUR_API_KEY");
headers.put("Content-Type", "application/json");

response = invokeurl
[
    url: url
    type: GET
    headers: headers
];
info "Response: " + response;
```

### POST Request (JSON body)

```deluge
url = "https://api.example.com/webhook";
headers = Map();
headers.put("Content-Type", "application/json");
headers.put("X-API-Key", "YOUR_API_KEY");

payload = Map();
payload.put("event", "deal_won");
payload.put("client_email", "client@example.com");
payload.put("amount", 1500);

response = invokeurl
[
    url: url
    type: POST
    headers: headers
    parameters: payload.toString()
];
info "POST response: " + response;
```

### Paystack Payment Verification

```deluge
void verifyPaystackPayment(String reference)
{
    paystackSecretKey = "sk_live_YOUR_KEY";
    url = "https://api.paystack.co/transaction/verify/" + reference;

    headers = Map();
    headers.put("Authorization", "Bearer " + paystackSecretKey);

    response = invokeurl [ url: url type: GET headers: headers ];

    status = response.get("data").get("status");
    amount = response.get("data").get("amount");
    email = response.get("data").get("customer").get("email");

    if(status == "success")
    {
        info "Payment verified: " + email + " paid " + (amount / 100);
    }
}
```

### Slack Notification via Webhook

```deluge
void sendSlackNotification(String message)
{
    slackWebhookUrl = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL";
    payload = Map();
    payload.put("text", message);
    headers = Map();
    headers.put("Content-Type", "application/json");

    invokeurl [ url: slackWebhookUrl type: POST headers: headers parameters: payload.toString() ];
}
// Usage: sendSlackNotification("New deal won: Hillary - USD 2500");
```

### WhatsApp via Meta Business API

```deluge
void sendWhatsAppMessage(String toPhone, String messageBody)
{
    phoneNumberId = "YOUR_PHONE_NUMBER_ID";
    accessToken = "YOUR_META_ACCESS_TOKEN";
    url = "https://graph.facebook.com/v17.0/" + phoneNumberId + "/messages";

    headers = Map();
    headers.put("Authorization", "Bearer " + accessToken);
    headers.put("Content-Type", "application/json");

    messageMap = Map();
    messageMap.put("messaging_product", "whatsapp");
    messageMap.put("to", toPhone);
    messageMap.put("type", "text");
    textMap = Map();
    textMap.put("body", messageBody);
    messageMap.put("text", textMap);

    invokeurl [ url: url type: POST headers: headers parameters: messageMap.toString() ];
}
```

---

## Setting Up Custom Functions in Zoho CRM

1. CRM > Setup > Automation > Functions > + New Function
2. Set name (no spaces), description, and arguments (String type for each input)
3. Write Deluge code, save, and test with a real record ID
4. Connect via: Setup > Automation > Workflow Rules > Actions > Custom Functions
5. Map arguments: `${Deals.ID}` maps to your `dealId` parameter

### Argument Mapping Reference

| CRM Field | Workflow Argument Syntax |
|---|---|
| Record ID | `${Deals.ID}` or `${Leads.ID}` |
| Field value | `${Deals.Stage}` |
| Owner ID | `${Deals.Owner.ID}` |

---

## Webhook Setup — Receiving External Data

### Option A: Zoho Flow Webhook (Recommended — no code)
1. Zoho Flow > Create Flow > Trigger = Webhook
2. Copy the generated webhook URL
3. Configure external system (Stripe, Paystack, Calendly) to POST to that URL
4. Map payload fields to Zoho actions in the flow builder

### Option B: CRM Custom Function Webhook
1. CRM > Setup > Developer Space > APIs > Webhooks
2. Create webhook > call your Custom Function
3. Map incoming payload fields to function arguments

---

## Scheduled Functions

Setup: CRM > Setup > Automation > Schedules > + New Schedule
Set frequency (Daily/Weekly/Monthly), time, and select the Custom Function.

### Daily — Overdue Follow-Up Alert (9am)

```deluge
void dailyOverdueFollowUpAlert()
{
    searchCriteria = "(Lead_Status:equals:New)";
    leads = zoho.crm.searchRecords("Leads", searchCriteria);
    overdueLeads = List();

    for each lead in leads
    {
        modifiedTime = lead.get("Modified_Time");
        if(isNull(modifiedTime)) { continue; }
        daysSince = zoho.currentdate.daysBetween(modifiedTime.toDate());
        if(daysSince >= 3)
        {
            overdueLeads.add(lead.get("Last_Name") + " — " + lead.get("Email"));
        }
    }

    if(overdueLeads.size() > 0)
    {
        message = "Leads needing follow-up today:\n";
        for each item in overdueLeads { message = message + "- " + item + "\n"; }
        sendmail
        [
            from: zoho.adminuserid
            to: "info@raeofsunshineconsulting.co"
            subject: "Daily Follow-Up Alert — " + overdueLeads.size() + " leads"
            message: message
        ]
    }
}
```

### Weekly — Inactive Deal Cleanup (Monday 8am)

```deluge
void weeklyInactiveDealCleanup()
{
    allDeals = zoho.crm.searchRecords("Deals", "(Stage:not_equal:Closed Won)");

    for each deal in allDeals
    {
        modifiedTime = deal.get("Modified_Time");
        dealId = deal.get("id");
        if(isNull(modifiedTime)) { continue; }

        daysSince = zoho.currentdate.daysBetween(modifiedTime.toDate());
        if(daysSince >= 30)
        {
            updateMap = Map();
            updateMap.put("CF_Inactivity_Flag", true);
            updateMap.put("CF_Days_Inactive", daysSince);
            zoho.crm.updateRecord("Deals", dealId, updateMap);
        }
    }
    info "Weekly deal cleanup complete.";
}
```

---

## Zoho Functions (Serverless) vs CRM Custom Functions

| Type | Where It Lives | When to Use |
|---|---|---|
| CRM Custom Function | CRM > Setup > Functions | Triggered by CRM workflow rules only |
| Zoho Functions (Serverless) | developer.zoho.com | Called by webhooks, external apps, or schedules from any Zoho app |
| Flow Actions | Zoho Flow | Connecting apps without code |

Use **Zoho Functions (Serverless)** when:
- You need the function callable from multiple Zoho apps (not just CRM)
- You need a persistent webhook endpoint that external systems can call
- You want to build a standalone API layer for a client's Zoho org

---

## API Security — Never Hardcode Keys

Always store API keys as Zoho CRM Configuration variables, not in function code.

Setup: CRM > Setup > Developer Space > APIs > Custom Variables

```deluge
// WRONG — key visible in code
paystackKey = "sk_live_xxxxxxxx";

// CORRECT — read from CRM custom variable
paystackKey = zoho.crm.getConfiguration().get("PAYSTACK_SECRET_KEY");
```

For client deployments: Create one configuration variable per API key.
Document all variable names in the client handover document.
