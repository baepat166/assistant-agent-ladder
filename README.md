# Assistant Agent Ladder

A simple pattern for running a **personal AI assistant as a team instead of a single model** — with a clear escalation ladder from cheap-and-local up to frontier-and-expensive.

The idea: the assistant you talk to is an **orchestrator**, not the one grinding the work. It receives your request, picks the *lowest* rung on the ladder that can plausibly handle the job, delegates, and relays the result. If a rung can't do it, the work is pushed one step up.

## Why

- **Cost & speed** — most tasks don't need a frontier model. Route the easy 80% to small/cheap workers and save the big model for the hard 20%.
- **Responsiveness** — the orchestrator stays free to keep talking to you while workers grind in the background.
- **Predictability** — "which model runs this?" stops being a coin flip and becomes a rule.

## The 5-rung ladder

Start at the lowest rung likely to succeed. Escalate only when a rung fails.

| Rung | Role | Typical model class | Good for |
|------|------|--------------------|----------|
| 1 | **Local small** | On-device / local LLM | quick lookups, short transforms, cheap high-volume work |
| 2 | **Mid cloud** | Fast mid-tier hosted model | general tasks, drafts, summaries, most day-to-day work |
| 3 | **Balanced** | Strong general hosted model | multi-step tasks, careful writing, light reasoning |
| 4 | **Frontier** | Top reasoning model | hard reasoning, tricky debugging, high-stakes output |
| 5 | **Orchestrator** | Same frontier model, *you* | plans, routes, verifies — does **not** grind the work itself |

> Rung 5 is the conductor, not a player. Its job is to decide *who* plays.

## Core rules

1. **Orchestrator never grinds.** The top agent delegates and relays. It only does trivial things itself (short replies, reading a file to decide who to assign).
2. **Climb only when needed.** Begin at the cheapest rung that might work; move up on failure, not by default.
3. **One session key per parallel job.** Cloud workers can run several jobs in parallel; each needs an isolated session so they don't collide.
4. **Local workers run one-at-a-time.** On-device models are CPU/GPU bound — serialize them, keep them for light/short work.
5. **Long jobs go to the background.** Fire the worker detached, poll its log periodically, keep the main chat responsive.

## Example config shape (generic)

Each rung is just an agent definition with a `primary` model and `fallbacks`. Names and providers are placeholders — swap in your own.

```jsonc
{
  "agents": {
    "list": [
      {
        "id": "worker-local",
        "displayName": "Worker (local small)",
        "model": { "primary": "local/small-model", "fallbacks": ["cloud/fast-mid"] }
      },
      {
        "id": "worker-mid",
        "displayName": "Worker (mid cloud)",
        "model": { "primary": "cloud/fast-mid", "fallbacks": ["cloud/mid"] }
      },
      {
        "id": "worker-balanced",
        "displayName": "Worker (balanced)",
        "model": { "primary": "cloud/balanced", "fallbacks": ["cloud/mid"] }
      },
      {
        "id": "worker-frontier",
        "displayName": "Worker (frontier)",
        "model": { "primary": "cloud/frontier", "fallbacks": ["cloud/balanced"] }
      },
      {
        "id": "orchestrator",
        "displayName": "Orchestrator (frontier)",
        "model": { "primary": "cloud/frontier", "fallbacks": ["cloud/mid"] }
      }
    ]
  }
}
```

## How a request flows

```
You ──▶ Orchestrator
           │  classify difficulty
           ▼
     lowest viable rung ──▶ do the work ──▶ result
           │ (if it fails)
           ▼
     next rung up ──▶ ... ──▶ Frontier
           │
           ▼
     Orchestrator relays the answer back to you
```

## Notes

- The ladder is a *default*, not a cage. A task you already know is hard can start higher.
- Keep a written rule for what each rung owns — otherwise routing drifts back to "always use the big model."
- Verify before claiming done: a rung's output is only "done" once you've actually checked it, not because the agent said so.

---

Built as part of a personal assistant setup. Framework only — no personal data, no secrets. Adapt the rungs and model classes to whatever providers you use.
