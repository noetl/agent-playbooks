# agent-playbooks

Playbooks that use [NoETL](https://github.com/noetl/noetl) as a local AI agent framework: a scheduled Claude agent on your own machine, written as YAML playbooks and run by the `noetl` CLI.

Documentation: [Tutorial](https://github.com/noetl/agent-playbooks/wiki/Tutorial) · [Architecture](https://github.com/noetl/agent-playbooks/wiki/Architecture) · [Configuration Reference](https://github.com/noetl/agent-playbooks/wiki/Configuration-Reference) · [Security](https://github.com/noetl/agent-playbooks/wiki/Security) · [Troubleshooting](https://github.com/noetl/agent-playbooks/wiki/Troubleshooting)

> **Requires noetl 5.1.0 or newer**, which ships `kind: agent` (noetl-tools 4.1.0). Install from apt, Homebrew or crates.io; see the [Tutorial](https://github.com/noetl/agent-playbooks/wiki/Tutorial#1-get-a-noetl-cli-with-kind-agent).

## How this differs from an always-on agent such as OpenClaw

| | always-on agent (e.g. OpenClaw) | these playbooks |
|---|---|---|
| Process | daemon running 24/7 with a heartbeat | nothing runs between jobs; a systemd timer starts `noetl run` |
| Agent definition | prompts + installed skills | a YAML playbook you can read and diff |
| What the model can touch | whatever the installed skills allow, often a shell | only the tools the playbook declares; each tool is another playbook |
| Cost when idle | heartbeat calls | zero |
| Billing | API key | API key (`framework: claude`) **or** your Claude subscription through the Claude Code CLI (`framework: claude-code`) |

It does not replace a chat-first assistant you message all day. It covers scheduled, bounded jobs: triage, digests, reports.

## Two ways to run Claude

| | `framework: claude` | `framework: claude-code` |
|---|---|---|
| Runs | Anthropic Messages API, called directly from Rust | the official `claude` CLI, headless (`claude -p`) |
| Billing | API credits (console.anthropic.com) | the account the CLI is signed in to, including Pro/Max |
| Tools | `tools:` entries, each bound to a playbook; NoETL runs every call | Claude Code's own tools, granted with `allowed_tools` / `mcp_config`; default none |
| Credential | `auth: {type: api_key, credential: anthropic}` | none in the playbook; uses the CLI sign-in |
| Bound | `max_turns` (default 8) | step `timeout` (default 300 s) |

## Quick start

1. Install noetl 5.1.0 or newer: `brew tap noetl/tap && brew install noetl`, the apt repo, `cargo install --bins noetl`, or `pip install "noetl>=5.3.2"` (details in the [Tutorial](https://github.com/noetl/agent-playbooks/wiki/Tutorial#1-get-a-noetl-cli-with-kind-agent)).
2. Pick a billing mode:
   - **API:** create a key at console.anthropic.com → API Keys, add credits, and store it:
     ```bash
     install -d -m 700 ~/.config/noetl
     ```
     Then put the key (starts with `sk-ant-api`) on one line in `~/.config/noetl/anthropic.key`, mode `600`.
   - **Subscription:** install the Claude Code CLI and run `claude` once to sign in.
3. Run a playbook:
   ```bash
   NOETL_CREDENTIAL_ANTHROPIC="$(cat ~/.config/noetl/anthropic.key)" noetl run playbooks/inbox-triage/triage_api.yaml --runtime local --set account=work
   ```
   ```bash
   noetl run playbooks/inbox-triage/triage_subscription.yaml --runtime local --set account=work
   ```

Both read stub mail from `playbooks/inbox-triage/tools/list_unread.yaml`, so they run without mailbox credentials. A run of `triage_api.yaml` uses about 1,100 input and 170 output tokens.

## Your real Gmail

`playbooks/gmail-triage/triage.yaml` triages several real Gmail accounts, read-only, with one OAuth credential per account. It needs noetl 5.3.0 or newer (`noetl credential`, `google_oauth` auth, and the `thinking: disabled` cost control). Setup: [Connect Gmail](https://github.com/noetl/agent-playbooks/wiki/Connect-Gmail).

```bash
noetl credential google gmail-work --client-secret-file ~/.config/noetl/google-client.json
```

```bash
noetl run playbooks/gmail-triage/triage.yaml --runtime local --set accounts=work,personal,family
```

## Run it every morning

`systemd/` has a user service and timer (07:30 daily, catches up after the machine was off):

```bash
mkdir -p ~/.config/systemd/user && cp systemd/inbox-triage.* ~/.config/systemd/user/
```

```bash
systemctl --user daemon-reload && systemctl --user enable --now inbox-triage.timer
```

Read results with `journalctl --user -u inbox-triage.service`. To run it while logged out: `sudo loginctl enable-linger $USER`.

## Layout

```
playbooks/
  gmail-triage/
    triage.yaml               # real Gmail, several accounts; see wiki "Connect Gmail"
    tools/list_unread.yaml    # http + auth: google_oauth, credential gmail-<account>
    tools/get_message.yaml    # From/Subject/Date/snippet only (format=metadata)
  inbox-triage/
    triage_api.yaml           # framework: claude — model calls tools, NoETL runs them
    triage_subscription.yaml  # framework: claude-code — NoETL fetches, Claude reads
    tools/list_unread.yaml    # agent tool (stub); swap in Gmail API http steps
systemd/
  inbox-triage.service
  inbox-triage.timer
  gmail-triage.service
  gmail-triage.timer
```

## Credentials and safety

- **No ambient keys.** A `framework: claude` step must name a credential alias. In local mode the alias `anthropic` is read from `NOETL_CREDENTIAL_ANTHROPIC`; an unset or empty value stops the step before any API call.
- **`claude-code` cannot fall back to API billing.** NoETL removes `ANTHROPIC_API_KEY` and `ANTHROPIC_AUTH_TOKEN` from the CLI's environment.
- **Model output is data.** Values the model passes to a tool playbook are never rendered as templates, so text quoted from an email cannot read your playbook variables.
- **Tools are opt-in.** `claude-code` steps get no tools unless `allowed_tools` lists them.

## License

Apache-2.0, see [LICENSE](LICENSE).
