# laya-triage

An OpenClaw plugin that puts **Laya**, Convai Innovations' open-source decision model, in front of your agent's main
model. Laya runs inside your own server, next to OpenClaw: no message leaves the machine and you need no API key.
Every incoming message is classified first — `thanks` / `spam` / `question` / `incident`, plus `urgent` yes/no — and the
main model only wakes up for the messages that need it:

| Laya's answer (score ≥ 0.9) | What happens |
|---|---|
| `thanks` | the plugin replies `בשמחה! 🙂` itself; the main model is never called |
| `spam` | the turn ends in silence |
| `question` / `incident` | the main model gets `[triage by Laya: <kind>, urgent <p>]` prepended, and the earlier verdicts of the same chat |
| any score below 0.9 | untouched: the normal agent path |

Every decision is logged: `laya thanks 1.00 urgent 0.25 86ms`. To watch only Laya's decisions:

```bash
openclaw logs --follow | grep "laya "
```

Laya sees each incoming message before OpenClaw hands it to the model (the `before_dispatch` hook), so thanks and
spam end there and a spammer gets nothing back. Commands such as `/new` go straight to OpenClaw, and heartbeat or
cron turns are never triaged.

This is the Laya edition of [jev-triage](https://github.com/CodeCave0000/jev-triage), the same plugin built on
TypeSafe's hosted Jev. If `jev-triage` is already installed from that video, it keeps working unchanged once
OpenClaw points at Laya; this repository is the same plugin with Laya's name in its log line and its tags.

## How it connects

OpenClaw's decision-model role talks to Laya through the official TypeSafe plugin: its `baseUrl` setting sends every
decision to a local server instead of TypeSafe, and `laya-serve` answers on the same `POST /v1/systemone` endpoint as Jev.

One catch. The TypeSafe plugin validates every answer strictly (`additionalProperties: false`), and Laya's answer
carries fields Jev does not send (`routing`, `answer_confidence`, `action`, extra `usage` fields), so the plugin rejects
it with `invalid-response` and the agent goes on as if no decision was made. `laya_shim.py` sits in between: it
forwards each request to Laya and returns only the fields Jev returns. One file, Python standard library only.

## Requirements

- OpenClaw **2026.9.6 or newer** (the decision-model role and the TypeSafe plugin's local-server mode); tested on **2026.9.7**
- the official TypeSafe plugin (the bridge; no TypeSafe account is needed for it)
- about 2 GB of free memory for Laya's multilingual checkpoint; a CPU is enough

## Install

Tested on 13 October 2026 with OpenClaw 2026.9.7, `@openclaw/typesafe` 2026.9.7 and Laya 0.4.6, on a 2-vCPU Hostinger
VPS with no GPU. Run everything in your OpenClaw terminal (on Hostinger: Docker Manager → your OpenClaw project → Terminal).

### 1. Laya and the adapter, inside the OpenClaw container

```bash
cd ~
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
uv venv laya
uv pip install --python laya/bin/python torch --index-url https://download.pytorch.org/whl/cpu
uv pip install --python laya/bin/python "laya[serve]"
curl -LO https://raw.githubusercontent.com/CodeCave0000/laya-triage/main/laya_shim.py
```

### 2. Point OpenClaw at the adapter

```bash
openclaw plugins install @openclaw/typesafe
openclaw config set plugins.entries.typesafe.config.baseUrl http://127.0.0.1:8712
openclaw config set agents.defaults.decisionModel typesafe/kev-latest     # "local server" in the Decision picker; the adapter pins the checkpoint
```

With `baseUrl` set, every request from the TypeSafe plugin goes to the local server, whatever the model label says, and
no TypeSafe key is used. If your agent already has a decision model from the Jev video (`typesafe/jev-latest`), you can
leave it as it is.

### 3. This plugin

```bash
openclaw plugins install git:github.com/CodeCave0000/laya-triage --force --accept-capabilities
openclaw config set plugins.entries.laya-triage.hooks.allowConversationAccess true
```

`--accept-capabilities` is required: a plugin that reads conversations gets real access, so OpenClaw refuses it
without explicit consent. `--force` acknowledges that the source is outside ClawHub's review.

**No `git` where OpenClaw runs?** (`git --version` says "not found".) Download the folder and install it by path instead:

```bash
curl -fsSL https://github.com/CodeCave0000/laya-triage/archive/refs/heads/main.tar.gz | tar xz -C /tmp
openclaw plugins install /tmp/laya-triage-main --force --accept-capabilities
```

### 4. Restart, then start Laya

Restart OpenClaw. In Docker (Hostinger's one-click install included), restart the **container** from Docker Manager and
give it a few minutes; `openclaw gateway restart` does not work inside the container. `openclaw plugins list` should
then show `typesafe` and `laya-triage` enabled.

A container restart stops everything that ran inside it, so after **every** restart start Laya and the adapter again:

```bash
cd ~
LAYA_HOST=127.0.0.1 LAYA_PORT=8711 LAYA_MODELS=multilingual LAYA_PRELOAD=1 nohup laya/bin/laya-serve > laya.log 2>&1 &
nohup laya/bin/python laya_shim.py 8712 8711 multilingual > shim.log 2>&1 &
```

`laya_shim.py <adapter port> <laya port> [checkpoint]`. The checkpoint pins every request to one Laya model, so a small
server keeps one model in memory. `multilingual` reads 100+ languages, Hebrew included; the English checkpoint is sharper
on English and useless on Hebrew.

## What to expect

- About 2 GB of memory for `laya-serve` with the multilingual model; the first start downloads it (~650 MB).
- About 0.4 to 0.6 seconds per decision on a 2-vCPU VPS, under 0.1 s on a GPU.
- Laya's base models are weaker than Jev on nuance (urgency, negation). On our Hebrew test set the outage was an
  `incident` at 1.00 but `urgent` only 0.28 where Jev said 0.97, and a double-billing complaint came back `thanks` at
  0.71. Their own benchmark file says the headline score comes from a fine-tuned checkpoint. Keep the 0.9 bar and let the
  main model handle the rest.

## Without the adapter

Laya 0.4.6 and newer can also drop the extra fields itself: start `laya-serve` with `LAYA_JEV_STRICT=1`, point `baseUrl`
straight at it (`http://127.0.0.1:8711`), and set `agents.defaults.decisionModel typesafe/multilingual` to pin the
checkpoint (the label after `typesafe/` reaches Laya unchanged). Checked on 17 October 2026 against the TypeSafe plugin's
own request and validation code from OpenClaw 2026.10.4: strict answers pass, answers without the flag are rejected.
The adapter stays the documented path because it is the one that ran on the VPS.

## Go back to Jev

```bash
openclaw config unset plugins.entries.typesafe.config.baseUrl
openclaw config set agents.defaults.decisionModel typesafe/jev-latest
```

Then restart the container. The Jev edition of this plugin is [jev-triage](https://github.com/CodeCave0000/jev-triage).

## Before you trust it

The 0.9 threshold is a starting point, not a measurement of your traffic. Run your own messages through it and
check the scores in the log first — OpenClaw's decision-model docs say the same. Laya can only return one of the
options you give it, and it can still return the wrong one: that is what the threshold is for.

MIT License.
