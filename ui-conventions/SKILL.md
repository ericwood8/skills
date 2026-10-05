---
name: ui-conventions
description: Personal desktop-dialog and toolbar UX conventions — keyboard access keys with conflict checking, Escape/Enter behavior, default-button focus, first-control focus on toolbars, tooltips on every toolbar button, and colorblind-safe status icons (shape-coded, not just color-coded) for success/warning/error messages, and input-control choice rules (check box for true/false, radio buttons for up to three choices, drop-down for more, number box for whole numbers, labels not text boxes for read-only text, no more than 20 fields on a tab) so users cannot type bad values. Framework-agnostic (applies to WPF, WinUI3, WinForms, or any desktop UI); see the winui3 skill for WinUI3-specific implementation techniques. Use when building or reviewing any dialog, form, toolbar/menu bar, or status/result message in a desktop app.
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

## 7. Pick the input control that cannot take a bad value

Apply to every screen, dialog and settings page in every project. A free-text box is the last choice,
for values that really are free text. If the valid values are known, the control offers them, so the user
cannot mistype one.

| The field holds | Use | Notes |
| --- | --- | --- |
| true / false | **Check box** | Caption is the field name as a question ("Dashboard?"). Checked writes `true`, unchecked writes `false`; never ask the user to type "true". |
| One of 2 or 3 fixed values | **Radio buttons** | Include a first choice "Not set (the default)" that writes a blank when blank is a valid value. |
| One of more than 3 fixed values, or choices with long descriptions | **Drop-down** (combo box, not editable) | Same "Not set" first choice. |
| Any of a fixed list (a comma-separated list of known names) | **Check boxes**, in a drop-down button's flyout when the list is long | The button shows what is ticked ("Api, WinUI3") or a prompt when none is. Single-choice drop-downs are wrong here: the user usually needs several. Keep names the list does not know when saving, so nothing is silently lost. |
| A whole number (year, port, count, version) | **Number box** with a minimum and maximum | Empty must stay possible when blank means the default. Ports 1 to 65535. |
| A value from the database (a table or column name) | A list picker over the real names, when a connection exists | Typing names is the fallback. |
| Real free text (a name, a path, a namespace) | Text box | |

- A value already stored that is not on the list (a hand-edited file) must survive: show no selection and
  keep the stored text until the user picks something else.
- Keep the list of valid values in one place in the non-UI layer (a table of key, kind, choices) so tests
  can check it against the code that parses the value, and so every screen offers the same list.
- Match case when loading a stored value ("sqlserver" selects "SQL Server"); write the canonical spelling.

## 8. Explanatory text is a label, never a text box

A text box looks like something to type in and takes a tab stop. Read-only explanatory text (what a tab
holds, what a field means, a hint under a field) is a label / text block (in WinUI3 a `TextBlock`, inside a
`Border` panel when it needs a background), which is not focusable and cannot be edited. Do not use a
read-only, disabled or placeholder-filled text box for it. Do not put the explanation inside the value:
a placeholder like "true: the plan also writes ..." makes the user wonder what to type. Caption, then the
explanation as its own small line, then the control.

- A tab description starts "This tab has ..." and is plain weight, not bold.
- Black text needs a light panel behind it; the screen's theme may be dark, so check contrast.

## 9. Captions have spaces between the words

A caption made from an identifier ("DashboardStrip", "NoApiTables") is shown with the words apart
("Dashboard Strip", "No Api Tables"), the way generated screens already caption column names. Use the
same word-splitting function for every screen so they agree.

## 10. At most 20 fields on a tab; group by what the user is deciding

A settings screen with one long tab is hard to scan. Split by topic (General, Namespaces, Tables,
Database, API, Output, Build ...), at most 20 fields per tab, each tab opening with its "This tab has ..."
description. A field no tab lists falls into the first tab so a new setting is never hidden, and a test
asserts that every setting is on exactly one tab and no tab is over the limit.

## 11. A setting the user turns on must say what else they have to do

A feature switch that only takes effect on the next generate, restart or rebuild says so in its own
explanation line. Saving a setting does nothing by itself, and a user who ticks a box and sees no change
assumes the feature is broken. Where a control depends on another (a "Stacks" list that decides what is
generated), say so beside it, and state what a blank value does.
