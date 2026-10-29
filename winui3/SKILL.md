---
name: winui3
description: Use when building, debugging or reviewing a WinUI3 / Windows App SDK unpackaged desktop app: build errors (MSB3027, WMC9997, WMC0001, CsWinRT1028), CommunityToolkit.Mvvm properties, ContentDialog problems (only one open, cannot reshow, width capped, dialog crashes on open), ComboBox, ListView, NumberBox, CalendarDatePicker, TabView, InfoBar, grid and toolbar styling, paging bar, and driving the app with UI Automation.
---

# WinUI3 (Windows App SDK)

This file holds the rules that apply to almost every WinUI3 task: build and project, MVVM, ContentDialog, and the resource trap that crashes a dialog on open. Read the file below only when your task matches it.

## Pick the file

| You are about to... | Read |
|---|---|
| Use or fix a control (ComboBox, ListView, NumberBox, CalendarDatePicker, TabView, TreeView, icons, AppBarButton, MenuFlyoutItem), or choose the input control for a field | [controls](reference/controls.md) |
| Style a CRUD grid, its toolbar, search bar or paging bar | [grid-and-toolbar-styling](reference/grid-and-toolbar-styling.md) |
| Drive the running app with UI Automation (patterns, control and dialog traps), take a screenshot of a screen, or read a control's default template | [driving-and-inspecting](reference/driving-and-inspecting.md) |

Related skills: `ui-conventions` (what the dialogs and controls should do), `verify-in-running-app` (the click-through procedure), `msbuild`, `self-contained-deployment`, `codegen-t4-templates`.

## Build and project

- **XAML comments cannot contain `--`** (it fails the XAML compiler with `WMC9997`/`WMC9999`, also from a code generator's own template comments). Use a colon, comma or a real em dash. Same trap in `.csproj` (see the msbuild skill).
- **Rebuild fails with `MSB3026`/`MSB3027` ("cannot access the file ... used by another process") when the app is running.** Not a code problem. Do not kill the process without asking (it may be the user's session); ask them to close it, then rebuild.
- **Build the x64 output before driving the app:** `dotnet build <App>.csproj -p:Platform=x64`. A default-platform build leaves `bin\x64\Debug\...` stale and the stale exe shows an old UI.
- **`WindowsAppSDKSelfContained` is separate from `SelfContained`** (see the self-contained-deployment skill): set the first `true` and the second `false`. With `SelfContained=false` use `<RuntimeIdentifier>win-x64</RuntimeIdentifier>` (singular), not `RuntimeIdentifiers`, or the build fails with "WindowsAppSDKSelfContained requires a supported Windows architecture".
- **`xmlns:` prefix collisions (`WMC0001: Unknown type 'X' in XML namespace 'using:...ViewModels'`)**: one prefix, one namespace per file; the message names the wrong namespace, which is the clue. Declare a second prefix (`xmlns:views="using:MyApp.Views"`).
- **`CsWinRT1028` ("class is not marked partial")** for a type reachable from WinRT types, such as a `DbContext` subclass: declare it `partial`.
- **`FolderPicker` cannot start at an arbitrary directory** (`SuggestedStartLocation` takes only a `PickerLocationId`). Use the Win32 `SHBrowseForFolder` with `BIF_NEWDIALOGSTYLE` and a `BFFM_INITIALIZED` callback. Do not use `FolderBrowserDialog`: `UseWindowsForms` conflicts with the XAML build items (`MC6000`).

## MVVM

- **Use classic `[ObservableProperty] private string _name;`, not partial properties** (`public partial string Name { get; set; }` failed with `CS9248`/`CS8050` even with `LangVersion` latest). Suppress the `MVVMTK0045` advisory with `<NoWarn>$(NoWarn);MVVMTK0045</NoWarn>` when not using Native AOT.
- **A classic `{Binding}` to an empty string shows the parent object's type name** (the template falls back to the inherited DataContext). Use `<DataTemplate x:DataType="x:String"><TextBlock Text="{x:Bind}" /></DataTemplate>`.
- **`x:Bind` converts `bool` to `Visibility`** by itself: no converter.
- **`ComboBox.SelectedValue` assigned before `ItemsSource` is filled does not select later** (the box renders blank). Do not set lookup-backed properties in the constructor; assign them in the async method right after that parent's options finish loading.

## ContentDialog

- **Only one can be open at a time**, even inside a button handler's deferral (`COMException: Only a single ContentDialog can be open at any time`). For a confirmation inside a dialog use a two-step confirm on the button ("Confirm Delete?") or a `Flyout` (a different popup layer).
- **A closed instance cannot be shown again.** `Hide()` then `ShowAsync()` on the same instance throws the same COMException. Create a fresh instance and carry the state (the record being edited) in a field.
- **Drilling from a dialog into another:** in the `ItemClick` handler `Hide()` the current dialog, then `await new ChildDialog(...) { XamlRoot = XamlRoot }.ShowAsync()`. Do not reopen the parent when the child closes: `Hide()` resolves the parent's `ShowAsync()`, which unblocks ancestors; three levels deep two dialogs call `ShowAsync()` at once and the process dies with a native fault in `Microsoft.UI.Xaml.dll`.
- **Footer buttons** (`PrimaryButtonText` etc.) are plain strings. The default template names them `PrimaryButton`, `SecondaryButton`, `CloseButton`: in `dialog.Opened` walk the visual tree (`VisualTreeHelper.GetChild`, match `FrameworkElement.Name`) and set `AccessKey` or `Focus(FocusState.Programmatic)`. Works for subclasses and ad-hoc dialogs.
- **Width is capped by theme resources, not by the dialog's `MinWidth`/`MaxWidth`** (a dialog stays about 550 px wide; content beyond is clipped). Override `ContentDialogMinWidth` (default about 320) and `ContentDialogMaxWidth` (about 548) in the dialog's own `Resources` (`<x:Double x:Key="ContentDialogMaxWidth">1400</x:Double>`); then the inner `ScrollViewer`'s `MinWidth` is the knob, with `HorizontalScrollBarVisibility="Auto"` as a safety net.
- **Keep the error `InfoBar` outside the dialog's `ScrollViewer`** (a `StackPanel` above it), or a validation message scrolls out of sight and "Save does nothing".
- **`ShowAsync()` returns when the dialog closes or `Hide()`s itself, not when its own async work is done.** With one shared (non-thread-safe) `DbContext` this causes "second operation was started on this context". Keep the dialog's load as a `Task` field (`Loaded += (_, _) => _loading = ViewModel.LoadAsync();`), give it `ShowAndWaitAsync()` that awaits `ShowAsync()` then `_loading`, and when it hides itself to open a child, also await a `TaskCompletionSource` that the drill-down handler completes in `finally`.

## Resources, header templates and crashes on open (found by pressing Edit)

- **A `StaticResource` in an attribute of the ROOT element of a ContentDialog (`PrimaryButtonStyle="{StaticResource X}"`) is read before the dialog's own `<ContentDialog.Resources>`**, so the dialog throws `XamlParseException: Cannot find a Resource with the Name/Key X [Line: 8 Position: 5]` the moment it is constructed, and the app dies (event `Application Error`, `Microsoft.UI.Xaml.dll`, code `0xc000027b`). The build succeeds and nothing before the click shows it. Declare the resources first, then set the property with a property element: `<ContentDialog.PrimaryButtonStyle><StaticResource ResourceKey="X" /></ContentDialog.PrimaryButtonStyle>`. To see the message, temporarily add `UnhandledException += (s, e) => File.WriteAllText(path, e.Message + e.Exception);` to the App constructor, run, read the file, remove the line.
- **A rich caption (a red asterisk after the header text):** a `Header` string cannot hold markup. Declare `<DataTemplate x:Key="RequiredHeader"><TextBlock><Run Text="{Binding}" /><Run Text=" *" Foreground="Red" /></TextBlock></DataTemplate>` once in the dialog's resources and set `HeaderTemplate="{StaticResource RequiredHeader}"` on each marked control (TextBox, ComboBox, NumberBox, CalendarDatePicker); the `Header="Caption"` string is the data. `{x:Bind}` with an empty path and `Foreground="{ThemeResource SystemFillColorCriticalBrush}"` in that template made the XAML compiler fail (`WMC0610`, XBF syntax error `0xc00cee3c`, pointing at the end of the element).
- Mark a required control for UI Automation with `AutomationProperties.IsRequiredForForm="True"` (read back as `IsRequiredForForm`). A ComboBox's header text is not exposed as a separate Text element, so check its star on a screenshot.
