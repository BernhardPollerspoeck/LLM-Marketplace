# PlusUi Text Controls

Controls for displaying and editing text: **Label** (static/dynamic text), **RichTextLabel** (mixed-style inline runs), **Link** (clickable hyperlink), **Entry** (text input), and **CodeEditor** (multi-line code editing with syntax highlighting). UI is built in C# with the fluent builder — no XAML.

## Table of contents

- [Conventions (read first)](#conventions-read-first)
- [Label](#label)
- [RichTextLabel](#richtextlabel)
- [Link](#link)
- [Entry](#entry)
- [CodeEditor](#codeeditor)
- [Common LLM mistakes](#common-llm-mistakes)

## Conventions (read first)

- **Every `SetX` has a matching `BindX`.** `Set*` writes a static value once; `Bind*` registers a live binding that re-runs on `PropertyChanged`. Property tables below only flag the rare property that has **no** `Bind*`.
- **`Bind*` takes a C# expression, never a property-name string.** Signature is `BindText(() => vm.Name)` — an `Expression<Func<T>>`. (The old `BindText(nameof(vm.Name), () => vm.Name)` form seen in some XML doc comments is NOT a real overload — do not use it.)
- **`Label`, `RichTextLabel`, `Link` are read-only bindings** (VM -> control). **`Entry`** and **`CodeEditor`** have a two-way `BindText` overload that also takes a setter `Action<T>` to push edits back.
- The `[GenerateShadowMethods]` source generator makes inherited `Set*`/`Bind*` return the **derived** type, so chaining stays typed (e.g. `new Label().SetText(...)` returns `Label`).
- `Label`, `Link`, `Entry` extend `UiTextElement` and share its text properties (`Text`, `TextSize`, `TextColor`, `FontFamily`, `FontWeight`, `FontStyle`, `HorizontalTextAlignment`, `TextWrapping`, `MaxLines`, `TextTruncation`). `RichTextLabel` extends `UiElement` and re-declares its own text defaults that apply to child runs.
- `TextWrapping` values: `NoWrap` (default), `Wrap` (character break), `WordWrap` (word boundaries). **Wrapping needs an explicit width** (`SetDesiredWidth`) or it has no boundary to wrap against.
- `HorizontalTextAlignment` (Left/Center/Right) positions text *inside* the element; `HorizontalAlignment` (Left/Center/Right/Stretch) positions the *element* in its parent. They are independent.

---

## Label

Non-editable text for static or dynamic content, with wrapping, truncation, and binding. `IsFocusable => false`, `AccessibilityRole.Label`.

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `Text` | `string?` | `null` | Content. Also feeds the accessibility label. |
| `TextSize` | `float` | `12` | Font size (points). |
| `TextColor` | `Color` | `White` | |
| `FontFamily` | `string` | theme | Font family name. |
| `FontWeight` | `FontWeight` | `Regular` | Thin/Light/Regular/SemiBold/Bold/Black… Defaults to Regular (400), not Bold. |
| `FontStyle` | `FontStyle` | `Normal` | Normal/Italic/Oblique. |
| `HorizontalTextAlignment` | `HorizontalTextAlignment` | `Left` | Text position within bounds. |
| `TextWrapping` | `TextWrapping` | `NoWrap` | Set to `Wrap`/`WordWrap` to enable wrapping. |
| `MaxLines` | `int?` | `null` | Caps visible lines; only honored when wrapping. |
| `TextTruncation` | `TextTruncation` | `None` | None/Start/Middle/End ellipsis. |
| `SupportsSystemFontScaling` | `bool` | global config | **No `Bind*`** — use `SetSupportsSystemFontScaling`. |
| `Margin` | `Margin` | zero | Outer spacing (inherited). |
| `HorizontalAlignment` / `VerticalAlignment` | enum | `Undefined` | Element placement in parent. |
| `DesiredWidth` / `DesiredHeight` | `float` | — | Explicit size. **Width is required for wrapping.** |
| `Background` | `IBackground` | `null` | Solid color or gradient. |
| `CornerRadius` | `float` | `0` | |
| `Opacity` | `float` | `1.0` | |
| `IsVisible` | `bool` | `true` | |
| `AccessibilityLabel` | `string?` | from `Text` | Screen-reader label. |
| `ShadowColor` / `ShadowOffset` / `ShadowBlur` | `Color` / `Point` / `float` | — | Text shadow. |

```csharp
// Static
new Label().SetText("The quick brown fox jumps over the lazy dog.");

// Styled
new Label()
    .SetText("Bold Accent Text")
    .SetFontWeight(FontWeight.Bold)
    .SetTextColor(PlusUiDefaults.AccentPrimary)
    .SetTextSize(18);

// Wrapping (needs both TextWrapping AND DesiredWidth)
new Label()
    .SetText("This long line wraps across multiple lines when WordWrap is on and width is limited.")
    .SetTextWrapping(TextWrapping.WordWrap)
    .SetDesiredWidth(360)
    .SetMaxLines(2)
    .SetTextTruncation(TextTruncation.End);

// One-way data binding (expression, not a string)
new Label().BindText(() => vm.UserName);
new Label().BindText(() => vm.Count, c => $"Items: {c}");   // generic overload with formatter
```

**Gotchas**
- `NoWrap` is the default — text never wraps unless you set `Wrap`/`WordWrap`.
- Wrapping silently does nothing without `SetDesiredWidth()`.
- Text is **clipped** to element bounds; oversized text does not overflow.
- `MaxLines` is ignored in `NoWrap` mode; truncation/ellipsis applies to the last visible line.
- In `NoWrap`, newlines and control characters are stripped/normalized to spaces; wrap modes honor explicit line breaks.
- Not focusable — inherited focus-ring properties exist but are inactive.

---

## RichTextLabel

Displays multiple styled segments (`TextRun`) inline within one control. Each run can override color, size, weight, style, and family; unset (`null`) run properties **inherit the label's defaults** (not theme defaults). `IsFocusable => false`, `AccessibilityRole.Label` (label is the concatenation of run text).

**RichTextLabel** properties (defaults supplied to runs that leave a value `null`):

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `TextSize` | `float` | `PlusUiDefaults.FontSize` | Default for runs with `FontSize == null`. |
| `TextColor` | `Color` | `PlusUiDefaults.TextPrimary` | Default for runs with `Color == null`. |
| `FontWeight` | `FontWeight` | `PlusUiDefaults.FontWeight` | |
| `FontStyle` | `FontStyle` | `PlusUiDefaults.FontStyle` | |
| `FontFamily` | `string?` | `null` | |
| `TextWrapping` | `TextWrapping` | theme | `NoWrap`, `Wrap` (character break), or `WordWrap` (word boundaries). |
| `HorizontalTextAlignment` | `HorizontalTextAlignment` | theme | |
| `MaxLines` | `int?` | `null` | Drops lines beyond the limit — **no ellipsis**. |
| `Margin` / `Background` / `DesiredWidth` / `DesiredHeight` / `IsVisible` / `Opacity` | inherited from `UiElement` | — | All have `Bind*`. |

Run management: `AddRun(TextRun)` appends; `SetRuns(IEnumerable<TextRun>)` **clears then adds**; `ClearRuns()`; `BindRuns(() => vm.Runs)` for reactive lists.

**TextRun** — construct with `new TextRun(text)`; style via fluent setters (each property is nullable to inherit):

| Method | Type | Bind |
|--------|------|------|
| `SetText` | `string` | `BindText(() => …)` (non-nullable `Func<string>`) |
| `SetColor` | `Color?` | `BindColor` |
| `SetFontSize` | `float?` | `BindFontSize` |
| `SetFontWeight` | `FontWeight?` | `BindFontWeight` |
| `SetFontStyle` | `FontStyle?` | `BindFontStyle` |
| `SetFontFamily` | `string?` | `BindFontFamily` |

```csharp
// Mixed colors / weights
new RichTextLabel()
    .AddRun(new TextRun("Hello "))
    .AddRun(new TextRun("World").SetColor(Colors.Blue).SetFontWeight(FontWeight.Bold));

// Price tag with per-run sizes (runs inherit TextSize=14 where not overridden)
new RichTextLabel()
    .SetTextSize(14)
    .SetTextWrapping(TextWrapping.WordWrap)
    .SetDesiredWidth(400)
    .AddRun(new TextRun("Price: "))
    .AddRun(new TextRun("$99").SetFontSize(28).SetFontWeight(FontWeight.Bold).SetColor(PlusUiDefaults.AccentSuccess))
    .AddRun(new TextRun(".99").SetFontSize(16).SetColor(PlusUiDefaults.AccentSuccess))
    .AddRun(new TextRun(" /month"));

// Reactive runs from the VM
new RichTextLabel().BindRuns(() => vm.Highlights);
```

**Gotchas**
- Run properties are nullable: `null` inherits from the **parent label**, not theme defaults.
- Newlines/control characters in run text are stripped automatically — pass clean text.
- `MaxLines` truncates by dropping whole lines; there is no ellipsis or visual cue.
- Runs flow inline left-to-right; there is no vertical stacking inside one label — use a `VStack` of labels for that.
- `SetRuns()` clears existing runs first; use `AddRun()` to append.
- `NoWrap` keeps everything on one line and can overflow the available width.

---

## Link

Clickable hyperlink that opens `Url` via the platform service, drawing an automatic underline. Extends `UiTextElement`; `IsFocusable => true`, `AccessibilityRole.Link`. Inherits all `UiTextElement` text properties (see Label).

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `Url` | `string?` | `null` | Opened on click. Must be non-empty or click is a no-op. |
| `UnderlineThickness` | `float` | `1f` | Underline stroke width. Underline cannot be disabled. |
| `Text` | `string?` | `null` | Required — a Link with no text is invisible. |
| `TextColor` | `Color` | `White` | **No default link color** — set one explicitly. |
| `FontWeight` | `FontWeight` | `Regular` | |
| `HorizontalTextAlignment` | enum | `Left` | Also shifts the underline. |
| `TextWrapping` / `MaxLines` / `TextTruncation` | — | — | Same semantics as Label. |
| `TabIndex` | `int?` | `null` | Tab order. |
| `TabStop` | `bool` | `true` | In tab navigation by default. |
| `Margin` / `HorizontalAlignment` / `VerticalAlignment` / `IsVisible` / `Background` | inherited | — | All have `Bind*`. |

```csharp
// Minimum: text + url (style it to look like a link!)
new Link()
    .SetText("Visit Website")
    .SetUrl("https://example.com")
    .SetTextColor(Colors.CornflowerBlue);

// Styled
new Link()
    .SetText("Documentation")
    .SetUrl("https://docs.example.com")
    .SetTextColor(PlusUiDefaults.AccentPrimary)
    .SetFontWeight(FontWeight.SemiBold)
    .SetUnderlineThickness(2);

// Bound url/visibility
new Link().SetText("Profile").BindUrl(() => vm.ProfileUrl).BindIsVisible(() => vm.IsLoggedIn);
```

**Gotchas**
- **No default link-blue** — without `SetTextColor` the link renders white. Use `Colors.Blue`, `Colors.CornflowerBlue`, or `PlusUiDefaults.AccentPrimary`.
- **Empty/null `Url` makes click do nothing** (`InvokeCommand` checks `IsNullOrEmpty`).
- Underline is always drawn; only thickness is adjustable — there is no underline-less Link.
- Under wrapping, the underline is drawn per line, not as one continuous rule.
- Always focusable and a tab stop — you cannot disable focus. Use a `Label` with a click handler if you need a non-focusable clickable text.
- `Url` must be a platform-valid scheme (e.g. `https://`); opening goes through `IPlatformService.OpenUrl`.

---

## Entry

Text input with cursor navigation, selection, clipboard (Ctrl+A/C/X/V), password masking, and optional multi-line mode. Extends `UiTextElement`; `IsFocusable => true`, `AccessibilityRole.TextInput`. Defaults: width 200, height 40, `BackgroundInput` background, `CornerRadius`, padding `Margin(12, 8)`.

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `Text` | `string?` | `null` | Usually two-way bound. |
| `Placeholder` | `string?` | `null` | Shown when empty. |
| `PlaceholderColor` | `Color` | `TextPlaceholder` | |
| `IsPassword` | `bool` | `false` | Masks displayed text. |
| `PasswordChar` | `char` | `'•'` | Mask glyph. |
| `MaxLength` | `int?` | `null` | Blocks typing past limit; truncates paste. |
| `Keyboard` | `KeyboardType` | `Default` | Mobile: Default/Text/Numeric/Email/Telephone/Url. |
| `ReturnKey` | `ReturnKeyType` | `Default` | Mobile return-key label (Go/Send/Search/Next/Done). |
| `IsMultiLine` | `bool` | `false` | Enter inserts newline. Auto-sets Min/MaxLines=5 if unset. |
| `MinLines` | `int` | `1` (5 if multi-line) | Clamped to ≥1. |
| `MaxLines` | `int?` | `null` (5 if multi-line) | Scrolls beyond this. |
| `Padding` | `Margin` | `Margin(12, 8)` | Internal padding (a `Margin`, not a float). |
| `FocusBorderColor` | `Color` | `AccentPrimary` | Border when focused. |
| `FocusBorderThickness` | `float` | `2` | |
| `SelectionColor` | `Color` | `Color(100,149,237,100)` | Selection highlight (semi-transparent). |
| `TextWrapping` | `TextWrapping` | `NoWrap` | Shadows the base property. |
| `TextSize` / `TextColor` | `float` / `Color` | `12` / `White` | Inherited. |
| `Background` / `CornerRadius` / `Margin` | inherited | input theme | |

`SetOnTextChanged(Action<string?>)` fires on every type/paste/delete (no `Bind` counterpart — it is a callback, not a property).

```csharp
// Two-way bound (most common): getter expression + setter action
new Entry()
    .SetPlaceholder("Enter username...")
    .SetDesiredWidth(300)
    .BindText(() => vm.Username, v => vm.Username = v);

// Password
new Entry()
    .SetIsPassword(true)
    .SetPasswordChar('*')
    .SetMaxLength(20)
    .SetPlaceholder("Password")
    .BindText(() => vm.Password, v => vm.Password = v);

// Multi-line: set Min/MaxLines BEFORE SetIsMultiLine to keep custom values
new Entry()
    .SetMinLines(3)
    .SetMaxLines(10)
    .SetIsMultiLine(true)
    .SetDesiredWidth(400)
    .SetPlaceholder("Type notes here...")
    .BindText(() => vm.Notes, v => vm.Notes = v);

// Mobile keyboard hints
new Entry()
    .SetKeyboard(KeyboardType.Email)
    .SetReturnKey(ReturnKeyType.Next)
    .SetPlaceholder("email@example.com")
    .BindText(() => vm.Email, v => vm.Email = v);

// Live search via callback
new Entry()
    .SetPlaceholder("Search...")
    .SetPadding(new Margin(12, 8))
    .SetOnTextChanged(text => vm.SearchQuery = text);
```

**Two `BindText` overloads:**
- `BindText(Expression<Func<string?>>, Action<string>)` — string two-way.
- `BindText<T>(Expression<Func<T>>, Action<T>, Func<T,string?>? toControl = null, Func<string,T>? toSource = null)` — typed with optional converters (e.g. `int` <-> string).

**Gotchas**
- **Order matters for multi-line:** `SetIsMultiLine(true)` auto-sets MinLines=5/MaxLines=5 *if not already set* — call `SetMinLines`/`SetMaxLines` first for custom values.
- Enabling multi-line resets `DesiredHeight` (height becomes `-1`) so it can size from Min/MaxLines — don't rely on `DesiredHeight` there.
- `SetPadding` takes a `Margin` (`Margin(left,top,right,bottom)` or `Margin(horizontal,vertical)`), never a single float.
- `MaxLength` silently truncates pasted text to fit; no warning.
- Clipboard (Ctrl+C/X/V) works on desktop (Windows/macOS/Linux) via the built-in `IClipboardService`; Android/iOS/Web have no implementation yet, so copy/cut/paste are silent no-ops there.
- Password masking uses `'•'` by default; set `PasswordChar` to override.
- Cursor blink is hardcoded at 500 ms on / 500 ms off.
- Selection is cleared on blur; single-line scrolls horizontally and multi-line shows an auto vertical scrollbar (not optional).
- `ReturnKey`/`Keyboard` only affect mobile platforms; desktop uses system defaults.
- Make `SelectionColor` opaque enough to read against custom backgrounds.

---

## CodeEditor

Multi-line editor for code: caret, selection, clipboard, block indent/outdent, auto-indent, line numbers, and syntax highlighting driven by a callback. Extends `UiTextElement`; `IsFocusable => true`, `AccessibilityRole.TextInput`. Defaults: width 480, height 320, `BackgroundInput` background, `CornerRadius`, padding `Margin(8, 6)`, `TextWrapping.NoWrap` (code does not wrap).

**The document is always a plain `string`.** Styling is never stored on the control — a highlighter recomputes it from the text on every change. That is the whole point of the design: because no styled offsets are persisted, inserting or deleting text in the middle can never desynchronize the styling, and every cursor/selection index stays a plain character index.

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `Text` | `string?` | `null` | Usually two-way bound. Tabs are normalized to spaces. |
| `TabSize` | `int` | `4` | Spaces per indent step. Clamped to ≥1. |
| `AutoIndent` | `bool` | `true` | Enter copies the current line's leading whitespace. |
| `ShowLineNumbers` | `bool` | `true` | Renders a gutter sized to the line count. |
| `LineNumberColor` | `Color` | `TextPlaceholder` | Gutter digits. |
| `CurrentLineColor` | `Color` | `Color(255,255,255,12)` | Active-line highlight; only drawn while focused. |
| `SelectionColor` | `Color` | `Color(100,149,237,100)` | Selection highlight (semi-transparent). |
| `CaretColor` | `Color` | `TextPrimary` | Caret bar. |
| `IsReadOnly` | `bool` | `false` | Blocks edits; selection/copy/navigation still work. |
| `Padding` | `Margin` | `Margin(8, 6)` | Internal padding (a `Margin`, not a float). |
| `FontFamily` / `TextSize` | `string?` / `float` | inherited | **Set a monospace family** (e.g. `"Consolas"`) — there is no monospace default. |

`SetOnTextChanged(Action<string?>)` fires on every edit (no `Bind` counterpart — it is a callback, not a property).

### Highlighting

Two mutually exclusive modes. **Setting one clears the other** — they never both feed the render path.

| Method | Callback | Span offsets | Use for |
|--------|----------|--------------|---------|
| `SetHighlighter` | `Func<string, List<StyleSpan>>` | Absolute in document | Simple, non-code styling (markers, search hits) |
| `SetLineHighlighter` | `LineHighlighter` delegate | Relative to the line | Code — caches per line, handles multi-line constructs |

```csharp
public readonly record struct StyleSpan(
    int Start, int Length,
    Color? Color = null, FontWeight? FontWeight = null, FontStyle? FontStyle = null);

// Returns the state the next line starts in. Fill `output`; do not keep a reference to it.
public delegate int LineHighlighter(string lineText, int lineIndex, int stateIn, List<StyleSpan> output);
```

`SetLineHighlighter` is the one to use for code. Each line is tokenized with the state the **previous** line ended in, which is what makes block comments and verbatim strings spanning lines work — a line cannot otherwise know it sits inside a comment opened above it. Results are cached per line, so editing one line of a large file re-tokenizes that line and then stops as soon as the carried state lines up again.

```csharp
// Line-based, stateful — the right default for code
const int Normal = 0, InBlockComment = 1;

new CodeEditor()
    .SetFontFamily("Consolas")
    .SetTabSize(4)
    .SetLineHighlighter((line, index, state, output) =>
    {
        if (state == InBlockComment)
        {
            var close = line.IndexOf("*/", StringComparison.Ordinal);
            if (close < 0)
            {
                output.Add(new StyleSpan(0, line.Length, CommentColor));
                return InBlockComment;          // still open — carry downward
            }
            output.Add(new StyleSpan(0, close + 2, CommentColor));
            return Normal;
        }

        foreach (var token in Tokenize(line))
            output.Add(new StyleSpan(token.Start, token.Length, ColorFor(token.Kind)));
        return Normal;
    })
    .BindText(() => vm.Code, v => vm.Code = v);

// Whole-document — fine for simple, stateless marking
new CodeEditor()
    .SetHighlighter(text =>
    {
        var spans = new List<StyleSpan>();
        var i = text.IndexOf("TODO", StringComparison.Ordinal);
        while (i >= 0)
        {
            spans.Add(new StyleSpan(i, 4, Colors.Red, FontWeight.Bold));
            i = text.IndexOf("TODO", i + 4, StringComparison.Ordinal);
        }
        return spans;
    })
    .BindText(() => vm.Notes, v => vm.Notes = v);

// Read-only viewer
new CodeEditor()
    .SetFontFamily("Consolas")
    .SetIsReadOnly(true)
    .SetShowLineNumbers(false)
    .SetLineHighlighter(MyHighlighter)
    .SetText(snippet);
```

### Keyboard

| Key | Behavior |
|-----|----------|
| `Tab` / `Shift+Tab` | Indent / outdent. Indents **every line** touched by a multi-line selection. |
| `Enter` | Newline, keeping the current indentation when `AutoIndent`. |
| `Backspace` | Removes a whole indent step inside leading indentation, otherwise one character. |
| `Home` | Toggles between the first non-whitespace character and column 0. `Ctrl+Home` to document start. |
| `Ctrl+A/C/X/V` | Select all, copy, cut, paste. |

**Gotchas**
- **Set a monospace `FontFamily` yourself** — the editor inherits the normal text font, so columns will not line up otherwise.
- **Tabs are normalized to `TabSize` spaces** on every input path (typing, paste, `SetText`). This keeps character index == rendered column, which is what makes click-to-caret and selection exact. There is deliberately no "use real tab characters" option.
- **`SetHighlighter` and `SetLineHighlighter` are mutually exclusive** — calling one nulls the other. Do not expect them to compose.
- **Line-highlighter spans are relative to the line**, document-highlighter spans are absolute. Mixing the two conventions up is the most common bug here.
- For the per-line cache to converge, the same `(lineText, stateIn)` must always yield the same result — keep the highlighter pure and do not close over mutable outside state.
- A highlighter that throws is caught: the affected line/document renders unstyled rather than taking the app down. It will not be reported — a silently uncolored file usually means an exception, not a tokenizer gap.
- **`Tab` is claimed only while focused and editable.** A read-only editor lets `Tab` move focus on, which is what you want in a form.
- **No undo/redo.** Deliberately left to the app — keep your own history on the bound VM property.
- No word wrap: `TextWrapping` is forced to `NoWrap`; long lines scroll horizontally.
- Overly long or out-of-range spans are clipped to the line, so a sloppy tokenizer cannot crash rendering.

---

## Common LLM mistakes

- Using `BindText(nameof(vm.X), () => vm.X)` — **not a real overload.** Bind takes only the expression: `BindText(() => vm.X)` (read-only) or `BindText(() => vm.X, v => vm.X = v)` (Entry two-way).
- Expecting text to wrap from `SetTextWrapping` alone — you **must** also `SetDesiredWidth`.
- Forgetting `SetTextColor` on a `Link` and getting invisible/white "links."
- Setting `MaxLines` on a `Label`/`RichTextLabel` in `NoWrap` mode (ignored).
- Passing a float to `Entry.SetPadding` — it requires a `Margin`.
- Calling `SetIsMultiLine(true)` before `SetMinLines`/`SetMaxLines` and being surprised by the 5/5 defaults.
- Assuming `RichTextLabel` run `null` properties fall back to theme defaults — they inherit from the **parent label**.
- Using `SetRuns()` expecting it to append — it clears first; use `AddRun()`.
- Confusing `HorizontalTextAlignment` (text within bounds) with `HorizontalAlignment` (element in parent).
- Reaching for `Entry` with `SetIsMultiLine(true)` to build a code editor — use `CodeEditor`, which adds highlighting, indent handling, and line numbers.
- Building a `CodeEditor` highlighter that returns **absolute** document offsets from `SetLineHighlighter` — line highlighters return **line-relative** offsets.
- Expecting `CodeEditor` to store styling so the user can select text and apply bold — there is no such model; styling is always derived from the text by the highlighter.
- Forgetting a monospace `SetFontFamily` on `CodeEditor` and getting misaligned columns.
