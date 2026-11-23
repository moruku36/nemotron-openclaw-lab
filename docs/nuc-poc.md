# Intel NUC proof of concept

[Home](../README.md) | [日本語](nuc-poc.ja.md)

Hardware preflight on the NUC is complete. Inference measurements and a full agent turn are pending; this is not a completed deployment. Start with synthetic English text: the NVIDIA 4B model card lists English and coding languages. Assess Japanese separately.

## 0. Check the lab host

Stage 0 checked the target CPU-only x86-64 class and RAM/storage headroom. The target is an Intel NUC in the 11th-generation Core i5 / 32 GB class. Verify resources and the loopback port again before each run. Use approved, versioned installs and a fresh unprivileged lab environment with no production state, credentials, SSH agent forwarding, or shared workspace. Download weights before the inference check; a public artifact download is not hosted inference. Do not sign in to cloud Ollama or configure hosted fallbacks.

## Preferred runtime: two disposable containers

Pin OCI image digests for Ollama and OpenClaw. Stage the images and exact model before the offline test. After staging, the disposable containers share a network namespace with only loopback available. Do not publish ports or mount host paths or the container engine socket. Keep separate temporary state, bound CPU/RAM/PIDs and request time, and process one request at a time. This arrangement does not prove OpenShell confinement.

## 1. Prove local inference

The local 4B Q4_K_M package downloads about 2.8 GB; runtime memory also includes context and runtime overhead. Start with 4B rather than 30B-A3B: fewer active parameters per token do not remove the larger weight set. Do not assume CUDA or a discrete GPU. Do not use a `:cloud` tag.

The commands below are an optional host-native probe, not the preferred OCI setup. In OCI use equivalent probes inside the isolated namespace after staging, without re-enabling egress. On the host, run the first block only if port 11434 is free. Run the second in another terminal on the NUC. Never stop an existing service to claim the port.

```bash
OLLAMA_HOST=127.0.0.1:11434 OLLAMA_NO_CLOUD=1 OLLAMA_NUM_PARALLEL=1 ollama serve
```

```bash
ollama pull nemotron-3-nano:4b-q4_K_M
ollama show nemotron-3-nano:4b-q4_K_M
curl --fail-with-body --max-time 180 \
  -H 'Content-Type: application/json' \
  --data '{"model":"nemotron-3-nano:4b-q4_K_M","stream":false,"think":false,"messages":[{"role":"user","content":"Reply with exactly: PONG"}],"options":{"num_ctx":4096,"num_predict":32}}' \
  http://127.0.0.1:11434/api/chat
ollama ps
```

Verify cloud-disabled logs, loopback-only listener, and CPU execution in `ollama ps` while loaded. `message.content` should be `PONG`. Record cold latency, load time, generation speed, and peak memory. Rerun warm only if useful. Throughput is `eval_count / (eval_duration / 1e9)`. The 180-second limit is a lab budget, not a performance claim. Stop on timeout, out-of-memory, or sustained swapping without switching to hosted inference.

## 2. Prove a full OpenClaw agent turn

For this optional host-native smoke test, use a fresh `nemotron-lab` profile. Check that `OPENCLAW_STATE_DIR` and `OPENCLAW_CONFIG_PATH` do not override it. Abort if the config path or any existing credentials, channels, or fallbacks point elsewhere. `ollama-local` is a marker, not an API secret.

```bash
export OLLAMA_API_KEY=ollama-local
openclaw --profile nemotron-lab config file
# Verify the path belongs to the lab before changing config.
openclaw --profile nemotron-lab config set models.providers.ollama.baseUrl http://127.0.0.1:11434
openclaw --profile nemotron-lab config validate
openclaw --profile nemotron-lab models list --provider ollama
timeout 180s openclaw --profile nemotron-lab infer model run --local \
  --model ollama/nemotron-3-nano:4b-q4_K_M --prompt 'Reply with exactly: PONG' --json
```

Use the native URL without `/v1`. The stateless `--local` probe does not need a Gateway or load agent tools. Verify the expected model, nonempty reply, and local transport. This is diagnostic only. Stage 2 passes only after one genuine OpenClaw agent turn on synthetic text using lab-only state, with evidence that the request reached the intended local model. An embedded agent run qualifies; a persistent Gateway is not required. Keep messaging, browser, file/exec access, schedules, and hosted fallbacks disabled. Measure larger context separately.

## 3. Gate NVIDIA sandbox work

Ubuntu 24.04 is an upstream onboarding target, but host prerequisites do not cover all inference needs. NVIDIA does not list 4B among the qualified Ollama defaults. Treat NemoClaw/OpenShell with 4B on this NUC as unverified.

Use the installed release's documented provider path. Current releases attach OpenShell providers to new Ollama-backed OpenClaw sandboxes; older ones can route differently. Do not reuse the host loopback URL inside a container or assume `inference.local` applies. Non-WSL onboarding may start a token-gated wildcard proxy on port 11435; stop before onboarding if reachability cannot be restricted to the intended private boundary. Do not relax the firewall, or mount host credentials, home folders, or the container engine socket.

Require the supported onboarding probe to make a real structured tool call. Then check lab workspace and local inference scoping with benign synthetic fixtures only. Tool-call JSON in plain text is not a pass. If this gate fails, stop at text-only inference.

## Evidence and rollback

Record versions, model digest and quantization, peak RAM/swap, response time, throughput, listener scope, and per-stage outcome. Keep raw logs and machine identifiers private. Stop only the two disposable lab containers or the host-native foreground processes started for this experiment and keep or remove lab state independently from production. Do not change the Mac mini.

## References

- [NVIDIA 4B model card](https://huggingface.co/nvidia/NVIDIA-Nemotron-3-Nano-4B-GGUF) · [Ollama 4B artifact](https://ollama.com/library/nemotron-3-nano:4b-q4_K_M) · [Ollama local FAQ](https://docs.ollama.com/faq)
- [OpenClaw Ollama provider](https://docs.openclaw.ai/providers/ollama) · [Profile paths](https://docs.openclaw.ai/help/environment) · [Inference CLI](https://docs.openclaw.ai/cli/infer)
- [NemoClaw prerequisites](https://docs.nvidia.com/nemoclaw/latest/user-guide/openclaw/get-started/prerequisites) · [Local inference options](https://docs.nvidia.com/nemoclaw/latest/user-guide/openclaw/inference/local-inference/choose-local-inference-server) · [Ollama onboarding](https://docs.nvidia.com/nemoclaw/latest/user-guide/openclaw/inference/local-inference/set-up-ollama) · [Platform support](https://docs.nvidia.com/nemoclaw/latest/user-guide/openclaw/reference/platform-support)
- [NVIDIA OpenShell architecture](https://docs.nvidia.com/openshell/about/architecture)
