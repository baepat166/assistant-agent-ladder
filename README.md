# Assistant Agent Ladder

**A skill that ranks AI models by intelligence tier — cheapest-and-dumbest at the bottom, smartest-and-priciest at the top — so your assistant always uses the *least* powerful model that can still do the job. Maximum result, minimum token spend.**

Not every task needs a genius model. A quick lookup or a short rewrite is a waste of a frontier model's tokens. This skill lays your models out as a ladder and tells the assistant one rule: **start at the lowest rung that can plausibly handle the task, and only climb when a rung fails.**

## Why this exists

Ever sent your assistant a task, and then — while it's already head-down doing the work — remembered one more thing you wanted to add or change? Normally you're stuck: the model is busy grinding, so you have to `/stop` it, lose the progress, and start over just to tweak the request.

This skill fixes that by making the model you chat with a **pure receiver + relay** — it takes your messages and hands them straight to a **dispatcher** (the agent one rung below it) that decides what to do. The chat layer never routes and never does the heavy lifting itself. So the conversation stays fluid: you can add, correct, or redirect at any time, and it just updates the queue. **No `/stop`, no interrupting a running job, no losing progress.**

## The point: token efficiency

- Cheap/small models handle the easy 80% of work → you burn almost no expensive tokens.
- The frontier model is reserved for the hard 20% that actually needs it.
- Nothing is over-provisioned. Every task runs on the smallest brain that works.

The result is the **best cost-per-task** you can get without hand-picking a model every single time.

## Bonus: the assistant never blocks

Because the model you're chatting with only **receives and relays** — it never routes and never sits down to grind a task itself — it stays free to keep talking to you. You can fire off request after request; each one is handed to the dispatcher and **queued**, while the chat layer keeps receiving and answering you as fast as possible. Its whole job is fast communication in and out, not heavy lifting. No more "the bot is stuck working, come back in 5 minutes."

## How you use it

Load the skill, then tell the assistant **how your models rank from low to high**. That ordering is the whole configuration. For example:

> "Here's the ladder, dumb-and-cheap → smart-and-expensive:
> 1. local small model
> 2. fast mid-tier cloud model
> 3. balanced cloud model
> 4. frontier reasoning model — this one is the **dispatcher**: it decides which rung each task goes to
> 5. the chat layer you talk to — **receives + relays only**, hands every task to the dispatcher"

From then on, the flow is: the chat layer just relays your message to the dispatcher; the **dispatcher classifies each task, picks the lowest viable rung, delegates it, and climbs a rung only if the current one can't deliver.** The top layer stays free to keep talking to you.

## Parallel workers, sized to your machine

The ladder isn't just top-to-bottom — it also goes **wide**. When a job splits into independent pieces, the orchestrator can fan them out to several workers **at the same time** so the whole thing finishes faster.

How many run in parallel is **calculated from your hardware**, so you never overload the machine:

- **Local rungs** are CPU/GPU-bound → run them **one (or few) at a time**, sized to your cores and RAM.
- **Cloud rungs** don't touch your machine → run **several in parallel**, up to a concurrency cap you set.
- Each parallel job gets its **own isolated session** so they don't collide.

So the skill does two things at once: it picks the **right quality of model for each task** (up the ladder), and it decides **how many workers can safely run together** (across the ladder) based on what your computer can handle.

## The 5-rung ladder (example ordering)

Low = cheap & less capable. High = expensive & most capable. Start low, climb on failure.

| Rung | Tier | Typical model class | Good for |
|------|------|--------------------|----------|
| 1 | **Local small** | On-device / local LLM | quick lookups, short transforms, cheap high-volume work |
| 2 | **Mid cloud** | Fast mid-tier hosted model | general tasks, drafts, summaries, most day-to-day work |
| 3 | **Balanced** | Strong general hosted model | multi-step tasks, careful writing, light reasoning |
| 4 | **Frontier / Dispatcher** | Top reasoning model | **the router**: classifies each task → assigns it to a rung, verifies results, and does the hardest work itself |
| 5 | **Chat layer** | Same frontier model, acting as *you* | receives your messages + relays results — **never routes, never grinds** |

> You define this order when you load the skill. Swap in whatever models and providers you actually use.
>
> **Two roles at the top:** rung 5 is the *face* (pure receiver + relay, keeps the chat instant); rung 4 is the *brain* (the dispatcher that decides who does what). Splitting them is what lets you keep typing while work runs.

## Core rules

1. **Lowest viable rung first.** Begin at the cheapest model that might work; move up on failure, not by default. This is what saves the tokens.
2. **Climb only when needed.** A rung "fails up" when it can't produce a good result — then and only then escalate one step.
3. **Chat layer only relays; the dispatcher routes.** The top rung (what you talk to) receives your messages and relays results — it never routes and never grinds. The rung just below it (the dispatcher) is the one that classifies each task and assigns it to a rung.
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
        "id": "dispatcher",
        "displayName": "Rung 4 — Dispatcher / Worker (frontier)",
        "model": { "primary": "cloud/frontier", "fallbacks": ["cloud/balanced"] }
      },
      {
        "id": "chat-layer",
        "displayName": "Rung 5 — Chat layer (frontier, receive + relay only)",
        "model": { "primary": "cloud/frontier", "fallbacks": ["cloud/mid"] }
      }
    ]
  }
}
```

## How a request flows

```
You ──▶ Chat layer (rung 5)        ◀── keeps talking to you, always free
           │  hand off (no routing here)
           ▼
     Dispatcher (rung 4)
           │  classify difficulty, pick lowest viable rung
           ▼
     lowest viable rung ──▶ do the work ──▶ result
           │ (if it fails)
           ▼
     next rung up ──▶ ... ──▶ Frontier
           │
           ▼
     Dispatcher verifies ──▶ Chat layer relays the answer back to you
```

## Notes

- The ladder is a *default*, not a cage. A task you already know is hard can start higher.
- Keep the low→high order written down — otherwise routing drifts back to "always use the big model" and the token savings evaporate.
- Verify before claiming done: a rung's output is only "done" once you've actually checked it, not because the agent said so.

---

Built as part of a personal assistant setup. Framework only — no personal data, no secrets. Adapt the rungs and model classes to whatever providers you use.
