# Cross-App Integrations — Zoho One Reference

Complete integration recipes for connecting Zoho apps. Each recipe includes
the trigger, the Deluge function or Flow config, and the exact fields used.

---

## Integration 1 — CRM Deal Won → Zoho Books Invoice

**Trigger:** CRM Workflow Rule on Deals — Stage changes to "Closed Won"
**Function location:** CRM → Setup → Automation → Workflow Rules → [Rule] → Custom Function

```deluge
void dealWonCreateInvoice(String dealId)
{
    orgId = "874099604"; // ROS Zoho Books org — replace with client org ID

    // Fetch deal
    deal = zoho.crm.getRecordById("Deals", dealId);
    if(isNull(deal)) { info "Deal not found"; return; }

    // Get contact
    contactMap = deal.get("Contact_Name");
    if(isNull(contactMap)) { info "No contact on deal"; return; }

    contactId = contactMap.get("id");
    contact = zoho.crm.getRecordById("Contacts", contactId);

    customerName = "";
    customerEmail = "";

    if(!isNull(contact))
    {
        customerName = contact.get("Full_Name");
        customerEmail = contact.get("Email");
    }

    dealName = deal.get("Deal_Name");
    amount = deal.get("Amount");
    if(isNull(amount)) { amount = 0; }

    // Build invoice
    invoiceMap = Map();
    invoiceMap.put("customer_name", customerName);
    invoiceMap.put("customer_email", customerEmail);
    invoiceMap.put("date", zoho.currentdate.toString("yyyy-MM-dd"));
    invoiceMap.put("due_date", zoho.currentdate.addDay(14).toString("yyyy-MM-dd"));
    invoiceMap.put("reference_number", dealId.toString());
    invoiceMap.put("notes", "Invoice auto-generated from CRM Deal: " + dealName);

    lineItem = Map();
    lineItem.put("name", dealName);
    lineItem.put("rate", amount);
    lineItem.put("quantity", 1);

    lineItems = List();
    lineItems.add(lineItem);
    invoiceMap.put("line_items", lineItems);

    // Create invoice
    response = zoho.books.createRecord("invoices", orgId, invoiceMap);
    invoiceId = response.get("invoice").get("invoice_id");

    // Save invoice ID back to CRM deal
    updateMap = Map();
    updateMap.put("Invoice_ID", invoiceId.toString());
    updateMap.put("Invoice_Status", "Draft");
    zoho.crm.updateRecord("Deals", dealId, updateMap);

    info "Invoice created: " + invoiceId;
}
```

**Required custom fields on Deals module:**
- `Invoice_ID` — Single Line Text
- `Invoice_Status` — Single Line Text

---

## Integration 2 — CRM Deal Stage → Zoho Projects

**Trigger:** CRM Workflow Rule on Deals — Stage changes to "Project Active"
**What it does:** Creates a Zoho Project, then saves the project ID back to the deal

```deluge
void dealActiveCreateProject(String dealId)
{
    portalId = "raeofsunshineconsulting"; // Replace with client portal ID

    deal = zoho.crm.getRecordById("Deals", dealId);
    if(isNull(deal)) { return; }

    dealName = deal.get("Deal_Name");
    amount = deal.get("Amount");
    if(isNull(amount)) { amount = 0; }

    contactMap = deal.get("Contact_Name");
    contactName = "";
    if(!isNull(contactMap)) { contactName = contactMap.get("name"); }

    closingDate = deal.get("Closing_Date");
    endDate = zoho.currentdate.addDay(30).toString("yyyy-MM-dd");
    if(!isNull(closingDate))
    {
        endDate = closingDate.toString("yyyy-MM-dd");
    }

    // Build project
    projectMap = Map();
    projectMap.put("name", dealName + " — " + contactName);
    projectMap.put("description", "Project value: USD " + amount
                   + "\nCRM Deal ID: " + dealId);
    projectMap.put("start_date", zoho.currentdate.toString("yyyy-MM-dd"));
    projectMap.put("end_date", endDate);

    response = zoho.projects.createProject(portalId, projectMap);
    projectId = response.get("project").get("id").toString();

    // Default task list creation
    taskListMap = Map();
    taskListMap.put("name", "Deliverables");
    zoho.projects.createTaskList(portalId, projectId, taskListMap);

    // Save project ID to CRM
    updateMap = Map();
    updateMap.put("Zoho_Project_ID", projectId);
    updateMap.put("Project_Portal", portalId);
    zoho.crm.updateRecord("Deals", dealId, updateMap);

    info "Project created: " + projectId;
}
```

**Required custom fields on Deals module:**
- `Zoho_Project_ID` — Single Line Text
- `Project_Portal` — Single Line Text

---

## Integration 3 — CRM Deal → Zoho Sign Agreement

**Trigger:** CRM Workflow Rule on Deals — Stage changes to "Proposal Accepted"
**Prerequisite:** Agreement template must be created in Zoho Sign first

```deluge
void proposalAcceptedSendAgreement(String dealId)
{
    deal = zoho.crm.getRecordById("Deals", dealId);
    if(isNull(deal)) { return; }

    contactMap = deal.get("Contact_Name");
    if(isNull(contactMap)) { return; }

    contactId = contactMap.get("id");
    contact = zoho.crm.getRecordById("Contacts", contactId);
    if(isNull(contact)) { return; }

    recipientEmail = contact.get("Email");
    recipientName = contact.get("Full_Name");
    dealName = deal.get("Deal_Name");
    amount = deal.get("Amount");

    if(isNull(recipientEmail)) { return; }

    // Sign template ID — get this from Zoho Sign → Templates
    templateId = "YOUR_SIGN_TEMPLATE_ID_HERE";

    signMap = Map();
    signMap.put("template_id", templateId);

    // Prefill fields in the template (must match field names in Sign template)
    fieldValues = Map();
    fieldValues.put("Client_Name", recipientName);
    fieldValues.put("Project_Name", dealName);
    fieldValues.put("Project_Value", amount.toString());
    fieldValues.put("Date", zoho.currentdate.toString("dd MMMM yyyy"));
    signMap.put("field_data", Map().put("field_text_data", fieldValues));

    recipient = Map();
    recipient.put("recipient_name", recipientName);
    recipient.put("recipient_email", recipientEmail);
    recipient.put("action_type", "SIGN");
    recipient.put("role", "Client");

    recipients = List();
    recipients.add(recipient);
    signMap.put("recipients", recipients);
    signMap.put("notes", "Please review and sign your service agreement for: " + dealName);

    response = zoho.sign.sendForSignature(signMap);
    requestId = response.get("requests").get("request_id");

    // Save Sign request ID to CRM
    updateMap = Map();
    updateMap.put("Sign_Request_ID", requestId.toString());
    updateMap.put("Agreement_Status", "Sent");
    zoho.crm.updateRecord("Deals", dealId, updateMap);

    info "Agreement sent: " + requestId;
}
```

**Required custom fields on Deals module:**
- `Sign_Request_ID` — Single Line Text
- `Agreement_Status` — Picklist: Sent / Signed / Declined

---

## Integration 4 — Zoho Books Invoice Paid → CRM Update

**Trigger:** Zoho Flow — Zoho Books Invoice status changes to "paid"
**Use Flow for this one** — Books doesn't have native CRM workflow triggers

**Zoho Flow Configuration:**
```
App: Zoho Books
Trigger: Invoice — Payment Received
↓
Filter: Invoice.status = "paid"
↓
App: Zoho CRM
Action: Search Records
  Module: Deals
  Search: Reference_Number = {{invoice.reference_number}}
↓
App: Zoho CRM
Action: Update Record
  Record ID: {{previous step result.id}}
  Fields:
    - Payment_Status = "Received"
    - Payment_Date = {{invoice.last_payment_date}}
    - Invoice_Status = "Paid"
```

---

## Integration 5 — Zoho Form → CRM Lead with Source Tracking

**Trigger:** Zoho CRM — Zoho Form submission (Forms are natively connected)
**What it does:** Adds lead source, enriches the lead, assigns to owner

```deluge
// Called via Workflow Rule on Leads — trigger: Created via Form
void enrichNewFormLead(String leadId)
{
    lead = zoho.crm.getRecordById("Leads", leadId);
    if(isNull(lead)) { return; }

    // Determine which form it came from (map field in form must send form name)
    formSource = lead.get("CF_Form_Source"); // Custom field populated by form
    if(isNull(formSource)) { formSource = "Website Form"; }

    // Set default follow-up task
    taskMap = Map();
    taskMap.put("Subject", "Follow up: " + lead.get("Last_Name"));
    taskMap.put("Due_Date", zoho.currentdate.addDay(1).toString("yyyy-MM-dd"));
    taskMap.put("Status", "Not Started");
    taskMap.put("Priority", "High");
    taskMap.put("What_Id", leadId);
    taskMap.put("se_module", "Leads");

    zoho.crm.createRecord("Tasks", taskMap);

    // Update lead status and source
    updateMap = Map();
    updateMap.put("Lead_Status", "New");
    updateMap.put("Lead_Source", formSource);
    updateMap.put("Rating", "Hot");
    zoho.crm.updateRecord("Leads", leadId, updateMap);

    info "Lead enriched: " + leadId;
}
```

---

## Integration 6 — CRM New Contact → Zoho Campaigns Subscriber

**Use Zoho Flow for this** — native connector, no code needed

**Zoho Flow Configuration:**
```
App: Zoho CRM
Trigger: New Record — Contacts
↓
Filter: Contact.Email is not empty
  AND Contact.Email_Opt_Out = false
↓
App: Zoho Campaigns
Action: Add Subscriber
  Mailing List: "All Contacts" (or segment by type)
  Email: {{contact.email}}
  First Name: {{contact.first_name}}
  Last Name: {{contact.last_name}}
  Custom Field — Client_Type: {{contact.cf_client_type}}
```

**Reverse — Unsubscribe sync (Campaigns → CRM):**
```
App: Zoho Campaigns
Trigger: Subscriber Unsubscribed
↓
App: Zoho CRM
Action: Search Records — Contacts — Email = {{subscriber.email}}
↓
App: Zoho CRM
Action: Update Record — Email_Opt_Out = true
```

---

## Integration 7 — Zoho WorkDrive Approval → CRM Update

**Use case:** Client approves a design in WorkDrive → CRM deal stage advances

```deluge
// Called via Zoho Flow webhook when WorkDrive review is approved
void workdriveApprovalReceived(Map payload)
{
    // Extract file and reviewer info from WorkDrive webhook
    fileId = payload.get("resource_id");
    reviewStatus = payload.get("status"); // "APPROVED" or "REJECTED"
    dealId = payload.get("custom_data").get("deal_id"); // Must be sent as metadata

    if(isNull(dealId)) { return; }

    if(reviewStatus == "APPROVED")
    {
        updateMap = Map();
        updateMap.put("Design_Approval_Status", "Approved");
        updateMap.put("Design_Approved_Date", zoho.currentdate.toString("yyyy-MM-dd"));
        // Optionally advance the stage
        updateMap.put("Stage", "Development");
        zoho.crm.updateRecord("Deals", dealId, updateMap);
    }
    else if(reviewStatus == "REJECTED")
    {
        updateMap = Map();
        updateMap.put("Design_Approval_Status", "Revision Requested");
        zoho.crm.updateRecord("Deals", dealId, updateMap);
    }

    info "WorkDrive approval processed: " + dealId + " — " + reviewStatus;
}
```
