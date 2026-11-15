# AGENTS.md

## Project
Hopscot: a cross-platform software KVM (one keyboard and mouse across machines
over LAN) for a college major project. It is a derivative of lan-mouse
(https://github.com/feschber/lan-mouse). We are NOT rewriting it. We merge a few
small upstream PRs, make small improvements, rebrand it to Hopscot, test on real
hardware and document the results.

License is GPL-3.0-or-later. See Attribution below.
The local folder is ~/Documents/projects/hopscot.

## Machines
- Laptop: Arch Linux, KDE Plasma 6.8.2 (Wayland), xdg-desktop-portal 1.22.1,
  xdg-desktop-portal-kde 6.8.2, libei 1.6.0, InputCapture portal v2.
  Main development and capture side. Codex runs here.
- Win11 and Win10 test boxes. Antigravity runs on all three machines.
- No second Linux machine. Do not set up Linux VMs.

## Build
Linux (KDE), development:
    cargo build --locked --no-default-features \
      --features gtk,libei_capture,libei_emulation,rdp_emulation
Linux (KDE), release:
    cargo build --release --locked --no-default-features \
      --features gtk,libei_capture,libei_emulation,rdp_emulation
Windows (no GUI), development:
    cargo build --locked --no-default-features
Windows (no GUI), release:
    cargo build --release --locked --no-default-features
If feature names fail, read Cargo.toml and report before editing.
Use development builds while developing; release builds for distribution and benchmarks.

## Git rules
- origin = our own repo (revanthlol/hopscot, NOT a GitHub fork).
  upstream = feschber/lan-mouse.
- Fetch upstream PRs with: git fetch upstream pull/<N>/head:pr-<N>-<name>
- One branch per task, named by task (pr-515-persist, sendinput-fix, systemd-unit).
- Never edit the same file from two machines at the same time.
- Do not force-push main. Do not rewrite shared history.
- Commit messages say what changed and why. Credit upstream PRs by number.

## Planned work (small only), in this order
1. Build stock and record baseline behavior (see Testing).
2. Test and merge upstream PR #515 (InputCapture v2 restore_token persistence).
   It stores input-capture.token under $XDG_CACHE_HOME/lan-mouse/ or
   ~/.cache/lan-mouse/ upstream; use hopscot/ after the rename.
3. Test and merge upstream PR #516: fail-fast on a refused SendInput call,
   not bounded retry. Emulation must then be re-enabled.
4. Test and merge upstream PR #533 (outgoing peer fingerprint verification).
   Fingerprint authentication is incoming-only until #533; transport is DTLS over UDP.
5. systemd user unit for autostart on the laptop.
6. Failure-test table: stock vs ours (see Testing).
7. Rename to Hopscot (see Rename). The human authorized preparing this on
   rename-hopscot before items 1-6 are complete; do not merge it into main.

## Out of scope. Do not start these without asking
Clipboard sync, file transfer, mDNS discovery, Windows service or UAC support,
GUI redesign, wire protocol changes, new crypto, rewriting session or zone logic,
macOS and X11 backends, Hyprland or wlroots support.

## Attribution (required, never remove)
- Keep LICENSE (GPL-3.0) and every existing copyright header and notice.
- README must say: Hopscot is based on lan-mouse by feschber (GPL-3.0), and
  list our changes and the upstream PRs we merged.
- Never hide or delete references to upstream in LICENSE, headers, README or
  commit history. A find-and-replace of "lan-mouse" must skip those.

## Rename to Hopscot (single dedicated commit on rename-hopscot)
Binary name: hopscot. Display name: Hopscot.
- Change user-facing names only: root package/binary name, GUI window title and
  strings, GTK application ID (use io.github.revanthlol.Hopscot), desktop file,
  icon names, systemd unit, config and data directory names, log prefixes, README.
- Keep internal workspace crate names (input-capture, lan-mouse-proto, etc.) as
  they are, to keep the diff small and future upstream merges possible.
- First run `grep -rni "lan.mouse\|lanmouse" --exclude-dir=target --exclude-dir=.git`
  and list every hit by category (rename / keep for attribution / internal) BEFORE
  editing anything. Show the human that list.
- If the config directory name changes, say so and give the migration command
  so existing paired devices are not lost.
- Build and run on the laptop and both Windows machines after the rename.

## Hard rules
- Do not change the wire protocol or crypto without asking the human.
- Never invent libei, portal or Win32 behavior from memory. Cite the portal docs
  (flatpak.github.io/xdg-desktop-portal) or the code, or say "unverified".
- Every PR merge needs a test result on real hardware, written down.
- Input capture must always be recoverable. Never ship code that can leave the
  laptop keyboard or mouse captured with no way back. Keep the release hotkey working.
- Do not claim something works unless you ran it. Say what you ran and what you saw.
- After any task, give the human a short summary: files changed, why, how to test.
  The human must be able to explain every change in a viva.
- Keep diffs small. No drive-by refactors, formatting sweeps or dependency swaps.

## Testing
Record results in ../hopscot-notes/ with date, commit hash, machines and network.
Scenarios, stock vs ours:
- Cursor crosses the edge both ways (laptop <-> Win11, laptop <-> Win10)
- Pull Wi-Fi mid-capture: how long until local input returns?
- Sleep and wake a Windows machine while connected
- Plug or unplug a monitor on the laptop while connected
- Restart the app: does the portal permission dialog reappear?
- Elevated window or UAC prompt on Windows (expected: input not delivered)
Note the network setup (Wi-Fi vs Ethernet), since it affects latency numbers.

## Known gotchas
- SendInput cannot reach elevated windows, UAC or the lock screen.
- Windows firewall may block the UDP port (default 4242, confirmed in
  lan-mouse-ipc/src/lib.rs: DEFAULT_PORT).
- Windows cursor can be invisible with no physical mouse attached.
- Barriers exist only on screen edges, so switching is edge-based.

## Upstream development guidelines (carried over)

Adapted from lan-mouse by feschber and its contributors. Upstream AGENTS.md
contributors include Jon Stelly, Ferdinand Schober and Florian S. Kluge;
see preserved upstream commit history (PRs #374, #444 and #490).
Hopscot-specific scope and commands above take precedence.

### Core principles
- Scope discipline: implement only what was requested; report follow-up work.
- Clarify ambiguous OS-specific requirements and document what was actually tested.
- Update README.md or DOC.md when touching public APIs or platform support.
- Use Result/Option, thiserror, descriptive logs, and concise comments for invariants.

### Terminology and architecture
- Client: a configured remote machine. Its active flag enables outgoing capture;
  it is distinct from the currently captured client and the peer's emulation state.
- Backend: an OS-specific capture or emulation implementation.
- Handle: a per-client identifier used to route events and track state.
- Pipeline: input-capture -> service -> lan-mouse-proto/DTLS over UDP ->
  remote service -> input-emulation. lan-mouse-ipc connects local frontends to
  the service (Unix sockets on Unix, loopback TCP port 5252 on Windows).
- input-capture uses ashpd + reis for libei; Windows uses low-level hooks.
  Compiled capture backends are tried in order: libei, layer-shell, X11,
  Windows, macOS. Explicit backend selection skips automatic selection;
  user cancellation stops fallback. Dummy capture is explicitly selectable.
- InputCapture tracks pressed_keys and maps positions to handles; its pending
  queue fans events out when multiple clients share an edge. Preserve its Stream pump.
- input-emulation replays events via consume/create/destroy/terminate, using
  libei, a RemoteDesktop portal fallback (rdp_emulation), or Windows SendInput.
  Destroy/terminate attempt to release held keys; this is not a hardware guarantee.
- input-event owns shared scancodes and event types; reuse these translations.
- lan-mouse-cli and lan-mouse-gtk are local frontends. src/capture.rs owns capture
  lifecycle; connect.rs outgoing DTLS; listen.rs incoming authentication; crypto.rs
  certificate/fingerprint utilities. Do not change protocol or crypto without approval.

### Features and async patterns
- Root Cargo.toml exposes application features; backend crates have their own
  features. build.rs files export OS/feature cfgs such as cfg(layer_shell).
- Prefer module-level gating to per-function stubs.
- New backends require explicit scope approval, feature/cfg wiring, and selection logs.
- Tokio uses existing stream handling, futures and async_trait. Avoid blocking
  the runtime; use spawn_blocking for unavoidable blocking work.

### Development checks
- Run commands from the repository root. Use the scoped build commands above.
- cargo build --locked -p <crate> builds a single crate.
- cargo test --workspace runs workspace tests (default features).
- cargo fmt --check checks formatting without a repository-wide rewrite.
- cargo clippy --workspace --all-targets --all-features checks lints where platform
  dependencies are installed; report missing dependencies rather than changing them.
- LAN_MOUSE_LOG_LEVEL=debug cargo run enables logging before the rename;
  use HOPSCOT_LOG_LEVEL after the rename.
- Add proportional utility/protocol tests for logic changes. Dummy backends can
  exercise limited paths without hardware, not prove OS integration or recovery.
- Test OS backends on their target machines or report manual checks as skipped.
- Verify key release on unexpected disconnect, without assuming it always succeeds.
