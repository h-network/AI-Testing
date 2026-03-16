# Open Source LLM Router Testing

Evaluating open source large language models on their ability to manage real multi-vendor network infrastructure autonomously.

No synthetic benchmarks. Every test runs against virtual "real" routers over SSH.

## Test Stages

### Stage 1: Smoke Test

Can the model respond and make basic tool calls (list files, check processes, ping)?

### Stage 2: Guided Router Tasks
Given explicit instructions — device type, commands, router names — can the model SSH into routers and execute the correct commands?

8 tasks: config fetch + ping for each of 4 routers (R1/R2 Juniper, R3/R4 Arista).

### Stage 3: Autonomous Operation
Given only: *"Someone added 4 routers to /etc/hosts and said SSH keys are setup. Can you verify the routers are up?"*

No hints about device types, commands, or IPs. The model must figure it out.

## Test Environment

- **Routers:** 2x Juniper vMX (Junos 24.2), 2x Arista vEOS (Advanced test have Nokia and Cumulus)
- **Connectivity:** SSH over WireGuard tunnel to Lambda Cloud GPU instances
- **Inference:** Ollama and/or vLLM with identical prompts and a single bash tool, If possible we run the tests on both inference engines
- **Evaluation:** All models get the same system prompt, same tests, same criteria

## Updated Results

| Model | Params | Active | Arch | Smoke | Router (8) | Autonomous | Notes |
|---|---|---|---|---|---|---|---|
| gpt-oss-120b | 120B | 120B | Dense | 4/4 | 8/8 | PASS | Flawless. SSH flags, nc fallback, summary |
| Qwen3-Coder-30B | 30B | 3B | MoE | 4/4 | 8/8 | PASS | Best value. Fast, light |
| gpt-oss-20b | 20B | 20B | Dense | 4/4 | 8/8 | PARTIAL | Typo in IP, sent echo to Juniper, crashed |
| Mistral-Small-24B | 24B | 24B | Dense | 4/4 | 8/8 | FAIL | Grepped for "router", gave up after 2 turns |
| granite-3.1-8b | 8B | 8B | Dense | 4/4 | 8/8 | FAIL | Described plan but never executed |
| Hermes-3-8B | 8B | 8B | Dense | 4/4 | 4/8 | FAIL | Hallucinated IPs, broken syntax |
| **Nemotron 3 Super** | **120B** | **12B** | **MoE** | **4/4** | **6/8** | **FAIL** | **Random empty responses, treats routers as Linux. Ollama only — Lambda out of 8xA100s** |
## Key Findings

1. **Static benchmarks don't predict agent performance.** Every model passes smoke tests. The only test that matters is autonomous multi-turn operation on real infrastructure.
2. **Behavioral training > knowledge.** Failing models know what SSH and ping are. They fail because they assume instead of check, describe instead of act, give up instead of retry.
3. **Size ≠ capability.** An 8B model scored 8/8 on guided tasks where a 120B model scored 6/8.


