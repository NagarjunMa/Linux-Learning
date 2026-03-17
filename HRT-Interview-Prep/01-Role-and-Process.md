# HRT: The Role and Interview Process

## What HRT Is

Hudson River Trading (HRT) is a high-frequency trading firm. The **SysTrade team** manages and optimizes HRT's live trading infrastructure — the servers, networks, and systems that execute trades in real time.

## The Two Engineering Divisions

| Division | Focus | Primary Challenge | Languages |
|----------|-------|------------------|-----------|
| **Trading Tech** | Live trading: exchange data, orders, FPGA/ASIC | **LATENCY** | ~70% C++, 30% Python |
| **Research & Dev** | Storage, data centers, GPU clusters, ETL | **SCALE** | ~70% Python, 30% C++ |

The Systems Software Engineer and Linux Engineer roles are **Platform roles in Trading Tech**.

## What the Role Actually Is

> HRT explicitly labels it "Systems Automation / Python" — they expect you to WRITE SOFTWARE that manages systems, not just run commands.

Responsibilities:
- Machine provisioning, monitoring, metrics, logging, configuration management
- Automation workflows in Python
- Installing/configuring hardware, remote administration, performance monitoring

## The 7 Reasons Candidates Fail

| Failure Mode | What HRT Cares About |
|-------------|---------------------|
| Shallow fundamentals | Using tools without understanding internals — they will drill vtables, Python list memory growth, kernel internals |
| Communication issues | HRT rates communication AND technical ability equally |
| Not listening / diving in too fast | Take time to repeat back the problem before solving |
| Avoiding real work | EVERYONE codes at HRT, including CTOs |
| Arrogance | "If you act like you already know everything, you're gone" |
| Indifference | Show you care about quality, not just getting the task done |
| Cheating | LLMs detected in early rounds → immediate rejection + permanent ban |

## The Complete Interview Process

### Round 0: Online Assessment (OA)
- Platform: CodeSignal or HackerRank
- 3–4 problems, medium to medium-hard LeetCode
- Completion speed matters, not just correctness
- For systems roles: may include Python scripting, not just algorithms
- **Budget: ~112 days total process** (vs 18 days for SWE)

### Round 1: Technical Phone Screen (30–45 min)
- Direct Linux command questions: "How do you find the highest CPU process?"
- May include short LeetCode medium
- CS fundamentals: OS concepts, memory, data structures
- **This is the round you're preparing for**

### Round 2: Technical Phone Screen #2 (45–60 min CoderPad)
- Deeper algorithms and CS fundamentals
- Python scripting/debugging live on CoderPad
- **SSH Debugging Round** — given access to a broken HRT cloud server to diagnose live

### Round 3+: Onsite (Full Day)
| Interview | What to Expect |
|-----------|---------------|
| 2hr Coding Session | Full mini-project in Python. Multi-file, real tooling. Not single LeetCode |
| 1hr System Design × 2 | Distributed systems, monitoring platforms, trading infrastructure |
| CS Fundamentals | OS internals, process management, memory, networking deep-dive |
| Team Fit / Culture | Curiosity, passion, listening, asking good questions |
| Resume Deep-Dive | Everything listed is fair game — expect internal workings questions |

## What HRT Explicitly Measures

1. **Deep fundamentals** — not tricks. True intuition: why was this tool invented?
2. **Systems-level knowledge** — memory, I/O, process management, virtual memory
3. **Collaboration** — do you take hints well? Are you open to different approaches?
4. **Teachability** — did you apply something the interviewer showed you earlier?
5. **Communication** — can you explain your work to a non-expert?
6. **Problem-solving methodology** — big picture first, map out steps, track progress
