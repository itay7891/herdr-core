# Private macOS ↔ Ubuntu lab over Tailscale

This is a **manual, opt-in deployment recipe**, not an enforcement feature or a completed security test. It targets the user's private Mac and Ubuntu lab, and does not require a public listener.

## Security boundaries

- `hc` exposes a **local Unix socket** on macOS/Linux. Do not forward it or publish it on TCP.
- Remote machine connections use **SSH over Tailscale**. Tailscale provides the private network path; SSH authenticates the host/user and carries the hc protocol.
- Do **not** expose SSH on a public interface. Confirm firewall policy and Tailscale ACL/grants before adding the machine. A Tailscale address alone does not prevent a separately exposed public SSH listener.
- Core and all agent programs inherit the remote Unix account's permissions. Use a dedicated **non-sudo** test user with no production credentials, no access to other users' homes, and only synthetic test repositories. `HC_SESSION` separates sessions, **not permissions**.
- Do not install unreviewed JavaScript mods. The GUI's `HERDR_MODS=0` switch is a temporary restriction, not a sandbox; some UI will be missing.
- Do not run `curl | sh`, `hc update`, `hc machine update`, or `hc machine add --install` until the binary, version, supply chain, and destination are reviewed. The remote update/install commands can copy executable code.
- Do not assume an AgentSentry MCP proxy sees direct `hc exec`, shell commands, or unrestricted file access.

## Prerequisites

1. Install Tailscale on both machines and enroll both in the intended tailnet.
2. Configure Tailscale ACL/grants to allow **only the Mac identity** to reach the Ubuntu lab machine's SSH port. Verify this with the tailnet admin configuration.
3. Configure SSH on Ubuntu to accept the dedicated test user only through the approved path; use key-based authentication and verified SSH host keys. Verify `sshd` binding and host firewall rules. No router port forwarding.
4. Build the **same audited `herdr-core` revision** locally for each OS. `itay7891/herdr` (Herdr 1 TUI) is not a compatible `hc` core. Confirm both binaries report `hc ...` and verify their hashes/build provenance.
5. Keep both machines' clocks synchronized; do not copy private keys into repositories.

## Mac SSH config example

Use the real Tailscale MagicDNS hostname or 100.x tailnet IP of Ubuntu; do not use a public hostname. Substitute the dedicated account and your SSH key:

```sshconfig
Host itayx-herdr-lab
    HostName REPLACE_WITH_UBUNTU_TAILSCALE_HOSTNAME
    User REPLACE_WITH_DEDICATED_NON_SUDO_USER
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    StrictHostKeyChecking yes
    BatchMode yes
```

Test the SSH path **before** registering it with hc:

```sh
tailscale status
ssh itayx-herdr-lab 'id -un; command -v hc; hc --version'
```

The first SSH connection may require interactive host-key verification; do that separately and compare the fingerprint through a trusted channel before enabling `BatchMode yes`. Never bypass host-key checks to make the test pass.

## Core connection (Mac)

Use a disposable test user, an isolated configuration/state home, and a non-production repository. Run `hc` from the verified local build:

```sh
hc --version
hc machine add itayx-herdr-lab --name ubuntu-lab
hc machine ls
hc exec --on ubuntu-lab -- id -un
hc run --bg --on ubuntu-lab --name hello -- sh -lc 'printf "herdr lab ready\\n"'
hc ls
```

Do not use `--install` in the machine-add command. If hc reports that the remote binary is missing/incompatible, **stop**, review the remote binary and install it manually from the pinned source. Do not approve an automatic copy before verifying its destination.

The `hc exec` test proves remote execution works, **not** that it is sandboxed or that all traffic is restricted to Tailscale. Verify actual SSH destination, listening ports, and firewall rules separately.

## GUI

Build `itay7891/herdr-app` from its own reviewed branch, and point `HERDR_CORE` to the verified compatible local `hc` binary. Do not use legacy Herdr TUI as the GUI's core. Run GUI with a scratch `HERDR_APP_HOME` first. Do not attach sensitive repositories until mods and the local control socket have been reviewed.

## Exit criteria

- [ ] Both Mac and Ubuntu use compatible pinned hc revisions
- [ ] No public SSH exposure; tailnet policy verified
- [ ] SSH host key verified and dedicated account has no sudo
- [ ] Core sockets/state accessible only to their intended Unix owner
- [ ] GUI connects to the correct hc binary, no download/update side effects
- [ ] Synthetic agent-to-agent handoff and reconnect tested
- [ ] Network egress observed and documented
- [ ] Independent review of mods, socket control and remote file operations
- [ ] AgentSentry tests include direct-shell bypass paths and explicitly document uncovered operations

**Do not treat these checkboxes as passed until actual tests are run on the target machines.**
