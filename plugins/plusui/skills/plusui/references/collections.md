# Collection & Data Controls

Controls that render collections of data: a virtualized list, a hierarchical tree, a tabular grid, and a tabbed container. All four follow PlusUi's fluent builder pattern, build their UI in C# (no XAML), and bind with **lambda expressions** (`() => vm.X`), never `nameof(...)` strings.

## Table of contents

- [Conventions](#conventions)
- [ItemsList&lt;T&gt;](#itemslistt)
- [TreeView](#treeview)
- [DataGrid&lt;T&gt;](#datagridt)
- [TabControl](#tabcontrol)
- [Common LLM mistakes](#common-llm-mistakes)

## Conventions

- **Set\* vs Bind\***: Every `Set*` sets a static value once; the matching `Bind*` registers a live binding that re-runs on `PropertyChanged`. The "Bind?" column below marks which properties have a `Bind*` counterpart.
- **One-way bind**: `Bind*(Expression<Func<T>> getter)` — e.g. `.BindText(() => vm.Title)`.
- **Two-way bind** (selection / editable values): `Bind*(Expression<Func<T>> getter, Action<T> setter)` — e.g. `.BindSelectedItem(() => vm.Sel, v => vm.Sel = v)`.
- **Virtualization**: `ItemsList`, `TreeView`, and `DataGrid` render only visible rows. Always constrain the viewport with `SetDesiredHeight`/`SetDesiredWidth` — without it the control lays out *all* items and you lose the virtualization benefit.
- **Reactive sources**: pass an `ObservableCollection<T>` (or any `INotifyCollectionChanged`) and the control refreshes automatically; do not rebuild it by hand.

---

## ItemsList&lt;T&gt;

High-performance virtualized list. Renders only the items currently visible in the viewport. Base: `UiLayoutElement`. Construct as `new ItemsList<T>()` where `T` is the item type.

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|:-----:|-------|
| `ItemsSource` | `IEnumerable<T>?` | `null` | yes | Data source. Responds to `ObservableCollection` changes automatically. |
| `ItemTemplate` | `Func<T, int, UiElement>` | `null` | yes | Builds a `UiElement` per item. Receives `(item, zeroBasedIndex)`. Must not return null. |
| `Orientation` | `Orientation` | `Vertical` | yes | `Vertical` (top-to-bottom) or `Horizontal` (left-to-right). |
| `ScrollOffset` | `float` | `0` | yes | Scroll position in **pixels** (not item index). Clamped to valid range. |
| `ScrollFactor` | `float` | `1.0` | yes | Wheel/touch scroll-speed multiplier. |
| `ShowScrollbar` | `bool` | `true` | yes | Scrollbar still only appears when content exceeds the viewport. |
| `Scrollbar` | `Scrollbar` | auto | no | Read-only internal scrollbar control. |

```csharp
// Simple string list
new ItemsList<string>()
    .SetItemsSource(Enumerable.Range(1, 8).Select(i => $"Item {i}").ToList())
    .SetItemTemplate((item, _) => new Label().SetText(item).SetMargin(new Margin(8, 6)))
    .SetDesiredHeight(180);
```

```csharp
// Object list with a composite row template + alternating background
new ItemsList<Person>()
    .SetItemsSource(new List<Person>
    {
        new("Ada Lovelace", "Engineer"),
        new("Alan Turing", "Mathematician"),
    })
    .SetItemTemplate((p, index) => new HStack()
        .SetSpacing(8)
        .SetMargin(new Margin(8, 6))
        .SetBackground(index % 2 == 0 ? PlusUiDefaults.BackgroundPrimary : PlusUiDefaults.BackgroundSecondary)
        .AddChild(new Label().SetText(p.Name).SetFontWeight(FontWeight.SemiBold))
        .AddChild(new Label().SetText(p.Role).SetTextColor(PlusUiDefaults.TextSecondary)))
    .SetDesiredHeight(160);
```

```csharp
// Reactive: ObservableCollection + per-item two-way binding inside the template
new ItemsList<TodoItem>()
    .BindItemsSource(() => vm.Todos)                 // ObservableCollection<TodoItem>
    .SetItemTemplate((todo, _) => new HStack()
        .SetMargin(new Margin(4, 8))
        .AddChild(new Checkbox().BindIsChecked(() => todo.IsDone, v => todo.IsDone = v))
        .AddChild(new Label().BindText(() => todo.Title).SetMargin(new Margin(8, 0, 0, 0))))
    .SetDesiredHeight(240);
```

**Rules**

- Set **both** `ItemsSource` and `ItemTemplate` — setting only one shows nothing.
- Use the `index` parameter for styling only (e.g. zebra striping), never for data lookups.
- Horizontal carousels: `.SetOrientation(Orientation.Horizontal)` and constrain height with `SetDesiredHeight`.

**Gotchas**

- `ItemTemplate` returning `null` for any item throws `InvalidOperationException`.
- Virtualization renders only visible items — do not assume every item exists in `Children`.
- `ScrollOffset` is pixels, not item indices.
- Without `SetDesiredHeight`/`Width` the list shows ALL items (no virtualization).
- Default orientation is `Vertical`; horizontal needs an explicit `SetOrientation`.
- The first measure pass sizes ALL items to compute total scroll extent — very large datasets may show a brief initial layout delay.

---

## TreeView

Hierarchical tree with lazy child loading, expand/collapse, optional connector lines, and virtualized rendering of visible nodes. Base: `UiLayoutElement<TreeView>` (implements `IScrollableControl`, `IInputControl`, `IHoverableControl`). Items are typed as `object` — register a **children selector per node type** so the tree knows how to descend.

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|:-----:|-------|
| `ItemsSource` | `IEnumerable<object>?` | `null` | yes | Root items. `INotifyCollectionChanged` triggers a rebuild. |
| `ItemTemplate` | `Func<object, int, UiElement>` | `null` | yes | Builds a `UiElement` per node. Receives `(item, depth)`. |
| `SelectedItem` | `object?` | `null` | yes (two-way) | Set by clicking node content (not the expander). |
| `Indentation` | `float` | `~20` | yes | Horizontal pixels per tree level (`PlusUiDefaults.TreeViewIndent`). |
| `ItemHeight` | `float` | `~32` | yes | Row height in pixels (`PlusUiDefaults.ItemHeight`). |
| `ExpanderSize` | `float` | `~16` | yes | Expand/collapse triangle size (`PlusUiDefaults.IconSize`). Drawn only when the node has children. |
| `AutoExpandInitialLevels` | `bool` | `false` | yes | Auto-expand first N levels on the *first* build only. `SetAutoExpandInitialLevels(true, depth: N)`. |
| `ShowLines` | `bool` | `false` | yes | Connector lines between parent/child. Off by default. |
| `LineColor` | `SKColor` | `(80,80,80)` | yes | Only rendered when `ShowLines` is true. |
| `LineThickness` | `float` | `1.0` | yes | Connector stroke thickness. |

Children selectors are **not** a bindable property — register one per concrete node type:

```csharp
.SetChildrenSelector<TItem>(item => item.Children.Cast<object>())
```

```csharp
// Single-type tree (TreeNode). The children selector is REQUIRED even here.
new TreeView()
    .SetItemsSource(new List<object>
    {
        new TreeNode("Fruits",
            new TreeNode("Apple"),
            new TreeNode("Banana")),
        new TreeNode("Vegetables",
            new TreeNode("Carrot")),
    })
    .SetChildrenSelector<TreeNode>(n => n.Children.Cast<object>())
    .SetItemTemplate((item, _) => new Label()
        .SetText(((TreeNode)item).Name)
        .SetVerticalAlignment(VerticalAlignment.Center))
    .SetItemHeight(30)
    .SetIndentation(28)
    .SetShowLines(true)
    .SetLineColor(new SKColor(120, 120, 120))
    .SetDesiredHeight(260);
```

```csharp
// Heterogeneous tree: register a selector per type that can have children.
new TreeView()
    .SetItemsSource(projects)
    .SetChildrenSelector<Project>(p => p.Tasks.Cast<object>())
    .SetChildrenSelector<ProjectTask>(t => t.SubTasks.Cast<object>())
    .SetItemTemplate((item, depth) => item switch
    {
        Project p => new Label().SetText(p.Name).SetFontWeight(FontWeight.Bold),
        ProjectTask t => new HStack().AddChildren(
            new Checkbox().BindIsChecked(() => t.IsComplete, v => t.IsComplete = v),
            new Label().SetText(t.Title)),
        _ => new Label().SetText(item.ToString())
    })
    .BindSelectedItem(() => vm.Selected, v => vm.Selected = v)   // two-way: getter + setter
    .SetDesiredHeight(400);
```

```csharp
// Imperative expand control + auto-expand the first 3 levels
var treeView = new TreeView()
    .SetItemsSource(fileSystem.RootFolders)
    .SetChildrenSelector<Folder>(f => f.SubFolders.Cast<object>())
    .SetItemTemplate((item, _) => new Label().SetText(((Folder)item).Name))
    .SetAutoExpandInitialLevels(true, depth: 3);

treeView.ExpandNode(someFolder);
bool isExpanded = treeView.IsNodeExpanded(someFolder);
treeView.ToggleNode(someFolder);
```

**Rules**

- Use the `depth` parameter in the template for level-dependent styling.
- Persist expansion across refreshes with `SetExpandedKeys(keys, keySelector)` — call it **before** `SetItemsSource`.
- After mutating data in place, call `Refresh()` to rebuild while preserving expansion state.

**Gotchas**

- You MUST call `SetChildrenSelector<T>()` for every type that can have children — even single-type trees won't expand without it.
- `BindSelectedItem` is two-way: it needs a getter lambda AND an `Action<object?>` setter. (Profile JSON examples that show `nameof(...)` are wrong — the API is lambda-based.)
- `AutoExpandInitialLevels` applies only to the first `BuildNodes()` after `SetItemsSource`; later `Refresh()` calls do not re-auto-expand.
- `ShowLines` is `false` by default — no connectors unless enabled.
- Keep `ItemHeight` consistent with the template's measured height or content misaligns.
- Clicks on the expander region (`depth × Indentation`, width `ExpanderSize`) toggle expansion; clicks on the template content select the node.

---

## DataGrid&lt;T&gt;

Virtualized tabular grid with typed columns, selection modes, alternating/per-row styling, and independent vertical/horizontal scrolling. Base: `UiLayoutElement<DataGrid<T>>` (implements `IScrollableControl`, `IInputControl`, `IHoverableControl`). Construct as `new DataGrid<T>()`.

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|:-----:|-------|
| `ItemsSource` | `IEnumerable<T>?` | `null` | yes | `INotifyCollectionChanged` refreshes automatically. |
| `Columns` | `List<DataGridColumn<T>>` | empty | no | Add via `AddColumn()`. See column types below. |
| `SelectionMode` | `SelectionMode` | `Single` | yes | `None`, `Single`, or `Multiple`. |
| `SelectedItem` | `T?` | `null` | yes (two-way) | Last selected item in `Multiple` mode. |
| `SelectedItems` | `List<T>` | empty | no | Read-only. Mutate via `SelectItem()`/`DeselectItem()`/`ClearSelection()`. |
| `RowHeight` | `float` | `32` | yes | Data row height. |
| `HeaderHeight` | `float` | `36` | yes | Header row height. |
| `CellPadding` | `Margin` | `Margin(8,4)` | yes | Applied to ALL cells including headers. |
| `AlternatingRowStyles` | `bool` | `true` | yes | Uses `EvenRowStyle`/`OddRowStyle`. |
| `EvenRowStyle` | `DataGridRowStyle?` | `null` | yes | Even-row style when alternating is on. |
| `OddRowStyle` | `DataGridRowStyle?` | `null` | yes | Odd-row style when alternating is on. |
| `RowStyleCallback` | `Func<T, int, DataGridRowStyle>?` | `null` | yes | Per-row styling; takes precedence over alternating styles. |
| `ShowRowSeparators` | `bool` | `true` | yes | Horizontal lines between rows. |
| `ShowColumnSeparators` | `bool` | `true` | yes | Vertical lines between columns. |
| `SeparatorColor` | `SKColor` | `(60,60,60)` | yes | Row/column separator color. |
| `SeparatorThickness` | `float` | `1` | yes | Separator line thickness. |
| `HeaderSeparatorColor` | `SKColor` | `(100,100,100)` | yes | Line between header and data. |
| `ScrollOffset` | `float` | `0` | yes | Vertical scroll position. |
| `HorizontalScrollOffset` | `float` | `0` | yes | Horizontal scroll position. |
| `ShowVerticalScrollbar` | `bool` | `true` | yes | Vertical scrollbar (below header). |
| `ShowHorizontalScrollbar` | `bool` | `true` | yes | Horizontal scrollbar (at bottom). |
| `VerticalScrollbar` | `Scrollbar` | auto | no | Read-only internal control. |
| `HorizontalScrollbar` | `Scrollbar` | auto | no | Read-only internal control. |

**Column types** (all `DataGridColumn<T>`, added via `AddColumn`): text, checkbox, button, link, image, combobox, datepicker, timepicker, progress, slider, editor, and template. Each configures with `SetHeader(string)` + a value binding + `SetWidth(...)`.

**Column widths**: `DataGridColumnWidth.Star(n)` (proportional), `.Absolute(px)` (fixed), `.Auto` (currently a 100px placeholder — true content measurement not yet implemented).

```csharp
// Text columns with star + absolute widths
new DataGrid<Product>()
    .SetItemsSource(products)
    .AddColumn(new DataGridTextColumn<Product>()
        .SetHeader("Name")
        .SetBinding(p => p.Name)                          // Func<T,string>
        .SetWidth(DataGridColumnWidth.Star(2)))
    .AddColumn(new DataGridTextColumn<Product>()
        .SetHeader("Price")
        .SetBinding(p => p.Price.ToString("C"))
        .SetWidth(DataGridColumnWidth.Absolute(100)))
    .SetSelectionMode(SelectionMode.Single)
    .SetRowHeight(34)
    .SetDesiredHeight(400);
```

```csharp
// Checkbox column (two-way binding) + template column
new DataGrid<TaskRow>()
    .SetItemsSource(tasks)
    .AddColumn(new DataGridCheckboxColumn<TaskRow>()
        .SetHeader("Done")
        .SetBinding(t => t.IsComplete, (t, v) => t.IsComplete = v)   // getter + setter
        .SetWidth(DataGridColumnWidth.Absolute(60)))
    .AddColumn(new DataGridTextColumn<TaskRow>()
        .SetHeader("Title")
        .SetBinding(t => t.Title)
        .SetWidth(DataGridColumnWidth.Star(1)))
    .AddColumn(new DataGridTemplateColumn<TaskRow>()
        .SetHeader("Status")
        .SetCellTemplate((t, _) => new Label()
            .SetText(t.IsComplete ? "Done" : "Pending")
            .SetTextColor(t.IsComplete ? Colors.Green : Colors.Gray)))
    .SetDesiredHeight(400);
```

```csharp
// Two-way selection binding + per-row styling by status
new DataGrid<Order>()
    .SetItemsSource(orders)
    .AddColumn(new DataGridTextColumn<Order>().SetHeader("Order #").SetBinding(o => o.Id))
    .AddColumn(new DataGridTextColumn<Order>().SetHeader("Customer").SetBinding(o => o.CustomerName))
    .SetSelectionMode(SelectionMode.Single)
    .BindSelectedItem(() => vm.SelectedOrder, o => vm.SelectedOrder = o)
    .SetRowStyleCallback((order, idx) => order.Status switch
    {
        "Pending"   => new DataGridRowStyle(new SolidColorBackground(new Color(255, 200, 100, 50))),
        "Completed" => new DataGridRowStyle(new SolidColorBackground(new Color(100, 255, 100, 50))),
        _           => new DataGridRowStyle()
    })
    .SetDesiredHeight(400);
```

**Rules**

- Add all columns *before* `SetItemsSource` for optimal first layout.
- Chain order: columns → `SetItemsSource` → selection/styling → `SetDesiredHeight`/`Width`.
- Star widths only distribute remaining space if at least one column uses `Star`.

**Gotchas**

- `SelectedItems` is read-only — never call `SelectedItems.Add()`; use `SelectItem()`/`DeselectItem()`/`ClearSelection()`.
- `BindSelectedItem` takes a getter lambda **and** a setter (`() => vm.X, x => vm.X = x`). It is two-way, unlike one-way `Bind*` methods elsewhere.
- `RowStyleCallback` overrides alternating styles, but selected rows ALWAYS get the blue highlight regardless.
- The `idx` in `RowStyleCallback` is the real index in `ItemsSource`, not the on-screen visual position.
- `CellPadding` also affects header cells.
- `DataGridColumnWidth.Auto` currently behaves as a fixed 100px.
- Vertical and horizontal scrollbars can coexist, reducing the viewport.
- Custom header templates are not supported — headers are text-only via `SetHeader()`.

---

## TabControl

Organizes content into tabbed panes; only the selected tab's content is rendered. Base: `UiLayoutElement` (implements `IInputControl`, `IFocusable`, `IKeyboardInputHandler`, `IHoverableControl`). Tabs are `TabItem` objects (`SetHeader`, `SetContent`, `SetIsEnabled`).

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|:-----:|-------|
| `SelectedIndex` | `int` | `-1` until first tab | yes (two-way) | Setting to `-1` or a disabled index is silently rejected. First added tab auto-selects. |
| `SelectedTab` | `TabItem` | `null` | yes | Change via `SetSelectedTab()`/`SetSelectedIndex()`. |
| `Tabs` | `IReadOnlyList<TabItem>` | empty | yes | Manage with `AddTab`/`RemoveTab`/`SetTabs`/`BindTabs`. |
| `TabPosition` | `TabPosition` | `Top` | yes | `Top`/`Bottom`/`Left`/`Right`. Controls keyboard nav axis. |
| `HeaderTextSize` | `float` | `PlusUiDefaults.FontSize` | yes | Header font size. |
| `HeaderTextColor` | `SKColor` | `TextPrimary` | yes | Inactive tab text. |
| `ActiveHeaderTextColor` | `SKColor` | `AccentSuccess` (green) | yes | Selected tab text — green, not the usual accent. |
| `DisabledHeaderTextColor` | `SKColor` | `BorderColor` | yes | Disabled tab text. |
| `HeaderBackgroundColor` | `SKColor` | `BackgroundPrimary` | yes | Whole header bar. |
| `ActiveTabBackgroundColor` | `SKColor` | `BackgroundHover` | yes | Selected tab background. |
| `HoverTabBackgroundColor` | `SKColor` | `BackgroundSecondary` | yes | Hovered tab background. |
| `TabIndicatorColor` | `SKColor` | `AccentSuccess` | yes | Active-tab indicator bar. |
| `TabIndicatorHeight` | `float` | `3` | yes | Indicator thickness; `0` hides it. |
| `TabPadding` | `Margin` | `Margin(16,10)` | yes | Inner padding per tab header. |
| `TabSpacing` | `float` | `0` | yes | Gap between tab headers. |
| `OnSelectedIndexChanged` | `Action<int>` | `null` | yes | Callback with new index. |
| `SelectionChangedCommand` | `ICommand` | `null` | yes | WPF-style command receiving new index. |

Inherited from `UiElement` (all bindable): `IsVisible`, `Margin`, `Background` (content area only, not header), `HorizontalAlignment`, `VerticalAlignment`.

```csharp
// Basic top tabs
new TabControl()
    .AddTab(new TabItem()
        .SetHeader("General")
        .SetContent(new VStack()
            .SetMargin(new Margin(12))
            .AddChild(new Label().SetText("General settings"))))
    .AddTab(new TabItem()
        .SetHeader("Advanced")
        .SetContent(new VStack()
            .AddChild(new Label().SetText("Advanced settings"))))
    .SetSelectedIndex(0);
```

```csharp
// Bottom bar, no indicator (mobile-style)
new TabControl()
    .SetTabPosition(TabPosition.Bottom)
    .SetHeaderBackgroundColor(new SKColor(30, 30, 30))
    .SetTabIndicatorHeight(0)
    .AddTab(new TabItem().SetHeader("Home").SetContent(homePage))
    .AddTab(new TabItem().SetHeader("Search").SetContent(searchPage))
    .AddTab(new TabItem().SetHeader("Profile").SetContent(profilePage));
```

```csharp
// Left rail with two-way SelectedIndex binding
new TabControl()
    .SetTabPosition(TabPosition.Left)
    .SetDesiredWidth(800)
    .SetHeaderBackgroundColor(new SKColor(25, 25, 25))
    .SetTabPadding(new Margin(20, 12))
    .AddTab(new TabItem().SetHeader("Dashboard").SetContent(dashboard))
    .AddTab(new TabItem().SetHeader("Reports").SetContent(reports))
    .BindSelectedIndex(() => vm.ActiveSection, i => vm.ActiveSection = i);
```

**Rules**

- Set `TabPosition` before adding tabs / first layout; changing it later requires `InvalidateMeasure()`.
- Disable a tab with `TabItem.SetIsEnabled(false)` — keyboard nav skips it and clicks are ignored.
- Use `SetOnSelectedIndexChanged(i => ...)` for callbacks or `SetSelectionChangedCommand(cmd)` for ICommand.

**Gotchas**

- Only the selected tab's content is drawn; others are measured but not rendered (by design).
- `ActiveHeaderTextColor` defaults to **green** (`AccentSuccess`), not the usual blue accent.
- `SelectedIndex` is `-1` before any tab exists; the first added tab becomes index 0 automatically.
- Setting `SelectedIndex` to a disabled tab is silently ignored.
- `TabItem.Icon` is declared but not rendered — reserved for future use.
- Keyboard nav axis follows position: left/right arrows for `Top`/`Bottom`, up/down for `Left`/`Right`.
- `BindSelectedIndex` needs the `Action<int>` setter for two-way; without it, VM changes won't write back to the view.

---

## Common LLM mistakes

- **Using `nameof(...)` in `Bind*` calls.** PlusUi binds with lambda expressions: `.BindText(() => vm.Title)`, `.BindSelectedItem(() => vm.Sel, v => vm.Sel = v)`. There is no string-path overload.
- **Forgetting `SetChildrenSelector<T>()` on `TreeView`** — the tree won't expand, even for a single node type.
- **Omitting `SetDesiredHeight`/`Width`** on `ItemsList`/`TreeView`/`DataGrid` — defeats virtualization; all items lay out.
- **Setting only `ItemsSource` or only `ItemTemplate`** on `ItemsList` — nothing renders without both.
- **Returning `null` from an `ItemTemplate`/`SetCellTemplate`** — `ItemsList` throws.
- **Mutating `DataGrid.SelectedItems` directly** — it is read-only; use `SelectItem`/`DeselectItem`/`ClearSelection`.
- **Treating `ScrollOffset` as an item index** — it is pixels.
- **Expecting `TabControl`'s active text to be blue** — it is green by default.
- **Forgetting the setter on two-way binds** (`BindSelectedItem`, `BindSelectedIndex`, `BindIsChecked`) — external VM updates won't propagate.
