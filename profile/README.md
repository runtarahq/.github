<p align="center">
  <h1 align="center">Runtara</h1>
  <p align="center">
    <strong>Durable Execution for Long-Running Workflows</strong>
  </p>
  <p align="center">
    Build workflows that survive crashes, restarts, and deployments.<br/>
    No more lost progress. No more manual recovery. Just reliable execution.
  </p>
</p>

<p align="center">
  <a href="https://www.gnu.org/licenses/agpl-3.0"><img src="https://img.shields.io/badge/License-AGPL%20v3-blue.svg" alt="License: AGPL v3"></a>
  <a href="https://www.rust-lang.org"><img src="https://img.shields.io/badge/rust-1.75+-orange.svg" alt="Rust"></a>
</p>

---

> **Beta Software**: Under active development. APIs may change.

## The Problem

Modern applications run workflows that take minutes, hours, or even days:

- **Order processing** — validate payment, reserve inventory, notify shipping
- **Data pipelines** — fetch, transform, load across multiple systems
- **Onboarding flows** — send welcome email, wait 3 days, send follow-up
- **Batch operations** — process 10,000 records with external API calls

**What happens when these workflows crash?**

With traditional approaches, you lose progress. You restart from the beginning. You process duplicates. You build complex recovery logic. You wake up at 3 AM because a 6-hour job failed at hour 5.

## The Solution

Runtara is a **durable execution engine**. Your workflows automatically checkpoint their progress to a database. When they crash, they resume exactly where they left off.

```rust
for (i, order) in orders.iter().enumerate() {
    // This checkpoint is the magic ✨
    // First run: saves state to database
    // After crash: returns saved state, skips to next iteration
    let result = sdk.checkpoint(&format!("order-{}", i), &state).await?;

    if result.existing_state().is_some() {
        continue; // Already processed, skip
    }

    process_order(order).await?;  // Only runs once per order, ever
}
```

**Crash at order 500 of 1000?** Restart, and processing continues from order 501.

## Key Ideas

### 1. Durability Without Complexity

You don't need message queues, dead letter queues, retry frameworks, or state machines. Just `checkpoint()` your progress and let Runtara handle the rest.

### 2. Sleep That Survives Restarts

Need to wait 3 days before sending a follow-up email? With Runtara, the process exits, and the platform wakes it up when it's time:

```rust
sdk.durable_sleep(Duration::from_days(3)).await?;
send_followup_email().await?;  // Runs 3 days later, guaranteed
```

No cron jobs. No external schedulers. The sleep is part of your workflow.

### 3. External Control

Pause a workflow while investigating an issue. Resume it when ready. Cancel if needed. All without losing state:

```rust
// In your admin dashboard
management_sdk.pause_instance("order-workflow-123").await?;

// Later, when ready
management_sdk.resume_instance("order-workflow-123").await?;
```

### 4. Native Performance

Workflows compile to native Rust binaries. They run in isolated OCI containers with full resource control. This isn't an interpreted workflow engine — it's compiled code with durability built in.

## Who Is This For?

**Platform Teams** building internal workflow infrastructure

**SaaS Products** that need reliable background job processing

**Integration Platforms** connecting multiple external systems

**Anyone** tired of building custom recovery logic for long-running processes

## Architecture at a Glance

```
Your Product (UI, API, Business Logic)
           │
           ▼
┌─────────────────────────────────────────────┐
│              RUNTARA PLATFORM               │
│                                             │
│  Management SDK ──▶ runtara-core            │
│    (start/stop)     (checkpoints, signals)  │
│                            │                │
│  runtara-environment ◀─────┘                │
│    (OCI runner)      │                      │
│                      ▼                      │
│             Workflow Instances              │
│           (compiled Rust binaries)          │
└─────────────────────────────────────────────┘
           │
           ▼
      PostgreSQL
   (durable state)
```

## Quick Start

```bash
# 1. Clone and build
git clone https://github.com/runtara/runtara.git
cd runtara && cargo build

# 2. Start the server
export RUNTARA_DATABASE_URL=sqlite://.data/runtara.db
cargo run -p runtara-core

# 3. Run an example (in another terminal)
cargo run -p durable-example --bin checkpoint_example
```

## Core Concepts

| Concept | What It Does |
|---------|-------------|
| **Checkpoint** | Saves workflow state. On crash, resume from here. |
| **Durable Sleep** | Workflow exits, platform wakes it at the right time. |
| **Signals** | Pause, resume, or cancel workflows externally. |
| **Instances** | Individual workflow executions with unique IDs. |
| **Tenants** | Isolation between different customers/environments. |

## Example: Resilient Order Processing

```rust
use runtara_sdk::RuntaraSdk;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut sdk = RuntaraSdk::localhost("order-batch-001", "tenant-1")?;
    sdk.connect().await?;
    sdk.register(None).await?;

    let orders = fetch_pending_orders().await?;

    for (i, order) in orders.iter().enumerate() {
        let state = serde_json::to_vec(&ProcessingState { index: i })?;
        let result = sdk.checkpoint(&format!("order-{}", order.id), &state).await?;

        // Handle external signals
        if result.should_cancel() { return Err("Cancelled".into()); }
        if result.should_pause() {
            sdk.suspended().await?;
            return Ok(());
        }

        // Skip if already processed (resuming after crash)
        if result.existing_state().is_some() { continue; }

        // This code only runs once per order
        validate_payment(&order).await?;
        reserve_inventory(&order).await?;
        notify_shipping(&order).await?;
    }

    sdk.completed(b"All orders processed").await?;
    Ok(())
}
```

## Workflow DSL (Optional)

Define workflows as JSON for no-code configuration:

```json
{
  "name": "Order Processing",
  "steps": {
    "validate": {
      "stepType": "Agent",
      "agentId": "http",
      "capabilityId": "request",
      "inputMapping": {
        "url": { "valueType": "immediate", "value": "https://api.example.com/validate" }
      }
    },
    "notify": { ... },
    "finish": { ... }
  },
  "executionPlan": [
    { "fromStep": "validate", "toStep": "notify" },
    { "fromStep": "notify", "toStep": "finish" }
  ]
}
```

JSON workflows compile to the same native binaries as hand-written Rust code.

## Built-in Integrations

| Agent | Use Case |
|-------|----------|
| **HTTP** | REST APIs with auth (Bearer, API Key, Basic) |
| **SFTP** | File transfers |
| **CSV/XML** | Data parsing and generation |
| **Transform** | Field mapping, filtering, merging |

Need something custom? Build your own agents with the `#[agent]` macro.

## Project Structure

```
runtara/
├── runtara-core          # Execution engine (checkpoints, signals, scheduling)
├── runtara-environment   # OCI container runner, image registry
├── runtara-sdk           # Client library for workflows
├── runtara-workflows     # JSON DSL compiler
├── runtara-agents        # Built-in HTTP, SFTP, CSV agents
└── durable-example       # Example workflows
```

## Development

```bash
cargo build              # Build all crates
cargo test               # Run tests
cargo run -p runtara-core        # Run execution engine
cargo run -p runtara-environment # Run container environment
```

See individual crate READMEs for detailed documentation.

## Status

Runtara is in **beta**. We're using it internally at [SyncMyOrders](https://syncmyorders.com) for production workflow automation, but the public API is still evolving.

What's stable:
- Core checkpoint/resume semantics
- Durable sleep
- Signal handling (pause/resume/cancel)

What's evolving:
- DSL syntax and agent interfaces
- Management API
- Multi-region deployment

## Contributing

Contributions welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

All contributors must sign our [Contributor License Agreement](CLA.md).

## License

[GNU Affero General Public License v3.0 (AGPL-3.0)](LICENSE)

For commercial licensing: hello@syncmyorders.com

---

<p align="center">
  <strong>Built with ❤️ by <a href="https://syncmyorders.com">SyncMyOrders</a></strong><br/>
  Copyright © 2025 SyncMyOrders Sp. z o.o.
</p>
