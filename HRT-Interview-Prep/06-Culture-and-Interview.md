# HRT Culture, Phone Screen Strategy, and Questions to Ask

## The 45-Minute Call: What to Expect

| Time | What Happens |
|------|-------------|
| 0–5 min | Intro / small talk. Walk through background — 90-second version |
| 5–20 min | Linux commands & troubleshooting. Direct, specific, command-level |
| 20–35 min | Resume deep-dive. Anything listed is fair game — they probe depth, not breadth |
| 35–42 min | Short scenario / problem. Broken system description → how would you fix it? |
| 42–45 min | Your questions for them. **This is evaluated too** |

## Call Mindset

**Think out loud always:** "First I'd check X because... then if that's clear I'd look at Y..." Silence = red flag.

**Methodology beats memorization:** If you don't know the exact command, describe the approach. They care how you think.

**Never fake it:** "I haven't used that specific tool, but here's how I'd approach it..." — HRT respects intellectual honesty.

**Specificity signals depth:** Don't say "I'd use monitoring tools." Say "I'd run `vmstat 1 5` to check if it's CPU-bound vs I/O-bound, then `perf top` to find the hot function."

**Show excitement:** HRT explicitly scores for passion. "This is exactly the kind of environment I want to work in."

## What Gets You Rejected

| Red Flag | Why |
|----------|-----|
| Diving in without restating the problem | "Candidates who don't understand the problem before diving in" |
| Shallow tool knowledge | "I use Ansible" → they'll ask: How does the connection mechanism work? What happens when a task fails? |
| Silence while coding | Signals poor collaboration instincts |
| Not showing excitement | "Someone that will treat this job like any other job should just get any other job" |
| Overclaiming on resume | If you put it on there, they will drill until they find your limit |
| Ignoring hints | If an interviewer suggests a direction, explore it |

## Questions to Ask (Pick 3–4)

### Show You Know the Team
- "The SysTrade team description says you 'optimize, manage and design' low-latency trading systems — where is the current focus? More active design work, or optimization of existing systems?"
- "What does 'low-latency' mean in your context — microseconds, nanoseconds? What's the order of magnitude you're working to improve?"
- "How does the SysTrade team interact with trading teams day-to-day — embedded or shared platform team?"

### Technical Depth
- "What's the primary Linux distribution you run on trading servers? How standardized is the stack?"
- "When a latency spike is detected in a live system, what does the incident workflow look like? Who gets paged first?"
- "How much of infrastructure automation is in Python versus shell? Do you use Salt, Ansible, or something internal?"
- "Is kernel bypass networking (DPDK, Solarflare) something this team manages?"

### Culture & Growth
- "What does the first 30–60 days look like on this team? What would I own early on?"
- "What's the on-call situation — how frequent are pages, and how are incidents handled?"
- "What does career growth look like — does HRT encourage rotation between systems and trading tech?"

## Your 90-Second Intro Template

```
"I'm a software engineer with [X] years of experience, strong background in Java and backend systems.
Most recently I've been working on [Y], where I managed [Z — scale metrics].
I've spent time at the intersection of application performance and Linux infrastructure —
profiling JVM latency, tuning kernel parameters, writing automation tooling in Python.

What draws me specifically to the SysTrade role is the combination of low-latency systems
engineering and live trading infrastructure. I want to work on systems where performance
is measured in microseconds and where infrastructure decisions directly affect the business."
```

## Answer Templates

### "What are you looking for?"
> "I'm looking for an environment where systems performance is a first-class concern — not a nice-to-have. I want to work with engineers who think deeply about how the kernel works, how networking works at the hardware level, and where every millisecond of optimization actually matters. HRT is exactly that. I'm also drawn to the culture where the CTO still writes production code — that's rare."

### "What's your weakness?"
> "Honestly, I'm less experienced with kernel-bypass networking — DPDK, Solarflare OpenOnload — I understand the concepts well and I've tuned standard Linux networking extensively, but I haven't operated kernel-bypass stacks in production. That said, it's one of the things I'm most excited to learn in this role."

## Night-Before Checklist

- [ ] Review command reference (file 02) — know the commands and reasoning
- [ ] Review crisis scenarios (file 03) — practice at least 3 out loud
- [ ] Review low-latency Q&A (file 04) — at least 2 answers ready
- [ ] Practice your 90-second intro out loud
- [ ] Write down 3 questions on a notepad
- [ ] Sleep — a tired brain doesn't debug systems well
