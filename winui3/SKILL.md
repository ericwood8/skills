---
name: winui3
description: Concrete WinUI3 (Windows App SDK) gotchas and working patterns learned building an unpackaged desktop app (CodeGenNew) — CommunityToolkit.Mvvm's ObservableProperty backing-field-vs-partial-property split, TreeView's real hierarchical-binding limitation, ContentDialog's single-open restriction (both overlapping dialogs and reusing the same closed instance for a reshow) and reach-in techniques (access keys, default-button focus), ComboBox.SelectedValue not retroactively selecting once ItemsSource populates asynchronously after construction, BitmapIcon vs ImageIcon, showing success/warning/error status icons via InfoBar or SvgImageSource, the native folder picker gap, self-contained deployment breaking runtime-compiled templates, driving a WinUI3 app externally with UI Automation for verification, AppBarButton's Label FontSize being hardcoded in its default template (not bound to the button's own FontSize), MenuFlyoutItem's Disabled visual state overriding a plain Foreground, verifying a default-template assumption against the WindowsAppSDK's own generic.xaml in the NuGet cache instead of guessing, why a rebuild fails with a file-lock error while the app itself is running, and the project owner's preferred grid/toolbar styling (bordered grids with row separator lines, shaded toolbar/pagination/search bars). Use when building, debugging, or reviewing a WinUI3/Windows App SDK app, especially unpackaged desktop ones.
---

# WinUI3 (Windows App SDK) gotchas

## Writing a `.xaml` comment? See the msbuild skill's double-hyphen trap first

`.xaml` files are strict XML, so a `<!-- ... -->` comment containing `--` (a natural em-dash writing
habit) fails the XAML compiler at build time (`WMC9997`/`WMC9999`), not at the moment you type it — and
it's easy to reintroduce over and over in one session since it doesn't look wrong. Full writeup (it hits
`.csproj` too) is in the msbuild skill's "XML comments can't contain `--`" section; the short version is:
use a real em dash (—), a colon, or a comma instead of `--` in any XAML comment.

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

## `Hide()`-ing a ContentDialog then `ShowAsync()`-ing the *same instance* again throws COMException

This is a different trap from the "only one open at a time" rule above: it's not about two distinct
dialogs overlapping, it's about **reshowing one dialog after it closed**. A drill-down flow (parent dialog
-> `Hide()` -> show child dialog -> child closes -> bring the parent back) is tempting to write as "call
`ShowAsync()` on `this` again" from inside the parent's own click handler, but that throws
`System.Runtime.InteropServices.COMException: "An async operation was not properly started. Only a single
ContentDialog can be open at any time."` even though nothing else is open at that moment — a `ContentDialog`
instance is not reusable once its show/hide lifecycle has completed once. Fix: construct a **fresh instance**
of the same dialog type for the reshow, carrying forward whatever state it needs (the record being edited):
```csharp
private async void OnChildRowClick(object sender, ItemClickEventArgs e)
{
    if (e.ClickedItem is not ChildGridRow { Entity: SalesInvoice invoice }) return;
    Hide();
    var child = new SalesInvoiceDetailMasterDialog(_context, invoice) { XamlRoot = XamlRoot };
    await child.ShowAsync();
    var reopened = new CustomerMonthlySummaryDetailMasterDialog(_context, _editing) { XamlRoot = XamlRoot };
    await reopened.ShowAsync();
}
```
Store whatever the constructor needs (e.g. the record being edited) in a field set once at construction, so
the reshow can rebuild the dialog with the same state.

## `ComboBox.SelectedValue` set before `ItemsSource` is populated does not retroactively select

A lookup/foreign-key drop-down whose `SelectedValue` is assigned in the constructor (from the record being
edited) while its `ItemsSource` is only populated later, asynchronously (a `LoadLookupsAsync()` call after
the dialog opens), renders **blank** — even though the value is technically set on the property. Typing in
the box then reveals the full item list, which makes it look like a plain `TextBox` that happens to
autocomplete, not a bound `ComboBox`. This is a real WinUI3/UWP limitation: `SelectedValue` only resolves
against whatever `ItemsSource` already contains at the moment it's assigned; it does not re-evaluate once
the list shows up later. Fix: don't set the lookup field in the constructor at all — defer that assignment
into the same async method that populates `ItemsSource`, right after that specific parent's options finish
loading:
```csharp
// constructor: skip assigning the lookup-backed property here, only plain scalar fields
// LoadLookupsAsync():
foreach (var p in parents)
{
    p.OptionsProp = await LoadOptionsForParentAsync(p); // populates ItemsSource
    if (_editing is not null)
        CustomerId = _editing.CustomerId; // assign the lookup value AFTER its ItemsSource exists
}
```
This applies to any control whose "selected value" binding depends on an `ItemsSource`/`ItemsSource`-like
collection that loads asynchronously after construction — not unique to a generated screen.

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

## When a default-template assumption doesn't hold, grep the WindowsAppSDK's own `generic.xaml`

A control's default `ControlTemplate` is not obvious from the public API surface — a property that
*sounds* like it should reach some piece of rendered text (or a `Foreground` that *sounds* like it should
apply) may not, because the template hardcodes a literal value or a `VisualState` overrides it. Guessing
from behavior alone burns a round trip with the person testing the app; the template itself is available
and searchable, offline, in the restored NuGet cache:
```
find ~/.nuget/packages/microsoft.windowsappsdk.winui -iname "generic.xaml"
# .../lib/native/Microsoft.UI/Themes/generic.xaml -- the one that matters for a WinUI3 (not UWP) app
```
Find the control's default style (`x:Key="Default<ControlName>Style"`), then its `ControlTemplate`, and
read the actual `Setter`/`TemplateBinding`/literal values on the named parts. Two concrete findings from
doing this, below — both were confirmed this way, not by trial and error.

### `AppBarButton`'s `Label` ignores the button's own `FontSize` — it's hardcoded to 12 in the template

Setting `FontSize="32"` on an `AppBarButton` looks like it should enlarge its caption text (and the icon
area does respond to it) — but the default template's label `TextBlock` (`x:Name="TextLabel"`) has a
**literal** `FontSize="12"` baked into the template XAML, not a `TemplateBinding` to the button's own
`FontSize`. The property silently does nothing for the label, in every label position (`Right`, on-top,
`Compact`) — confirmed by reading every `TextLabel` declaration in `DefaultAppBarButtonStyle`, none of them
bind `FontSize`. There is no style-`Setter`-only fix (you cannot target a literal value inside a template
without replacing the whole template, which is large and WindowsAppSDK-owned). The working fix: after the
button's own template has applied (its `Loaded` event), reach into the visual tree for the `TextBlock`
named `"TextLabel"` and set its `FontSize` from the button's own `FontSize` directly:
```csharp
private void OnAppBarButtonLoaded(object sender, RoutedEventArgs e)
{
    if (sender is AppBarButton { FontSize: var fontSize } button
        && FindDescendant<TextBlock>(button, "TextLabel") is { } label)
        label.FontSize = fontSize;
}

private static T? FindDescendant<T>(DependencyObject root, string name) where T : FrameworkElement
{
    int count = VisualTreeHelper.GetChildrenCount(root);
    for (int i = 0; i < count; i++)
    {
        var child = VisualTreeHelper.GetChild(root, i);
        if (child is T match && match.Name == name) return match;
        if (FindDescendant<T>(child, name) is { } found) return found;
    }
    return null;
}
```
Wire `Loaded="OnAppBarButtonLoaded"` on each `AppBarButton` that needs this. This keeps the `FontSize` set
in XAML as the one place that number lives — the handler just propagates it to where the template forgot to.

### A disabled `MenuFlyoutItem`'s text color: `Foreground` alone is not enough — override the `ThemeResource`

Setting `Foreground` on a `MenuFlyoutItem` with `IsEnabled="false"` (a common way to show a non-clickable
label row, e.g. a status/summary line at the top of a right-click menu) does not change its rendered
color. The `Disabled` `VisualState` in the default template sets `TextBlock.Foreground` from
`{ThemeResource MenuFlyoutItemForegroundDisabled}`, and that `Setter` wins over the plain `Foreground`
property once the item is disabled. The fix is to override that specific resource key **on the item's own
`Resources` dictionary** — a `ThemeResource`/`StaticResource` lookup checks the element itself before
falling back to the app-wide theme, so a per-instance override there beats the default without touching
any global resource:
```csharp
var item = new MenuFlyoutItem { Text = "...", IsEnabled = false, Foreground = redBrush };
item.Resources["MenuFlyoutItemForegroundDisabled"] = redBrush; // the part that actually works
```
The same pattern applies to any other disabled-state color that doesn't respond to a plain property
setter — find the exact resource key the control's `Disabled` `VisualState` sets (per the technique above)
and override that key instead of the property.

## Rebuilding while the app is running fails with a file-lock error, not a compile error

`dotnet build` on a WinUI3 app project that is currently running (or attached in Visual Studio) fails with
`MSB3026`/`MSB3027` ("Could not copy ... The process cannot access the file ... because it is being used by
another process"), retried ~10 times before erroring out — for the `.exe`/`apphost.exe` itself and every
dependent project DLL it references. This is not a code problem and re-reading the diff won't explain it;
check whether the app's own process is running before assuming a build regression. Don't kill the running
process without asking — it may be the user's own active session (they could be mid-review of the exact
change just made) — ask them to close it (or confirm before ending it yourself), then rebuild.

## `BitmapIcon` needs a real `Uri`; `ImageIcon` accepts any `ImageSource`

If icons are loaded from an embedded resource stream (`BitmapImage` populated via `SetSourceAsync`, no
backing file/Uri at all), `<BitmapIcon UriSource="...">` won't work — its `UriSource` is null since the
image was never constructed from a URI. Use `<ImageIcon Source="{x:Bind SomeBitmapImage}">` instead;
`ImageIcon.Source` is a plain `ImageSource` and works with any already-constructed `BitmapImage`,
regardless of how its pixels were loaded.

## Showing success/warning/error status icons (shape-coded, not just color-coded)

See the ui-conventions skill for the general convention (green check / amber triangle+`!` / red circle+X,
distinguished by shape so it doesn't rely on color alone) and its bundled `assets/status-*.svg` icons.
Two ways to actually show it in WinUI3, in order of preference:

**Prefer the built-in `InfoBar` control first.** It already has a `Severity` property
(`Success`/`Warning`/`Error`/`Informational`) that renders the correct shape-coded, theme-aware,
accessible icon with zero custom assets:
```xml
<InfoBar IsOpen="{x:Bind ViewModel.HasResult, Mode=OneWay}"
         Severity="{x:Bind ViewModel.ResultSeverity, Mode=OneWay}"
         Title="{x:Bind ViewModel.ResultTitle, Mode=OneWay}"
         Message="{x:Bind ViewModel.ResultMessage, Mode=OneWay}" />
```
For a modal popup like a "Template generation failed:" dialog, put the `InfoBar` *inside* the
`ContentDialog`'s content instead of building a custom icon+text header — this keeps the existing modal
button flow (OK/Retry/Cancel) while getting the icon for free. For a non-blocking result (most "it worked" /
"here's a warning" cases don't actually need to be modal), consider dropping `ContentDialog` entirely and
showing the `InfoBar` inline in the window — fewer clicks, same information.

**Use the bundled SVG assets directly only when `InfoBar`'s built-in styling doesn't fit** (e.g. a custom
non-InfoBar layout that still needs the icon). WinUI3 can load an SVG straight from disk via
`SvgImageSource` + `Image` (or wrap it in `ImageIcon`, per the `BitmapIcon`-vs-`ImageIcon` note above —
`ImageIcon.Source` accepts any `ImageSource`, and `SvgImageSource` is one):
```xml
<ImageIcon Width="20" Height="20">
    <ImageIcon.Source>
        <SvgImageSource UriSource="ms-appx:///Assets/status-error.svg" />
    </ImageIcon.Source>
</ImageIcon>
```
(Copy the relevant `assets/status-*.svg` file from the ui-conventions skill into the project's own `Assets/`
folder as `Content`/`Resource`, since `ms-appx:///` resolves against the app package, not the skill
directory.)

## Grid/toolbar styling preference: bordered grids, shaded toolbars

The project owner's stated preference for every grid on a WinUI3 CRUD screen (list pages, and a
master/detail dialog's read-only child grids alike): a visible line around the whole grid and a line
between each row, so it reads as a table instead of loose rows of text — plus a shaded background on the
Add New/Refresh bar, the search bar, and the pagination bar, all sharing one matching look so the three
toolbars read as a family instead of plain unstyled `StackPanel`s. Requested 2026-09-27 after seeing the
generated screens live; CodeGenNew.Templates\WinUI3_MasterScreen_v1.tt and
WinUI3_DetailMasterScreen_v1.tt now bake this in for every future generated screen, so a hand-built WinUI3
screen (or one from another codegen tool) should match it too, for visual consistency:

```xml
<!-- Grid: Border around the whole thing, another Border under the header, one per row.
     The outer Grid.RowDefinitions row this whole Border sits in MUST be "*", not "Auto" --
     see the pitfall below; it's the easy way to lose this layout entirely. -->
<Border BorderBrush="{ThemeResource ControlStrokeColorSecondaryBrush}" BorderThickness="1" CornerRadius="4">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" /> <!-- header -->
            <RowDefinition Height="*" />    <!-- rows -->
        </Grid.RowDefinitions>
        <Border Grid.Row="0" BorderBrush="{ThemeResource ControlStrokeColorSecondaryBrush}" BorderThickness="0,0,0,1" Padding="8">
            <!-- header content -->
        </Border>
        <ListView Grid.Row="1" Padding="0">
            <ListView.ItemTemplate>
                <DataTemplate>
                    <Border BorderBrush="{ThemeResource ControlStrokeColorSecondaryBrush}" BorderThickness="0,0,0,1" Padding="8">
                        <!-- row content -->
                    </Border>
                </DataTemplate>
            </ListView.ItemTemplate>
        </ListView>
    </Grid>
</Border>

<!-- Toolbar (Add New/Refresh, search, or pagination): same shaded Border on every one -->
<Border Background="{ThemeResource SolidBackgroundFillColorSecondaryBrush}"
        BorderBrush="{ThemeResource ControlStrokeColorSecondaryBrush}" BorderThickness="1" CornerRadius="4" Padding="8">
    <StackPanel Orientation="Horizontal" Spacing="8"> <!-- buttons/fields --> </StackPanel>
</Border>
```

`ControlStrokeColorSecondaryBrush`/`SolidBackgroundFillColorSecondaryBrush` are theme resources, so the
border/shade follow the app's light/dark theme automatically instead of a hardcoded color — no custom
resource dictionary needed.

### Pitfall 1: the Fluent2 `Card*` brushes are the wrong choice here — they're invisible on a plain page

The natural first reach for "themed card-style shading" is `CardBackgroundFillColorDefaultBrush`/
`CardStrokeColorDefaultBrush` (the brushes Settings-style "Card" UI uses) — but they render as **no visible
shading at all** on an ordinary page background, and this genuinely shipped once before being caught by a
live user report. Confirmed by reading the WindowsAppSDK's own `generic.xaml` (the technique documented
above, under "grep the WindowsAppSDK's own generic.xaml"): in the Light theme dictionary,
`CardBackgroundFillColorDefault` is `#B3FFFFFF` — a 70%-opacity **white** fill, meant to sit on top of
another surface (like `SolidBackgroundFillColorSecondary`), not directly on the page. Layered on an already-
white page it's indistinguishable from no background at all; `CardStrokeColorDefault` is similarly faint
(`#0F000000`, ~6% opacity black). Use the fully-opaque pair instead for anything meant to read as visible
shading against a plain page: `SolidBackgroundFillColorSecondaryBrush` (`#EEEEEE` Light / `#1C1C1C` Dark) and
`ControlStrokeColorSecondaryBrush` (`#29000000` Light / `#18FFFFFF` Dark, noticeably more visible than
`ControlStrokeColorDefault`/`CardStrokeColorDefault`/`DividerStrokeColorDefault`, all ~6-9% opacity). The
general lesson: a "faint tint over another surface" brush and a "flat, opaque, meant-for-the-page" brush are
different tools, and picking the Card* one for the latter job looks fine in a quick review (or even a zoomed-
in screenshot) but reads as "no shading at all" at normal viewing — check theme dictionary hex/alpha values
directly rather than trusting the resource name.

### Pitfall 2: an outer `Grid.RowDefinitions` row of `Auto` around a `*`-sized `ListView` grows unbounded

Boxing a `ListView` in a `Border` (per the pattern above) adds a wrapping `Grid`/`Border` layer between the
`ListView` and whatever `Grid.RowDefinitions` row it used to sit in directly. If that **outer** row is left
(or accidentally changed to) `Height="Auto"` instead of `Height="*"`, the `Border`/inner `Grid`/`ListView`
all size to their natural content instead of being handed a bounded height — so the inner `*` row (meant to
make the `ListView` fill available space and scroll) has no space to fill, and the `ListView` instead grows
to show **every** row unclipped. On a page whose own root `Grid` has no `ScrollViewer`, this pushes anything
below it (a pagination bar in the next row) off the bottom of the window entirely — it doesn't error, it just
silently disappears from view, which is exactly what shipped once before a live user report caught it
("as if the grid was not capped off" was the precise, correct diagnosis). The fix is always to make sure the
row that directly bounds the `ListView`'s box is `*`, not `Auto`, all the way up the parent chain to
wherever the page actually has room to constrain it.

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
