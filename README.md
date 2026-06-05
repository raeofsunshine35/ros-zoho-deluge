# ros-zoho-deluge

> A Claude Code Skill for Zoho One automation — Deluge scripting, cross-app integrations, Zoho Flow recipes, and API connections.

Built by [Rae of Sunshine Consulting](https://raeofsunshineconsulting.co) — the first production-ready Claude skill specifically for the Zoho One ecosystem.

---

## What This Skill Does

When installed, this skill teaches Claude to:

- **Write Deluge scripts** for Zoho CRM, Books, Projects, Sign, Campaigns, WorkDrive, and Desk
- **Build cross-app integrations** — CRM deal won → Books invoice, CRM active → Projects, CRM proposal accepted → Sign agreement
- **Configure Zoho Flow** automation recipes between any Zoho apps
- **Call external APIs** from Deluge — Paystack, Stripe, Slack, WhatsApp, and custom webhooks
- **Set up scheduled functions** — daily follow-up alerts, weekly cleanup tasks
- **Debug Deluge errors** — null safety, rate limits, field API names, and execution logs
- **Handle webhooks** — receive data from external systems into Zoho

---

## Install

### Option 1 — npx (Recommended)
```bash
npx skills add Raeofsunshine35/ros-zoho-deluge
```

### Option 2 — Manual
```bash
# Clone or download this repo
git clone https://github.com/Raeofsunshine35/ros-zoho-deluge.git

# Copy to your Claude skills directory
cp -r ros-zoho-deluge ~/.claude/skills/
```

Restart Claude Code. The skill loads automatically.

---

## Skill Structure

```
ros-zoho-deluge/
├── SKILL.md                          ← Main skill file (loaded by Claude)
├── README.md                         ← This file
├── LICENSE                           ← MIT
├── references/
│   ├── deluge-patterns.md            ← Null safety, data types, CRM/Books/Projects patterns
│   ├── cross-app-integrations.md     ← 7 complete integration recipes with full code
│   └── api-and-functions.md          ← invokeURL, webhooks, scheduled functions, security
└── scripts/
    ├── crm-to-books.deluge           ← Deal Won → Books Invoice (copy-paste ready)
    ├── project-from-deal.deluge      ← Deal Active → Zoho Project (copy-paste ready)
    └── new-lead-notify.deluge        ← New Lead → Email + Task (copy-paste ready)
```

---

## Trigger Phrases

Claude activates this skill when you say things like:

- "Write a Deluge script to..."
- "Automate between CRM and Books"
- "When a deal is won, create an invoice"
- "Build a Zoho Flow for..."
- "Fix this Deluge error"
- "How do I call an external API from Deluge?"
- "Set up a scheduled function"
- "Connect CRM to Zoho Projects"
- "Send a Sign document when the deal moves to..."

---

## Integration Map

| From | To | Method | Script |
|---|---|---|---|
| CRM Deal Won | Zoho Books Invoice | Deluge | `crm-to-books.deluge` |
| CRM Deal Active | Zoho Projects | Deluge | `project-from-deal.deluge` |
| CRM New Lead | Email + Task | Deluge | `new-lead-notify.deluge` |
| CRM Proposal Accepted | Zoho Sign | Deluge | See `cross-app-integrations.md` |
| Books Invoice Paid | CRM Update | Zoho Flow | See `cross-app-integrations.md` |
| CRM New Contact | Campaigns Subscriber | Zoho Flow | See `cross-app-integrations.md` |
| Paystack Webhook | CRM Contact | invokeURL | See `api-and-functions.md` |

---

## ROS Org Reference

Default org IDs used in scripts (replace with client values):

| System | ID |
|---|---|
| Zoho Books Org | `874099604` |
| Zoho Projects Portal | `raeofsunshineconsulting` |

---

## Related Skills

This skill works alongside the full ROS Zoho skill suite:

- [`ros-zoho-crm-builder`](https://github.com/Raeofsunshine35/ros-zoho-crm-builder) — CRM pipeline setup, fields, Blueprints
- [`ros-zoho-sites-builder`](https://github.com/Raeofsunshine35/ros-zoho-sites-builder) — Zoho Sites website builds
- [`ros-zoho-commerce`](https://github.com/Raeofsunshine35/ros-zoho-commerce) — Zoho Commerce store setup
- [`ros-trainercentral`](https://github.com/Raeofsunshine35/ros-trainercentral) — Zoho TrainerCentral course builds

---

## License

MIT — free to use, adapt, and redistribute.

---

## About

Built by **Davia-Rae Johnson**, Founder of [Rae of Sunshine Consulting](https://raeofsunshineconsulting.co).
Enterprise Infrastructure Architecture firm serving founders, coaches, real estate professionals,
and service providers across Ghana, the Caribbean, United States, Canada, and the global diaspora.
