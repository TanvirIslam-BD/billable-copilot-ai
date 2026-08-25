# Billable Copilot AI

> **Autonomous AI scope intelligence and unbilled work detector for freelancers, developers, and studios.**  
> Native Model Context Protocol (MCP) tool inside Claude Desktop, ChatGPT, and web workspace.

[![MCP Protocol v2.4](https://img.shields.io/badge/MCP-Protocol%20v2.4-10B981?style=flat-square)](https://modelcontextprotocol.io)
[![Status](https://img.shields.io/badge/Status-Production%20Ready-4F46E5?style=flat-square)]()
[![Theme](https://img.shields.io/badge/Design-Void%20Dark%20%2B%20Clean%20SaaS-0F172A?style=flat-square)]()
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)

---

## ⚡ Overview

Freelancers and studios lose 15–25% of their billable earnings to unbilled scope creep—informal Slack messages, round-4 revisions, and "quick favors" that fall outside the signed Statement of Work (SOW).

**Billable Copilot AI** connects directly to your contract AST. Whenever a client asks for something new, Billable checks it against your agreement in sub-180ms, calculates unbilled hours, and writes the polite reply that gets you paid.

```
                  ┌──────────────────────┐
                  │   Billable MCP       │
                  │   (tools + auth)     │
                  └──────────┬───────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
   Web Dashboard      User's Claude        User's ChatGPT
   (app.html)         (their own app)      (their own app)
```

---

## ✨ Key Features

### 1. 🎯 Autonomous Scope Analyzer
- Paste any incoming client request (e.g., *"Can we also add blog search?"*).
- Instantly matches against signed contract clauses (§2.1 Exclusions, §5.1 Revision Cap).
- Returns definitive verdicts: **`In Scope`**, **`Out of Scope`**, or **`Ambiguous`**.
- Computes estimated revenue impact (e.g., `+$400.00`) and schedule adjustments (`+2 days`).

### 2. 📄 Contract AST Parser & Scope Inspector
- Parses raw contract PDFs, SOWs, and email agreements into structured JSONB ASTs.
- Interactive toggle between **Parsed Scope (JSONB)** and **Raw Contract Text**.
- Live tracking of revision allowances with small meters (`2 / 3 used`).

### 3. 💬 Dynamic Tone Switcher & Polite Reply Generator
- Drafts ready-to-send change order emails with 1-click clipboard copy.
- Supports 3 tone modes:
  - **Warm & Collaborative**: Friendly, partnership-focused wording.
  - **Neutral & Professional**: Objective, standard agency phrasing.
  - **Firm & Formal**: Contract-strict phrasing for difficult scope boundaries.

### 4. 📊 Change Orders Kanban Pipeline
- Visual pipeline: `Draft` → `Sent / Pending` → `Accepted` → `Paid / Collected`.
- Monospace right-aligned currency figures with tabular numerals.

### 5. 💰 Append-Only Revenue Recovery Ledger
- Monthly accounting audit report: **Caught ($2,340)** / **Billed ($1,900)** / **Waived ($440)** / **Collected ($1,840)**.
- Cryptographic SHA-256 audit hashes for every unbilled event log.
- 1-click CSV audit log export.

### 6. 🌐 Persistent AI Copilot Rail
- Right-hand contextual rail always present on desktop.
- Context injection: passes `visible_state` (screen, contract ID, revisions used, unbilled amounts).
- Real-time animated Scope Radar waves tracking contract health (*Healthy*, *Warning*, *At Risk*).

---

## 🖥️ Screen Architecture

The application is structured into two core experiences:

### Marketing Landing Page (`index.html`)
- **Theme**: Dark Obsidian / Void Onyx (`#050709`).
- **Signature Interactive Scope Deck 2.0**: Live hardware simulator with lime/emerald client ask bubbles, Copilot AST breakdown, circular radial gauge, and custom scope sandbox testing.
- **High-Fashion Editorial Photography**: Curated Unsplash imagery across hero, 3D bento grid, testimonials, pricing, and panoramic pre-footer.
- **Full SEO & Structured Data**: Complete Open Graph, Twitter Cards, `robots.txt`, `sitemap.xml`, and Schema.org JSON-LD (`WebApplication`, `Organization`, `FAQPage`).

### Web Console (`app.html`)
- **Theme**: Neo-Clean Light SaaS Dashboard (`#F8FAFC` background with pure white cards and indigo/emerald accents).
- **Screens**:
  1. **Today / Overview**: KPI sparkline cards, 6-month trend multi-line curve chart, requests analyzed donut breakdown, and risk tables.
  2. **Contracts**: Active contract AST scope viewer with JSONB vs Raw toggle.
  3. **Requests Stream**: Filterable stream of client checks across Email, Slack, and WhatsApp.
  4. **Change Orders**: Kanban pipeline tracking quotes to payments.
  5. **Recovery Ledger**: Monthly audit report with append-only event trail.
  6. **MCP & Settings**: Claude Desktop and ChatGPT configuration generator.

---

## 🚀 Quick Start (Local Setup)

### 1. Clone the repository
```bash
git clone https://github.com/TanvirIslam-BD/billable-copilot-ai.git
cd billable-copilot-ai
```

### 2. Start local static server
Using Python:
```bash
python -m http.server 3000
```
Or using Node.js / `npx serve`:
```bash
npx serve -l 3000 .
```

### 3. Open in your browser
- **Landing Page**: [http://localhost:3000](http://localhost:3000)
- **App Console**: [http://localhost:3000/app.html](http://localhost:3000/app.html)

---

## 🔌 Claude Desktop MCP Integration

Add the following configuration to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "billable": {
      "command": "npx",
      "args": ["-y", "@billable/mcp-server@latest"],
      "env": {
        "BILLABLE_API_KEY": "your_billable_api_key_here"
      }
    }
  }
}
```

### MCP Tool Surface
- `add_contract`: Upload raw text or file → parsed JSONB scope out.
- `check_request`: Request text + contract → verdict (`in_scope`, `out_of_scope`, `ambiguous`), clause reference, reasoning, suggested amount, and delay.
- `draft_reply`: Generates polite message based on `tone` (`warm` | `neutral` | `firm`).
- `create_change_order`: Creates formal PDF change order and tracking entry.
- `get_recovery_report`: Returns monthly financial summary (caught, billed, waived, collected).

---

## 📁 Repository Structure

```
billable-copilot-ai/
├── assets/                       # Official branding PNGs and images
│   ├── logo-horizontal.png
│   ├── logo-icon.png
│   └── logo-app-badge.png
├── favicon.svg                   # Vector logo favicon
├── favicon.png                   # Retina PNG favicon
├── og.png                        # Open Graph social preview
├── index.html                    # Dark Mode Marketing Landing Page & Scope Deck
├── app.html                      # 5-Screen Web Console Dashboard & AI Copilot Rail
├── DASHBOARD_BUILD_GUIDE.md      # Comprehensive architecture & build guide
├── robots.txt                    # Search crawler instructions
├── sitemap.xml                   # XML sitemap for SEO
└── README.md                     # Project documentation
```

---

## 🎨 Design System Tokens

| Element | Color Code | Purpose |
|---|---|---|
| **Void Obsidian (Landing)** | `#050709` / `#0D1117` | High-contrast dark marketing theme |
| **Clean Surface (App)** | `#F8FAFC` / `#FFFFFF` | High-density readability dashboard |
| **Brand Indigo** | `#4F46E5` / `#6366F1` | Primary actions and active navigation |
| **Recovered Money** | `#00F59B` / `#10B981` | Recovered revenue, success states |
| **Out-of-Scope Flag** | `#EF4444` / `#FF3B30` | Scope creep warnings and risk alerts |
| **Ambiguous Warning** | `#F59E0B` | Unclear scope requiring clause review |

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).
