---
name: winui3
description: Concrete WinUI3 (Windows App SDK) gotchas and working patterns learned building an unpackaged desktop app (CodeGenNew) — CommunityToolkit.Mvvm's ObservableProperty backing-field-vs-partial-property split, TreeView's real hierarchical-binding limitation, ContentDialog's single-open restriction and reach-in techniques (access keys, default-button focus), BitmapIcon vs ImageIcon, the native folder picker gap, self-contained deployment breaking runtime-compiled templates, and driving a WinUI3 app externally with UI Automation for verification. Use when building, debugging, or reviewing a WinUI3/Windows App SDK app, especially unpackaged desktop ones.
---

# WinUI3 (Windows App SDK) gotchas

## CommunityToolkit.Mvvm: use classic backing-field `[ObservableProperty]`, not partial properties

The newer partial-property style —
```csharp
public partial class FooViewModel : ObservableObject
{
    [ObservableProperty]
    public partial string Name { get; set; }
}
```
failed to compile in a WinUI3 project (`CS9248 "must have an implementation part"`, `CS8050`) even with
`<LangVersion>latest</LangVersion>` forced explicitly. Root cause not identified. The classic style built
and ran correctly the first time:
```csharp
public partial class FooViewModel : ObservableObject
{
    [ObservableProperty]
    private string _name;
}
```
This does raise the `MVVMTK0045` advisory (partial properties are recommended for Native AOT/trimmed
scenarios) — safe to suppress via `<NoWarn>$(NoWarn);MVVMTK0045</NoWarn>` for a project that isn't using
Native AOT/trimming.

## TreeView has no simple hierarchical-binding mode across mixed item types

`TreeView.ItemsSource` bound to a flat root collection, with `TreeView.ItemTemplate` wrapping a
`TreeViewItem` whose own `ItemsSource` points at a child collection, is a real, working pattern **only
when every level is the same type** (a single `x:DataType` DataTemplate can't switch shape per level
without a `ItemTemplateSelector`). Putting `ItemTemplate` directly on the inner `TreeViewItem` (instead of
just `ItemsSource`) produces `WMC0075`/`WMC0011` "Unknown member" errors — the child template is supposed
to come from the *same* `TreeView.ItemTemplate`, reapplied recursively, not a separate one per level.

For a two-level tree where the two levels are naturally different shapes (e.g. tables and their columns),
it's often simpler and more robust to skip the recursive-template API entirely: keep the TreeView's real
items flat (one level), and render the "children" as plain non-interactive content *inside* the parent
row's own DataTemplate (an `ItemsControl` with `Visibility` bound to an expand/collapse flag). This also
sidesteps selection/right-click ever applying to a "child" — since it was never a real tree node in the
first place, there's nothing to select.

## ContentDialog: only one can be open at a time — plan around it up front

Showing a second `ContentDialog` while one is already open (even mid-deferral, inside a `PrimaryButtonClick`
handler) throws `COMException: "Only a single ContentDialog can be open at any time."` This bites two
common patterns:
- A confirmation prompt shown from inside another dialog's button handler (e.g. "this folder doesn't
  exist, create it?"). Fix: don't nest — either do a two-step confirm inline in the same dialog (first
  click warns, second click on the same input proceeds), or resolve it without a modal at all.
- A "type a name"/delete-confirmation prompt opened from a dialog that is itself a `ContentDialog` (e.g. a
  template-management screen's "New..."/"Delete" buttons). Fix: use a `Flyout` instead of a nested
  `ContentDialog` for the prompt — a `Flyout` is a different popup layer and coexists fine with an already-
  open `ContentDialog`. For a delete confirmation specifically, an inline arm/confirm on the button itself
  (first click changes its own label to "Confirm Delete?", second click actually deletes) avoids a modal
  entirely.

## Reaching into ContentDialog's own footer buttons (access keys, default-button focus)

`PrimaryButtonText`/`SecondaryButtonText`/`CloseButtonText` are plain strings — there's no direct XAML
hook to set `AccessKey` or focus on them. But the default `ContentDialog` template names its generated
buttons **"PrimaryButton"**, **"SecondaryButton"**, **"CloseButton"** (stable since UWP), so you can reach
them via a visual-tree walk once the dialog is open:
```csharp
dialog.Opened += (_, _) =>
{
    var primary = FindButtonByName(dialog, "PrimaryButton");
    primary.AccessKey = "S";
    primary.Focus(FocusState.Programmatic); // if it's the DefaultButton
};
```
(`FindButtonByName` = a small recursive `VisualTreeHelper.GetChild` walk checking `FrameworkElement.Name`.)
This works uniformly for both a compiled `ContentDialog` subclass and an ad-hoc `new ContentDialog { ... }`
built entirely in code — the `Opened` event fires on any instance either way.

## `BitmapIcon` needs a real `Uri`; `ImageIcon` accepts any `ImageSource`

If icons are loaded from an embedded resource stream (`BitmapImage` populated via `SetSourceAsync`, no
backing file/Uri at all), `<BitmapIcon UriSource="...">` won't work — its `UriSource` is null since the
image was never constructed from a URI. Use `<ImageIcon Source="{x:Bind SomeBitmapImage}">` instead;
`ImageIcon.Source` is a plain `ImageSource` and works with any already-constructed `BitmapImage`,
regardless of how its pixels were loaded.

## `Windows.Storage.Pickers.FolderPicker` can't open at a specific starting directory

Its `SuggestedStartLocation` only accepts a fixed `PickerLocationId` enum (Desktop, Downloads,
ComputerFolder, etc.) — there's no way to point it at an arbitrary path the app already knows about (e.g.
"reopen where the developer last picked"). For a full starting-directory control, drop to the native
Win32 `SHBrowseForFolder` (with `BIF_NEWDIALOGSTYLE` for the modern resizable look) via P/Invoke instead —
its `BFFM_INITIALIZED` callback message lets you set the initial selection explicitly. This sidesteps
`System.Windows.Forms.FolderBrowserDialog` too, which is simpler but pulls in `UseWindowsForms`, and that
property conflicts with a WinUI3 project's own XAML `Page` build items (`MC6000` "must include
PresentationCore, PresentationFramework").

## Self-contained deployment can break anything that compiles code at runtime

See the self-contained-deployment skill for the general principle; the concrete trigger in a WinUI3 app is
`WindowsAppSDKSelfContained` (the Windows App SDK's own native runtime) being a **separate** setting from
`SelfContained` (.NET's own runtime bundling) — you can and often should set the former `true` and the
latter `false`. Also note: once `SelfContained=false`, a plain `dotnet build`/`dotnet run` without an
explicit `-p:Platform=x64` (or a matching `<RuntimeIdentifier>` set explicitly, singular not plural) may
fail the Windows App SDK's own build targets with `"WindowsAppSDKSelfContained requires a supported
Windows architecture."` — fix by using `<RuntimeIdentifier>win-x64</RuntimeIdentifier>` (singular) rather
than `<RuntimeIdentifiers>win-x64</RuntimeIdentifiers>` (plural, publish-oriented) so a single RID is
always resolved even for an ordinary build.

## UI Automation can drive an unpackaged WinUI3 app from outside, for real verification

Classic `System.Windows.Automation` (`UIAutomationClient`/`UIAutomationTypes`, the same API WPF apps are
automated with) works against a running WinUI3 app's window with no special setup — WinUI3 exposes
standard UIA peers. From PowerShell:
```powershell
Add-Type -AssemblyName UIAutomationClient, UIAutomationTypes
$proc = Start-Process -FilePath $exePath -PassThru
$mainWin = [System.Windows.Automation.AutomationElement]::RootElement.FindFirst(
    [System.Windows.Automation.TreeScope]::Children,
    (New-Object System.Windows.Automation.PropertyCondition(
        [System.Windows.Automation.AutomationElement]::ProcessIdProperty, $proc.Id)))
```
From there, `FindFirst`/`FindAll` with a `NameProperty`/`ControlTypeProperty` condition locate buttons,
text fields, etc.; `InvokePattern` clicks buttons; `ValuePattern.Current.Value` reads a TextBox's actual
displayed value (its `Current.Name` is just its accessible label, not its content). This is genuinely
useful for confirming a fix actually works end-to-end (e.g. "does this dialog really show the previously
saved value") rather than reasoning about it from source alone.

Two sharp edges: `AutomationElement.FocusedElement` is **system-wide**, not scoped to your process — after
killing/relaunching a test instance, bring the target window to the foreground first
(`user32.dll!SetForegroundWindow`) and prefer checking `element.Current.HasKeyboardFocus` on a specific
element scoped to your own window over trusting the global focused-element pointer. And avoid
`System.Windows.Forms.SendKeys` for key input in this kind of test — it injects real system-wide keyboard
input and can land on whatever window the person at the keyboard is actually using; drive the target
through UIA's own patterns (`InvokePattern`, `ValuePattern`, `TogglePattern`) instead, which stay scoped to
the element you found.
