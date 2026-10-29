# Desktop (WinUI3) apps

Read this when: you click through a WinUI3 app against a scratch database, or a dialog or grid does not respond to the script.

The UI Automation techniques, the control and dialog traps (ComboBox, TabItem, TextBox focus, scrolling a tall dialog, sort-arrow button names) and the screenshot method are in the `winui3` skill, `reference/driving-and-inspecting.md`. This file is the verification procedure.

- Drive with UI Automation from PowerShell, not screenshots: `Start-Process` the built exe (set password environment variables in the same script; use the x64 exe, see the `winui3` skill), wait about 6 s, find the window by process id, find controls by `NameProperty`, invoke or select them, wait about 4 s, list `Descendants` names. A loaded grid shows its row cells; an empty one only headers and "Page 1 of 1". Kill the process at the end. A first launch can show an empty grid if a previous instance is still shutting down: rerun before concluding.
- Read a cell after a formatting change (the `CurrencyFormatter` detail is in the `winui3` skill, `reference/controls.md`).
- A WinUI3 app reads `appsettings.json` from `bin`: edit that copy to point at the scratch database and put it back in `finally`.
- Add a test that parses generated XAML as XML (`XDocument.Parse`) when a template has never been built in a sample.
