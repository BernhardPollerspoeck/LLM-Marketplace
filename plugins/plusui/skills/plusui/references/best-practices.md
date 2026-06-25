# PlusUi Best Practices (Do & Don't)

Cross-cutting guidance for writing maintainable, correct, performant PlusUi apps.
Each rule pairs a **Do** with the matching **Don't**. For per-control pitfalls, see the
"Common LLM mistakes" section at the end of each control reference.

> All binding examples use PlusUi's **expression** API (`Bind*(() => vm.X)`). There is **no**
> `nameof`-based overload — `BindText(nameof(vm.X), () => vm.X)` does **not** compile. Some older
> repo docs still show the `nameof` form; it is outdated.

## Quick checklist

- [ ] Logic lives in the ViewModel; the Page only builds UI.
- [ ] Use `Bind*` for anything that changes; `Set*` alone is a one-time static value.
- [ ] Keep the layout hierarchy shallow — reach for `Grid` instead of deeply nested stacks.
- [ ] Build controls fresh in `Build()`; never cache a control in a field.
- [ ] Use `async`/`await` in commands; never `.Result` / `.Wait()`.
- [ ] Release resources in `Dispose()` or `Disappearing()`; use the page lifecycle.
- [ ] Prefer built-in controls before writing a custom one.
- [ ] Custom controls: `partial class` + `[GenerateShadowMethods]`, and a `Bind*` for every `Set*`.
- [ ] Colors are `SKColor` (SkiaSharp) — `SKColors.*` / `new SKColor(r, g, b)`.

---

## 1. MVVM: logic in the ViewModel, not the Page

The Page's only job is to compose the UI tree in `Build()`. State, data access, and commands
belong in the ViewModel (`ObservableObject` + CommunityToolkit source generators).

**Do**

```csharp
public partial class ProductViewModel(IProductService products) : ObservableObject
{
    [ObservableProperty] private string _productName = "";
    [ObservableProperty] private decimal _price;

    [RelayCommand]
    private async Task LoadProductAsync(int id)
    {
        var product = await products.GetProductAsync(id);
        ProductName = product.Name;
        Price = product.Price;
    }
}

public class ProductPage(ProductViewModel vm) : UiPageElement(vm)
{
    protected override UiElement Build() => new VStack(
        new Label().BindText(() => vm.ProductName),
        new Label().BindText(() => vm.Price, p => $"${p:F2}")   // formatter overload
    );
}
```

**Don't** — do work or block in the page:

```csharp
public class ProductPage(IProductService service) : UiPageElement(null!)
{
    protected override UiElement Build()
    {
        var product = service.GetProduct(1).Result;   // BAD: blocking + logic in the view
        return new Label().SetText(product?.Name);
    }
}
```

---

## 2. Binding: `Bind*` for live values, `Set*` for static

If a value comes from the ViewModel and can change, **bind** it. `Set*` writes a value once and
never updates. Every `Set*` has a matching `Bind*` — never wire `INotifyPropertyChanged` by hand.

| Intent | API |
|---|---|
| One-way (VM → View) | `.BindText(() => vm.Title)` |
| One-way with formatting | `.BindText(() => vm.Count, c => c.ToString())` |
| Two-way (input controls) | `.BindText(() => vm.Query, v => vm.Query = v)` |
| Command | `.SetCommand(vm.SaveCommand)` (generated from `[RelayCommand] Save`) |

**Do**

```csharp
new Label().BindText(() => vm.Title);                         // updates on change
new Entry().BindText(() => vm.Query, v => vm.Query = v);      // two-way
new Slider().BindValue(() => vm.Volume, v => vm.Volume = v);  // two-way
new Button().SetText("Save").SetCommand(vm.SaveCommand);
```

**Don't**

```csharp
new Label().SetText(vm.Title);          // BAD: static snapshot, never updates
new Entry().BindText(() => vm.Query);   // BAD: one-way only — user edits won't flow back
new Button().SetOnClick(() => vm.Save()); // avoid for VM logic — prefer SetCommand(vm.SaveCommand)
```

- `[RelayCommand] private void Save()` generates `SaveCommand` — bind `vm.SaveCommand`, not `vm.Save`.
- Use `[RelayCommand] private async Task SaveAsync()` for async work; it generates `SaveCommand`.

---

## 3. Build controls in `Build()` — never cache references

`Build()` produces the tree; the framework owns the control instances and may rebuild. Drive
changes through bindings, not by holding a control and mutating it later.

**Do**

```csharp
protected override UiElement Build() => new VStack(
    new Label().BindText(() => vm.Title),
    new Button().SetText("Click").SetCommand(vm.ClickCommand)
);
```

**Don't**

```csharp
private Label? _title;                         // BAD: caching a control
protected override UiElement Build()
{
    _title = new Label().SetText("Title");
    return _title;
}
public void UpdateTitle(string t) => _title?.SetText(t);  // BAD: mutate via binding instead
```

To change what a label shows, change the bound VM property — the binding updates the UI.

---

## 4. Keep the layout hierarchy shallow

Deeply nested `VStack`/`HStack` chains hurt readability and measure/arrange performance. Use
`Grid` (rows/columns) for 2-D layouts instead of nesting stacks inside stacks.

**Do**

```csharp
new Grid()
    .AddColumn(Column.Star)
    .AddColumn(Column.Auto)
    .AddRow(Row.Auto)
    .AddChild(new Label().BindText(() => vm.Name), row: 0, column: 0)
    .AddChild(new Button().SetText("Edit").SetCommand(vm.EditCommand), row: 0, column: 1);
```

**Don't**

```csharp
new VStack(new HStack(new VStack(new HStack(   // BAD: over-nesting
    new Label().SetText("Deep!")))));
```

- Use `Spacing` for gaps between children instead of wrapping each child in a margin container.
- Pick the right container: `VStack`/`HStack` for 1-D lists, `Grid`/`UniformGrid` for 2-D,
  `ItemsList<T>` for data-driven collections.

---

## 5. Async commands — never block the UI thread

**Do**

```csharp
[ObservableProperty] private bool _isLoading;

[RelayCommand]
private async Task LoadDataAsync()
{
    IsLoading = true;
    try { Items = await _service.GetDataAsync(); }
    finally { IsLoading = false; }
}
```

**Don't**

```csharp
[RelayCommand]
private void LoadData()
{
    Items = _service.GetDataAsync().Result;   // BAD: blocks the UI thread
}
```

---

## 6. Use the page lifecycle; release resources

Pages expose `Appearing()`, `Disappearing()`, `OnNavigatedTo(object?)`, and `OnNavigatedFrom()`.
Start/stop work there, and dispose anything you own.

**Do**

```csharp
public class DashboardPage(DashboardViewModel vm) : UiPageElement(vm), IDisposable
{
    private Timer? _timer;

    public override void Appearing()
    {
        base.Appearing();
        _timer = new Timer(_ => vm.Refresh(), null, 0, 1000);   // start work when visible
    }

    public override void Disappearing()
    {
        base.Disappearing();
        _timer?.Dispose();                                       // stop when hidden
        _timer = null;
    }

    public void Dispose() => _timer?.Dispose();
}
```

**Don't** start timers, subscriptions, or polling in the constructor and never stop them — that
leaks across navigation.

---

## 7. Prefer built-in controls

PlusUi ships a large control set (see the other reference files). Check for an existing control
before building your own — `TabControl`, `DataGrid<T>`, `TreeView`, `ComboBox<T>`, `Grid`,
`UniformGrid`, pickers, etc. Build a custom control only when nothing fits.

---

## 8. Custom controls: `partial` + `[GenerateShadowMethods]`, and pair Set/Bind

Reusable controls compose existing ones (usually via `UserControl`) and are `partial` with
`[GenerateShadowMethods]` so fluent shadow methods are generated. For each `Set*` you add,
add a matching `Bind*`.

**Do**

```csharp
using PlusUi.core.Attributes;

[GenerateShadowMethods]
public partial class LabeledValue : UserControl
{
    internal string Caption { get; set; } = "";
    public LabeledValue SetCaption(string value) { Caption = value; InvalidateMeasure(); return this; }
    public LabeledValue BindCaption(Expression<Func<string>> expr) { /* register binding */ return this; }
}
```

**Don't** redefine properties that already exist on the base (`Background`, `CornerRadius`,
`Margin`, …) — use the inherited `Set*`/`Bind*` instead:

```csharp
public partial class MyControl : UiElement
{
    internal Color BackgroundColor { get; set; }   // BAD: Background already exists on UiElement
}
```

See [usercontrol.md](usercontrol.md) and [architecture.md](architecture.md).

---

## 9. Margin vs. Padding, and consistent spacing

- `Margin` = space **outside** the element. `Padding` = space **inside** (e.g. around a button's text).
- Define spacing constants instead of sprinkling magic numbers.

```csharp
public static class Spacing
{
    public static readonly Margin Small  = new(4);
    public static readonly Margin Medium = new(8);
    public static readonly Margin Large  = new(16);
}

new Button()
    .SetMargin(Spacing.Medium)            // space around the button
    .SetPadding(new Margin(12, 6));       // space inside, around the text
```

---

## 10. Colors are SkiaSharp `SKColor`

```csharp
new Label().SetTextColor(SKColors.White);
new Solid(null, 1).SetBackground(new SKColor(45, 45, 45));
```

Don't use WPF `Brush`/`Color`, hex strings, or CSS color names. See [theming.md](theming.md).

---

## Common LLM mistakes (summary)

- `BindText(nameof(vm.X), () => vm.X)` — **no such overload**; use `BindText(() => vm.X)`.
- `SetText(vm.X)` for a value that changes — static; use `BindText(() => vm.X)`.
- One-arg `BindText` on an `Entry`/input you expect to edit — that's one-way; add the setter.
- Binding `vm.Save` instead of the generated `vm.SaveCommand`.
- Caching controls in fields and mutating them — drive UI through bound VM properties.
- `.Result` / `.Wait()` in a command — block the UI thread; use async commands.
- Re-implementing a built-in control; redefining inherited properties on a custom control.
- Using `Color`/hex where the API wants `SKColor`.
