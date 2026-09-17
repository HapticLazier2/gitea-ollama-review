# Self-Hosted AI Code Review for Gitea

Run AI-generated code review comments on Gitea pull requests using a
small, local LLM — no GitHub Copilot, no external API calls, no data
leaving your network. A dedicated host runs Ollama plus Gitea's
`act_runner`, wired into Gitea Actions so it posts a review comment on
every PR automatically.

Built because Gitea has no built-in AI review feature (there's an open
upstream feature request for it, but the maintainers' stated direction is
a generic bot/agent framework rather than a native feature) — this bolts
one on with things Gitea already supports: Actions, a self-hosted runner,
and a repo API.

## How it works

1. A pull request is opened, synced, or reopened.
2. Gitea Actions triggers the workflow on your self-hosted runner.
3. The workflow fetches the PR's diff against its base branch.
4. The diff is sent to a local Ollama endpoint running a code-focused model.
5. The model's response is posted back as a PR comment via Gitea's API.

Everything runs on infrastructure you control — the only external
network calls needed are to pull the model once and for routine OS
updates.

## Requirements

- A Gitea instance with **Actions enabled** and admin access to the
  **Runners** settings page.
- A Linux host or VM to run `act_runner` + Ollama, with network access to
  your Gitea instance and the internet (for the initial model pull).
  CPU-only works fine for an async, non-real-time workload; a GPU speeds
  up inference but isn't required.
- Recommended minimum: 4+ CPU cores and 8GB+ free RAM for a 7B quantized
  model (see [Choosing a model](#choosing-a-model)). Scale up for larger
  models.
- `git`, `curl`, and `jq` on the runner host (installed in step 6).

## Reference variables

The instructions below use placeholders — substitute your own values
throughout.

| Placeholder | Meaning | Example |
|---|---|---|
| `<GITEA_URL>` | Base URL of your Gitea instance | `http://192.168.20.100:3000` |
| `<RUNNER_NAME>` | Name to register the runner under | `ci-runner` |
| `<RUNNER_TOKEN>` | Registration token from Gitea's Runners admin page | — |
| `<MODEL_NAME>` | Ollama model tag | `qwen2.5-coder:7b` |

## Installation

### 1. (Optional) Network isolation

If you're running this in a homelab or otherwise want the runner on its
own segmented network, the pattern is: a dedicated VLAN/subnet for the
runner, default-deny firewall rules with narrow exceptions for
(a) reaching Gitea specifically, (b) outbound TCP 80/443 for model pulls
and updates, and (c) DNS — plus outbound NAT for that subnet to the
internet. The specifics (VLAN tagging, bridge setup, firewall rule
syntax) depend entirely on your hypervisor and firewall, so they're not
reproduced here.

**If your runner host already has network access to Gitea and the
internet, skip straight to step 2.**

### 2. Provision the runner host

Any Linux VM or machine works. A single network interface with a route
to Gitea and the internet is all that's required.

> **Gotcha (fresh Debian/Ubuntu VMs):** if you get locked out of
> `sudo`/`su` because the initial user wasn't added to `sudoers`,
> recover via GRUB: interrupt boot, append `rw init=/bin/bash` to the
> kernel line, then at the resulting root shell:
> ```bash
> mount -o remount,rw /
> passwd root
> usermod -aG sudo <your-user>
> ```
> Reboot normally afterward (not via the bypassed init).

Verify reachability before continuing:

```bash
curl -I <GITEA_URL>          # expect a response from Gitea
curl -I https://ollama.com   # confirms outbound internet access
```

### 3. Install Ollama and pull a model

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull <MODEL_NAME>
ollama run <MODEL_NAME>      # smoke test
```

Runs in CPU-only mode automatically if no GPU is available.

### 4. Install and register `act_runner`

> **Gotcha:** Gitea's runner project renamed its download artifact from
> `act_runner` to `gitea-runner`. The releases API path is unchanged
> (`.../repos/gitea/act_runner/releases/latest`), but the binary itself
> now downloads from `dl.gitea.com/gitea-runner/<version>/...`, not
> `dl.gitea.com/act_runner/...`. The old URL just returns nothing, which
> looks like a network problem but isn't.

```bash
# Download the current release for your architecture from
# dl.gitea.com/gitea-runner/<version>/...
chmod +x gitea-runner-*
sudo mv gitea-runner-* /usr/local/bin/act_runner
```

Get a registration token from Gitea's **Runners** admin page, then:

```bash
act_runner register \
  --instance <GITEA_URL> \
  --token <RUNNER_TOKEN> \
  --name <RUNNER_NAME> \
  --labels ubuntu-latest:host
```

> **Gotchas:**
> - Copy the registration token carefully and in full — a truncated or
>   mistyped token produces a persistent, unhelpful
>   `unknown: runner registration token not found` error with no other
>   symptoms.
> - Flags must have no stray whitespace. `-- name <RUNNER_NAME>` (space
>   after `--`) is parsed as the argument-terminator plus positional
>   args, not a `--name` flag, and fails with
>   `accepts at most 0 arg(s), received 4`. Use `--name` exactly, no
>   internal spaces.
> - `--labels ubuntu-latest:host` registers a **host-only** runner (jobs
>   run directly on this machine, no Docker required). If you'd rather
>   run jobs in containers, install Docker and use a
>   `ubuntu-latest:docker://...` label instead — the rest of this guide
>   assumes host-only.

### 5. Run `act_runner` as a systemd service

Config lives at `/etc/act_runner/config.yaml`, registration state at
`/etc/act_runner/.runner`.

> **Gotcha:** `act_runner generate-config`'s default output ships a
> **non-empty** `runner.labels` list of Docker-based labels. Per the
> config's own comment, a non-empty list here *overrides* whatever
> labels were registered in `.runner` — so even with a host-only
> registration, the daemon looks for Docker and fails with
> `daemon Docker Engine socket not found and docker_host config was
> invalid` on a host that doesn't have it. Fix:

```yaml
# /etc/act_runner/config.yaml
runner:
  labels: []   # empty list -> falls back to the registered label(s)
```

Create a systemd unit (`/etc/systemd/system/act_runner.service`) that
runs `act_runner daemon` in this directory, then:

```bash
sudo systemctl enable --now act_runner
sudo journalctl -u act_runner -f
# expect: "runner: <RUNNER_NAME>, ... declare successfully"
```

Confirm the runner shows as online on Gitea's **Runners** admin page.

### 6. Install shell dependencies for the workflow

The workflow below avoids `actions/checkout` and other JS-based actions
— a host-only runner has no container providing a Node.js runtime, so
plain shell steps keep it dependency-light:

```bash
sudo apt update && sudo apt install -y git curl jq
```

### 7. Add the review workflow to a repo

Commit [`.gitea/workflows/ai-review.yml`](.gitea/workflows/ai-review.yml)
to any repo you want reviewed. It triggers on PR open/sync/reopen,
fetches the diff, sends it to the local Ollama endpoint
(`http://localhost:11434`), and posts the response as a PR comment using
Gitea Actions' auto-issued per-job token (`secrets.GITHUB_TOKEN`, kept
under that name for GitHub-compatibility) — no manual secret setup
needed per repo.

Set `MODEL` at the top of the workflow file to match whatever you pulled
in step 3.

Notes:
- The diff is truncated (12,000 characters by default) before hitting
  the model — smaller/CPU-only models have real context and latency
  limits; raise this if your model and hardware can handle it.
- `stream: false` waits for the full response in one shot, which is
  simpler for a CI step but blocks the whole request until generation
  finishes — budget roughly 30-90 seconds for a 7B CPU model on a
  reasonably sized diff, more for larger models or diffs.

## Choosing a model

Any model Ollama can run works; a code-specialized model gives
noticeably better reviews than a general-purpose one at the same size.
`qwen2.5-coder:7b` (~4-5GB quantized) is a reasonable default for
CPU-only inference — it's a good fit for an async workload where a
30-90 second delay is acceptable, unlike a chat interface. Larger models
(14B, 32B) give better reviews at the cost of RAM and latency; smaller
models trade the reverse.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `curl` to `dl.gitea.com/act_runner/...` returns nothing | Artifact renamed to `gitea-runner` | Use `dl.gitea.com/gitea-runner/<version>/...` |
| `unknown: runner registration token not found` | Token copied truncated / mistyped | Re-copy the full token exactly |
| `act_runner register` fails with `accepts at most 0 arg(s)` | Stray space after `--` in a flag | Use `--name` with no internal whitespace |
| `daemon Docker Engine socket not found` on a Docker-less runner | Non-empty default `runner.labels` in `config.yaml` overrides the registered label | Set `runner.labels: []` |
| Locked out of `sudo`/`su` on a fresh VM | User never added to `sudoers` | GRUB recovery (`rw init=/bin/bash`) → reset password/group → normal reboot |
| Workflow times out or gives truncated/low-quality reviews | Diff or model too large for available CPU/RAM | Lower the diff character limit, or use a smaller/faster model |

## Customization

- **Model:** change `<MODEL_NAME>` in step 3 and `MODEL` in the workflow file.
- **Prompt:** edit the `PROMPT` variable in the workflow to change review
  focus, tone, or length.
- **Diff size limit:** adjust the `head -c 12000` truncation to fit your
  model's context window.
- **Scope:** enable the workflow per repo (as written) or copy it into
  an org-wide template if your Gitea version supports shared workflows.

## Roadmap ideas

- [ ] Post inline comments on specific diff lines instead of one summary comment.
- [ ] Skip review on trivial changes (e.g. docs-only, formatting-only diffs).
- [ ] Cache/reuse context across commits on the same PR instead of a full re-review each push.

## Related projects

This isn't the only take on self-hosted AI review for Gitea — worth
checking these out too, in case one fits your needs better:

- **[kekxv/AiReviewPR](https://github.com/kekxv/AiReviewPR)** — the
  closest existing match: Gitea Actions + Ollama, same core idea as this
  repo.
- **[ccsert/opencode-review-gitea](https://github.com/ccsert/opencode-review-gitea)** —
  a full agent framework (TypeScript, Docker or source install) with
  line-level inline comments and multiple LLM backends including Ollama.
- **[TerraScan](https://spaceterran.com/posts/terrascan-self-hosted-ai-code-review-gitea/)** —
  runs as a Docker container via Gitea Actions, supports Ollama plus
  cloud providers, posts inline comments.
- **[Self-Hosted AI Code Review Bot](https://dev.to/signal-weekly/build-a-self-hosted-ai-code-review-bot-with-ollama-and-gitea-webhooks-592p)** —
  same goal, different trigger: a standing Python/Flask service driven
  by Gitea **webhooks** rather than Actions, Docker Compose stack.
- **[AI Gitea Bot](https://forum.gitea.com/t/ai-gitea-bot-open-source-ai-powered-code-reviews-for-your-self-hosted-gitea/12030)** —
  a persistent Spring Boot service (Docker image) with Ollama support
  and inline reply-in-context.
- **[Nikita-Filonov/ai-review](https://github.com/Nikita-Filonov/ai-review)** —
  general-purpose multi-provider review tool covering GitHub, GitLab,
  Bitbucket, Azure DevOps, and Gitea, not Gitea-specific.
- **[Gitea + Claude](https://gmcd.dev/blog/self-hosted-ai-code-reviews-gitea-claude/)** —
  same Gitea Actions approach, cloud-based (Claude) rather than local
  inference.

**What's different here:** no Docker, no Node/Python/Spring runtime, no
agent framework — a single workflow file using plain `git`/`curl`/`jq`
against Ollama's HTTP API, on a host-only `act_runner`. Every project
above needs at least one of Docker, a language runtime, or a standing
service. If you want the smallest possible dependency footprint and are
fine with a single summary comment instead of inline line-level
comments, this is the tradeoff this repo makes; if you want richer
review output and don't mind the extra moving parts, one of the above
may suit you better.

## Acknowledgments

- [Ollama](https://ollama.com) for local model serving.
- [Qwen2.5-Coder](https://github.com/QwenLM/Qwen2.5-Coder) as an example review model.
- Gitea's `act_runner` project.
