# First NUC trial: 2026-11-24 JST

[Home](../README.md) | [日本語](first-trial.ja.md)

- Direct: the fixed synthetic reply matched exactly over local Ollama 0.40.17 with `nemotron-3-nano:4b-q4_K_M`. The loaded model used CPU only at 16,384 context; Ollama reported 3.5 GB. Actual cold latency and throughput were not retained.
- Agent: a separate OpenClaw 2026.11.7 Gateway reached local readiness and completed a turn. A loopback observer verified one forwarded request to the exact model with no tool schemas. The visible text did not satisfy the exact-reply assertion; the reply itself was not retained, so absence and different text were not distinguished. This is routing evidence, not an end-to-end pass.
- Boundary: the inference namespace had loopback only after staging. No host binds or published ports. An attempted catalog refresh failed offline; no successful external inference. Both disposable containers, temporary model state and newly staged image references were removed. The existing NUC service remained running with the same restart count. Production OpenClaw on the separate Mac mini was not changed.

A focused follow-up would retain the bounded synthetic reply and stop reason, and read the routing counters even when the reply assertion fails. No long-running setup or NVIDIA NemoClaw/OpenShell validation was performed.
