# Assistant Agent Ladder

**A skill that ranks AI models by intelligence tier — cheapest-and-dumbest at the bottom, smartest-and-priciest at the top — so your assistant always uses the *least* powerful model that can still do the job. Maximum result, minimum token spend.**

Not every task needs a genius model. A quick lookup or a short rewrite is a waste of a frontier model's tokens. This skill lays your models out as a ladder and tells the assistant one rule: **start at the lowest rung that can plausibly handle the task, and only climb when a rung fails.**

## The point: token efficiency

- Cheap/small models handle the easy 80% of work → you burn almost no expensive tokens.
- The frontier model is reserved for the hard 20% that actually needs it.
- Nothing is over-provisioned. Every task runs on the smallest brain that works.

The result is the **best cost-per-task** you can get without hand-picking a model every single time.

## How you use it

Load the skill, then tell the assistant **how your models rank from low to high**. That ordering is the whole configuration. For example:

> "Here's the ladder, dumb-and-cheap → smart-and-expensive:
> 1. local small model
> 2. fast mid-tier cloud model
> 3. balanced cloud model
> 4. frontier reasoning model
> 5. me (the orchestrator) — you only route, you don't grind"

From then on the assistant self-routes: it classifies each task, picks the lowest viable rung, delegates, and climbs a rung only if the current one can't deliver.

## The 5-rung ladder (example ordering)

Low = cheap & less capable. High = expensive & most capable. Start low, climb on failure.

| Rung | Tier | Typical model class | Good for |
|------|------|--------------------|----------|
| 1 | **Local small** | On-device / local LLM | quick lookups, short transforms, cheap high-volume work |
| 2 | **Mid cloud** | Fast mid-tier hosted model | general tasks, drafts, summaries, most day-to-day work |
| 3 | **Balanced** | Strong general hosted model | multi-step tasks, careful writing, light reasoning |
| 4 | **Frontier** | Top reasoning model | hard reasoning, tricky debugging, high-stakes output |
| 5 | **Orchestrator** | Same frontier model, acting as *you* | plans, routes, verifies — does **not** grind the work itself |

> You define this order when you load the skill. Swap in whatever models and providers you actually use.

## Core rules

1. **Lowest viable rung first.** Begin at the cheapest model that might work; move up on failure, not by default. This is what saves the tokens.
2. **Climb only when needed.** A rung "fails up" when it can't produce a good result — then and only then escalate one step.
3. **Orchestrator never grinds.** The top rung delegates and relays. It only does trivial things itself (short replies, reading a file to decide who to assign).
4. **One session key per parallel job.** Cloud workers can run several jobs in parallel; each needs an isolated session so they don't collide.
5. **Local workers run one-at-a-time.** On-device models are CPU/GPU bound — serialize them, keep them for light/short work.
6. **Long jobs go to the background.** Fire the worker detached, poll its log periodically, keep the main chat responsive.

## Example config shape (generic)

Each rung is just an agent definition with a `primary` model and `fallbacks`, listed low → high. Names and providers are placeholders — swap in your own.

```jsonc
{
  "agents": {
    "list": [
      {
        "id": "worker-local",
        "displayName": "Rung 1 — Worker (local small)",
        "model": { "primary": "local/small-model", "fallbacks": ["cloud/fast-mid"] }
      },
      {
        "id": "worker-mid",
        "displayName": "Rung 2 — Worker (mid cloud)",
        "model": { "primary": "cloud/fast-mid", "fallbacks": ["cloud/mid"] }
      },
      {
        "id": "worker-balanced",
        "displayName": "Rung 3 — Worker (balanced)",
        "model": { "primary": "cloud/balanced", "fallbacks": ["cloud/mid"] }
      },
      {
        "id": "worker-frontier",
        "displayName": "Rung 4 — Worker (frontier)",
        "model": { "primary": "cloud/frontier", "fallbacks": ["cloud/balanced"] }
      },
      {
        "id": "orchestrator",
        "displayName": "Rung 5 — Orchestrator (frontier)",
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
- Keep the low→high order written down — otherwise routing drifts back to "always use the big model" and the token savings evaporate.
- Verify before claiming done: a rung's output is only "done" once you've actually checked it, not because the agent said so.

---

Built as part of a personal assistant setup. Framework only — no personal data, no secrets. Adapt the rungs and model classes to whatever providers you use.
