# Persistent roles and wake on mention

Opt-in persistent roles keep a stable Buzz identity and channel membership when
no native agent is running. An authorized `@mention` durably queues work, asks a
configured launcher to find/start the role, waits for readiness, and submits the
message once. One bridge daemon serves all roles. Existing live-session mirroring
continues unchanged when no persistent roles are configured.

## Configure

Add tables like `examples/persistent-roles.toml` to your existing config. Keep
credentials in the existing bridge-managed secret source. A role id is also its
identity id: an existing `[identities.<id>]` can be reused, or normal automatic
provisioning creates it. Do not reuse an identity consumed by another listener.
Run `plan --json`, then normal provisioning/reconciliation after reviewing the
mapping. The channel **must already exist**; explicit ids are adopted, never
inferred from labels. Normal membership/ownership permissions are still required.
Offline role profiles use the existing profile/declaration pipeline, so they
appear in Buzz's mention picker without an LLM process. Identity keys stay with
the bridge; changing a runtime never changes the role's reply identity.

`project` and `role` are normalized ids, independent of repo paths, native session
ids and workspace labels. Two projects sharing a repository must have distinct
project ids, directories, identities and channels. Multiple roles of one project
share its explicit channel. Preserve config, secrets, inbox and launcher journals
across restarts. Do not clone them into a second running bridge.

The existing `respond_to` policy applies before enqueue and again before delivery.
Use `owner-only` unless a broader allowlist is intentional. Only structured Nostr
pubkey mentions trigger activation; message text never selects a project,
launcher or command. `since` explicitly enrolls old events; otherwise enrollment
starts at the first poll. A two-second cursor overlap plus a permanent dedup
ledger handles ordinary replay. A full 200-event page raises `history_gap` and
holds the cursor: the current Buzz CLI has no reliable pagination contract here.
An operator must recover older history before advancing enrollment; do not simply
clear state. `status --json` exposes poll errors and pending outcomes.

For a roles-only deployment, set `bridge.exclude_spaces = ["*"]`; role tables
are independent of that filter. This avoids creating channels for project home
labels (which may contain invisible suffixes) and transient worktree Spaces.
For a mixed deployment, retain the usual include/exclude filters. Membership
removal uses the union of bindings sharing a channel; an offline persistent
project prevents that channel being archived when its former Space closes.
On a Herdr snapshot outage the daemon keeps polling roles but skips topology
reconciliation, so it cannot mistake an outage for closed live Spaces.

## Launcher contract, version 1

`launcher` is an argv array, never shell text. Buzzr sends one JSON document on
stdin and reads one JSON response from stdout (55-second timeout, 64 KiB accepted
response). Child errors are reduced to fixed diagnostics; bridge credentials are
removed from its environment. The launcher obtains its own quota credentials
from an external source and must not include them in responses or logs.

Ensure request:

```json
{"version":1,"op":"ensure","project":"demo","role":"studio-lead","activation_id":"demo-lead"}
```

An idempotent ensure must reuse a live session, serialize concurrent activation,
and only consult quota when a new process is needed. Return `ready` with an opaque
`session` object, or `busy`, `starting`, `blocked`. Optional diagnostics: `reason`
(`quota`, `unknown_capacity`, `auth_error`, `compatibility`, `startup`, `busy`, `approval`,
`configuration`, `ambiguous_activation`, `ambiguous_delivery`, `unavailable`),
`route`, `fallback`, `retry_at` (Unix reset time). Status exposes fallback and
unknown outcomes. Busy/startup are retried on later polls; blocked requests wait
at least 60 seconds. No repeated native spawn occurs on known exhausted capacity.

Deliver adds `op: "deliver"`, a stable SHA-256 `request_id`, the returned `session`
and `content` (original message plus project/role context and a one-use reply
command). Revalidate the terminal identity and readiness before writing input.
Return `delivered` only after accepted submission; this acknowledges delivery,
not task completion. `busy`/`starting`/`blocked` mean **definitely not submitted**
and permit retry. Any ambiguous outcome must return `uncertain`. Persist a
request-id ledger before side effects. Reply commands use the bridge's explicit
runtime directory and a scoped token, with no Buzz credential in the prompt.

Terminal input and a local journal cannot form one atomic transaction. Buzzr
writes `sending` before delivery; a crash there becomes `uncertain`, holds later
requests for that role, and posts a fixed notice in the original Buzz thread.
It never claims exactly-once completion across that boundary. A successful reply
using the issued token confirms delivery. Otherwise, inspect the session and use:

```sh
buzzr role-resolve --request REQUEST_HASH --disposition delivered
# Only after confirming the prompt was NOT submitted / activation did not occur:
buzzr role-resolve --request REQUEST_HASH --disposition retry
```

Recovery invokes launcher `op: "resolve"` with `request_id` and `disposition`.
The launcher updates its own ledger first and returns `delivered` or `queued`;
then Buzzr updates its inbox. This is an operator assertion, not an automatic
retry strategy. Never use `retry` to clear an outcome you have not investigated.
Messages are retained on quota, auth/availability, startup and delivery failures.
The permanent ledger grows with received messages; no arbitrary count pruning
can safely replace it. Local state contains message content and is mode 0600.

## Herdr Projects adapter

`scripts/projects_launcher.py --config /absolute/path/launcher.json` implements
this contract using Herdr Projects, not a parallel project scheduler. Python
3.11+ is required. See `examples/projects-launcher.json`.

It checks both Herdr client/server >=0.9.1 and Herdr Projects >=0.2.25,<0.3.0.
The socket is explicit and caller `HERDR_*` context is cleared. One coordinator
binding is allowed per project. Coordinators are discovered in their canonical
project home; workers are discovered from their recorded native project thread.
Multiple matches hold for inspection. Idle/done must be observed across polls;
working queues, blocked/unknown waits for the user, and known trust/hook dialogs
are refused without answering them. Activation/delivery journals use per-role
file locks and atomic writes. Lost activation results hold instead of spawning
another process. No other panes or shared services are stopped.

For an absent coordinator:

1. Call `ag-role ROLE --plan --cwd PROJECT_HOME` from the model-lanes fork. It
   reads the role's ordered external policy and proxy-account capacity.
2. Map the selected route to a configured Herdr Projects profile. Verify its
   native kind and **exact argument vector**, and reject YOLO/legacy extra args.
3. Invoke `herdr-projects --root ROOT open PROJECT --profile PROFILE --tab
   --socket SOCKET`. Existing upstream logic resumes only a matching kind and
   profile; changing provider starts fresh. No native session id is transferred.

Herdr Projects launches canonical `claude`/`codex` commands, not arbitrary
executables. `native_launchers` explicitly attests that Herdr's shell resolves
those commands to the managed launchers used by model-lanes (same proxy account
pool and normal permissions). Verify that setup before enabling. A mismatched
profile or unrecognized executable fails closed. This adapter does not silently
route around the proxy. Model-lanes --plan is available in
https://github.com/terry-li-hm/herdr-model-lanes/pull/1.

`mode: "thread"` uses native `thread start --task-file -` with the existing role
instruction file. Herdr Projects still owns worker placement, task records,
reports and scheduling. Completed/exited workers can get a new thread; project
memory and earlier reports remain in the project. No Claude session id is passed
to Codex. An unresolved activation is retained for operator reconciliation.

Worker cold starts default to blocked. Upstream
[issue #63](https://github.com/eliasstravik/herdr-projects/issues/63) describes
pending briefs submitted before startup is safely recognized; set
`worker_startup_verified: true` **only after** checking/fixing that on the deployed
Herdr/native versions and trusted working directories. A coordinator runtime
adaptation should also keep trust decisions with the user, overriding the advice
in the upstream coordinator skill ([#62](https://github.com/eliasstravik/herdr-projects/issues/62)).
The adapter's screen guard protects its own delivery; it cannot intercept the
upstream ticker's initial worker brief.

Create project homes with native `herdr-projects new`, and add role references,
project context and runtime adaptation to PROJECT.md. Leave the shared role
library intact. Different providers read the same durable project context and
reports; this is not cross-provider conversation resume. Existing project
coordinator tools must call this adapter's ensure/deliver JSON interface for
quota-aware role workers. Raw `hp open`, `hp thread start/restart`, Herdr UI
launches, and arbitrary shell commands are **not globally intercepted**. Likewise,
existing Herdr Projects schedules do not gain wake-on-mention behavior from this
plugin; Buzz mentions use the adapter directly.

Upstream `open --tab` focuses the created/reused project Space and applies its
normal coordinator name. This adapter does not add session handoff or rename
other agents. Native Herdr recognition remains intact, and Buzz identity does
not depend on activity-based agent names. Herdr Projects issue
[#64](https://github.com/eliasstravik/herdr-projects/issues/64) can group worktree
Spaces of separate projects sharing a repo under the wrong sidebar parent;
explicit project/channel ids prevent cross-project message delivery regardless.

## Validation and rollout

Run `cargo fmt --check`, `cargo clippy --all-targets --all-features --locked -- -D
warnings`, `cargo test --all-targets --locked`, and `python3 -m unittest discover
-s scripts -p 'test_*.py'`. Tests use fake relay/native surfaces: they cover offline
roles, owner checks, replay, busy queues, failures, crash ambiguity, stable replies,
project isolation, profile/quota gates and concurrent activation locks.

Build with `cargo build --release` and use a linked development checkout when
ready to deploy. Installing a fork by GitHub reference before it publishes its
own release binaries can run the old released bridge instead. Review the
existing daemon/other consumers and migrate **one** bridge instance; do not run a
second listener against the same identities. Do not restart a shared Herdr server
as part of this plugin rollout. Complete one owner-authored offline mention →
native readiness → reply round trip on a compatible host before calling the
production integration verified.
