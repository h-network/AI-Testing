# Open Source LLM Evaluation for Autonomous Network Engineering

## Introduction

We evaluated open source large language models for their ability to autonomously manage multi-vendor network infrastructure. The goal: find models that can SSH into routers, execute the right commands, handle errors, and report findings — without hand-holding.

This is not a benchmarking exercise on synthetic datasets. Every test ran against real network devices (Juniper and Arista) in a live lab environment, executing real commands over SSH.

## Evaluation Approach

Models were tested across four progressively harder stages:

1. **Basic capability** — can the model respond and make tool calls?
2. **Guided tasks** — given explicit instructions (device type, commands), can it execute correctly?
3. **Autonomous tasks** — given a vague objective with no hints, can it figure out the steps?
4. **Complex reasoning** — can it investigate, troubleshoot, and adapt when things go wrong?

All models were served via vLLM using Docker containers with tool calling enabled. Each model was tested with identical prompts and a single bash tool for command execution.

## Models Tested

We evaluated 16 open source models across two inference frameworks (Ollama and vLLM):

| Model | Parameters | Architecture | Parser |
|---|---|---|---|
| openai/gpt-oss-120b | 120B | Dense | openai |
| openai/gpt-oss-20b | 20B | Dense | openai |
| Qwen/Qwen3-Coder-30B-A3B-Instruct | 30B (3B active) | MoE | qwen3_xml |
| mistralai/Mistral-Small-24B-Instruct-2501 | 24B | Dense | mistral |
| ibm-granite/granite-3.1-8b-instruct | 8B | Dense | granite |
| NousResearch/Hermes-3-Llama-3.1-8B | 8B | Dense | hermes |
| ibm-granite/granite-20b-functioncalling | 20B | Dense | granite-20b-fc |
| Salesforce/xLAM-7b-r | 7B | Dense | xlam |
| microsoft/phi-4 | 14B | Dense | hermes |
| tencent/Hunyuan-A13B-Instruct | 13B | MoE | hermes |
| internlm/internlm2_5-7b-chat | 7B | Dense | internlm |
| allenai/Olmo-3-7B-Instruct | 7B | Dense | olmo3 |
| Qwen/Qwen3-32B | 32B | Dense | hermes |
| meta-llama/Llama-3.1-8B-Instruct | 8B | Dense | — |
| cohere/command-r:35b | 35B | Dense | — |
| deepseek-ai/DeepSeek-R1-14B | 14B | Dense | — |

## Results

### Stage 1: Basic Tool Calling

All models that support tool calling passed basic tests — responding to messages and making simple tool calls. This stage is not a differentiator.

### Stage 2: Guided Network Tasks

When given explicit context ("R1 is Juniper, use `show configuration`"), most models performed well. However, even at this stage, differences appeared:

- One model introduced a typo in an IP address
- One model ran the wrong command for certain tasks
- The best models produced identical, correct commands every time

### Stage 3: Autonomous Operation

This is where models were separated. Given only "someone added routers to /etc/hosts, verify they're up," the model had to:

- Find the router entries
- Determine how to reach them
- Verify connectivity
- Report findings

**Only 2 out of 6 models completed this task.** The failures were revealing:

- One model grepped for "router" in /etc/hosts, found nothing (entries were named R1-R4), and gave up
- One model described what it *would* do but never executed any commands
- One model hallucinated IP addresses it had never seen
- One model tried but sent Linux commands to network devices, causing errors

The two models that succeeded — **gpt-oss-120b** and **Qwen3-Coder-30B** — both read the full hosts file, extracted the IPs, and systematically verified each router.

### Stage 4: Complex Reasoning

**gpt-oss-120b** demonstrated genuine troubleshooting instincts:

- When SSH connections failed, it tried alternative verification methods (`nc` port check)
- Used proper SSH flags (`-oBatchMode=yes`, `-oConnectTimeout=5`) for non-interactive sessions
- Produced a formatted summary table with per-router status
- Made zero errors across all tests

### Full Results

| Model | VRAM | Smoke | Router | Autonomous | Notes |
|---|---|---|---|---|---|
| **gpt-oss-120b** | 63GB | 4/4 | 8/8 | COMPLETED | Flawless. SSH flags, `nc` fallback, markdown summary |
| **Qwen3-Coder-30B-A3B** | ~18GB | 4/4 | 8/8 | COMPLETED | Best value. MoE, fast, light |
| gpt-oss-20b | ~40GB | 4/4 | 8/8 | INCOMPLETE | Typo in IP, sent `echo` to Juniper, crashed |
| Mistral-Small-24B | ~48GB | 4/4 | 8/8 | FAILED | Grepped for "router", gave up after 2 turns |
| granite-3.1-8b | ~16GB | 4/4 | 8/8 | FAILED | Described plan but never executed commands |
| Hermes-3-8B | ~16GB | 4/4 | 4/8 | FAILED | Hallucinated IPs, broken command syntax |
| granite-20b-fc | — | 4/4 | — | — | All tool calls failed on vLLM |
| xLAM-7b | — | 4/4 | — | — | All tool calls failed on vLLM |
| phi-4 | — | 4/4 | — | — | No tool calling support |
| Hunyuan-A13B | — | — | — | — | Failed to start on vLLM |
| internlm2-7b | — | — | — | — | No tool calling support (Ollama) |
| Olmo-3-7B | — | 4/4 | — | — | All tool calls failed on vLLM |
| Qwen3-32B | — | 4/4 | — | — | Partial tool calling only |
| Llama-3.1-8B | — | 4/4 | — | — | Partial tool calling only |
| DeepSeek-R1-14B | — | 4/4 | — | — | HTTP 400 on all tool calls |
| command-r:35b | — | 4/4 | — | — | Passed Ollama, not tested on vLLM |

## Key Findings

### 1. Static benchmarks are meaningless for agent tasks

Every model passed basic tool calling tests. The only test that matters is autonomous multi-turn operation on real infrastructure. Leaderboard scores do not predict real-world agent performance.

### 2. Size matters — but not linearly

A 120B dense model was flawless where its 20B variant (same architecture) made errors and crashed. However, a 30B MoE model with only 3B active parameters also completed autonomous tasks successfully, using a fraction of the resources.

### 3. Most models cannot operate autonomously

Of 16 models tested, only 3 could reliably execute network tasks without explicit instructions. Most either gave up on the first obstacle, hallucinated information, or described actions instead of taking them.

### 4. Behavior matters more than knowledge

The failing models knew what SSH and ping are. They failed because of behavioral patterns — assuming instead of checking, describing instead of acting, giving up instead of retrying. This suggests that behavioral fine-tuning on a small number of high-quality examples could close the gap between capable and incapable models.

### 5. Framework affects results

The same model can produce different results depending on the inference framework. Tool calling support varies significantly between Ollama and vLLM, and some models that appear non-functional on one framework work correctly on another with the right configuration.

### 6. Reasoning models underperform on tool calling

Models specifically designed for chain-of-thought reasoning performed poorly on tool calling tasks. They tend to reason about what they would do rather than executing tools. For agent workloads, action-oriented models outperform thinkers.

## Practical Recommendations

### For deploying network automation agents:

- **Test on real infrastructure**, not benchmarks. A model's tool calling score tells you nothing about whether it can manage your routers.
- **Use multi-turn autonomous tests** as the primary evaluation. Single-turn guided tests are trivially easy for most models.
- **Consider MoE architectures** for resource-constrained deployments. They can match dense model performance at a fraction of the compute cost.
- **Separate knowledge from behavior**. Use RAG or external knowledge APIs for vendor-specific facts. Train the model on *how to act*, not *what to know*.
- **Don't trust small models with autonomous operation** unless they've been fine-tuned for it. General-purpose instruction tuning is not sufficient.

### For fine-tuning domain-specific agents:

- You don't need massive datasets. Behavioral patterns can be trained with hundreds of high-quality multi-turn examples, not thousands of facts.
- Generate training data by running tasks through capable models (or cloud APIs) on real lab topologies. The outputs become the training set for smaller, deployable models.
- Validate fine-tuned models with the same autonomous test suite used for evaluation. The pipeline is the same for benchmarking and quality assurance.

## Hardware Considerations

| Deployment | Minimum GPU | Suitable Models |
|---|---|---|
| Development / Testing | 24GB (RTX 4090, 5090) | MoE models, small dense models |
| Production — Lite | 32GB (5090) | Fine-tuned MoE models |
| Production — Premium | 80GB+ (A100, GH200) | Full-size dense models |

All tested models run fully airgapped with pre-downloaded weights. No runtime internet dependency.

## Conclusion

Autonomous network management with open source LLMs is viable today, but model selection is critical. The majority of available models cannot operate autonomously on real infrastructure. Through systematic evaluation on real devices, we identified models that can — and more importantly, identified the behavioral gaps that fine-tuning can close.

The path to a reliable self-hosted network engineering agent is not finding a perfect model. It's finding a capable base model, training it on the right behaviors, and pairing it with a solid knowledge layer. The tools to do this are all open source and the compute cost is minimal.
