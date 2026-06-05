---
name: ros-zoho-deluge
description: >
  Use for all Zoho Deluge scripting, Zoho Flow automation, Zoho Functions, and
  cross-app Zoho One integration tasks. Triggers: "write a Deluge script", "Deluge
  function", "Zoho Flow", "automate between Zoho apps", "CRM to Books automation",
  "Books to Projects", "trigger an email from CRM", "Zoho Functions", "custom
  function", "scheduled function", "webhook", "API call from Deluge", "connect Zoho
  apps", "Zoho One automation", "cross-app workflow", "Deluge error", "null safety",
  "invoke URL", "Books invoice trigger", "CRM deal won trigger", or any request to
  write, fix, or explain Deluge code or Zoho automation logic. Works across: Zoho
  CRM, Zoho Books, Zoho Projects, Zoho Sign, Zoho Campaigns, Zoho WorkDrive, Zoho
  Mail, Zoho Desk, and Zoho Flow. Use alongside ros-zoho-crm-builder for CRM-specific
  context and ros-zoho-commerce for Commerce automation.
---

# ROS Zoho Deluge & Automation Skill

Complete reference for writing, debugging, and deploying Deluge scripts and
cross-app Zoho One automations — for Rae of Sunshine Consulting and all clients
on the Zoho ecosystem.

---

## Reference Files

```
ros-zoho-deluge/
├── SKILL.md                        ← You are here — read first
├── references/
│   ├── deluge-patterns.md          ← Production Deluge patterns, null safety, gotchas
│   ├── cross-app-integrations.md   ← CRM↔Books, CRM↔Projects, CRM↔Sign, Flow recipes
│   └── api-and-functions.md        ← invokeURL, custom functions, webhooks, scheduled tasks
└── scripts/
    ├── crm-to-books.deluge         ← Create invoice in Books when CRM deal is Won
    ├── deal-won-trigger.deluge     ← Full deal-won automation sequence
    ├── new-lead-notify.deluge      ← Slack/email notification on new lead
    └── project-from-deal.deluge    ← Auto-create Zoho Project when deal reaches Active
```

Load references as needed. For a Deluge script task → `deluge-patterns.md`.
For cross-app logic → `cross-app-integrations.md`. For APIs/webhooks → `api-and-functions.md`.

---

## Zoho Automation Decision Tree

Before writing any code, determine the right tool:

| Situation | Use This |
|---|---|
| Simple field update, email, or task when record changes | **Workflow Rule** (no code needed) |
| Multi-step process with conditions and branching | **Blueprint** in CRM |
| Custom logic that Workflow Rules can't handle | **Deluge Custom Function** |
| Connecting two Zoho apps (e.g. CRM deal → Books invoice) | **Deluge Function** or **Zoho Flow** |
| Connecting Zoho to a non-Zoho app | **Zoho Flow** (if connector exists) or **invokeURL** |
| Recurring/scheduled task (nightly sync, weekly report) | **Scheduled Function** |
| External system sending data into Zoho | **Webhook** + **Custom Function** |
| Real-time response to an external event | **Zoho Flow webhook trigger** |

**Rule:** Use the simplest tool that solves the problem. Workflow Rules before Blueprints, Blueprints before Deluge, Deluge before Flow, Flow before custom webhooks.

---

## Deluge Fundamentals

### Language Basics

Deluge is Zoho's proprietary scripting language. It looks like Python but has important differences:

```deluge
// Variables — no type declaration needed
leadName = "Hillary Dunkley";
invoiceAmount = 500.00;
isActive = true;

// String concatenation — use + operator
message = "Hello " + leadName + ", your invoice is ready.";

// Maps (like dictionaries/objects)
contactMap = Map();
contactMap.put("Last Name", "Johnson");
contactMap.put("Email", "info@raeofsunshineconsulting.co");

// Lists
serviceList = List();
serviceList.add("CRM Setup");
serviceList.add("Website Build");

// Conditional
if(invoiceAmount > 1000)
{
    tier = "Premium";
}
else
{
    tier = "Standard";
}

// For loop
for each service in serviceList
{
    info "Service: " + service;
}
```

### Critical Differences from Other Languages

| Concept | Deluge Behaviour |
|---|---|
| Null check | Use `isNull(variable)` — never assume a field has a value |
| Boolean | `true` / `false` (lowercase) |
| String quotes | Double quotes `"` only — single quotes cause errors |
| Comments | `//` for single line — no block comments |
| Semicolons | Required at end of each statement |
| Map access | `mapVariable.get("Key")` — not dot notation |
| Date format | `"yyyy-MM-dd"` — always use this format for date fields |
| Info/debug | `info "message";` — outputs to execution log |

---

## Null Safety — The #1 Source of Deluge Errors

**Always check for null before using a value.** Null errors crash functions silently.

```deluge
// WRONG — will crash if Lead_Source is empty
source = lead.get("Lead_Source");
message = "Source is: " + source; // ERROR if null

// CORRECT — null-safe pattern
source = lead.get("Lead_Source");
if(isNull(source) || source == "")
{
    source = "Unknown";
}
message = "Source is: " + source;
```

### Null-Safe Map Get Helper Pattern

Use this pattern every time you pull a value from a record:

```deluge
// Safe getter — returns default if null or empty
leadName = input.get("Last_Name");
if(isNull(leadName) || leadName == "")
{
    leadName = "Valued Client";
}
```

### Null-Safe Record Fetch

```deluge
// Fetch a single record — always check if it returned anything
dealRecord = zoho.crm.getRecordById("Deals", dealId);
if(isNull(dealRecord) || dealRecord.isEmpty())
{
    info "Deal not found: " + dealId;
    return;
}
dealName = dealRecord.get("Deal_Name");
```

---

## Common Deluge Patterns

### Pattern 1 — Create a Record in Another Module

```deluge
// Create a Zoho Books invoice when a CRM Deal is Won
void createInvoiceFromDeal(String dealId)
{
    // Fetch the deal
    deal = zoho.crm.getRecordById("Deals", dealId);
    if(isNull(deal)) { return; }

    // Get related contact
    contactId = deal.get("Contact_Name").get("id");
    contact = zoho.crm.getRecordById("Contacts", contactId);

    // Build the invoice map
    invoiceMap = Map();
    invoiceMap.put("customer_name", contact.get("Full_Name"));
    invoiceMap.put("customer_email", contact.get("Email"));
    invoiceMap.put("date", zoho.currentdate.toString("yyyy-MM-dd"));
    invoiceMap.put("payment_terms", 15); // Net 15

    // Line items
    lineItem = Map();
    lineItem.put("name", deal.get("Deal_Name"));
    lineItem.put("rate", deal.get("Amount"));
    lineItem.put("quantity", 1);

    lineItems = List();
    lineItems.add(lineItem);
    invoiceMap.put("line_items", lineItems);

    // Create in Zoho Books (org ID required)
    orgId = "874099604"; // ROS Zoho Books Org ID
    response = zoho.books.createRecord("invoices", orgId, invoiceMap);
    info "Invoice created: " + response;
}
```

### Pattern 2 — Send an Email via Zoho Mail

```deluge
// Send a notification email when a lead is assigned
void notifyLeadAssigned(String leadId)
{
    lead = zoho.crm.getRecordById("Leads", leadId);
    if(isNull(lead)) { return; }

    ownerEmail = lead.get("Owner").get("email");
    leadName = lead.get("Last_Name");

    if(isNull(ownerEmail)) { return; }

    subject = "New Lead Assigned: " + leadName;
    body = "Hi,\n\nA new lead has been assigned to you:\n\n"
         + "Name: " + leadName + "\n"
         + "Email: " + lead.get("Email") + "\n"
         + "Phone: " + lead.get("Phone") + "\n\n"
         + "Log in to review: https://crm.zoho.com\n\n"
         + "Rae of Sunshine Consulting";

    sendmail
    [
        from: zoho.adminuserid
        to: ownerEmail
        subject: subject
        message: body
    ]
}
```

### Pattern 3 — Create a Zoho Project from a Deal

```deluge
// Auto-create a Zoho Project when a deal reaches "Project Active" stage
void createProjectFromDeal(String dealId)
{
    deal = zoho.crm.getRecordById("Deals", dealId);
    if(isNull(deal)) { return; }

    contactName = deal.get("Contact_Name").get("name");
    dealName = deal.get("Deal_Name");
    amount = deal.get("Amount");

    projectMap = Map();
    projectMap.put("name", dealName + " — " + contactName);
    projectMap.put("description", "Auto-created from CRM Deal. Value: USD " + amount);
    projectMap.put("start_date", zoho.currentdate.toString("yyyy-MM-dd"));

    // Portal ID for ROS
    portalId = "raeofsunshineconsulting";
    response = zoho.projects.createProject(portalId, projectMap);

    projectId = response.get("project").get("id").toString();

    // Save project ID back to the CRM deal
    updateMap = Map();
    updateMap.put("Project_ID", projectId);
    zoho.crm.updateRecord("Deals", dealId, updateMap);

    info "Project created: " + projectId;
}
```

### Pattern 4 — Send a Zoho Sign Document

```deluge
// Send an agreement for signature when deal moves to "Proposal Accepted"
void sendSignDocument(String dealId)
{
    deal = zoho.crm.getRecordById("Deals", dealId);
    if(isNull(deal)) { return; }

    contactId = deal.get("Contact_Name").get("id");
    contact = zoho.crm.getRecordById("Contacts", contactId);

    recipientEmail = contact.get("Email");
    recipientName = contact.get("Full_Name");

    if(isNull(recipientEmail)) { return; }

    // Build Sign request — template ID must be created in Zoho Sign first
    signMap = Map();
    signMap.put("template_id", "YOUR_SIGN_TEMPLATE_ID");

    recipient = Map();
    recipient.put("recipient_name", recipientName);
    recipient.put("recipient_email", recipientEmail);
    recipient.put("action_type", "SIGN");
    recipient.put("role", "Client");

    recipients = List();
    recipients.add(recipient);
    signMap.put("recipients", recipients);

    response = zoho.sign.sendForSignature(signMap);
    info "Sign document sent: " + response;
}
```

### Pattern 5 — Update CRM Record from External Webhook

```deluge
// Webhook receiver — called when payment confirmed by Paystack/Stripe
void handlePaymentWebhook(Map payload)
{
    // Extract data from webhook payload
    email = payload.get("data").get("customer").get("email");
    amount = payload.get("data").get("amount");
    reference = payload.get("data").get("reference");

    if(isNull(email)) { return; }

    // Find matching contact in CRM
    searchCriteria = "(Email:equals:" + email + ")";
    contacts = zoho.crm.searchRecords("Contacts", searchCriteria);

    if(contacts.isEmpty()) { return; }

    contactId = contacts.get(0).get("id");

    // Update contact with payment info
    updateMap = Map();
    updateMap.put("Payment_Reference", reference);
    updateMap.put("Last_Payment_Amount", amount / 100); // Paystack sends in kobo/cents
    updateMap.put("Payment_Status", "Paid");

    zoho.crm.updateRecord("Contacts", contactId, updateMap);
    info "Contact updated with payment: " + reference;
}
```

---

## Zoho Flow — When to Use It Instead of Deluge

Use **Zoho Flow** when:
- You need to connect Zoho to a non-Zoho app (Slack, Google, WhatsApp, etc.)
- The logic is simple and visual (no complex conditions)
- You want non-developers to maintain the automation
- You need a pre-built connector (Flow has 800+)

Use **Deluge** when:
- The logic is complex (loops, conditionals, data transformation)
- You need to manipulate data before sending it somewhere
- You need to read from multiple Zoho modules before acting
- Performance matters (Deluge runs faster than Flow for heavy operations)

### Zoho Flow Recipe — CRM Lead to Campaigns Subscriber

```
Trigger: Zoho CRM — New Record in Leads
↓
Filter: Lead Status = "Qualified"
↓
Action: Zoho Campaigns — Add Subscriber to List
  → List: "Qualified Prospects"
  → Email: {{Lead.Email}}
  → First Name: {{Lead.First_Name}}
  → Last Name: {{Lead.Last_Name}}
```

### Zoho Flow Recipe — Books Invoice Paid to CRM Update

```
Trigger: Zoho Books — Invoice Status Changed
↓
Filter: Status = "Paid"
↓
Action: Zoho CRM — Update Record
  → Module: Deals
  → Search by: Deal_Name = {{Invoice.Reference_Number}}
  → Update: Payment_Status = "Received"
  → Update: Invoice_Paid_Date = {{Invoice.Last_Payment_Date}}
```

---

## Cross-App Integration Map (Zoho One)

The most common integrations ROS builds for clients:

| From | To | Trigger | What Gets Created/Updated |
|---|---|---|---|
| CRM Deal Won | Zoho Books | Stage = "Closed Won" | Invoice created automatically |
| CRM Deal Active | Zoho Projects | Stage = "Project Active" | Project created, deal updated with project ID |
| CRM Deal Active | Zoho Sign | Stage = "Proposal Accepted" | Agreement sent for signature |
| Zoho Books Invoice Paid | Zoho CRM | Invoice status = Paid | Deal/Contact updated with payment date |
| Zoho Form Submission | Zoho CRM | New form entry | Lead created with source = form name |
| Zoho CRM New Lead | Zoho Campaigns | New qualified lead | Added to nurture sequence |
| Zoho Campaigns Unsubscribe | Zoho CRM | Email opt-out | Contact email opt-out field = true |
| Zoho Books New Client | Zoho CRM | New contact in Books | Contact created/updated in CRM |

---

## Debugging Deluge

### Read the Execution Log
Every Custom Function has an execution log. Always check it before declaring a function broken:
- CRM → Setup → Automation → Functions → [Function Name] → Logs

### Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `null pointer` | Accessing a field on a null record | Add `isNull()` check before access |
| `invalid data type` | Passing string where number expected | Use `input.toLong()` or `input.toDecimal()` |
| `record not found` | Wrong module name or ID | Check module API name in CRM Setup → Modules |
| `permission denied` | Function running as wrong user | Check profile permissions for the automation |
| `API limit exceeded` | Too many API calls in a loop | Add `sleep(1000)` between iterations |
| `invalid date format` | Date string format mismatch | Always use `"yyyy-MM-dd"` format |
| `map key not found` | `.get()` on a key that doesn't exist | Use `map.containsKey("key")` before `.get()` |

### Debug Pattern — Log Everything

```deluge
info "=== Function Started ===";
info "Input dealId: " + dealId;

deal = zoho.crm.getRecordById("Deals", dealId);
info "Deal fetched: " + deal;

if(isNull(deal))
{
    info "ERROR: Deal is null — stopping";
    return;
}

info "Deal name: " + deal.get("Deal_Name");
info "=== Function Complete ===";
```

---

## ROS Org IDs & Connection Reference

Always verify which org you're connected to before running functions:

```
Zoho Books Org ID:        874099604
Zoho Projects Portal:     raeofsunshineconsulting
Zoho CRM:                 raeofsunshineconsulting.zoho.com
GHS Currency ID:          5811667000000204007
```

**For client orgs:** Never hardcode ROS IDs into client functions. Pass org ID as a parameter or read from a custom field.

---

## Integration with Other ROS Skills

| Task | Also Load |
|---|---|
| Building the CRM pipeline the Deluge is for | `ros-zoho-crm-builder` |
| Commerce/store automation | `ros-zoho-commerce` |
| TrainerCentral enrollment trigger | `ros-trainercentral` |
| Zoho Sites form → CRM webhook | `ros-zoho-sites-builder` |
| Books invoice workflow | Load `references/cross-app-integrations.md` |
| Troubleshooting a specific error | Load `references/deluge-patterns.md` |
