### Section 7: Documentation & Leadership

#### 7.1 Incident runbook outline

- **Title & scope**
  - e.g., `API Latency and Errors – Prod Runbook`.

- **Sections**
  - **Symptoms**: what alerts trigger this runbook; user-facing signals.
  - **Quick checks**:
    - Links to:
      - Grafana dashboards.
      - Kibana/CloudWatch Log insights queries.
      - `kubectl` snippets for common checks.
  - **Decision tree**:
    - If nodes `NotReady` → follow Node Health subsection.
    - If only 1 service affected → follow Service Health subsection.
  - **Standard actions**:
    - Steps for rollback, scaling, known mitigations.
  - **Escalation**:
    - When to bring in DB team, networking, security.
  - **Post-incident steps**:
    - Ensure tickets created and postmortem scheduled.

#### 7.2 On-call escalation guide outline

- **Purpose**
  - Define who responds, in what order, and expectations.

- **Contents**
  - **Roles**:
    - Primary on-call, secondary on-call, incident commander, comms lead.
  - **Contact routes**:
    - PagerDuty/ops tool, Slack channels, phone/SMS escalation.
  - **Severities & response times**:
    - SEV-1: P0 global outage, 24/7 paging, 5-min ack.
    - SEV-2: Major partial outage, 15-min ack, business hours priority.
  - **Escalation paths**:
    - When primary doesn’t respond → auto page secondary after N minutes.
    - When engineering blocked → involve vendor support or infra leadership.
  - **Handovers**:
    - Shift-change procedures; status handoff templates.

#### 7.3 Mentoring & communication

- **Mentoring junior engineers**
  - Pair them on **incident retros** so they see reasoning, not just commands.
  - Encourage them to write or update **runbooks** after incidents; review together.
  - Use **post-incident “learning sessions”** to walk through timelines and tools.
  - Give them contained ownership: e.g., “You own logging improvements for service X.”

- **Communicating incidents**

  - **To executives / non-technical stakeholders**:
    - Focus on **impact, timeline, and mitigation**:
      - “From 10:02–10:18 UTC, ~30% of requests failed for EU customers due to a database capacity issue. We rolled back a configuration change and increased capacity. We’re now implementing safeguards so this config cannot be changed without review.”
    - Avoid deep technical jargon; use visuals where helpful.
  - **To engineering teams**:
    - Provide detailed **technical analysis**:
      - Metrics graphs, specific root cause (e.g., HPA + DB pool misconfiguration), exact fixes.
    - Link to runbooks, PRs, and follow-up work items.
    - Encourage feedback: what tooling or docs would have made this easier?
