# 🚀 Project Plan: System Design Learning Website
> Personal notes → Markdown files → GitHub Pages → Accessible Anywhere

---

## 🎯 Goal

Build a **personal system design knowledge base** that:
- Lives as Markdown (README) files in a GitHub repo
- Auto-publishes as a website on every `git push`
- Is fully writable in VS Code with Claude Code
- Grows organically as you study each roadmap phase

---

## 🛠️ Tech Stack

| Layer | Tool | Why |
|-------|------|-----|
| **Site Generator** | [Docusaurus v3](https://docusaurus.io/) | Markdown-first, React, dark mode, search built-in |
| **Diagrams** | Mermaid.js | Architecture diagrams inside Markdown |
| **Hosting** | GitHub Pages | Free, auto-deploy, custom domain ready |
| **CI/CD** | GitHub Actions | Auto-build & publish on every push |
| **Editor** | VS Code + Claude Code | AI-assisted note writing |
| **Version Control** | GitHub | Source of truth |

> **Simpler alternative:** Use [MkDocs + Material theme](https://squidfunk.github.io/mkdocs-material/) — pure Python, zero JS setup needed.

---

## 📁 Repository Structure

```
system-design-notes/
│
├── .github/
│   └── workflows/
│       └── deploy.yml                  ← GitHub Actions auto-deploy
│
├── docs/                               ← All your Markdown notes
│   ├── intro.md                        ← Home / landing page
│   │
│   ├── phase-1-foundations/
│   │   ├── networking.md
│   │   ├── os-concepts.md
│   │   ├── data-structures.md
│   │   └── java-deep-dive.md
│   │
│   ├── phase-2-core-concepts/
│   │   ├── scalability-patterns.md
│   │   ├── caching.md
│   │   ├── cap-theorem.md
│   │   └── rate-limiting.md
│   │
│   ├── phase-3-distributed-systems/
│   │   ├── consensus-raft-paxos.md
│   │   ├── distributed-transactions.md
│   │   └── data-consistency.md
│   │
│   ├── phase-4-databases/
│   │   ├── relational-deep-dive.md
│   │   ├── nosql-patterns.md
│   │   └── storage-engines.md
│   │
│   ├── phase-5-messaging/
│   │   ├── kafka-deep-dive.md
│   │   ├── event-driven-patterns.md
│   │   └── stream-processing.md
│   │
│   ├── phase-6-microservices/
│   │   ├── ddd-bounded-contexts.md
│   │   ├── api-design.md
│   │   └── service-mesh.md
│   │
│   ├── phase-7-observability/
│   │   ├── logging-metrics-tracing.md
│   │   ├── slo-sla-sli.md
│   │   └── chaos-engineering.md
│   │
│   ├── phase-8-security/
│   │   ├── auth-authz.md
│   │   └── owasp-compliance.md
│   │
│   ├── phase-9-cloud-native/
│   │   ├── kubernetes-deep-dive.md
│   │   ├── cloud-patterns.md
│   │   └── infrastructure-as-code.md
│   │
│   └── phase-10-case-studies/
│       ├── url-shortener.md
│       ├── news-feed.md
│       ├── youtube.md
│       ├── uber.md
│       ├── whatsapp.md
│       └── payment-system.md
│
├── static/
│   └── img/                            ← diagrams, screenshots
│
├── docusaurus.config.js                ← Site config
├── sidebars.js                         ← Navigation
├── package.json
└── README.md                           ← Repo overview
```

---

## 🗓️ Execution Plan (4 Phases)

---

### Phase A — Setup & Scaffold (Week 1) ✅

**Goal:** Get a working website live on GitHub Pages before writing a single note.

**Steps:**

1. **Create GitHub repo**
   ```
   Repo name: system-design-notes
   Visibility: Public (required for free GitHub Pages)
   ```

2. **Scaffold Docusaurus locally**
   ```bash
   npx create-docusaurus@latest system-design-notes classic
   cd system-design-notes
   npm run start       # verify locally at localhost:3000
   ```

3. **Configure `docusaurus.config.js`**
   ```js
   title: 'System Design Notes'
   tagline: '15 Years of Java Backend → Architecture Mastery'
   url: 'https://<your-github-username>.github.io'
   baseUrl: '/system-design-notes/'
   ```

4. **Create GitHub Actions deploy workflow**
   - File: `.github/workflows/deploy.yml`
   - Trigger: push to `main` branch
   - Action: build Docusaurus → publish to `gh-pages` branch

5. **Enable GitHub Pages**
   - Repo Settings → Pages → Source: `gh-pages` branch

6. **Verify deployment**
   - Push one commit → confirm site is live at `https://<username>.github.io/system-design-notes/`

**Claude Code usage in this phase:**
- Ask Claude Code to generate `deploy.yml` GitHub Actions file
- Ask Claude Code to scaffold the full folder structure automatically

---

### Phase B — Note Writing Workflow (Ongoing, Week 2+) 📝

**Goal:** Establish a fast, repeatable workflow to write notes in VS Code and publish instantly.

**Daily Workflow:**
```
Study topic
  → Open VS Code
  → Write/update .md file with Claude Code help
  → git add . && git commit -m "notes: kafka consumer groups"
  → git push
  → GitHub Actions builds & deploys (~2 min)
  → Site is live ✅
```

**VS Code Extensions to Install:**

| Extension | Purpose |
|-----------|---------|
| `Markdown Preview Enhanced` | Live preview while writing |
| `Markdown All in One` | Shortcuts, auto TOC generation |
| `Mermaid Preview` | Preview architecture diagrams inline |
| `GitLens` | Git history inside VS Code |
| `Claude Code` | AI assistant for note writing |

**Claude Code Prompts for Note Writing:**

| Task | What to ask Claude Code |
|------|------------------------|
| Explain a concept | "Explain Kafka consumer groups with Java examples. Format as Markdown." |
| Draw a diagram | "Create a Mermaid sequence diagram for the Kafka producer-consumer flow." |
| Expand rough notes | "I wrote this rough outline — expand it with details, trade-offs, and code." |
| Add Java example | "Add a Spring Boot Resilience4j circuit breaker Java code example." |
| Review completeness | "Review my notes on consistent hashing and add any missing key concepts." |
| Write case study | "Write a system design case study for URL Shortener using this template." |

**Markdown Note Template — Use for Every Topic:**

```markdown
# Topic Name

## 📌 What is it?
Brief definition in your own words.

## 🤔 Why does it exist? (Problem it solves)

## ⚙️ How it works (Deep Dive)

## 🏛️ Architecture Diagram
(Mermaid diagram here)

## 💻 Java Code Example

## ⚡ Trade-offs
| Pro | Con |
|-----|-----|

## 🔗 When to use / When NOT to use

## 🔗 Related Topics

## 📚 References
```

---

### Phase C — Content Build-Out (Weeks 2–52) 📚

**Goal:** Systematically fill in all 10 phases of the roadmap, one topic at a time.

**Weekly content schedule:**

| Weeks | Phase | Topics |
|-------|-------|--------|
| 1–4 | Phase 1 | Networking, OS concepts, JVM internals, Data structures for SD |
| 5–10 | Phase 2 | Scalability, Caching, CAP theorem, Rate limiting |
| 11–16 | Phase 3 | Raft/Paxos, SAGA pattern, CDC, CRDTs |
| 17–20 | Phase 4 | SQL sharding, Cassandra, DynamoDB, LSM Trees |
| 21–24 | Phase 5 | Kafka deep dive, Flink, Stream processing |
| 25–28 | Phase 6 | DDD, gRPC, API Design, Service Mesh |
| 29–32 | Phase 7 | OpenTelemetry, SLO/SLA, Chaos Engineering |
| 33–35 | Phase 8 | Zero Trust, OWASP Top 10, HashiCorp Vault |
| 36–39 | Phase 9 | Kubernetes, Terraform, GitOps |
| 40–52 | Phase 10 | 20 Case Studies (1 system per week) |

**Case Study Note Template:**

```markdown
# System Design: [System Name]

## Requirements Clarification
### Functional
- ...
### Non-Functional (scale, latency, availability)
- ...

## Back-of-Envelope Estimation
- QPS: ...
- Storage: ...
- Bandwidth: ...

## High-Level Design
(Mermaid diagram)

## Data Model

## Deep Dive: [Core Component]

## Bottlenecks & Scaling Strategy

## Failure Scenarios & Reliability

## Trade-offs Made
```

---

### Phase D — Polish & Enhancements (Month 6+) ✨

**Goal:** Make the site polished, searchable, and shareable.

- [ ] Add **Algolia DocSearch** — full-text search (free for open-source docs)
- [ ] Add **custom domain** — e.g. `notes.yourname.dev` (~₹800/yr)
- [ ] Add **Progress tracker** page — visual checklist of completed topics
- [ ] Add **Interview cheatsheets** — one-page quick reference per phase
- [ ] Enable **tags** on each page (Docusaurus built-in)
- [ ] Add **"Last updated"** timestamp on every note
- [ ] Add **Giscus comments** — GitHub Discussions per page
- [ ] **Share on LinkedIn** — your live notes as a public portfolio

---

## 🔄 GitHub Actions Deploy File

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Build website
        run: npm run build

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./build
```

---

## 📐 Daily Claude Code Workflow in VS Code

```
1. Open topic's .md file in VS Code
2. Write a rough outline or paste what you've read/studied
3. Ask Claude Code:
   "Expand this into detailed notes with Java examples and a Mermaid diagram"
4. Review, edit, add your own real-world insights
5. git push → site live in 2 minutes
```

---

## ✅ Week 1 Checklist (Start Here)

- [ ] Create GitHub repo: `system-design-notes` (Public)
- [ ] Scaffold Docusaurus: `npx create-docusaurus@latest system-design-notes classic`
- [ ] Run locally and verify: `npm run start`
- [ ] Configure `docusaurus.config.js` — title, URL, baseUrl
- [ ] Create full folder structure for all 10 phases
- [ ] Create `.github/workflows/deploy.yml`
- [ ] Enable GitHub Pages in repo Settings → Pages → `gh-pages` branch
- [ ] Push and verify live URL works
- [ ] Install VS Code extensions (Markdown Preview, Mermaid, Claude Code)
- [ ] Write first note: `docs/phase-1-foundations/networking.md`
- [ ] Push and see it live on GitHub Pages 🎉

---

## 💰 Cost

| Item | Cost |
|------|------|
| GitHub repo + Pages | **Free** |
| Docusaurus | **Free** |
| GitHub Actions | **Free** (2000 min/month) |
| Algolia DocSearch | **Free** (open-source docs) |
| Custom domain (optional) | ~₹800/year |
| **Total** | **₹0/month** |

---

## 🏁 End Result (12 Months)

| Deliverable | Status |
|-------------|--------|
| Live searchable website | ✅ `https://username.github.io/system-design-notes` |
| 10 phases of deep-dive notes | ✅ Written, versioned, yours forever |
| 20 case studies with diagrams | ✅ Ready for interview revision |
| Personal portfolio piece | ✅ Share on LinkedIn, resume |
| Accessible from any device | ✅ Phone, tablet, laptop — anywhere |

---

> *"The best way to learn system design is to teach it — even if the only student is future you."*
