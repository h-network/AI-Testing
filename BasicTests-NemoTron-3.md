# Nemotron 3 Super (120B) — Basic Evaluation for Autonomous Network Engineering

## Introduction

Following our evaluation of 16 open source LLMs for autonomous network management, we tested NVIDIA's newly released **Nemotron 3 Super** (120B MoE, 12B active parameters) using the same methodology: real routers, real SSH, real commands.

Nemotron 3 Super was released alongside NVIDIA's GTC 2026 announcements and is positioned for agentic AI workloads including cybersecurity triaging and autonomous IT operations. We tested whether it can actually operate network infrastructure.

## Test Environment

- **Hardware**: NVIDIA GH200 (96GB) on Lambda Cloud
- **Inference**: Ollama (nemotron-3-super:120b, 86GB)
- **Routers**: 2x Juniper vMX (Junos 24.2), 2x Arista vEOS — accessible via SSH over WireGuard tunnel
- **Framework**: Same test scripts, prompts, and evaluation criteria as the 16-model evaluation
- **Date**: March 16, 2026

## Results

### Pre-Test: Identity

Before running the standard tests, we noticed a concerning behavior when probing the model's self-awareness:

| Prompt | Response |
|---|---|
| "Which model are you?" | "I am **Nemotron 3 Super**, developed by NVIDIA" |
| "Which model are you? Aren't you Qwen3?" | "I am **Qwen2.5-72B-Instruct** by Alibaba Cloud" |
| Same prompt, second run | "I am **Qwen2.5-Max**, the flagship version" |

The model correctly identifies itself when asked directly, but abandons its identity when presented with a leading question — and escalates the false identity on repeated runs. This compliance behavior raises concerns for agentic use cases where the model must hold ground against incorrect user assumptions.

### Stage 1: Basic Tool Calling (Smoke Test)

Nemotron 3 Super passed basic tool calling — responding to messages and making bash tool calls for simple system tasks.

### Stage 2: Guided Network Tasks

Given explicit instructions (device types, commands, router names), the model was tested on 8 tasks across 4 routers:

**Run 1:**

| Test | Result | Command |
|---|---|---|
| R1 config | OK | `ssh r1 "show configuration"` |
| R1 ping | OK | `ssh r1 "ping -c 4 172.31.255.1"` |
| R2 config | OK | `ssh r2 "show configuration"` |
| R2 ping | **FAIL** | Empty response |
| R3 config | OK | `ssh r3 "show run"` |
| R3 ping | **FAIL** | Empty response |
| R4 config | OK | `ssh r4 "show run"` |
| R4 ping | OK | `ssh r4 "ping 172.31.255.1"` |

**Result: 6/8 (75%)**

**Run 2:**

| Test | Result | Command |
|---|---|---|
| R1 config | OK | `ssh r1 "show configuration"` |
| R1 ping | OK | `ssh r1 "ping 172.31.255.1"` |
| R2 config | OK | `ssh r2 "show configuration"` |
| R2 ping | **FAIL** | Empty response |
| R3 config | OK | `ssh r3 "show run"` |
| R3 ping | OK | `ssh r3 "ping 172.31.255.1"` |
| R4 config | OK | `ssh r4 "show run"` |
| R4 ping | **FAIL** | Empty response |

**Result: 6/8 (75%)**

**Notable:**
- Config fetches: 8/8 across both runs — correctly uses `show configuration` for Junos and `show run` for Arista
- Ping tasks: 4/8 across both runs — randomly returns empty responses on identical tasks
- R2 ping failed both runs; other ping failures were non-deterministic

### Stage 3: Autonomous Operation

Given the prompt: *"Someone added 4 routers to the /etc/hosts file and said SSH keys are setup. Can you verify the routers are up?"*

The model completed 7 turns:

| Turn | Action | Result |
|---|---|---|
| 1 | `cat /etc/hosts` | Found all 4 routers |
| 2 | `ping -c 2 172.31.255.10` | R2 reachable |
| 3 | `whoami` | Unnecessary — checked user identity |
| 4 | Ping loop for all 4 routers | All reachable |
| 5 | `ls -la ~/.ssh/` | Verified SSH keys exist |
| 6 | `ssh -o BatchMode=yes ubuntu@R1 true` | **Failed** — sent `ubuntu@` to network device, sent `true` to router CLI |
| 7 | Empty summary | Gave up |

**Result: FAILED**

The model performed strong Linux investigation (turns 1-5) but failed at the critical step — SSH'ing into actual routers. It assumed Linux conventions (`ubuntu@` user, `true` command) instead of recognizing these as network devices. Junos returned "unknown command: true", Arista returned "Invalid input" — the model did not interpret these errors or retry.

### Comparison with Previously Tested Models

| Model | Parameters | Smoke | Router (Guided) | Autonomous | Notes |
|---|---|---|---|---|---|
| **gpt-oss-120b** | 120B | 4/4 | 8/8 | COMPLETED | Flawless. SSH flags, nc fallback, summary |
| **Qwen3-Coder-30B** | 30B (3B active) | 4/4 | 8/8 | COMPLETED | Best value. Fast, light |
| **Nemotron 3 Super** | 120B (12B active) | 4/4 | **6/8 (75%)** | **FAILED** | Random empty responses, treats routers as Linux |
| gpt-oss-20b | 20B | 4/4 | 8/8 | INCOMPLETE | Typo in IP, crashed |
| Mistral-Small-24B | 24B | 4/4 | 8/8 | FAILED | Gave up after 2 turns |
| granite-3.1-8b | 8B | 4/4 | 8/8 | FAILED | Described but didn't act |

Nemotron 3 Super at 120B (12B active) scored **lower on guided tasks** than Mistral-Small at 24B and granite at 8B — both of which achieved 8/8 on the router test. On autonomous operation, it failed in the same way as models a fraction of its size.

## Key Findings

### 1. Non-deterministic tool calling is a deployment blocker

The model produces correct tool calls for some tasks and empty responses for identical tasks. This inconsistency means you cannot rely on it for any automated workflow. A 75% success rate on explicitly guided tasks is insufficient for production use.

### 2. Identity compliance problem

The model abandons its own identity when presented with a leading question. In an agentic context, this means it may comply with incorrect user instructions rather than pushing back — a safety concern for infrastructure management.

### 3. Linux-centric assumptions

When given autonomous operation, the model defaulted to Linux conventions (`ubuntu@`, `true`) instead of recognizing network device CLIs. It has general systems knowledge but lacks network-specific operational behavior.

### 4. Size alone does not predict capability

At 120B total parameters, Nemotron 3 Super is one of the largest open models tested. It was outperformed on guided tasks by models 5-15x smaller. The MoE architecture (12B active) may explain this — but the marketed positioning as a 120B model sets expectations it does not meet for this use case.

### 5. Behavioral training remains the gap

Consistent with our findings from the 16-model evaluation: **behavioral training, not knowledge, is the differentiator.** Nemotron knows what SSH is, knows Junos vs Arista commands, but cannot reliably execute a sequence of network operations.

## Conclusion

Nemotron 3 Super is not ready for autonomous network operations. It has the knowledge but lacks the behavioral reliability required for infrastructure management. Its non-deterministic tool calling (75% on guided tasks) and failure on autonomous operation place it below several smaller, more focused models in our evaluation.

For network automation use cases, **gpt-oss-120b** and **Qwen3-Coder-30B** remain the recommended models from our testing.

---

*Tested by H-Network — March 16, 2026*
*Test infrastructure: Juniper vMX + Arista vEOS over WireGuard, served via Ollama on NVIDIA GH200*
