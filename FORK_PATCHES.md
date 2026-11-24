# Fork-owned divergences

This file records intentional behavior owned by `janbam/herdr`, relative to `herdrdev/herdr`. Upstream version lag and temporary experiments are not fork patches. Read the integration and maintenance rules in [AGENTS.md](AGENTS.md#fork-owned-behavior-in-janbamherdr) before integrating upstream.

## Active patches

### legacy-shift-enter: Opt-in legacy host key interpretation

**Tracking:** [janbam/herdr#1](https://github.com/janbam/herdr/issues/1). Owned by this fork, not an upstream submission.

**Purpose:** Keep physical Shift+Enter usable as newline input in Pi when the host sends `ESC CR`. Captures from GNOME Terminal through tmux 3.4 produced the same bytes locally and over SSH: Shift+Enter was `ESC CR`, plain Enter and Ctrl+Enter were `CR`, and physical Alt+Enter was `ESC 01`. Requesting modifyOtherKeys Mode 2 did not change those captures. SSH environment detection was therefore not a sufficient fix for this setup.

The Pi fork accepts raw `ESC CR` as a legacy Shift+Enter alias. Herdr normally interprets it as Alt+Enter and re-encodes it for a negotiated application pane, activating Pi's Alt+Enter follow-up binding instead. The fork provides an explicit host interpretation policy rather than globally redefining conventional Alt+Enter.

**Configuration:** Add the setting to the TUI client's config, in an existing `[terminal]` section if present:

```toml
[terminal]
legacy_shift_enter = true
```

**Protected behavior:**

- Default `false` preserves conventional decoding. When enabled, only the exact original host bytes `ESC CR` become semantic Shift+Enter, before Herdr shortcuts, input leases, and pane encoding.
- Plain Enter, LF/Ctrl+J, other host bytes, explicit Kitty and modifyOtherKeys reports, and pasted text retain their existing interpretation. Enabling the setting intentionally gives up conventional Alt+Enter for the ambiguous `ESC CR` sequence.
- The setting comes from the client's local config, including with endpoint-provided keybindings. Valid config reloads apply it; an invalid `[terminal]` section keeps the previous value.
- Pane encoding still follows the application's live protocol. Kitty and modifyOtherKeys Mode 2 panes receive their corresponding Shift+Enter reports. Legacy shell panes retain the existing plain-Enter encoding for Shift+Enter, not an invented newline. Direct attach remains byte-transparent.
- Do not infer this mapping from `TERM`, `TMUX`, SSH, or missing Kitty support. Do not replace it with unconditional raw-byte forwarding or require a Pi or terminal-configuration change.

**Implementation:** `TerminalConfig` in [src/config/model.rs](src/config/model.rs), client-local configuration in [src/client/shell/state.rs](src/client/shell/state.rs) and [src/client/shell/config.rs](src/client/shell/config.rs), and host normalization in [src/client/shell/input.rs](src/client/shell/input.rs). The default config example lives in [src/main.rs](src/main.rs).

**User documentation:** [Keyboard guide](docs/next/website/src/content/docs/keyboard.mdx#legacy-shiftenter-in-this-fork) and [config reference](docs/next/website/src/data/config-reference.json). Keep the Keyboard guide's translated counterparts aligned.

**Regression checks:** Run `just test-one legacy_shift_enter` and `just test-one client::shell::config::tests`. The existing [input conformance tests](src/client/shell/tests/input_conformance.rs) exercise actual host framing, client dispatch, server pane input, and live pane encoding, including split reads, protocol transitions, opt-in versus default behavior, neighboring inputs, paste, and shortcut dispatch. Jan also confirmed that the opt-in fixes the live Pi workflow. Repeat that physical check when an upstream integration changes the host input or pane encoding paths.

**Upstream integration and retirement:** Preserve this contract if upstream moves or rewrites its implementation. An upstream negotiation improvement alone is not an equivalent replacement: the captured `ESC CR` alias must still reach a negotiated Pi pane as Shift+Enter when opted in. Retire this patch only after an upstream replacement preserves the opt-in behavior and regression checks, or Jan explicitly approves changing the contract. Record the replacement reference and any configuration migration here when retiring it.
