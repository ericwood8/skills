---
name: winui3
description: Rules and working patterns for WinUI3 (Windows App SDK) unpackaged desktop apps — CommunityToolkit.Mvvm properties, ContentDialog limits (one open at a time, no reshow of an instance, width caps, drill-down, shared DbContext), ComboBox/ListView/NumberBox/CalendarDatePicker/TabView/InfoBar/AppBarButton/MenuFlyoutItem behaviors, XAML comment and xmlns traps, bordered grid and toolbar styling, folder picker, self-contained settings, build file-lock errors, and driving the app with UI Automation. Use when building, debugging or reviewing a WinUI3/Windows App SDK app.
---

# WinUI3 (Windows App SDK)

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

## Controls

- **`ComboBox`: never set `HorizontalAlignment="Stretch"` on it.** It makes the drop-down chevron vanish or double. Leave the default or set `Width`/`MinWidth`. If a chevron misbehaves, check this first; no custom `ComboBox` style is needed. Code generators must not emit it.
- **`ListView`: use `SelectionMode="Single"` for any grid a person clicks in.** `None` makes clicks look dead and the arrow keys jump to the wrong row. Never `IsEnabled="False"` to make it read-only (grey, swallows clicks): use `IsItemClickEnabled="True"` and `ItemClick`, and give each row object an `Entity` reference instead of parsing display strings (`e.ClickedItem`).
- **`ListView` in a `ScrollViewer` can cut the last row** (containers have a 40 px minimum height). Give `ListViewItem` an `ItemContainerStyle` (`BasedOn="{StaticResource DefaultListViewItemStyle}"`) with `MinHeight` 0 and add a little bottom `Padding` on the content panel.
- **`NumberBox.Value` is a `double`**: bind a `double`/`double?` property (or a converter, or `Text`); empty is `NaN`, so check `double.IsNaN`; convert to `decimal` for money. Inline spin buttons need about 150 px beyond the digits (about 170 px for two digits), or the text area clips to nothing and typing seems dead. Test with a screenshot of a typed value. A decimal column needs a code-behind `DecimalFormatter` (`FractionDigits` = scale, an `IncrementNumberRounder`) to show `77.00`.
- **`CurrencyFormatter` cannot be declared in XAML** (create it in code-behind) **and rejects text typed without the symbol** (`7777.77` reverts). Use a small `partial` class implementing `INumberFormatter2` and `INumberParser` that formats with `CurrencyFormatter` and parses with it first, then a `DecimalFormatter` (`_currency.X(text) ?? _plain.X(text)` for `ParseDouble`, `ParseInt`, `ParseUInt`).
- **`CalendarDatePicker`**: `Date` is a `DateTimeOffset?`; `DateFormat="{}{month.integer}/{day.integer}/{year.full}"` (leading `{}` escapes the braces). Convert with `new DateTimeOffset(dateTime)` in and `.Value.Date + storedTimeOfDay` out so editing never zeroes a stored time. No clear button: add a "Clear" button that sets null. Do not set `MinDate`/`MaxDate` from a project-wide year range if existing rows can hold older dates.
- **`TabView` for a long form**: `IsAddTabButtonVisible="False" TabWidthMode="SizeToContent" CanReorderTabs="False" CanDragTabs="False"` with `TabViewItem IsClosable="False"`; give each page's `ScrollViewer` a fixed `Height`; write `&` in a header as `&amp;`. `TabItemsSource` with templates is unreliable: build the `TabViewItem`s in the constructor.
- **TreeView with different shapes per level:** the recursive `ItemTemplate` works only when every level is the same type (an inner `ItemTemplate` gives `WMC0075`/`WMC0011`). For two different levels keep the real items flat and render the children as non-interactive content inside the parent row's `DataTemplate` (an `ItemsControl` with an expand flag).
- **`BitmapIcon` needs a real `Uri`; `ImageIcon.Source` takes any `ImageSource`** (use it for a `BitmapImage` loaded from a stream, or for `SvgImageSource`).
- **Status icons:** prefer `InfoBar` (`Severity` Success/Warning/Error/Informational gives a shape-coded, accessible icon; put it inside a `ContentDialog`'s content, or inline instead of a modal). Use the ui-conventions skill's `assets/status-*.svg` through `<ImageIcon><ImageIcon.Source><SvgImageSource UriSource="ms-appx:///Assets/status-error.svg"/>` only when `InfoBar` does not fit; copy the file into the project's `Assets/` first.
- **`AppBarButton.Label` ignores the button's `FontSize`**: the template's `TextLabel` has a literal `FontSize="12"`. On the button's `Loaded` event find the `TextBlock` named `TextLabel` in the visual tree (a recursive `VisualTreeHelper.GetChild` walk) and set its `FontSize` from the button's.
- **A disabled `MenuFlyoutItem` ignores `Foreground`**: its `Disabled` visual state sets the color from `MenuFlyoutItemForegroundDisabled`. Override that key on the item's own `Resources`: `item.Resources["MenuFlyoutItemForegroundDisabled"] = brush;`. Same pattern for any disabled-state color.
- **Pagination bar `ComboBox` in a `UserControl`:** set `ItemsSource` and `SelectedItem` in the constructor after `InitializeComponent()` (not `x:Bind`); update `SelectedItem` from the dependency property's changed callback; `SelectionChanged` also fires for programmatic selection, so raise the page-size event only when the size differs from the property.

## Input controls that stop bad data (rules in the ui-conventions skill)

- **True/false:** `CheckBox` with `IsChecked="{x:Bind IsChecked, Mode=TwoWay}"` on a `bool` property that reads and writes the stored string.
- **Up to three choices:** `RadioButtons` with `ItemsSource`, `SelectedItem="{x:Bind SelectedChoice, Mode=TwoWay}"` and an `ItemTemplate` (`DataTemplate` with `x:DataType`, kept in `Resources`). The selected-item property must tolerate a null set and return null for a stored value not on the list.
- **More than three:** a non-editable `ComboBox` with `DisplayMemberPath="Label"` and the same binding.
- **Any of a list:** a `DropDownButton` whose `Flyout` holds an `ItemsControl` of `CheckBox`es; bind `Content` to a summary string and raise `PropertyChanged` for it; guard against feedback (a flag around the loop that sets each box from the loaded value).
- **Whole number:** `NumberBox` with `Minimum`, `Maximum`, `SpinButtonPlacementMode="Compact"`, `ValidationMode="InvalidInputOverwritten"`, `Value="{x:Bind NumberValue, Mode=TwoWay}"` on a `double` where `NaN` means blank. One `DataTemplate` can hold every control kind and show the one that applies.
- **Read-only text:** `TextBlock` (in a `Border` for a panel), never a read-only `TextBox` (tab stop, looks editable). In a dialog forced to `RequestedTheme="Dark"`, black text needs a light `Border` background.

## Grid and toolbar styling (the project owner's preference)

Every grid on a CRUD screen (list pages and a master/detail dialog's child grids): a visible line around the whole grid and between rows, plus one matching shaded look on the Add New/Refresh bar, the search bar and the pagination bar. Use opaque theme resources so light/dark follow the app:

```xml
<Border BorderBrush="{ThemeResource ControlStrokeColorSecondaryBrush}" BorderThickness="1" CornerRadius="4">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" /> <!-- header -->
            <RowDefinition Height="*" />    <!-- rows -->
        </Grid.RowDefinitions>
        <Border Grid.Row="0" BorderBrush="{ThemeResource ControlStrokeColorSecondaryBrush}" BorderThickness="0,0,0,1" Padding="8"> <!-- header --> </Border>
        <ListView Grid.Row="1" Padding="0">
            <ListView.ItemTemplate>
                <DataTemplate>
                    <Border BorderBrush="{ThemeResource ControlStrokeColorSecondaryBrush}" BorderThickness="0,0,0,1" Padding="8"> <!-- row --> </Border>
                </DataTemplate>
            </ListView.ItemTemplate>
        </ListView>
    </Grid>
</Border>

<Border Background="{ThemeResource SolidBackgroundFillColorSecondaryBrush}"
        BorderBrush="{ThemeResource ControlStrokeColorSecondaryBrush}" BorderThickness="1" CornerRadius="4" Padding="8">
    <StackPanel Orientation="Horizontal" Spacing="8"> <!-- buttons/fields --> </StackPanel>
</Border>
```

- **Do not use the `Card*` brushes** (`CardBackgroundFillColorDefaultBrush`, `CardStrokeColorDefaultBrush`): they are translucent tints meant to sit on another surface (`#B3FFFFFF` and about 6% black in Light) and show no shading on a plain page. `SolidBackgroundFillColorSecondaryBrush` (`#EEEEEE` Light / `#1C1C1C` Dark) and `ControlStrokeColorSecondaryBrush` (`#29000000` / `#18FFFFFF`) are opaque and visible. Check hex/alpha in the theme dictionary, not the resource name.
- **The row that bounds the boxed `ListView` must be `Height="*"`, not `Auto`**, all the way up to where the page has room. With `Auto` the `ListView` grows to show every row and, on a page with no `ScrollViewer`, pushes the pagination bar off the bottom without any error.

## Find a control's real template

When a default-template assumption fails, search the SDK's own `generic.xaml` instead of guessing: `find ~/.nuget/packages/microsoft.windowsappsdk.winui -iname generic.xaml` (the `lib/native/Microsoft.UI/Themes/generic.xaml` one). Find `x:Key="Default<Control>Style"`, then the `ControlTemplate`, and read the `Setter`/`TemplateBinding`/literal values on the named parts.

## Driving the running app (UI Automation)

- `System.Windows.Automation` works against an unpackaged WinUI3 app. From PowerShell: `Add-Type -AssemblyName UIAutomationClient, UIAutomationTypes`, `Start-Process -PassThru`, then `AutomationElement.RootElement.FindFirst(Children, PropertyCondition(ProcessIdProperty, $proc.Id))`; find by `NameProperty`/`ControlTypeProperty`; `InvokePattern` clicks; `ValuePattern.Current.Value` reads a TextBox's content (`Current.Name` is only its label). The harness is in `Docs/Verification` (`UiAutomation.psm1`).
- `AutomationElement.FocusedElement` is system-wide: bring the window forward (`user32!SetForegroundWindow`) and check `Current.HasKeyboardFocus` on a specific element. Never use `SendKeys` (it types into whatever window the person is using); use `InvokePattern`, `ValuePattern`, `TogglePattern`.
- `ScrollPattern.SetScrollPercent(-1, 100)` scrolls a dialog to its end. A UIA `BoundingRectangle` is the visible (clipped) rectangle, which is how a cut-off row shows in numbers. `CopyFromScreen` screenshots include any window in front.
- `Find-Name 'Rows per page'` returns the TextBlock beside the box; get the box with `@(Find-Type 'ComboBox')[0]`. `SelectionPattern` gives the selected item; `Expand-El` then `FindAll(Descendants, ListItem)` lists options; the page label is `Find-Like 'Page * of *'`; wait about 3 s after each change.

## Paging bar

- `PageSize` is an `[ObservableProperty]`; `TotalRows` and `HasLoaded` notify `PageLabel`, which reads `"Nothing found."` when loaded with no rows, else `"Page 2 of 5 (98 rows)"`.
- After the search, if the page is empty with `totalCount > 0` and `PageNumber > TotalPages`, set `PageNumber = TotalPages` and search again; otherwise deleting the last row of the last page shows "Page 3 of 2" and an empty grid.
- Two templates writing the same file (`PaginationBar.xaml`) must stay identical: a test that cuts the section out of both and compares them catches drift.
- A live paging test: `Docs/Verification/Test-WinUI3Paging.ps1` (scratch copy, extra rows so the last page holds one row, point the app at the copy through `appsettings.json` next to the exe and restore it in `finally`, page to the end, delete through the confirmation dialog, assert the label). The dialog's primary button has the same name as the row's (`Delete`): take the Delete button whose `GetRuntimeId()` was not in the list taken before the click.
