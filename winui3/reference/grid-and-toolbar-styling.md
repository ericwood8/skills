# Grid, toolbar and paging bar

Read this when: you style a CRUD screen's grid and its toolbar, search bar or pagination bar, or the grid pushes the paging bar off the page.

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

- **Do not use the `Card*` brushes** (`CardBackgroundFillColorDefaultBrush`, `CardStrokeColorDefaultBrush`): they are translucent tints that sit on another surface (`#B3FFFFFF` and about 6% black in Light) and show no shading on a plain page. `SolidBackgroundFillColorSecondaryBrush` (`#EEEEEE` Light / `#1C1C1C` Dark) and `ControlStrokeColorSecondaryBrush` (`#29000000` / `#18FFFFFF`) are opaque and visible. Check hex/alpha in the theme dictionary, not the resource name.
- **The row that bounds the boxed `ListView` must be `Height="*"`, not `Auto`**, all the way up to where the page has room. With `Auto` the `ListView` grows to show every row and, on a page with no `ScrollViewer`, pushes the pagination bar off the bottom without any error.

## Paging bar

- `PageSize` is an `[ObservableProperty]`; `TotalRows` and `HasLoaded` notify `PageLabel`, which reads `"Nothing found."` when loaded with no rows, else `"Page 2 of 5 (98 rows)"`.
- After the search, if the page is empty with `totalCount > 0` and `PageNumber > TotalPages`, set `PageNumber = TotalPages` and search again; otherwise deleting the last row of the last page shows "Page 3 of 2" and an empty grid (the rule for every stack: codegen-t4-templates `reference/paging-and-grids.md`).
- Two templates writing the same file (`PaginationBar.xaml`) must stay identical: a test that cuts the section out of both and compares them catches drift.
- A live paging test: `Docs/Verification/Test-WinUI3Paging.ps1` (scratch copy, extra rows so the last page holds one row, point the app at the copy through `appsettings.json` next to the exe and restore it in `finally`, page to the end, delete through the confirmation dialog, assert the label). The dialog's primary button has the same name as the row's (`Delete`): take the Delete button whose `GetRuntimeId()` was not in the list taken before the click.
