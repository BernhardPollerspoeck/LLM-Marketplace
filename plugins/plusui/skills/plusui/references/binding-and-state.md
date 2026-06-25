# PlusUi Data Binding & State

How PlusUi connects ViewModel state to the visual tree. PlusUi has **no XAML** — UI is built in C# with a fluent builder, and binding is done with C# lambda expressions, not string property paths.

## Table of contents

- [Mental model](#mental-model)
- [The ViewModel: ObservableObject + INotifyPropertyChanged](#the-viewmodel)
- [Bind* methods: what they accept](#bind-methods-what-they-accept)
- [How the expression path is extracted](#how-the-expression-path-is-extracted)
- [One-way binding (VM → View)](#one-way-binding-vm--view)
- [Two-way binding (VM ↔ View) for inputs](#two-way-binding-vm--view-for-inputs)
- [Value converters](#value-converters)
- [Commands (button binding)](#commands-button-binding)
- [Non-observable callbacks](#non-observable-callbacks)
- [How binding context flows page → children](#how-binding-context-flows-page--children)
- [Control binding cheat-sheet](#control-binding-cheat-sheet)
- [Common LLM mistakes](#common-llm-mistakes)
- [Gotchas](#gotchas)
- [Old patterns (do not use)](#old-patterns-do-not-use)

## Mental model

- Every `UiElement` has a `Context` (the binding source, an `INotifyPropertyChanged`). The page sets it to the ViewModel and it flows down to all children automatically.
- `Set*` methods set a **static** value once. `Bind*` methods register a **live** binding that re-runs when the bound property raises `PropertyChanged`.
- Rule (enforced project-wide): every `Set*` method has a matching `Bind*` method.
- Binding is one-way (VM → control) by default. Input controls add a **two-way** `Bind*` overload that also takes a setter `Action<T>` to write the user's edits back to the VM.
- Internally each `Bind*` call compiles the lambda to a getter, extracts the property path, and creates a `PathBindingTracker` that subscribes to `PropertyChanged` along that path.

## The ViewModel

PlusUi uses **CommunityToolkit.Mvvm**. ViewModels are `partial class : ObservableObject` (which implements `INotifyPropertyChanged`). Source generators turn fields into observable properties and methods into commands.

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class CounterViewModel(ICounterService service) : ObservableObject
{
    [ObservableProperty]                       // generates public int Count { get; set; } + PropertyChanged
    private int _count;

    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(Display))] // raise PropertyChanged for Display when Name changes
    private string _name = "";

    public string Display => $"{Name}: {Count}"; // computed property, no backing field

    partial void OnCountChanged(int value)      // generated hook, runs after Count is set
        => service.Persist(value);

    [RelayCommand]                              // generates IncrementCommand : ICommand
    private void Increment() => Count++;

    [RelayCommand]                              // async → IsRunning, CanExecute support
    private async Task LoadAsync() => Count = await service.GetAsync();
}
```

Key rules:
- The backing field name `_count` generates property `Count` (underscore stripped, PascalCased).
- A computed/derived property only updates the UI if something raises its `PropertyChanged` — use `[NotifyPropertyChangedFor(nameof(...))]` on the source property.
- `[RelayCommand] void Foo()` → property `FooCommand`. `[RelayCommand] Task FooAsync()` → `FooCommand` (the `Async` suffix is dropped).
- A ViewModel does **not** have to use CommunityToolkit — any `INotifyPropertyChanged` works — but the toolkit is the standard.

## Bind* methods: what they accept

Every `Bind*` takes a **lambda expression**, `Expression<Func<T>>` — never a property-name string.

```csharp
new Label().BindText(() => vm.Name);          // CORRECT — expression
```

The expression serves two purposes at once:
1. It is **compiled** into a getter `Func<T>` that produces the current value.
2. Its tree is **parsed** to discover which property names to subscribe to for change notification.

Because it is a real C# expression, it is compile-time checked and refactor-safe. You can:
- read a simple property: `() => vm.Title`
- read a nested chain: `() => vm.Settings.Theme.Accent`
- compute inline: `() => $"${vm.Price:F2}"`, `() => vm.IsBusy ? "Loading" : "Ready"`, `() => vm.Count.ToString()`

## How the expression path is extracted

`ExpressionPathService.GetPropertyPath` produces the segments that `PathBindingTracker` subscribes to:

| Expression | Extracted path | Subscribes to |
|------------|---------------|---------------|
| `() => vm.Title` | `["Title"]` | `vm.PropertyChanged` for `Title` |
| `() => vm.Settings.Theme` | `["Settings", "Theme"]` | `vm` for `Settings`, `vm.Settings` for `Theme` (re-subscribes if `Settings` swapped) |
| `() => $"{vm.First} {vm.Last}"` | `["First", "Last"]` (set of all members) | both `First` and `Last` |
| `() => vm.IsOn ? vm.A : vm.B` | `["IsOn", "A", "B"]` | all three |
| `() => count` (local, not on VM) | `["count"]` | nothing observable — value is read once |

Rules the parser applies:
- A clean property **chain** ending at the captured closure (the `vm` variable) has its first segment (`vm`) stripped, leaving the real property path.
- For **complex** expressions (conditionals, string interpolation, method calls, `new`), it collects every distinct member name in the tree so a change to any of them refreshes the binding.
- For nested chains the `PathBindingTracker` re-subscribes downstream segments when an intermediate object instance is replaced — so `vm.Settings.Theme` keeps working after `vm.Settings = new(...)`.

## One-way binding (VM → View)

Pass a single lambda. The control updates whenever the bound property (or any member in the expression) changes.

```csharp
new Label().BindText(() => vm.Title)
new Label().BindText(() => $"${vm.Price:F2}")          // formatted inline
new Label().BindTextColor(() => vm.IsError ? Colors.Red : Colors.White)
new Label().BindIsVisible(() => vm.HasResults)
new ProgressBar().BindProgress(() => vm.Progress)
new Image().BindSource(() => vm.AvatarUrl)
```

`Label`/`UiTextElement` also has a typed one-way formatter overload:

```csharp
new Label().BindText(() => vm.Count, c => $"{c} items");   // Func<T,string?> formatter
```

## Two-way binding (VM ↔ View) for inputs

Interactive controls add a `Bind*` overload that **also takes a setter** `Action<T>`. The getter pushes VM → control; the setter pushes user edits control → VM. When the user types/toggles/drags, the control invokes the setter, which assigns the VM property (raising `PropertyChanged`, which the binding observes).

```csharp
// Entry (text input)
new Entry()
    .SetPlaceholder("Enter username...")
    .BindText(() => vm.Username, v => vm.Username = v)

// Toggle (switch)
new Toggle().BindIsOn(() => vm.Enabled, v => vm.Enabled = v)

// Slider
new Slider().BindValue(() => vm.Volume, v => vm.Volume = v)

// Checkbox
new Checkbox().BindIsChecked(() => vm.AcceptTerms, v => vm.AcceptTerms = v)

// RadioButton (setter optional)
new RadioButton().BindIsSelected(() => vm.IsOptionA, v => vm.IsOptionA = v)

// ComboBox<T>
new ComboBox<string>()
    .SetItemsSource(new[] { "Red", "Green", "Blue" })
    .BindSelectedItem(() => vm.Color, v => vm.Color = v)

// DatePicker / TimePicker (two-arg overload is the two-way one)
new DatePicker().BindSelectedDate(() => vm.Date, v => vm.Date = v)
new TimePicker().BindSelectedTime(() => vm.Time, v => vm.Time = v)
```

> The single-argument form (e.g. `BindText(() => vm.X)` on an `Entry`, or `BindSelectedDate(() => vm.X)`) is **one-way only** — the control displays the value but never writes back. Always pass the setter for editable fields.

## Value converters

When the VM type differs from the control's native type, use the generic overload with `toControl` (source → control) and `toSource` (control → source) converters. Available on `Entry.BindText<T>`, `Toggle.BindIsOn<T>`, `Slider.BindValue<T>`, `Checkbox.BindIsChecked<T>`.

```csharp
// Bind an int VM property to a text Entry
new Entry().BindText(
    () => vm.Age,
    v => vm.Age = v,
    toControl: age => age.ToString(),
    toSource:  text => int.TryParse(text, out var n) ? n : 0);

// Bind an enum to a Toggle
new Toggle().BindIsOn(
    () => vm.Mode,
    v => vm.Mode = v,
    toControl: mode => mode == Mode.Dark,
    toSource:  on => on ? Mode.Dark : Mode.Light);
```

## Commands (button binding)

`Button` exposes `Command` (an `ICommand`), `CommandParameter`, and a plain `OnClick` action. On click it runs `OnClick` then, if `Command.CanExecute(parameter)` is true, `Command.Execute(parameter)`.

```csharp
// Bind a RelayCommand-generated command by reference (most common)
new Button().SetText("Save").SetCommand(vm.SaveCommand)

// With a parameter
new Button()
    .SetText("Delete")
    .SetCommand(vm.DeleteCommand)
    .SetCommandParameter(item.Id)

// Bind the command itself if it can change at runtime
new Button().BindCommand(() => vm.CurrentAction)
new Button().BindCommandParameter(() => vm.SelectedId)

// No-VM lambda handler
new Button().SetText("+").SetOnClick(() => vm.Count++)
```

- `SetCommand` takes the command **object** (`vm.SaveCommand`), not a lambda — it is not data that changes, so a static set is correct. Use `BindCommand(() => vm.X)` only when the command instance is swapped at runtime.
- A `RelayCommand` with a `CanExecute` predicate disables execution automatically; the button still renders but `Execute` is skipped when `CanExecute` is false.
- `CommandParameter` is passed to both `CanExecute` and `Execute`. Pattern from the demo sidebar: a single `NavigateToDemoCommand` with a per-row `SetCommandParameter(name)`.

## Non-observable callbacks

For fire-and-forget reactions without binding to a VM property, controls offer `SetOn*` callbacks. These do **not** require `INotifyPropertyChanged`.

```csharp
new Entry().SetOnTextChanged(t => Console.WriteLine(t))
new Slider().SetOnValueChanged(v => vm.ApplyVolume(v))
new Checkbox().SetOnIsCheckedChanged(b => ...)
new ComboBox<string>().SetOnSelectionChanged(v => vm.Selected = v ?? "none")
```

## How binding context flows page → children

You normally **never set `Context` manually** — it propagates automatically.

1. A page is `UiPageElement(INotifyPropertyChanged vm)`; the VM is stored as `ViewModel`.
2. `BuildPage()` calls your `Build()` to create the tree, then sets `tree.Context = ViewModel`.
3. `Context` is overridden on `UiLayoutElement` so its setter assigns the same context to **every child recursively**. So the whole visual subtree shares the page's ViewModel as binding source.
4. Setting `Context` (un)subscribes from the old/new context's `PropertyChanged` and refreshes all registered bindings and `PathBindingTracker`s.

```csharp
public class ProductPage(ProductViewModel vm) : UiPageElement(vm)   // vm becomes the Context
{
    protected override UiElement Build() => new VStack(
        new Label().BindText(() => vm.Name),          // resolves against vm
        new Entry().BindText(() => vm.Sku, v => vm.Sku = v),
        new Button().SetText("Save").SetCommand(vm.SaveCommand));
}
```

Because the lambdas already capture `vm` directly via the closure, the binding works through that closure. The flowed `Context` is what drives `PropertyChanged` subscription/refresh. For nested item templates (e.g. `ItemsList<T>`), the item element's `Context` is set to the item, so bind against the item there.

## Control binding cheat-sheet

| Control | One-way | Two-way (with setter) | Notes |
|---------|---------|------------------------|-------|
| `Label` / text | `BindText(() => vm.X)`, `BindText(() => vm.X, fmt)` | — | also `BindTextColor`, `BindTextSize`, `BindFontWeight` |
| `Entry` | — | `BindText(() => vm.X, v => vm.X = v)` | converter overload `BindText<T>(.., toControl, toSource)` |
| `Toggle` | — | `BindIsOn(() => vm.X, v => vm.X = v)` | converter overload `BindIsOn<T>` |
| `Checkbox` | — | `BindIsChecked(() => vm.X, v => vm.X = v)` | converter overload `BindIsChecked<T>` |
| `RadioButton` | `BindIsSelected(() => vm.X)` | `BindIsSelected(() => vm.X, v => vm.X = v)` | setter optional; `BindGroup`, `BindValue` |
| `Slider` | — | `BindValue(() => vm.X, v => vm.X = v)` | `BindMinimum` / `BindMaximum`; converter `BindValue<T>` |
| `ComboBox<T>` | `BindItemsSource(() => vm.Items)` | `BindSelectedItem(() => vm.X, v => vm.X = v)`, `BindSelectedIndex(() => vm.I, v => vm.I = v)` | |
| `DatePicker` | `BindSelectedDate(() => vm.X)` | `BindSelectedDate(() => vm.X, v => vm.X = v)` | `DateOnly?`; `BindMinDate` / `BindMaxDate` |
| `TimePicker` | `BindSelectedTime(() => vm.X)` | `BindSelectedTime(() => vm.X, v => vm.X = v)` | `TimeOnly?` |
| `Button` | `BindCommand(() => vm.Cmd)` | — | prefer `SetCommand(vm.Cmd)`; `SetCommandParameter` |
| any `UiElement` | `BindIsVisible`, `BindOpacity`, `BindBackground`, `BindMargin`, etc. | — | generated for every `Set*` |

## Common LLM mistakes

- **Passing a property-name string.** `BindText(nameof(vm.Title), () => vm.Title)` is the OLD API. The current API is `BindText(() => vm.Title)` — one lambda, no `nameof`.
- **Using `Set*` for dynamic data.** `new Label().SetText(vm.Title)` captures the value once and never updates. Use `BindText(() => vm.Title)`.
- **Forgetting the setter on inputs.** `new Entry().BindText(() => vm.Name)` is a COMPILE ERROR — `Entry.BindText` has no one-parameter overload. Editable fields require both a getter and a setter: `BindText(() => vm.Name, v => vm.Name = v)`.
- **Plain auto-properties on the VM.** A property must raise `PropertyChanged` (use `[ObservableProperty]`) or the UI never refreshes. A computed property needs `[NotifyPropertyChangedFor(nameof(Computed))]` on its inputs.
- **`SetCommand(() => vm.Save())`.** `SetCommand` takes an `ICommand` (`vm.SaveCommand`), not a lambda. For a lambda handler use `SetOnClick(() => ...)`.
- **Wrong command name.** `[RelayCommand] void Save()` generates `SaveCommand`; `[RelayCommand] Task LoadAsync()` generates `LoadCommand` (the `Async` suffix is dropped).
- **Setting `Context` by hand.** It flows from the page automatically; do not assign `element.Context`.
- **Caching control references** to push updates manually. Build fresh in `Build()` and bind; do not store `Label` fields and call `SetText` later.

## Gotchas

- Bindings refresh only via `INotifyPropertyChanged`. There is a legacy string-keyed fallback (`RegisterPathBinding` also registers each segment for manual `UpdateBindings()`), but in app code rely on `[ObservableProperty]`.
- Collections: `SetItemsSource(...)` with a plain `List<T>` is a one-time snapshot. For live add/remove use an `ObservableCollection<T>` and the collection control's items-source binding.
- Nested chains (`() => vm.A.B.C`) re-subscribe correctly when an intermediate (`A` or `B`) instance is replaced — that is supported, not a bug to work around.
- Local variables in the lambda (not on an `INotifyPropertyChanged`) are read once and never tracked.
- Two-way write-back fires through the control's internal setter list on user interaction (e.g. `Entry` on text change, `Toggle` on tap, `Slider` on drag/arrow-key); programmatic VM changes flow the other way through the getter.
- `Bind*` returns the control (fluent), so chain it: `new Entry().SetPlaceholder("...").BindText(() => vm.X, v => vm.X = v)`.

## Old patterns (do not use)

Some older docs show string-based binding. These are obsolete — the live API is expression-only.

```csharp
// OBSOLETE — do not generate this
new Label().BindText(nameof(vm.Title), () => vm.Title);
new Entry().BindText(nameof(vm.Q), () => vm.Q, value => vm.Q = value);

// CURRENT
new Label().BindText(() => vm.Title);
new Entry().BindText(() => vm.Q, value => vm.Q = value);
```
