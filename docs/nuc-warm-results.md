# Nemotron × OpenClaw on a CPU-only NUC: cold and resident turns

[日本語](nuc-warm-results.ja.md) · November 24, 2026 (JST)

**The embedded OpenClaw agent returned the three exact requested answers. Keeping Nemotron resident removed almost all measured load time, but did not make the agent turn faster in this run.**

| Fresh agent turn | Model before chat | Exact reply | Agent CLI | Ollama total | Ollama load | Prompt evaluation (reported tokens / time) | First visible native frame |
| --- | --- | --- | ---: | ---: | ---: | ---: | ---: |
| 1. English control | Not resident | `PONG` (4 B) | 95.428 s | 77.690 s | 2.049 s | 2,055 / 75.369 s | 77.430 s |
| 2. Japanese | Resident | `了解` (6 B) | 103.437 s | 84.184 s | 1.268 ms | 2,058 / 83.953 s | 84.004 s |
| 3. English control, same user text | Resident | `PONG` (4 B) | 106.937 s | 88.624 s | 0.946 ms | 2,053 / 88.228 s | 88.281 s |

Each turn made exactly one native Ollama `POST /api/chat` through the loopback observer from an actual `openclaw agent --local` run. Ollama's visible text and OpenClaw's final payload matched byte for byte with no trimming; both reported `stop`. All three requests had zero tool definitions and zero native tool calls. The CLI column measures command start to exit. The first-frame column starts at native request dispatch at the observer, not at CLI startup.

**Setup.** Intel NUC, Core i5-1135G7, CPU only; OpenClaw 2026.11.7, Ollama 0.40.17 and Nemotron 3 Nano 4B Q4_K_M (`nemotron-3-nano:4b-q4_K_M`). The Linux/amd64 image manifests and configs and the model bytes used the same verified digest pins as the earlier cold trial. Model manifest: `sha256:6cc467f054393a55e98a74098abde0c762ffb6d1d8cd64becf30458f38886197`. Context was 16,384 tokens, with two model threads, a 32-token output cap, thinking off and two-minute keep-alive. One Ollama instance stayed up; each turn had a fresh OpenClaw container and session. A native `GET /api/ps` immediately before each chat verified that the runner was absent for turn 1 and resident on CPU for turns 2 and 3 (0 B VRAM). Inputs were `Reply with exactly: PONG`, `次の二文字だけ返してください：了解`, and the first English input again. The repeat provides an English-to-English comparison without treating Japanese as the same workload.

**Resources and cleanup.** The Ollama and OpenClaw caps remained 2 CPUs / 8 GiB and 1 CPU / 2 GiB, respectively, in the same loopback-only network namespace, with no added mounts or published ports. Ollama's cgroup used 155.598 / 168.616 / 177.459 CPU-seconds over the three respective agent intervals. Its whole-run RAM peak was 4,199.19 MiB (4.101 GiB); the three fresh OpenClaw peaks were 1,031.71 / 1,028.95 / 1,026.11 MiB. Peak swap was 0 B for both roles. These are per-container cgroup measurements, not host totals. Trial containers, temporary image references and the private model copy were removed; the pre-existing Docker container and image inventories matched before and after.

**Interpretation.** The earlier cold run on the same embedded route reached `PONG` separately (94.732 s CLI, 75.913 s Ollama). In the new batch, prompt evaluation dominated even with the runner resident; the same-text warm English sample was slower end to end. The automatic agent envelope can change between fresh sessions, and runner residency does not establish prompt-cache reuse. Ollama's reported prompt count is not a count of cache hits or necessarily of newly evaluated tokens. These are individual samples, not a causal estimate or a throughput benchmark. The Japanese result verifies one exact two-character reply, not general Japanese quality. The cause of the earlier Gateway mismatch remains unresolved. Production use and NemoClaw/OpenShell Stage 3 remain unverified.
