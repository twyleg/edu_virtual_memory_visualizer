# Java Memory Lab

Java Memory Lab is a small, self-contained teaching tool for visualizing a deliberately simplified memory model for introductory Java programming.

The project is intended for classroom use. Its main purpose is to help students answer questions such as:

- Where does a local variable live?
- What is a stack frame?
- What happens to the stack when methods call other methods?
- What happens when primitive values are passed as parameters?
- What does a return value do?
- Where do objects live?
- What is actually stored in a reference variable?
- Why can modifying an object through a parameter affect the original object?
- How do arrays of objects relate to references and heap objects?

The visualizer is **not a JVM simulator** and must not be turned into one accidentally. It uses an intentionally simplified, deterministic 32-bit teaching model so that the memory state remains easy to reason about on a projector or interactive board.

---

## Project status

The current application is a single standalone HTML file:

```text
index.html
README.md
```

`index.html` contains all HTML, CSS, JavaScript, the built-in examples, and an embedded specification for external JSON configurations.

There is currently:

- no build step,
- no package manager,
- no backend,
- no framework,
- no external runtime dependency,
- no Java execution,
- no server-side code.

The file can be opened directly in a browser and can also be hosted as a GitHub Page.

For GitHub Pages, publishing the repository root is sufficient.

---

# Core design principle

The visualizer separates two things that are often mixed together in educational tools:

1. **Java source code**
2. **Explicit memory snapshots**

The application does **not** execute or interpret the Java source.

Instead, each visualization state explicitly describes what memory should look like at a particular source-code line. This makes the visualizer deterministic and allows examples to be authored specifically for teaching.

Conceptually:

```text
Java source
    |
    | source line number
    v
Memory state JSON
    |
    v
Visualizer
```

A state is therefore best understood as a slide in an execution animation rather than the result of a Java interpreter.

This is intentional.

---

# Simplified memory model

The most important assumption in the entire project is the virtual machine model used for teaching.

## 32-bit word-addressed memory

Memory contains **32 memory cells**.

Each visible memory cell represents one complete **32-bit word**.

Memory is therefore **word-addressed**, not byte-addressed.

Addresses are simple decimal values:

```text
0
1
2
3
...
31
```

The tool deliberately hides concepts such as individual bytes and endianness. The goal is to teach where variables live and how references work before introducing lower-level memory representation.

## Memory regions

The default layout is:

| Region | Addresses | Direction |
|---|---:|---|
| Static / Class Data | `0..7` | grows upward |
| Heap | `8..23` | grows upward |
| Stack | `24..31` | grows downward |

Examples 12 and 13 enable shared heap/stack space. Their boundary starts at address `24` and moves as allocations grow or frames return; unused heap cells can become stack cells. Static addresses `0..7` remain reserved. Other examples and custom configurations without `memory.shared` keep fixed ranges.

Undefined cells always contain:

```text
0x00000000
```

## One cell per value

For this model, all of the following occupy exactly one 32-bit memory cell:

- primitive local variables,
- method parameters,
- object references,
- array elements,
- object fields,
- static/class fields.

This is a pedagogical simplification.

For example, Java's real `boolean`, `double`, object layouts, JVM headers, padding, register allocation, escape analysis, JIT optimizations, and implementation-specific details are deliberately not modeled.

Do not “fix” this into a realistic HotSpot memory layout unless the teaching model itself is intentionally being redesigned.

---

# Stack model

The stack initially occupies addresses `24..31` and grows downward. With shared heap/stack space enabled, it can continue into free addresses below `24`.

Example:

```text
31  main.a
30  main.b
29  add.result
28  add.x
27  add.y
```

Each method invocation may define a logical **stack frame**.

A stack allocation can reference its frame using the `frame` property. All cells belonging to one frame should be contiguous.

The UI draws a frame outline and frame label around those cells.

When a method returns, its frame simply disappears from the next snapshot.

## Important teaching convention

Stack frames are logical teaching structures. They should make method lifetime and ownership of local variables easy to see.

They are not intended to reproduce the physical native stack layout of a real JVM.

---

# Heap model

Objects and arrays initially live in the heap region, addresses `8..23`. With shared heap/stack space enabled, allocations can use free space between the two regions.

The heap grows upward.

Object fields are placed in consecutive cells in declaration order.

Example:

```java
class Person {
    int age;
    boolean student;
    float height;
}
```

may be represented as:

```text
8   Person.age
9   Person.student
10  Person.height
```

The simplified model does not add:

- object headers,
- class pointers,
- alignment,
- padding,
- garbage-collector metadata.

---

# References

A reference is represented by the **word address of the referenced heap object**.

Example:

```text
Stack
31  person = 8
             |
             v
Heap
8   Person.age
9   Person.student
10  Person.height
```

In JSON, the numeric value and visual relationship are represented independently:

```json
{
  "id": "main.person",
  "label": "person",
  "region": "stack",
  "frame": "main",
  "address": 31,
  "value": 8,
  "pointsTo": "person.age"
}
```

`pointsTo` references the `id` of the first cell of the logical heap object.

This lets the UI highlight both the reference and the referenced object/cell.

---

# Static / Class Data

The region at addresses `0..7` is called **Static / Class Data**.

It exists to provide an intuitive place for Java `static` fields.

It should not be described as Java's literal equivalent of a C `.data` or `.bss` section. The actual JVM runtime representation is implementation-dependent.

For teaching purposes, a static field can simply be modeled as:

```json
{
  "id": "Counter.total",
  "label": "Counter.total",
  "region": "static",
  "address": 0,
  "value": 4
}
```

---

# JSON memory-state format

The canonical JSON format is also embedded as an HTML comment near the beginning of `index.html`.

That embedded specification should remain synchronized with this README whenever the format changes.

A typical configuration looks like this:

```json
{
  "title": "Example",
  "subtitle": "Optional subtitle",
  "memory": {
    "static": { "start": 0, "size": 8 },
    "heap":   { "start": 8, "size": 16 },
    "stack":  { "start": 24, "size": 8 }
  },
  "states": [
    {
      "line": 3,
      "note": "Optional explanatory text.",
      "highlight": "main.number",
      "frames": [
        { "id": "main", "label": "main()" }
      ],
      "memory": [
        {
          "id": "main.number",
          "label": "number = 10",
          "region": "stack",
          "frame": "main",
          "address": 31,
          "value": 10
        }
      ]
    }
  ]
}
```

## Top-level fields

### `title`

Optional display title.

### `subtitle`

Optional subtitle.

### `modelNote`

Still supported internally, but currently hidden in the UI.

Keep this field for compatibility because an explanatory banner may be restored later.

### `memory`

Defines the three memory ranges:

```json
"memory": {
  "static": { "start": 0, "size": 8 },
  "heap":   { "start": 8, "size": 16 },
  "stack":  { "start": 24, "size": 8 }
}
```

### `memory.shared`

Optional boolean, default `false`. When `true`, the adjacent heap and stack ranges form one shared pool. Their configured ranges define the initial display boundary. Heap allocations grow upward and stack allocations grow downward; their frontiers must not cross. The UI moves the boundary for each snapshot and places stack frames in the stack section, including borrowed cells. The display still contains 32 cells in the built-in examples.

### `states`

Ordered list of complete memory snapshots.

States are **not deltas**.

Every state must contain the complete set of memory cells that should be defined at that moment. Everything else is automatically zero.

---

# State fields

## `line`

Required.

The 1-based source-code line highlighted for this state.

```json
"line": 7
```

## `collision`

Optional metadata for a blocked allocation at the heap/stack boundary:

```json
"collision": { "region": "stack", "address": 23, "label": "dive(8): depth = 8" }
```

`region` is the requesting region (`heap` or `stack`). `address` must be the immediately adjacent word beyond the requesting region’s occupied allocation frontier, occupied by the other region. This frontier can move when shared space is enabled. `label` describes the attempted allocation. The attempted cell is not added to `memory`; existing cells remain intact. The UI displays a warning and highlights both sides of the boundary in combined and separated views.

Custom configurations retain their own region sizes; the 32-cell layout is the built-in default.

## `memory`

Required.

Complete list of currently defined cells.

## `note`

Optional.

Explanatory text associated with the state.

The note infrastructure still exists in the application, but the general Note panel has deliberately been removed to save screen space. Shared heap/stack examples display their current note in a compact banner above the memory grid.

**Do not delete note support.** It may be reintroduced later.

## `highlight`

Optional.

The `id` of a memory cell that should be selected/highlighted by default.

## `frames`

Optional stack-frame metadata.

Example:

```json
"frames": [
  { "id": "main", "label": "main()" },
  { "id": "add", "label": "add()" }
]
```

Each stack memory entry can then use:

```json
"frame": "add"
```

---

# Memory-cell fields

Each defined memory cell has this structure:

```json
{
  "id": "main.number",
  "label": "number = 10",
  "region": "stack",
  "frame": "main",
  "address": 31,
  "value": 10,
  "pointsTo": "optional.target.id"
}
```

## `id`

Required unique identifier within the state.

Use stable and descriptive identifiers such as:

```text
main.number
add.x
person.age
people.0
Counter.total
```

## `label`

Optional human-readable text displayed next to the memory cell.

## `region`

Required. One of:

```text
static
heap
stack
```

## `address`

Required word address.

## `value`

Required 32-bit numeric value.

Accepted conceptual range:

```text
-2147483648 .. 4294967295
```

Negative values are rendered as two's-complement values in hexadecimal and binary display modes.

## `frame`

Optional and valid only for stack cells.

Must reference a frame defined in that state's `frames` array.

## `pointsTo`

Optional.

References another memory-cell `id` and represents a logical reference relationship.

---

# Return-value convention

The project currently uses a deliberately explicit teaching convention for return values.

In the parameters/return example:

- the caller reserves the destination cell for `result`,
- the callee has a fixed stack frame containing its parameters and local variables,
- a separate transfer cell immediately above the callee frame temporarily contains the return value,
- after the callee returns, that value is copied into the caller's reserved result cell.

This is a **visual teaching convention**, not a claim about JVM internals.

Maintain this convention unless the corresponding example is intentionally redesigned.

---

# User interface

## Main layout

The application has two primary panes:

```text
Java source | Virtual memory
```

On sufficiently wide screens, the layout prefers approximately a **50/50 split**.

The memory pane has a minimum width so the memory representation stays readable. When horizontal space becomes constrained, the source pane shrinks first and gains horizontal scrolling.

On small screens the layout becomes stacked.

## Source pane

Features:

- Java syntax highlighting implemented locally in vanilla JavaScript,
- current source line highlighting,
- automatic vertical scrolling to the active line,
- horizontal scrolling for long lines.

Do not introduce an online syntax-highlighting dependency merely for convenience. Offline/self-contained operation is currently a feature.

## Memory pane

Features:

- independent vertical scrolling,
- decimal addresses,
- selectable value format:
  - hexadecimal,
  - decimal,
  - binary,
- cell labels,
- selected-cell highlighting,
- referenced-target highlighting,
- stack-frame outlines,
- usage counters per region.

The source remains visible while the memory pane scrolls.

---

# Combined vs separated memory view

A central teaching feature is the **Separate memory regions** button.

The visualizer initially presents memory as one continuous block.

This avoids introducing stack/heap/static segmentation before students are ready for it.

When the button is pressed, the memory animates into three visibly separate regions:

```text
Static / Class Data
Heap
Stack
```

The button then changes to **Combine memory** and reverses the animation.

Important visual detail:

When memory is combined, the spacing at region boundaries has been tuned to **1 px** so the boundary visually matches ordinary row spacing without looking either disconnected or unnaturally compressed.

Preserve this unless deliberately redesigning the memory grid.

---

# Sidebar

The application uses a collapsible sidebar on the left.

## Collapsed state

A narrow permanent rail remains visible.

It contains:

- burger-menu button,
- numbered quick-access buttons for all built-in examples.

The compact example buttons:

- use muted colors so they do not compete visually with the teaching content,
- highlight the active example subtly,
- expose the example name via tooltip.

## Expanded state

The expanded sidebar contains:

### Top

- **Load Java…**
- **Load JSON…**

These allow custom source/config pairs to be loaded using browser file inputs. This is intentionally compatible with opening the page directly via `file://` and therefore does not depend on `fetch()`.

### Middle

A full labeled list of built-in examples.

### Bottom

Display/settings controls:

- light/dark theme toggle,
- word display format selector.

The sidebar expands from the permanent rail rather than floating over the main page on desktop.

---

# Themes

The application supports light and dark modes.

The initial theme respects the browser/system preference where possible.

The chosen theme is remembered using browser storage when available.

When changing colors, verify both modes.

Important semantic color roles include:

- static/class data,
- heap,
- stack,
- selected cell,
- referenced target,
- active source line,
- stack-frame outline.

Do not optimize only for dark mode.

---

# Built-in examples

At the time of this README, `index.html` contains thirteen built-in examples in the `BUILTIN_EXAMPLES` array.

They are deliberately ordered so each example introduces only one or two new concepts.

Every built-in example starts at its `public static void main(String[] args)` declaration with an empty memory snapshot. All cells are zero and no modeled variables are allocated yet. Stepping forward then introduces the program’s allocations.

## 1 — Hello World

Purpose:

- show that a program can execute while the simplified memory remains empty,
- introduce the visualizer before introducing variables.

No user-defined variables are modeled.

## 2 — Local variables

Introduces:

- primitive local variables,
- the first `main()` stack frame,
- one memory cell per value.

Uses:

- `int`,
- `boolean`,
- `float`.

The float's cell stores its 32-bit IEEE-754 bit pattern. The label still presents the source-level value so floating-point representation does not become the lesson.

## 3 — Stack frames

Introduces:

- calling a method without parameters,
- a second stack frame,
- local variables belonging to different frames,
- frame removal when a method returns.

## 4 — Call stack

Each of `main()`, `first()`, `second()`, and `third()` declares two local `int` variables. At maximum depth, four two-cell frames fill the eight stack cells at addresses `24..31`. Returning removes both locals of a frame together.

Introduces:

- a chain of several nested method calls,
- multiple simultaneous stack frames,
- stack frames being pushed and popped.

## 5 — Different callers

Every method declares two local `int` variables. Calling `show()` from either `first()` or `second()` produces three two-cell frames, using six stack cells. Each return removes both locals together, and the second caller reuses the first caller’s freed stack space.

Introduces:

- the same method being called from different callers,
- the fact that a method's own frame does not depend on arbitrary access to its caller's locals,
- call order changes the active call stack.

Terminology note:

The method that invoked the current method is its **caller**. A method it invokes is its **callee**.

## 6 — Parameters & return

Introduces:

- primitive parameters copied into the callee frame,
- separate parameter cells,
- local calculation,
- the project's explicit return-transfer convention,
- copying a return value into a caller-reserved result cell.

## 7 — Local parameter assignments

Immediately follows the parameters/return example. `main()` stores `a = 5` and `b = 3`, then calls `change(a, b)`. The callee receives copies in separate parameter cells `x` and `y`, then computes local `result = x + y` (8) in its own frame. Assigning `x = 99` and `y = 77` changes only those cells; the caller’s variables remain `5` and `3` throughout.

After the callee returns, its parameter cells and local `result` cell disappear. The caller prints its unchanged values. This example isolates parameter assignment before introducing heap objects; Example 10 later revisits primitive copying alongside the object-reference example.

## 8 — First heap object

Introduces:

- a complex datatype (`Person`),
- object fields on the heap,
- a local reference on the stack,
- stack-to-heap reference visualization.

`Person` currently has:

```java
int age;
boolean student;
float height;
```

## 9 — Array of Persons

Uses four `Person` instances with the same `Person` class as Example 8.

Introduces:

```text
stack reference -> array object -> Person objects
```

Important distinction:

The local variable `people` is a reference on the stack. The `Person[]` object is on the heap, and its array elements contain references to individual `Person` objects.

## 10 — Primitive parameter copy

Introduces Java's pass-by-value behavior for primitive values.

A primitive argument is copied into the callee's parameter cell. Modifying the parameter does not modify the original caller variable.

## 11 — Object reference parameter

Pairs directly with Example 10.

Java still passes the argument by value, but the copied value is an object reference.

Therefore:

- caller variable and callee parameter occupy separate stack cells,
- both cells contain the same heap address,
- changing an object field through the callee reference changes the shared heap object.

This example should not be described as “Java passes objects by reference”. Java passes the reference **by value**.

---

## 12 — Stack growth into free heap space

An `int[4]` occupies heap addresses `8..11`, leaving ample free space. Its reference is at stack address `31`. Ten recursive calls to `dive(int depth)` grow the stack down to address `21`, safely passing the initial boundary at `24`. Addresses `12..20` remain free between heap and stack.

The visible boundary moves so borrowed addresses `21..23` have stack coloring and frame labels in both combined and separated views. Recursion stops at depth 10, and subsequent snapshots show the frames returning and the boundary moving back. A visible explanation accompanies each step.

## 13 — Heap / stack collision

Uses the same bounded recursive method, with an `int[16]` instead. The array fills heap addresses `8..23`. Its reference occupies stack address `31`. Seven recursive calls fill stack addresses `30..24`. The next call would need address `23`, already occupied by the array's last element.

The final snapshot keeps all existing values intact and displays a collision warning, highlighting heap address `23` and stack address `24`. The attempted allocation is blocked. Step backward to remove the warning and revisit the growth of the stack.

Both examples illustrate opposing growth within shared space in the 32-cell teaching model. Real JVMs manage heap and stack separately. The eight static cells remain reserved even when unused.

---

# Built-in example implementation

Built-in examples are currently stored directly inside `index.html` in the JavaScript constant:

```javascript
BUILTIN_EXAMPLES
```

Each example contains:

```javascript
{
    id: "...",
    label: "...",
    source: "...Java source...",
    config: { ...memory-state configuration... }
}
```

The sidebar's full list and the compact numbered rail are generated from this same collection, so their numbering remains synchronized automatically.

When adding a built-in example:

1. Add it to `BUILTIN_EXAMPLES` at the intended position.
2. Give it a stable `id`.
3. Give it a numbered `label` consistent with its position.
4. Include source code and complete state configuration.
5. Verify source line numbers against the exact embedded source string.
6. Verify all addresses satisfy region constraints.
7. Verify stack-frame cells are contiguous.
8. Step through every state manually.
9. Test light and dark mode.
10. Test combined and separated memory views.

---

# Validation

The application validates loaded JSON before rendering it.

Among other things, it checks:

- required memory/state structures,
- allowed regions,
- valid addresses,
- 32-bit numeric values,
- duplicate addresses inside one state,
- frame references,
- stack-only frame assignment,
- contiguous cells for each stack frame.

Validation failures are displayed in the UI.

When changing the schema, update all of the following together:

1. validation code,
2. renderer,
3. built-in examples,
4. embedded HTML specification comment,
5. this README.

---

# Source-code highlighting

Syntax highlighting is intentionally implemented without third-party dependencies.

The lightweight tokenizer highlights common Java constructs including:

- keywords,
- primitive/common types,
- class names,
- method calls,
- strings,
- chars,
- numbers,
- booleans and `null`,
- annotations,
- comments.

It is not intended to be a full Java parser.

Do not rely on syntax highlighting for program semantics.

---

# Hidden Note infrastructure

Earlier versions displayed a dedicated Note pane for the current state.

The visible pane has been removed to increase screen space, but the infrastructure remains intentionally present:

- state `note` fields,
- hidden note DOM elements,
- render logic that updates them.

This is deliberate.

Do not remove the feature merely because it is currently hidden. It may return in a later UI iteration.

---

# Custom examples

The UI supports loading:

1. a Java source file,
2. a JSON memory-state file.

These files are independent.

The Java file provides the source pane. The JSON file provides the memory states and source-line mapping.

Because the app uses browser file inputs rather than network requests, custom examples work when `index.html` is opened locally.

A useful future repository structure might be:

```text
index.html
README.md
examples/
    01-hello-world/
        Main.java
        memory.json
    02-local-variables/
        Main.java
        memory.json
    ...
```

The current built-in examples remain embedded in `index.html`; moving them into external files would require deciding how to retain direct `file://` compatibility. GitHub Pages could load them with `fetch()`, but a locally opened HTML file may not be allowed to do so by the browser.

If refactoring examples out of `index.html`, preserve a good local-development workflow or explicitly introduce a tiny local web server requirement.

---

# Guidance for Codex / future contributors

## Preserve the educational abstraction

The most important rule is:

> Optimize for conceptual clarity, not JVM implementation fidelity.

Do not automatically replace the simple model with JVM slots, native stack addresses, object headers, compressed oops, registers, garbage collector details, or actual HotSpot object layout.

Those may become advanced modes later, but they should not silently alter the introductory model.

## Keep the app self-contained when practical

The current single-file implementation is useful because it:

- works offline,
- works from `file://`,
- can be copied easily,
- can be hosted directly on GitHub Pages,
- has no dependency/toolchain maintenance.

A larger architecture is acceptable if it provides clear value, but do not introduce React/Vite/npm/etc. only for cosmetic refactoring.

## Prefer declarative examples

Memory states should remain explicit snapshots.

Do not infer memory from Java source unless a separate feature is intentionally introduced.

Explicit states are what make the teaching sequence author-controlled.

## Do not make state snapshots incremental

The current model is intentionally:

```text
state = complete memory snapshot
```

not:

```text
state = changes since previous state
```

Complete snapshots are verbose but much easier to author, inspect, debug, and reason about.

## Preserve external JSON compatibility

If internal representation changes, either:

- keep the documented JSON format working, or
- deliberately version/migrate the format.

Do not silently break custom classroom examples.

## Treat example order as curriculum

The built-in examples are not merely demo data. They are a teaching progression.

New examples should normally introduce one new concept at a time and reuse familiar structures from previous examples.

## Be careful with terminology

Use:

- caller = method that invoked the current method,
- callee = method invoked by the current method,
- Java passes arguments by value,
- object variables contain references,
- copying a reference still points to the same object.

Avoid statements such as:

> Java passes objects by reference.

## Keep UI presentation calm

The application is intended to be projected in class.

Existing design decisions intentionally keep secondary controls subdued:

- numbered example rail uses muted colors,
- settings live in the sidebar,
- the main screen prioritizes source and memory,
- explanatory Note UI is hidden,
- memory-region separation is an explicit teaching action rather than always visible.

New UI elements should not compete visually with source code and memory.

---

# Suggested manual regression checklist

After meaningful changes to `index.html`, test at least the following.

## Basic startup

- Page loads without console errors.
- Example 1 loads automatically.
- Source is visible.
- Memory contains zeros.

## Example navigation

- Expanded-sidebar example list works.
- Collapsed numbered example buttons work.
- Active example styling is synchronized.
- All built-in examples load.

## Step navigation

- Back/Next enable and disable correctly.
- Active source line changes correctly.
- Memory changes with states.
- Source scroll follows the active line.

## Memory

- Addresses display in decimal.
- Hex/Decimal/Binary word formatting works.
- Stack frames render correctly.
- Reference highlighting works.
- Memory pane scrolls independently.

## Memory regions

- Initial memory view is combined.
- `Separate memory regions` animation works.
- `Combine memory` reverses it.
- Combined region boundaries do not show a visually excessive gap.

## Layout

- Wide desktop: source/memory approximately 50/50.
- Medium width: memory retains readable minimum width.
- Source becomes horizontally scrollable rather than crushing memory.
- Narrow/mobile layout remains usable.

## Sidebar

- Collapsed rail remains visible.
- Burger opens/closes sidebar.
- Burger does not float over content.
- Quick example numbers have useful tooltips.

## Theme

- Light mode readable.
- Dark mode readable.
- Active line, frames, references, and selection remain distinguishable in both.

## Custom files

- Loading a `.java` file updates source.
- Loading a JSON file updates states.
- Invalid JSON/configuration produces a useful validation error.
- Custom mode does not accidentally select a built-in example.

---

# Potential future directions

Possible enhancements that fit the current architecture include:

- restoring the Note panel as an optional/collapsible teaching aid,
- additional examples for static fields,
- arrays of primitives,
- `null` references,
- aliasing with two references to one object,
- nested objects,
- recursion,
- garbage/unreachable objects,
- optional arrows between reference cells and heap objects,
- editable example metadata,
- keyboard navigation through states,
- presentation/fullscreen mode,
- external example packs for GitHub Pages,
- a separate advanced memory model while retaining the current beginner model.

Any advanced mode should coexist with, not silently replace, the current simplified model.

---

# Hosting with GitHub Pages

Because the project is static, GitHub Pages is straightforward.

Typical setup:

1. Put the current visualizer in the repository root as `index.html`.
2. Commit `README.md`.
3. Push the repository to GitHub.
4. In the repository settings, enable GitHub Pages from the intended branch/root directory.

No build action is required for the current architecture.

---

# Summary for Codex

If you are an automated coding agent modifying this repository, start with these assumptions:

1. `index.html` is the complete application.
2. This is a teaching visualizer, not a JVM emulator.
3. Memory is a 32-cell, 32-bit, word-addressed virtual model.
4. Initial ranges: static = `0..7`, heap = `8..23`, stack = `24..31`; `memory.shared` allows heap/stack boundary movement.
5. One modeled variable/reference/field/array element = one memory cell.
6. Stack grows downward; heap/static grow upward.
7. Memory states are complete declarative snapshots tied to source line numbers.
8. Built-in examples form a deliberate curriculum.
9. Custom Java + JSON loading must keep working.
10. Stack-frame metadata and reference relationships are first-class concepts.
11. The Note infrastructure is hidden, not obsolete.
12. The UI should stay calm, projector-friendly, responsive, and usable in light/dark mode.
13. Combined memory is the initial view; separation is an explicit teaching action.
14. Preserve the current no-build/offline-friendly setup unless there is a compelling reason to change it.
15. When changing the configuration schema, update validation, rendering, built-ins, the embedded spec, and this README together.

