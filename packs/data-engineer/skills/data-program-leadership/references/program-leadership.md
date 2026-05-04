# Program Leadership

This reference provides deep, actionable guidance for leading data programs across maturity stages, team structures, delivery practices, and multi-team strategy.

## Data maturity model

### Stage 1: Starting

Characteristics: scattered data across production databases, spreadsheets, and SaaS tools. No centralized analytics. Decisions driven by gut feeling or manual exports.

Team shape: 1-2 people. Typically a tech lead acting as data architect plus one engineer covering initial data engineering work. PM or lead gathers business requirements informally.

What to focus on:

- Stand up a single analytics replica or lightweight warehouse (Postgres replica, managed DW, or ClickHouse).
- Build 1-2 dashboards covering 2-3 high-impact metrics (DAU, revenue, completion rate).
- Interview 3-5 end users to discover the business questions that matter most.
- Find a project champion/sponsor in leadership.

What to avoid:

- Data Mesh, lakehouse architecture, or any multi-team platform design.
- Hiring a data scientist before you have reliable reporting.
- Spending more than 2 weeks on architecture design before building anything.

Concrete indicators you are here: no dedicated data team, analytics queries run against production, reports are spreadsheets or manual exports, metric definitions differ between teams.

### Stage 2: Scaling

Characteristics: centralized reporting exists for at least one domain. Formal data pipeline (dbt or equivalent) is running. At least one dashboard is trusted and used weekly.

Specialized roles appearing: dedicated data analyst (first hire priority), data engineer maintaining pipelines, power users in business teams trained on the BI tool.

Formal practices:

- Metric definitions documented in a data catalog (even a Notion page).
- dbt tests and data freshness monitoring.
- Access control separating raw data from mart layers.
- Change management for metric definitions (PR review process).

What to focus on:

- Expand to 3-4 domain marts (learning, sales, marketing, content).
- Train 1-2 power users per team to self-serve reports in the BI tool.
- Establish the 5 governance rules (metric ownership, naming conventions, access control, data freshness, change management).

### Stage 3+: Leading

Characteristics: self-service analytics across most teams. Automated data quality checks. Data-driven culture where features are planned with baseline metrics and KPI targets before building.

What to focus on:

- Metrics layer (define core metrics once, reuse everywhere).
- Advanced analytics: cohort analysis, predictive models, A/B test frameworks.
- Consider CDC for near real-time pipelines where batch latency is insufficient.
- Executive dashboard aggregating cross-domain metrics.

### How to assess current stage

| Indicator | Stage 1 | Stage 2 | Stage 3+ |
|-----------|---------|---------|----------|
| Where analytics queries run | Production DB | Dedicated analytics DB | Self-serve platform with governed access |
| Metric definitions | Undefined or conflicting | Documented for key metrics | Catalog with ownership, lineage, freshness SLA |
| Data team | None (engineers moonlight) | 1-3 dedicated people | Dedicated team with specialized roles |
| Report creation | Engineering builds every report | Power users create some reports | Business teams self-serve 80%+ of needs |
| Data quality | Unknown until someone complains | Automated tests on key tables | Comprehensive monitoring with alerting |

---

## 14 common pitfalls

These pitfalls are drawn from decades of data project experience. Data projects almost never fail because of technology; they fail because of people and process.

1. **Executives think BI is "easy."** Leadership imposes unrealistic timelines because they underestimate complexity. Even defining "active user" can take weeks of debate. Educate stakeholders with concrete examples of hidden complexity.

2. **Wrong technology choice.** Choosing tools based on familiarity rather than fit. Research before committing. A small evaluation committee spending one focused week prevents years of pain.

3. **Too many requirements gathered.** Spending months collecting hundreds of pages of requirements that no one reads while developers sit idle. Gather enough to start (a few weeks), then continue discovery in parallel with building.

4. **Too few requirements gathered.** Engineers guess what users need and build the wrong thing. Use iterative delivery: build a slice, show users, collect feedback, adjust, repeat.

5. **Reports with unvalidated data.** Showing numbers that contradict what users already know destroys trust immediately and is extremely hard to recover from. Cross-check against existing spreadsheets and manual reports before launch. When discrepancies exist, explain why rather than hiding them.

6. **Inexperienced consultants.** Consulting firms send experts to pitch but assign juniors to deliver. Always ask: "Will the person who pitched be the person who builds?"

7. **Offshore outsourcing without control.** Time zone gaps, language barriers, and reduced quality oversight compound quickly. If outsourcing, invest in tight communication cadence and clear acceptance criteria.

8. **Full ownership given to consultants.** The project becomes a black box with no internal visibility into real progress. Always assign an internal PM or tech lead with direct oversight, not just status report consumption.

9. **No knowledge transfer.** Consultants or single contributors build the system and leave. No one internal can fix bugs or add features. Ensure at least two people understand every critical system. Documentation is mandatory, not optional. Bus factor must be greater than one.

10. **Budget cuts mid-project.** Funding disappears near completion, forcing cuts to people, testing, or infrastructure. If cuts are unavoidable, reduce scope (fewer data sources in phase 1) rather than cutting quality or validation.

11. **Hard deadline, work backward.** Fixed deadlines force rushed discovery, skip validation, and prevent stakeholder collaboration. Push back with specifics: "I can deliver metric X by end of month, but the full analytics suite needs 2-3 months to do correctly."

12. **Modeling the warehouse by source data structure.** Copying production schema directly into the analytics layer produces tables optimized for CRUD, not analysis. Model based on business questions: "What does the business need to answer?" Design fact and dimension tables accordingly.

13. **Poor performance.** If the new dashboard loads slower than the spreadsheet it replaces, users will reject it regardless of additional capability. Benchmark response times and ask users what latency is acceptable before launch.

14. **Over-design or under-design.** Too little design leads to missing scalability and security gaps. Too much design leads to analysis paralysis and unnecessary complexity. Rule of thumb: if you have spent more than two weeks designing architecture without building anything, you are over-designing.

---

## 6 tips for delivery success

1. **Right investment level.** Sometimes spending twice the time and money builds a solution that returns 50 times the value. At minimum, one person on the team must understand data architectures well enough to make sound decisions. Rushing past this creates systems with wrong results, poor performance, and compliance gaps.

2. **Involve users early, show results often.** Show output quickly and continuously. Start with the smallest, easiest, highest-priority data source. Build one simple dashboard, show users, iterate. Users who see value early become project advocates instead of skeptics.

3. **Add value to reports and dashboards.** When migrating from spreadsheets or manual reports, the new system must show users something they could not see before: trends over time, cohort analysis, drill-downs by segment. If the new tool just recreates the old one, users will ask why they paid the cost.

4. **Let end users build prototypes.** Train power users (the best Excel/Sheets person on each team) to use the BI tool. When users build their own reports, they feel ownership and adopt naturally without coercion.

5. **Find a project champion/sponsor.** A senior executive who advocates internally, secures resources, facilitates cross-functional collaboration, and manages resistance. Without a champion, data projects die from budget cuts, resource starvation, or organizational apathy.

6. **Target 80% efficiency (20% buffer).** Plan for team members to spend roughly 80% of allocated time on project work. The remaining 20% covers meetings, learning, unexpected tasks, and rest. Minimize waiting time by ensuring the right people are assigned to the right tasks with proper training before they start.

---

## Team roles and organizational design

### Core roles for data programs

| Role | Responsibility | Essential at Stage |
|------|---------------|-------------------|
| Data Architect | High-level architecture design, technology selection, governance policies | Stage 1 (tech lead covers) |
| Data Engineer | Build, test, maintain pipelines and transformations | Stage 2 |
| Data Steward | Data quality, governance, access rights, cleansing rules | Stage 3 |
| DBA | Operations, backup, recovery, performance tuning | Stage 2 (DevOps/SRE covers) |
| Data Analyst | Analyze data, find patterns, create reports, communicate insights | Stage 2 (first dedicated hire) |
| Data Scientist | ML models, predictive/prescriptive analytics, A/B testing | Stage 3+ |
| Business Analyst | Bridge between data team and business users, gather and translate requirements | Stage 2 (PM covers initially) |
| Project Manager | Coordinate team, track progress, resolve blockers | Stage 1 (existing PM) |
| Data Privacy Officer | Compliance with regulations (GDPR, sector-specific rules) | Stage 2 |
| Data Governance Manager | Define policies for data quality, security, availability | Stage 3 |
| Data Quality Manager | Monitor completeness, consistency, accuracy | Stage 3 |
| Domain Data Product Owner | Manage lifecycle of data products within a domain | Stage 3 (Data Mesh) |
| Data Mesh Architect | Design overall mesh, coordinate across domains | Stage 3 (Data Mesh) |
| Platform Engineer | Build and maintain shared data infrastructure | Stage 3 (Data Mesh) |
| DataOps Engineer | Data lifecycle automation, quality monitoring | Stage 3 |
| Security and Compliance Officer | Regulations, data security standards | Stage 2 |
| UX Designer (Platform) | Interface design for internal data tools | Stage 3 (Data Mesh) |

### Role combinations for small teams

At stage 1-2, most roles are vacant. Make temporary combinations explicit:

| Person | Covers roles |
|--------|-------------|
| Tech lead | Data Architect + partial Data Engineer + Business Analyst (gathers requirements from stakeholders) |
| Backend engineer | Data Engineer (builds initial pipelines) + partial DBA |
| PM | Project Manager + partial Business Analyst |
| Power users (1-2 per business team) | Partial Data Analyst (self-serve reporting) |

Priority first hire when budget allows: Data Analyst. They prove value fastest by creating reports and dashboards, and they bridge the gap between engineering and business teams.

### Data Mesh roles (stage 3+ with 100+ engineers)

Data Mesh requires three separate teams:

**Domain teams** -- each business domain (sales, content, marketing) has its own data roles scoped to that domain, plus a Domain Data Product Owner managing data product lifecycle.

**Self-serve data infrastructure platform team** -- builds the shared platform: platform architect, platform engineers, data engineers, DevOps, DataOps, security officer, product owner, UX designer, support and training.

**Federated computational governance team** -- defines global standards: governance lead, governance specialist, privacy officer, security analyst, quality specialist, domain representatives, data architect, DataOps engineer.

### When NOT to build pure Data Mesh

Do not attempt pure Data Mesh with fewer than 100 engineers. The organizational overhead of three specialized teams with dozens of roles is not justified until the central data team is a clear bottleneck across many domains. For teams of 20-50 engineers serving 5-6 business teams, use "Data Mesh Lite" (centralized infrastructure, domain-defined metrics).

| Criterion | Data Mesh Requirement | Typical mid-size team |
|-----------|----------------------|----------------------|
| Engineering scale | 100+ engineers, central team is bottleneck | 20-50 engineers, no data team yet |
| Per-domain staffing | 5-10 data people per domain | Total engineering is 20-50, not just data |
| DDD experience | Strong domain-driven design culture | Team has not practiced DDD |
| Platform + governance teams | 15-20+ specialized roles across 2 centralized teams | No budget for dedicated teams |
| Cultural readiness | Organization-wide commitment to data ownership | Varies, often unclear |

---

## Multi-team data strategy

### Data Mesh Lite: comparison table

When multiple teams need data but the organization is too small for pure Data Mesh, borrow the four principles with centralized implementation.

| Principle | Pure Data Mesh | Data Mesh Lite |
|-----------|---------------|----------------|
| **Domain Ownership** | Each domain builds and manages its own data pipelines | Each domain **defines** metrics and business rules. Central engineering **implements** in one shared project |
| **Data as Product** | Formal data contracts, versioned APIs, self-contained data products | dbt schema YAML with metric definitions, owner, approval date. Data catalog (Notion or markdown) |
| **Self-Serve Platform** | Custom-built platform for domain teams | BI tool (Metabase/Superset) with trained power users per domain. Engineering only involved for new data sources or metric definitions |
| **Federated Governance** | Dedicated governance team with 10+ roles | 5 simple rules enforced by tech lead (see governance rules below) |

### 3-layer dbt architecture for multi-team

Three layers are necessary. Skipping the intermediate layer causes painful refactoring later.

| Layer | Purpose | Naming | Contents |
|-------|---------|--------|----------|
| Raw (staging) | Data as-is from sources, no transforms | `raw_` prefix | Deduplicated source tables (e.g., `raw_users`, `raw_sessions`, `raw_payments`) |
| Intermediate | Business entities, cleaned, joined, deduped | `int_` prefix | Enriched entities (e.g., `int_user_sessions`, `int_course_progress`, `int_exam_scores`) |
| Mart (per domain) | Domain-specific aggregations, ready for BI | `mart_` prefix with domain | Domain marts with fact and dimension tables (e.g., `mart_learning.fact_dau`, `mart_sales.fact_mrr`) |

Shared dimensions live in the intermediate layer and are referenced by all marts: `dim_user`, `dim_date`, `dim_course`, `dim_product_line`, `dim_market`.

### 5 governance rules with enforcement

| Rule | Description | Enforcement mechanism |
|------|------------|----------------------|
| 1. Metric ownership | Every metric must have an owner team and definition in the data catalog. No ad hoc metrics without catalog entry | Tech lead reviews new metric additions |
| 2. Naming convention | dbt models use `raw_`, `int_`, `mart_` prefixes. Metrics use snake_case. Tables use `fact_` or `dim_` prefix | Automated dbt CI/CD check |
| 3. Access control | Each team queries only its own mart plus shared dimensions. No direct raw layer access except engineering | Database RBAC (role per domain) |
| 4. Data freshness | Every dashboard displays "last updated" timestamp. Alert when data exceeds SLA (e.g., >2 hours for hourly, >24 hours for daily) | Monitoring and alerting system |
| 5. Change management | Metric definition changes require owner team approval before deployment | Pull request review process with domain team sign-off |

### Analytics engine selection

Columnar OLAP databases (ClickHouse, BigQuery, Redshift) outperform row-based OLTP databases for analytics workloads. Key advantages of a dedicated analytics engine:

- Sub-second queries on large datasets through vectorized execution and columnar storage.
- 10-40x compression ratios reducing storage costs.
- Materialized views for automatic pre-aggregation on insert (no scheduled refresh needed).
- Support for high concurrent dashboard usage without impacting production workloads.
- Native dbt adapter support.

Recommendation: start with batch sync (scheduled hourly or daily) from production database to analytics engine. Evolve to CDC (Change Data Capture) only when near real-time latency is required.

### Ingestion strategy evolution

| Phase | Method | Latency | Complexity |
|-------|--------|---------|------------|
| Initial | Batch sync via scheduled scripts or Airbyte | 1-24 hours | Low |
| Growth | CDC via Debezium or PeerDB streaming WAL changes | Near real-time | Medium |
| Advanced | Event streaming via Kafka for click-stream and behavioral data | Real-time | High |

---

## Phased rollout planning

### Phase 1 (months 1-2): first domain, dashboard, metric framework

Run two tracks in parallel:

**Track A -- Quick wins on existing systems:** Business teams begin aggregating available metrics (DAU, completion rate, revenue) immediately using current tools (production queries, spreadsheets, existing analytics). Establish baseline numbers. Map planned features to target metrics. Define KPIs per module before building features.

**Track B -- Analytics foundation:** Set up the analytics engine. Batch sync the first domain data. Build the first mart with dbt (fact tables + shared dimensions). Deploy BI tool with the first dashboard. Validate numbers against existing manual reports.

Deliverable: one domain has a trusted dashboard. Business team has a "feature to metric to KPI" framework in use. Baseline numbers are established.

Risk: unvalidated data (pitfall 5), performance worse than spreadsheets (pitfall 13), modeling by source schema (pitfall 12).

### Phase 2 (months 3-4): additional domains, data catalog, power user training

- Sync additional data sources (payment systems, marketing APIs).
- Build 2-3 additional domain marts.
- Create data catalog documenting all metrics with definitions, owners, sources, refresh frequency.
- Train 1-2 power users per team on the BI tool.
- Establish RBAC (role-based access control per domain).
- Build shared dimensions ensuring cross-domain consistency (especially `dim_user`).

Deliverable: 3-4 teams have their own dashboards. Power users self-serve basic reports. Data catalog exists.

Risk: metric definition conflicts across teams, engineering bottleneck from simultaneous requests from multiple teams, scope creep.

### Phase 3 (months 5-6): remaining domains, metrics layer, monitoring

- Build executive dashboard (cross-domain aggregation: revenue + engagement + content quality + growth).
- Implement dbt metrics layer (define core metrics once, prevent duplicate definitions).
- Deploy automated data quality monitoring and freshness alerting.
- Set up dbt CI/CD (automated tests before deployment).
- Performance tuning: optimize slow queries, add materialized views, review partition strategies.

Deliverable: full analytics platform with self-serve BI for all teams, guaranteed data quality, and sub-second dashboard load times.

Risk: bus factor if only 1-2 engineers understand the system, advanced features adding complexity without proportional value.

---

## Domain discovery

### Domain identification approach

Map business teams to data domains. Each domain represents a bounded set of business questions:

| Domain | Business questions | Typical data sources |
|--------|-------------------|---------------------|
| Learning/Product | How do users engage? Which features work? When do users churn? | Application database, event logs |
| Content/Academic | Which content is effective? Where are learners struggling? | LMS, assessment systems |
| Sales/Revenue | What is revenue? Who is churning? What is LTV? | Payment system, subscription DB, CRM |
| Marketing/Growth | Which channels convert? What is CAC? Which campaigns have ROI? | Ad platforms, analytics tools, email tools |
| B2B/Enterprise | Which organizations are active? License utilization? Renewal rate? | CRM, organization DB |
| Executive | Is the business healthy? Growing or shrinking? Where to invest? | Aggregation of all other domains |

Prioritize domains where: (a) schema is already known, (b) business urgency is highest, (c) a champion exists on the business side. Start with one domain that meets all three criteria.

### Schema discovery: known vs unknown

For each domain, classify data sources:

- **Known schema:** Production databases you control. Can inspect tables, relationships, and data volumes directly. Ready to build marts immediately.
- **Unknown schema:** Third-party tools, external APIs, other teams' databases, or systems where no one on the data team has explored the schema. Require discovery interviews before mart design.

Risk: unknown schemas should be postponed to a later phase. Attempting to build marts for domains with unknown schemas leads to rework and delays.

### Shared dimensions

Shared dimensions connect all domains and prevent data silos. `dim_user` is the most critical shared dimension because it links engagement (learning) with revenue (sales) with acquisition (marketing).

Critical columns for `dim_user`:

- `user_id` (primary key, present in all domain tables)
- `signup_date` (required for cohort analysis)
- `market_id` or `country` (required for geographic breakdowns)
- `subscription_tier` (free/premium/trial, required for revenue analysis)
- `org_id` (null for B2C, populated for B2B, required to separate segments)
- `acquisition_source` (UTM parameters, referral, required for marketing attribution)

If `dim_user` is missing key columns, no downstream mart can produce breakdowns by those dimensions. Investigate the user table schema in the first week.

### Discovery interview template

Use this template for each domain with unknown schema:

**Data sources:**

- What systems and tools does the team currently use?
- Where does the most important data live?
- Is there API access for third-party tools?

**Key questions:**

- What are the top 3 business questions the team wants to answer with data?
- How are those questions answered today (direct queries, manual reports, intuition)?
- How frequently does the team need to see updated data (real-time, daily, weekly)?

**Metrics:**

- What metrics does the team currently track? (name, definition, source, frequency)
- What metrics does the team wish it could track but cannot today?

**Access and users:**

- Who on the team will view reports? (roles and names)
- Who is the strongest spreadsheet user (power user candidate)?
- Is any data sensitive (PII, financial) requiring restricted access?

---

## Stakeholder alignment

### Leadership expectations management

Set expectations explicitly during the initial pitch:

- Timeline: a useful first dashboard takes 4-8 weeks, not days. A full analytics suite takes 3-6 months.
- Scope: phase 1 covers one domain with 2-3 key metrics. Broader coverage comes in later phases.
- Investment: at least one engineer needs 2-4 weeks of dedicated learning time before making architecture decisions.

Reframe every technical decision as business impact. Do not say "we need a Star Schema with SCD Type 2." Say "after two weeks, marketing will have a real-time dashboard tracking conversion rates instead of waiting until month-end to count manually."

### Define KPIs per module before building features

Before building any analytics feature, require the business team to specify:

1. What is the current baseline number?
2. What KPI target are we aiming for?
3. Which feature is expected to move this metric?
4. How will we measure the impact (A/B test or before/after comparison)?

### Measure impact post-feature

After shipping a feature, compare the target metric before and after. This creates a feedback loop that proves data project value and justifies continued investment:

- Baseline (pre-feature): DAU = X, Completion Rate = Y%
- Target: DAU +10%, Completion Rate +10%
- Actual (post-feature): measure and report the delta
- Breakdowns: per market, per module, per product line

### Business question to data requirement mapping

| Business question | Required metric | Required dimensions | Data source |
|-------------------|----------------|--------------------|-----------  |
| "How many users are active?" | DAU/WAU/MAU | date, market, product line | Session/engagement data |
| "Are users completing courses?" | Completion rate | course, level, market | Progress tracking data |
| "How much revenue are we generating?" | MRR/ARR | market, subscription tier, product line | Payment system |
| "Which acquisition channel works best?" | CAC, conversion rate | channel, campaign, market | Ad platforms + signup data |
| "Is the business healthy?" | Unit economics (LTV/CAC), retention, NPS | market, cohort | Cross-domain aggregation |

Every business question maps to specific metrics, dimensions, and data sources. If the data source for a question is unknown, that question belongs in a later phase.
