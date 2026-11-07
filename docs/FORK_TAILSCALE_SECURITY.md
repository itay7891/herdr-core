# Fork deployment policy: Tailscale-first Herdr Core

**Status:** Initial security review, not an enforced network policy. This file is guidance only. Do not treat this branch as production hardened.

## Verified from upstream source

- `hc` runs arbitrary commands with the operating-system permissions of its account.
- On Unix, the API listens on a local Unix socket, not a public TCP listener. Existing state directories are not automatically permission-repaired.
- SSH transports the remote core protocol. Adding a machine can copy and execute a core binary on the remote machine, after an interactive prompt (or without one with `--install`).
- `HC_SESSION` separates state but does **not** isolate privileges; the same OS account can access sessions.
- Local socket clients and trusted mods can command the runtime; there is no per-client policy broker.
- `hc update` uses HTTPS and a checksum from the release service, not a separately signed manifest.

## Proposed Mac ↔ Ubuntu topology

Mac Herdr App → locally built `hc` → SSH **over Tailscale** → Ubuntu `hc` → agents in isolated Git worktrees.

Recommended controls before production:

1. Use a dedicated unprivileged Ubuntu OS account for agents. Never run hc as root.
2. Use a specific SSH Host alias pointing to the Ubuntu machine's Tailscale address or MagicDNS name, with `IdentitiesOnly yes`, `StrictHostKeyChecking yes`, and a dedicated key. Verify the host key out of band.
3. Use Tailscale grants/ACLs to allow only the intended Mac identity/device to reach the Ubuntu SSH service. Limit the Ubuntu SSH listener with host firewall rules to `tailscale0` as appropriate; verify without locking out administrative access.
4. Do not forward raw hc Unix sockets, expose them over TCP, or place sockets in shared directories. Audit permissions of existing state/config directories and protect saved terminal output.
5. Disable unreviewed mods and avoid shell access to secrets. Mod sandboxing and per-command authorization are **not** implemented by this policy.
6. Pin reviewed builds and do not use unattended `hc update`, auto-install or `hc machine add --install` in production. Ensure both machines have compatible, locally reviewed builds.
7. Use a scratch `HC_SESSION`, scratch XDG directories, fake credentials, and a disposable Git repo for integration tests. Do not mistake sessions for a sandbox.
8. Treat direct shell/network commands as outside AgentSentry MCP enforcement unless an independently tested OS-level control mediates them.

## Acceptance tests to implement

- Verify GUI finds the compatible `hc` binary and can attach/read/resize local terminal.
- Verify remote connection succeeds through Tailscale SSH with host-key verification and no public SSH exposure.
- Verify denied source cannot SSH to Ubuntu under Tailscale policy; verify TCP/SSH not reachable from the public interface.
- Verify a compromised same-user process can access the Unix socket; mitigate by account isolation rather than claiming socket password protection.
- Verify remote binary installation/update is not silently triggered.
- Verify agent delegation, blocked/idle status, reconnect, and cancellation using test projects.
- Verify network egress of GUI/core/mods; no unapproved telemetry or updater requests.

## Current limitations

No host firewall, Tailscale ACL, mod sandbox, or runtime command restriction has been installed. This is an auditable deployment checklist for implementation and verification.
