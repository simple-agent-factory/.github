# Simple Agent Factory

> An **AI-driven, self-healing test-automation factory** for Java SOAP services.
> Three Claude agents detect bugs in production, fix the code, ship the patch, and close the ticket, without a human in the loop.

---

## demo recordings

* QA Tester agent: ![demo_agent_qa.mp4](./docs/demo_agent_qa.mp4?raw=true)
* Software Developer agent: ![demo_agent_dev.mp4](./docs/demo_agent_dev.mp4?raw=true)

## The pitch in one sentence

A faulty SOAP operation is detected against its written spec, a Jira bug is filed, a developer agent clones the repo, edits the Java source with `Math.*Exact` safe arithmetic, pushes to `main`, GitHub Actions deploys the fat JAR to a DigitalOcean droplet behind Nginx + Let's Encrypt, and a release agent verifies the deployment and closes the ticket, end to end, on its own.

---

## Runtime architecture

```mermaid
flowchart LR
    subgraph Orchestrator["Orchestrator (Python)"]
        ORCH["orchestrator.py<br/>--agent qa | dev | release"]
    end

    subgraph Agents["Claude Opus agents"]
        QA["Agent 1<br/>QA Engineer"]
        DEV["Agent 2<br/>Software Developer"]
        REL["Agent 3<br/>Release Engineer"]
    end

    subgraph Ticketing["Ticketing & state (blackboard)"]
        JIRA[("Jira Cloud<br/>project SSI<br/>[AID]-prefixed bugs")]
    end

    subgraph SCM["Source & CI/CD"]
        GH[("GitHub repo<br/>01_soap_arithmetics")]
        GHA["GitHub Actions<br/>build · deploy · health-check"]
    end

    subgraph Prod["Production (DigitalOcean droplet)"]
        NGINX["Nginx + Let's Encrypt<br/>TLS on :443"]
        JAR["systemd unit: arithmetics<br/>Jakarta XML WS on :8081"]
        WSDL>"WSDL @ ai-maxxing.cc/arithmetics?wsdl"]
    end

    ORCH --> QA
    ORCH --> DEV
    ORCH --> REL

    QA -- "introspects WSDL<br/>+ runs SOAP tests" --> WSDL
    QA -- "create Bug<br/>(To Do)" --> JIRA

    JIRA -- "next To Do bug" --> DEV
    DEV -- "git clone / push" --> GH
    DEV -- "transition<br/>In Progress" --> JIRA

    GH -- "push to main" --> GHA
    GHA -- "scp + systemctl restart" --> JAR
    JAR --> NGINX
    NGINX --> WSDL

    GHA -- "run status" --> ORCH
    JIRA -- "In Review bugs" --> REL
    GHA -- "conclusion: success" --> REL
    REL -- "comment + Done" --> JIRA
```

---

## The bug lifecycle, end to end

```mermaid
sequenceDiagram
    autonumber
    participant Spec as arithmetics_specifications.md
    participant QA as Agent 1, QA
    participant SOAP as ai-maxxing.cc (SOAP)
    participant Jira
    participant Dev as Agent 2, Dev
    participant GH as GitHub
    participant GHA as GitHub Actions
    participant Droplet as DigitalOcean droplet
    participant Rel as Agent 3, Release

    QA->>Spec: read expected behaviour
    QA->>SOAP: fetch WSDL + run test cases
    SOAP-->>QA: multiply(2,3) = 5   (wrong!)
    QA->>Jira: create Bug [AID] (To Do)

    loop until no To Do bugs remain
        Dev->>Jira: fetch next [AID] To Do
        Dev->>Jira: transition → In Progress
        Dev->>GH: clone repo
        Dev->>Dev: locate method, apply Math.multiplyExact
        Dev->>GH: commit + push to main
    end

    GH->>GHA: trigger build + deploy
    GHA->>Droplet: scp JAR → systemctl restart
    Droplet-->>GHA: /health 200 OK

    Rel->>GHA: poll latest run on main
    GHA-->>Rel: conclusion = success
    Rel->>Jira: comment + transition → Done
```

---

## Jira workflow, the agents' shared memory

The three agents never talk to each other directly. They coordinate via **Jira issue status**, which acts as a durable blackboard.

```mermaid
stateDiagram-v2
    [*] --> ToDo : Agent 1 (QA) creates Bug
    ToDo --> InProgress : Agent 2 (Dev) claims it
    InProgress --> InReview : Agent 2 pushes fix
    InReview --> Done : Agent 3 (Release) verifies CI/CD
    InReview --> InReview : deploy failed → comment, keep open
```

Every issue carries the `[AID]` prefix in its summary, a soft access control enforced in agent code so the shared `SSI` project stays uncontaminated by other use cases.

---

## Repositories in this org

| # | Repo | Role | Stack |
|---|------|------|-------|
| 0 | [`0_orchestration`](../../0_orchestration) | Sequencing the full pipeline (`qa → dev-loop → CI/CD poll → release`) | Python 3.11 |
| 1 | [`01_soap_arithmetics`](../../01_soap_arithmetics) | The target service under test, Java SOAP arithmetic with intentional bugs | Java 21, Jakarta XML WS 4.0, Maven, Ansible |
| 2 | [`02_agent1_qa`](../../02_agent1_qa) | QA Engineer, WSDL introspection + dynamic SOAP testing + Jira reporting | Python, Zeep, Anthropic SDK |
| 3 | [`03_agent2_dev`](../../03_agent2_dev) | Software Developer, clones repo, locates the bug, patches the Java, pushes | Python, GitPython, Anthropic SDK |
| 4 | [`04_agent3_release`](../../04_agent3_release) | Release Engineer, verifies the GitHub Actions run, closes the ticket | Python, GitHub REST API, Anthropic SDK |

---

## What each agent can do (tool surface)

```mermaid
flowchart TB
    subgraph QA["Agent 1, QA Engineer"]
        QA1["fetch_wsdl_operations"]
        QA2["run_soap_test"]
        QA3["create_jira_bug"]
    end

    subgraph DEV["Agent 2, Software Developer"]
        D1["transition_jira_issue"]
        D2["list_repo_files"]
        D3["read_repo_file"]
        D4["write_repo_file"]
        D5["commit_and_push"]
    end

    subgraph REL["Agent 3, Release Engineer"]
        R1["get_latest_github_run"]
        R2["get_github_run_jobs"]
        R3["add_jira_comment"]
        R4["transition_jira_issue"]
    end
```

Each agent runs a standard Claude tool-use loop (`claude-opus-4-6`), the LLM picks tools, the Python harness executes them, results are fed back until the model emits `end_turn`.

---

## The intentional defects (demo scenarios)

The deployed service ships with three planted bugs so the agents always have something to find:

| Operation | Bug | Symptom | Expected fix |
|---|---|---|---|
| `subtract(a, b)` | Computes `a - 2b` | `subtract(10, 3) → 4` | `Math.subtractExact(a, b)` |
| `multiply(a, b)` | Uses `Math.addExact` | `multiply(2, 3) → 5` | `Math.multiplyExact(a, b)` |
| `divide(a, b)` | No zero-check | `divide(1, 0) → raw exception` | guard `if (b == 0)` + SOAP Fault |

---

## CI/CD pipeline

```mermaid
flowchart LR
    PUSH[push to main] --> BUILD
    subgraph BUILD["build job"]
        B1[setup-java 21 Temurin] --> B2[mvn -B package]
        B2 --> B3[upload soap-arithmetics-1.0.0.jar artifact]
    end
    BUILD --> DEPLOY
    subgraph DEPLOY["deploy job (main only)"]
        D1[download artifact] --> D2[write DROPLET_SSH_KEY]
        D2 --> D3[scp JAR to droplet]
        D3 --> D4[systemctl restart arithmetics]
        D4 --> D5{curl /health 200?}
    end
    D5 -- yes --> OK[deploy success]
    D5 -- no --> FAIL[workflow fails<br/>Agent 3 leaves ticket In Review]
```

---

## Infrastructure stack

| Layer | Technology | Notes |
|---|---|---|
| **Reasoning** | Claude Opus (Anthropic API) | Tool-use loop per agent, system prompt cached |
| **Ticketing** | Jira Cloud REST API v3 | Project `SSI`, all AID issues prefixed `[AID]` |
| **VCS** | GitHub + dedicated machine account | Push via SSH from Agent 2 |
| **CI** | GitHub Actions | `build` (every push/PR) + `deploy` (main only) |
| **Runtime** | OpenJDK 21 + Jakarta XML WS 4.0 | Fat JAR, `Math.*Exact` for overflow-safe arithmetic |
| **Hosting** | DigitalOcean droplet (Ubuntu) | systemd unit `arithmetics` on `:8081` |
| **Edge** | Nginx + Let's Encrypt (Certbot) | TLS on `:443`, auto-renewing via systemd timer |
| **Provisioning** | Ansible (`ansible/bootstrap.yml`) | Two-phase Nginx config to bootstrap the cert |
| **Domain** | `ai-maxxing.cc` | A record → droplet IP |

---

## Security posture (MVP-honest)

- **Ticket scoping**, every Jira create / read / update is gated on the `[AID]` prefix, enforced in agent code. Cross-contamination with non-AID tickets in the shared `SSI` project is rejected at the tool call.
- **Credential scoping**, each agent gets only what it needs: QA has Jira write + WSDL read, Dev has GitHub push via a dedicated machine account's SSH key, Release has read-only Jira + read-only GitHub Actions PAT.
- **Audit trail**, every action is recorded in Jira (issue history + comments) and GitHub (commit log + Actions runs). Nothing is lost.
- **MVP caveats**, no human-in-the-loop, no automatic loopback if a fix regresses, single-pass logic. Documented intentionally.

---

## Running the full pipeline

```bash
# one shot, all four stages
python 0_orchestration/orchestrator.py

# or just one stage, for debugging
python 0_orchestration/orchestrator.py --agent qa
python 0_orchestration/orchestrator.py --agent dev
python 0_orchestration/orchestrator.py --agent release
```

Each agent reads its own `.env` (see `<agent>/.env.example`) and requires `ANTHROPIC_API_KEY`. The orchestrator chains them and polls GitHub Actions between Stage 2 and Stage 4.

---

## Why this matters

Classical test automation is brittle: a renamed field, a tweaked operator, a missing exception type, and the whole script breaks. This factory replaces that brittleness with **agentic flexibility**: each agent reads natural-language specs, introspects live contracts, reasons over real code, and uses the same enterprise tools (Jira, Git, CI) a human team would. The result is a closed remediation loop where the only artifact a human ever needs to read is the final, auto-closed Jira ticket.
