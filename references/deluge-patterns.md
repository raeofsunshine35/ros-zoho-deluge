# Deluge Patterns — Production Reference

Production-ready Deluge patterns for Zoho One. Built from real ROS client deployments.
Load this file when writing or debugging any Deluge function.

---

## Core Data Types and Conversion

```deluge
// String to Number
numValue = stringVar.toLong();       // Integer
decValue = stringVar.toDecimal();    // Float/decimal

// Number to String
strValue = numVar.toString();

// Date handling
today = zoho.currentdate;
todayStr = today.toString("yyyy-MM-dd");
futureDate = today.addDay(30).toString("yyyy-MM-dd");

// Boolean from string
isTrue = (stringVar == "true");
```

---

## Map Patterns

```deluge
// Safe map creation and population
dataMap = Map();
dataMap.put("Field_API_Name", value);

// Safe map get with fallback
getValue(Map m, String key, String fallback)
{
    val = m.get(key);
    if(isNull(val) || val.toString() == "")
    {
        return fallback;
    }
    return val.toString();
}

// Check before get
if(myMap.containsKey("some_key"))
{
    value = myMap.get("some_key");
}

// Nested map access — safe pattern
outerMap = record.get("Owner");
if(!isNull(outerMap))
{
    ownerId = outerMap.get("id");
    ownerName = outerMap.get("name");
    ownerEmail = outerMap.get("email");
}
```

---

## List Patterns

```deluge
// Create and populate
itemList = List();
itemList.add("First Item");
itemList.add("Second Item");

// Check if empty before iterating
if(!itemList.isEmpty())
{
    for each item in itemList
    {
        info item;
    }
}

// Get by index (0-based)
firstItem = itemList.get(0);

// List size
count = itemList.size();
```

---

## Zoho CRM — Read Patterns

```deluge
// Get single record by ID
record = zoho.crm.getRecordById("Leads", recordId);

// Search records
// Criteria format: "(Field_API:operator:value)"
results = zoho.crm.searchRecords("Contacts", "(Email:equals:test@test.com)");
results = zoho.crm.searchRecords("Deals", "(Stage:equals:Closed Won)");
results = zoho.crm.searchRecords("Leads", "(Lead_Source:equals:Website)");

// Get all records (max 200 per call — use pagination for more)
allLeads = zoho.crm.getRecords("Leads");

// Get related records
relatedContacts = zoho.crm.getRelatedRecords("Contacts", "Deals", dealId);
```

---

## Zoho CRM — Write Patterns

```deluge
// Create a new record
newRecord = Map();
newRecord.put("Last_Name", "Johnson");
newRecord.put("Email", "davia@raeofsunshineconsulting.co");
newRecord.put("Lead_Source", "Website");
newRecord.put("Phone", "+233597121610");

response = zoho.crm.createRecord("Leads", newRecord);
newId = response.get("id");

// Update an existing record
updateMap = Map();
updateMap.put("Stage", "Proposal Sent");
updateMap.put("Proposal_Date", zoho.currentdate.toString("yyyy-MM-dd"));

zoho.crm.updateRecord("Deals", dealId, updateMap);

// Add a note to a record
noteMap = Map();
noteMap.put("Note_Title", "Automated Note");
noteMap.put("Note_Content", "Invoice created automatically on deal won.");
noteMap.put("Parent_Id", dealId);
noteMap.put("se_module", "Deals");

zoho.crm.createRecord("Notes", noteMap);
```

---

## Zoho Books — Patterns

```deluge
// Always requires orgId — ROS: "874099604"
orgId = "874099604";

// Create invoice
invoiceMap = Map();
invoiceMap.put("customer_name", "Hillary Dunkley");
invoiceMap.put("customer_email", "hello@hillaryencourages.ca");
invoiceMap.put("date", zoho.currentdate.toString("yyyy-MM-dd"));
invoiceMap.put("due_date", zoho.currentdate.addDay(14).toString("yyyy-MM-dd"));
invoiceMap.put("payment_terms", 14);
invoiceMap.put("reference_number", "PROJ-001");

lineItem = Map();
lineItem.put("name", "Website Build — Phase 1");
lineItem.put("description", "Homepage, About, Services pages");
lineItem.put("rate", 1500.00);
lineItem.put("quantity", 1);
lineItem.put("tax_id", ""); // Add tax ID if applicable

lineItems = List();
lineItems.add(lineItem);
invoiceMap.put("line_items", lineItems);

response = zoho.books.createRecord("invoices", orgId, invoiceMap);
invoiceId = response.get("invoice").get("invoice_id");

// Get invoice status
invoice = zoho.books.getRecordById("invoices", invoiceId, orgId);
status = invoice.get("invoice").get("status");

// List contacts in Books
contacts = zoho.books.getRecords("contacts", orgId, Map());
```

---

## Zoho Projects — Patterns

```deluge
// Portal ID for ROS: "raeofsunshineconsulting"
portalId = "raeofsunshineconsulting";

// Create project
projectMap = Map();
projectMap.put("name", "Hillary Encourages — Website Build");
projectMap.put("description", "Full Squarespace website build and optimization.");
projectMap.put("start_date", zoho.currentdate.toString("yyyy-MM-dd"));
projectMap.put("end_date", zoho.currentdate.addDay(30).toString("yyyy-MM-dd"));

response = zoho.projects.createProject(portalId, projectMap);
projectId = response.get("project").get("id").toString();

// Create a task in a project
taskMap = Map();
taskMap.put("name", "Homepage Build");
taskMap.put("description", "Build homepage with hero, services, CTA sections");
taskMap.put("start_date", zoho.currentdate.toString("yyyy-MM-dd"));
taskMap.put("end_date", zoho.currentdate.addDay(7).toString("yyyy-MM-dd"));

zoho.projects.createTask(portalId, projectId, taskMap);
```

---

## Scheduled Functions

Scheduled functions run on a timer. Set them in CRM → Setup → Automation → Schedules.

```deluge
// Example: Run every Monday at 9am — flag leads with no activity in 14 days
void weeklyLeadCleanup()
{
    // Get all open leads
    allLeads = zoho.crm.getRecords("Leads");

    for each lead in allLeads
    {
        modifiedTime = lead.get("Modified_Time");
        leadId = lead.get("id");

        if(isNull(modifiedTime)) { continue; }

        // Check if last modified was more than 14 days ago
        daysSinceModified = zoho.currentdate.daysBetween(modifiedTime.toDate());

        if(daysSinceModified >= 14)
        {
            updateMap = Map();
            updateMap.put("Lead_Status", "Inactive");
            updateMap.put("Inactivity_Flag", true);
            zoho.crm.updateRecord("Leads", leadId, updateMap);
        }
    }
    info "Weekly lead cleanup complete.";
}
```

---

## Rate Limits and Performance

| Operation | Limit | Safe Practice |
|---|---|---|
| API calls per day | 10,000 (Free) / 25,000 (Standard) / 100,000 (One) | Batch updates, avoid loops |
| Records per getRecords() call | 200 | Paginate with `from_index` for larger sets |
| Function execution time | 30 seconds | Break large jobs into scheduled batches |
| Email sends per day | 50 via `sendmail` | Use Zoho Campaigns for bulk email |

```deluge
// Pagination for large data sets
pageIndex = 1;
batchSize = 200;
hasMore = true;

while(hasMore)
{
    params = Map();
    params.put("from_index", ((pageIndex - 1) * batchSize) + 1);
    params.put("to_index", pageIndex * batchSize);

    batch = zoho.crm.getRecords("Leads", "id", "asc", params);

    if(batch.size() < batchSize)
    {
        hasMore = false;
    }

    for each lead in batch
    {
        // process...
    }

    pageIndex = pageIndex + 1;
}
```

---

## Field API Names Reference

Field API names in Deluge are **not** the same as the display labels. Always check:
- CRM → Setup → Modules and Fields → [Module] → [Field] → API Name

Common gotchas:

| Display Label | API Name |
|---|---|
| First Name | First_Name |
| Last Name | Last_Name |
| Phone | Phone |
| Mobile | Mobile |
| Lead Source | Lead_Source |
| Deal Name | Deal_Name |
| Closing Date | Closing_Date |
| Account Name | Account_Name |
| Contact Name | Contact_Name (returns a Map, not a string) |
| Owner | Owner (returns a Map with id, name, email) |
| Created Time | Created_Time |
| Modified Time | Modified_Time |

**Lookup fields return Maps.** Always `.get("id")` or `.get("name")` from them:
```deluge
contactMap = deal.get("Contact_Name");  // Returns: {id: "xxx", name: "Hillary"}
contactId = contactMap.get("id");
contactName = contactMap.get("name");
```
