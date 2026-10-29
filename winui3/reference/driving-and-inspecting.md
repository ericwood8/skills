# Driving, inspecting and seeing a WinUI3 app

Read this when: you drive the running app with UI Automation, look at a screen, or need a control's real default template.

The step-by-step click-through against a scratch database (launch the app, point it at the copy, read the database back, clean up) is in the `verify-in-running-app` skill, `reference/desktop-winui3.md`. This file holds the UI Automation techniques and the control and dialog traps.

## Find a control's real template

When a default-template assumption fails, search the SDK's own `generic.xaml` instead of guessing: `find ~/.nuget/packages/microsoft.windowsappsdk.winui -iname generic.xaml` (the `lib/native/Microsoft.UI/Themes/generic.xaml` one). Find `x:Key="Default<Control>Style"`, then the `ControlTemplate`, and read the `Setter`/`TemplateBinding`/literal values on the named parts.

## Driving the running app (UI Automation)

- `System.Windows.Automation` works against an unpackaged WinUI3 app. From PowerShell: `Add-Type -AssemblyName UIAutomationClient, UIAutomationTypes`, `Start-Process -PassThru`, then `AutomationElement.RootElement.FindFirst(Children, PropertyCondition(ProcessIdProperty, $proc.Id))`; find by `NameProperty`/`ControlTypeProperty`; `InvokePattern` clicks; `ValuePattern.Current.Value` reads a TextBox's content (`Current.Name` is only its label). The harness is in `Docs/Verification` (`UiAutomation.psm1`).
- `AutomationElement.FocusedElement` is system-wide: bring the window forward (`user32!SetForegroundWindow`) and check `Current.HasKeyboardFocus` on a specific element. Never use `SendKeys` (it types into whatever window the person is using); use `InvokePattern`, `ValuePattern`, `TogglePattern`.
- `ScrollPattern.SetScrollPercent(-1, 100)` scrolls a dialog to its end. A UIA `BoundingRectangle` is the visible (clipped) rectangle, which is how a cut-off row shows in numbers. `CopyFromScreen` screenshots include any window in front.
- `Find-Name 'Rows per page'` returns the TextBlock beside the box; get the box with `@(Find-Type 'ComboBox')[0]`; the page label is `Find-Like 'Page * of *'`; wait about 3 s after each change.

## Control and dialog traps when driving the app

- Use the exe in `bin\x64\Debug\...` (search `bin` for `*.exe`; the name may differ from the project's), not a stale `bin\Debug` one.
- ComboBox: `SelectionPattern.Current.GetSelection()` gives the selected item (the `ValuePattern` value is empty); `ExpandCollapsePattern.Expand()` (the harness helper is `Expand-El`) then `FindAll(Descendants, ListItem)` lists options. TabItem: `SelectionItemPattern.Select()`, then list `Edit` controls by `Name`.
- Dialog traps: a TextBox commits on focus loss (`SetValue`, then `SetFocus` on the Search button, then `Invoke`); a grid far down a tall dialog has an empty `BoundingRectangle` until you `SetScrollPercent(NoScroll, 100)` its scroll ancestor (`Invoke` works off-screen, a real click does not); a header `Button`'s `Name` includes the sort arrow (`Customer PO ▼`), so match `-like "Customer PO*"`; a ContentDialog that keeps itself open can be clicked again.

## See a screen

- **See a screen:** open it with UI Automation (a navigation item is `SelectionItemPattern`, a button `InvokePattern`; the list page's Edit button opens the detail dialog), then save the window to a PNG with `System.Drawing` `Graphics.CopyFromScreen` over the window's `BoundingRectangle` and open the file with the Read tool. A dialog is not a child of the main window's element tree in a stale handle: find the window again by process id after it opens.
