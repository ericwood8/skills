---
name: ui-conventions
description: Personal desktop-dialog and toolbar UX conventions — keyboard access keys with conflict checking, Escape/Enter behavior, default-button focus, first-control focus on toolbars, tooltips on every toolbar button, and colorblind-safe status icons (shape-coded, not just color-coded) for success/warning/error messages. Framework-agnostic (applies to WPF, WinUI3, WinForms, or any desktop UI); see the winui3 skill for WinUI3-specific implementation techniques. Use when building or reviewing any dialog, form, toolbar/menu bar, or status/result message in a desktop app.
---

# Desktop UI conventions

Apply these to every screen in every desktop app, not just one project. They come from concrete requests
made across a real project (CodeGenNew) after the first pass shipped without them.

## 1. Keyboard access keys (hotkeys) on buttons

- Give every dialog's footer buttons a mnemonic letter: **C** for Cancel or Copy, **O** for OK/Open/Done,
  **S** for Save. Pick a different, non-conflicting letter for anything else on the same screen (e.g. a
  "Test" button can take **T**).
- **Always check for conflicts within the same screen/scope before assigning a letter.** Two controls
  that are both visible and focusable at the same time must never share a mnemonic. A modal dialog is
  its own scope — its access keys don't need to avoid the letters used by the window underneath it, since
  the OS/framework's access-key manager scopes to whatever's currently active (the dialog shadows the
  parent window's own access keys while it's open).
- The convention is a leading `&` (or the framework's equivalent, e.g. WinUI3's `AccessKey` property) on
  the mnemonic letter in the caption, e.g. `&Cancel`, `&OK`, `&Save`.
- **The platform's own default is to underline the mnemonic only while Alt is held down** (true of Win32,
  WPF, and WinUI3 alike) — that's expected, not a bug, and matches how every native Windows app already
  behaves. If a permanently-visible bold/underlined letter (not just on Alt-hold) is specifically wanted,
  that requires replacing the framework's built-in string-based button captions with custom button content
  built from separate text runs (one bold, one normal) — a bigger, separate change from just wiring up the
  access key itself. Don't conflate the two asks; implement the access key first, then ask before doing
  the heavier custom-content rework.

## 2. Escape = Cancel, Enter = Save/OK

Most native dialog frameworks (WPF `Window`/WinUI3 `ContentDialog`) already give you this for free:
- Escape always triggers the Cancel/Close action.
- Enter triggers whichever button is marked as the dialog's default button — including when focus is
  inside a TextBox, which is exactly the scenario a form-style dialog needs (type values, press Enter to
  submit).

Check whether the framework already does this before writing custom `KeyDown` handling. Reimplementing it
by hand risks subtly diverging from the platform convention users already expect (e.g. handling Enter only
when a Button has focus, missing the TextBox case).

## 3. Default-button focus

When a dialog opens, keyboard focus should already be on whichever button is marked as the default one
(the one Enter would trigger) — not on the first input field, and not nothing. For a dialog with only one
button (a plain info/confirmation dialog), that single button gets the initial focus.

This has to be done explicitly in most frameworks — a dialog's default-button styling (bold border, etc.)
does not automatically mean it holds keyboard focus. Set focus programmatically when the dialog opens/loads.

## 4. First-control focus on a toolbar/menu bar

The top-level command bar/toolbar/menu bar of the main window should give keyboard focus to its first
button as soon as it's shown, so a keyboard-only user can immediately Tab or Enter without first having to
click into the toolbar. Do this once, when the toolbar first loads.

## 5. Tooltips on every toolbar button

Every button on a top-level toolbar/command bar should have a tooltip explaining what it does — standard
practice, not optional polish. Include the access key in the tooltip text (e.g. "Connect to a SQL Server
database (Alt+C)") so the keyboard shortcut is discoverable without needing to hold Alt first.

## 6. Status icons: shape-coded, not just color-coded

Any success/warning/error message — a result popup, an inline validation message, a status bar entry —
gets an icon whose *shape* carries the meaning, not just its color. Color alone (a green dot vs. an amber
dot vs. a red dot) is unreliable for colorblind users (protanopia/deuteranopia make red/green/amber hard to
tell apart at a glance) and is the reason this convention exists at all — it's the same fix
[microsoft/skills#398](https://github.com/microsoft/skills/pull/398) made to a review-output legend, applied
here to actual app UI instead of markdown text.

| Meaning | Shape | Color | Asset |
| --- | --- | --- | --- |
| Pass / success / yes | Rounded square + checkmark | Green | [assets/status-success.svg](assets/status-success.svg) |
| Needs attention / warning / caution | Triangle + `!` | Amber/yellow | [assets/status-warning.svg](assets/status-warning.svg) |
| Blocking issue / error / no | Circle + X, on a light/white background | Red | [assets/status-error.svg](assets/status-error.svg) |

These three SVGs are simple, flat, single-color icons deliberately kept easy to re-theme (swap the fill/
stroke colors) or redraw at a different size — they're a starting point, not a locked design. See the
winui3 skill for how to wire this into an actual WinUI3 dialog (prefer the built-in `InfoBar` control's
`Severity` property over loading these as custom images when the framework already gives you this for
free).
