Excellent — that’s exactly how to make it a **teaching-grade reference**, not just a memory jogger.
Below is the updated **Markdown version** with:

✅ Proper syntax highlighting (` ```python `, ` ```expr `)
✅ Full inline comments on every code example so you understand what each part does
✅ Preserved clickable table of contents
✅ Clear visual hierarchy for skimming in Notion, Obsidian, VS Code, or GitHub

---

# 🎛️ TouchDesigner Syntax Cheat Sheet

*A quick-reference for linking, referencing, and automating parameters without syntax headaches.*

---

## 🧭 Table of Contents

1. [Operator Lookups & Paths](#-operator-lookups--paths)
2. [Parameter Read / Write](#-parameter-read--write)
3. [Reading from CHOPs](#-reading-from-chops)
4. [Reading from DATs (Tables/Text)](#-reading-from-dats-tablestext)
5. [TOP Pixel / Value Access](#-top-pixel--value-access)
6. [CHOP Export vs Bind vs Expression](#-chop-export-vs-bind-vs-expression)
7. [Panel (UI) Values](#-panel-ui-values)
8. [Time, Frames & Timing Utilities](#-time-frames--timing-utilities)
9. [Math Helpers (tdu)](#-math-helpers-tdu)
10. [Ternary / Logic Patterns](#-ternary--logic-patterns)
11. [Pulses](#-pulses)
12. [Mod / Extensions / Storage](#-mod--extensions--storage)
13. [Callbacks You’ll Actually Use](#-callbacks-youll-actually-use)
14. [DAT ↔ CHOP Conversions](#-dat--chop-conversions)
15. [SOP / TOP / CHOP “Glue” Operators](#-sop--top--chop-glue-operators)
16. [Common Index & Casting Pitfalls](#-common-index--casting-pitfalls)
17. [Replicator / Clone Basics](#-replicator--clone-basics)
18. [File / System Bits](#-file--system-bits)
19. [Handy One-Liners](#-handy-one-liners)
20. [Best-Practice Matrix](#-best-practice-matrix)

---

## 🔹 Operator Lookups & Paths

**Why it matters:** Operators are the building blocks of TouchDesigner. Referencing them correctly is how everything connects.

**Example use case:** Trigger a movie TOP when a button is pressed.

```python
op('null1')                # Gets the operator named "null1"
ops('geo*')                # Returns all operators whose names start with "geo"
op('.')                    # Refers to the current component (self)
op('..')                   # Refers to the parent component
op('/')                    # Refers to the project root
parent(2)                  # Goes up two parent levels
me.name                    # Returns this operator’s name
me.digits                  # Extracts trailing digits from operator’s name (useful in clones)
```

---

## 🔹 Parameter Read / Write

**Why it matters:** Every knob and field in TD is a parameter. Learn how to get and set them programmatically.

```python
# Set the 'tx' parameter of transform1 to 2.5
op('transform1').par.tx = 2.5

# Get the evaluated numeric value (as a float)
val = op('transform1').par.tx.eval()

# Access the raw expression string (if any)
expr_str = op('transform1').par.tx.expr

# Check how the parameter is linked: CONSTANT, EXPRESSION, EXPORT, or BIND
mode = op('transform1').par.tx.mode
```

*Use this for runtime control or when automating parameter changes.*

---

## 🔹 Reading from CHOPs

**Why it matters:** CHOPs hold time-based data like LFOs, sensors, or sliders.

```python
# In a parameter expression (evaluated each frame)
op('noise1')['chan1']      # Returns current sample value of channel 'chan1'

# In Python
ch = op('noise1').chan('chan1')  # Get channel object
val = ch.eval()                  # Get current evaluated value
frame_50 = ch[50]                # Access a specific sample index (50th frame)
```

*Common for driving animations or reacting to live input.*

---

## 🔹 Reading from DATs (Tables/Text)

**Why it matters:** DATs store structured data (text, lists, tables).

```python
# Get the cell at row 2, column 0
cell = op('table1')[2, 0]

# Get the string value inside that cell
value = cell.val

# Get an entire row or column as a list of cells
row = op('table1').row(2)
col = op('table1').col(0)
```

*Perfect for cue lists, configuration tables, or text content.*

---

## 🔹 TOP Pixel / Value Access

**Why it matters:** Sometimes you need to grab a pixel’s RGB values directly (e.g., image analysis).

```python
# Get RGBA values of pixel at x=100, y=50
r, g, b, a = op('cam1').pixel(100, 50)

# Get dimensions of a TOP
w = op('cam1').width
h = op('cam1').height
```

⚠️ *Avoid heavy per-pixel reads; use an Analyze TOP + TOP to CHOP when possible.*

---

## 🔹 CHOP Export vs Bind vs Expression

**Why it matters:** These are your three main ways to link things in TD.

| Method         | Use When                                        | Example                         |
| -------------- | ----------------------------------------------- | ------------------------------- |
| **Export**     | You need a live stream of data                  | Drive rotation with an LFO CHOP |
| **Bind**       | You want one master parameter to control others | Global opacity control          |
| **Expression** | You need quick conditional logic                | Toggle states or simple maths   |

```python
# --- Export (CHOP → Parm) ---
op('lfo1')['chan1'].export(op('geo1').par.tx)   # Channel drives parameter
op('lfo1')['chan1'].unexport(op('geo1').par.tx) # Stop exporting

# --- Bind (two parameters stay linked) ---
p = op('geo1').par.tx
p.bindExpr = "op('controller1').par.val"        # Sets up live binding expression
p.mode = ParMode.BIND                           # Enable bind mode

# --- Expression (inline logic) ---
op('switch1').par.index = 1 if op('button1').panel.state else 0
```

---

## 🔹 Panel (UI) Values

**Why it matters:** Panel COMPs give you real-time user interaction data.

```python
op('button1').panel.state      # 1 when pressed, 0 when released
op('slider1').panel.u          # Normalised horizontal position (0–1)
op('container1').panel.inside  # True if mouse is inside container
```

**Example:**
Use in expressions — `1 if op('button1').panel.state else 0` — to toggle visuals.

---

## 🔹 Time, Frames & Timing Utilities

**Why it matters:** Many effects depend on time-based logic and synchronisation.

```python
# Run a function in 1 second (1000ms)
run("op('fade1').par.enable = 1", delayMilliSeconds=1000)

# Access time values
current_frame = absTime.frame
seconds = me.time.seconds
fps = me.time.rate
```

*Use `run()` for delayed actions, or `absTime` for project-wide sync.*

---

## 🔹 Math Helpers (tdu)

**Why it matters:** tdu provides fast helper functions for remapping and clamping values.

```python
# Clamp a value between 0 and 1
tdu.clamp(x, 0, 1)

# Remap slider u (0–1) to LED brightness (0–255)
brightness = tdu.remap(op('slider1').panel.u, 0, 1, 0, 255)

# Linearly interpolate between two values
mix = tdu.lerp(a, b, 0.5)

# Find normalised position of x between two ranges
percent = tdu.unlerp(0, 10, x)
```

*Cleaner and safer than writing manual math expressions.*

---

## 🔹 Ternary / Logic Patterns

**Why it matters:** Build compact “if this then that” expressions.

```expr
1 if op('button1').panel.state else 0  # Returns 1 when pressed, 0 when not
'Play' if op('toggle1').panel.state else 'Stop'  # Returns strings conditionally
```

*Keep expressions readable; complex logic should move to Python callbacks.*

---

## 🔹 Pulses

**Why it matters:** Pulses are used for triggering one-time events.

```python
# Correct: triggers a reload pulse on a Movie File In TOP
op('moviefilein1').par.reload.pulse()

# Wrong: don’t do this; it won’t actually trigger
# op('moviefilein1').par.reload = 1
```

*Used for buttons, resets, and other one-off actions.*

---

## 🔹 Mod / Extensions / Storage

**Why it matters:** These make projects modular, reusable, and stateful.

```python
# Call a method from a custom extension
op('controller').ext.ControllerEXT.DoThing(123)

# Store a custom variable persistently on an operator
op('base1').store('score', 17)

# Retrieve stored value (with default if missing)
score = op('base1').fetch('score', default=0)

# Access shared modules by name
mod('utils').RemapValue(0.5)
```

*Best practice for large projects with shared logic.*

---

## 🔹 Callbacks You’ll Actually Use

**Why it matters:** Execute DATs respond to changes or inputs in your network.

### CHOP Execute DAT

```python
def onValueChange(channel, sampleIndex, val, prev):
    # Runs when any CHOP channel value changes
    if channel.name == 'trigger' and val > 0.5:
        op('movie1').par.play = True  # Start playing when trigger activates
    return
```

### Panel Execute DAT

```python
def onOffToOn(panelValue):
    # Fires when button goes from off → on
    op('switch1').par.index = 1
    return
```

*Used for button presses, sensor inputs, or triggers.*

---

## 🔹 DAT ↔ CHOP Conversions

**Why it matters:** You’ll often want to move data between tables and CHOPs.

```python
# Get value from a table cell (row 2, col 0)
val = float(op('table1')[2,0].val)

# Write CHOP channel values into a table
t = op('outTable')
t.clear()
for i, v in enumerate(op('lfo1').chan('chan1').vals):
    t.appendRow([i, v])  # Row: index, value
```

*Efficiently bridges static and dynamic data.*

---

## 🔹 SOP / TOP / CHOP “Glue” Operators

**Why it matters:**
TouchDesigner’s power comes from being able to **translate data between operator families** — geometry (SOPs), images (TOPs), motion/data (CHOPs), and tables/text (DATs).
“Glue operators” are those whose *whole job* is to **convert one data type into another** so you can cross those boundaries.

Think of them like *adapters* between worlds:

```
TOP (image) → CHOP (data) → SOP (geometry)
```

Each bridge lets you drive one kind of content with another.

---

### 🧩 Common Glue Operator Examples

| Source             | Target                                    | Glue Operator                                                           | Example Use Case                                |
| ------------------ | ----------------------------------------- | ----------------------------------------------------------------------- | ----------------------------------------------- |
| **CHOP → TOP**     | **TOP to CHOP** (reverse)                 | Use **Ramp TOP** to create a gradient that represents an audio waveform | Drive brightness of a texture from audio volume |
| **TOP → CHOP**     | `TOP to CHOP`                             | Convert pixel brightness or colour into a CHOP channel                  | Control an LED ring with webcam brightness      |
| **TOP → SOP**      | `TOP to SOP`                              | Use pixel positions or brightness to displace geometry                  | Make a mesh move based on image intensity       |
| **SOP → CHOP**     | `SOP to CHOP`                             | Extract point positions as CHOP channels                                | Drive particles or OSC data with point motion   |
| **DAT → CHOP**     | `DAT to CHOP`                             | Convert table cells to channel values                                   | Animate parameters from spreadsheet data        |
| **CHOP → DAT**     | `CHOP to DAT`                             | Convert channels to tables for logging                                  | Save sensor readings into a CSV                 |
| **CHOP → SOP**     | `Trail SOP` or `Limit SOP` with CHOP data | Plot sensor or audio waveforms as 3D geometry                           | Draw real-time curves in space                  |
| **CHOP → COMP/UI** | Export CHOP values to panel parameters    | Link sliders and LEDs to live data                                      | Make UI elements react to signals               |

---

### 🧠 Key Concept

Data is *fluid* in TD — CHOPs give you continuous signals, DATs give you structured logic, SOPs define form, and TOPs define colour or light.
Learning to **glue them together** is what turns small patches into full media systems.

---

### 💡 Example Network

> Imagine an audio-reactive mesh:

```
Audio CHOP → Math CHOP (scale) → CHOP to SOP → Render TOP → Composite TOP
```

Here:

* The CHOP carries sound amplitude.
* The Math CHOP normalises it.
* The CHOP to SOP turns the values into vertex heights.
* The Render TOP outputs the result visually.

That’s a “glue chain”.

---

## 🔹 Common Index & Casting Pitfalls

**Why it matters:** Prevent “string vs float” errors and ensure clean type handling.

```python
# Convert string from table cell to float
val = float(op('table1')[2,0].val)

# Always use .eval() when reading parameter values in Python
x = op('transform1').par.tx.eval()
```

*TD often returns strings — always cast when doing maths.*

---

## 🔹 Replicator / Clone Basics

**Why it matters:** Automate the creation of repeated UI or component structures.

```python
def onCreate(comp):
    # Called whenever a new replicated COMP is created
    comp.par.tx = comp.digits * 2  # Offset each clone by its index
    return
```

*Great for thumbnails, gallery grids, or modular setups.*

---

## 🔹 File / System Bits

**Why it matters:** For working with file paths or system-specific logic.

```python
project.folder       # Path to the current project directory
project.file         # Full path to the .toe file
system.platform      # Returns 'macOS', 'win', or 'linux'
```

*Use for saving/loading files relative to your project.*

---

## 🔹 Handy One-Liners

**Why it matters:**
These snippets are small, reusable expressions or Python calls you’ll use dozens of times while patching — like “quick glue” for logic, math, or automation.
They’re meant for parameter fields, callbacks, or short Execute DATs.

---

### 🧠 Common Contexts

* Inside a **parameter expression** (e.g. `tx` on a Transform TOP).
* Inside a **CHOP Execute DAT** to drive logic.
* In **callbacks** for triggers, fades, or toggles.

---

### 📋 One-Liner Examples (with context)

```expr
# Clamp the value of an LFO between -1 and 1
tdu.clamp(op('lfo1')['chan1']*5, -1, 1)
```

→ Prevents unwanted overshoot or distortion.

```expr
# Toggle state from a button
1 if op('button1').panel.state else 0
```

→ Clean way to return 1 when a button is pressed, 0 otherwise.

```python
# Pulse a reload trigger on a Movie TOP (Python)
op('movie1').par.reload.pulse()
```

→ Instantly reloads a video file (useful for dynamic media players).

```expr
# Remap slider position (0–1) to range -10 → 10
tdu.remap(op('slider1').panel.u, 0, 1, -10, 10)
```

→ Converts a UI control range to a useful numeric domain.

```python
# Delay an action by 200 ms
run("op('fade1').par.enable=1", delayMilliSeconds=200)
```

→ Adds a small “temporal cushion” for sequencing or soft transitions.

---

### 🔧 Tips

* One-liners are great **glue** for simple logic, but if you find yourself writing more than 1–2 conditions, move that logic into an Execute DAT or extension.
* You can chain functions with commas or parentheses:

  ```expr
  abs(tdu.clamp(op('lfo1')['chan1'], -0.5, 0.5))
  ```

  (Clamp and then get absolute value.)

---

## 🔹 Best-Practice Matrix

**Why it matters:**
TouchDesigner offers *many* ways to connect operators, but each has its own trade-offs.
This matrix helps you choose the right method for stability, readability, and performance.

---

### 🧭 Concept Overview

| Link Type           | Nature               | Editable In UI | Evaluation Speed   | Good For                                     | Example                    |
| ------------------- | -------------------- | -------------- | ------------------ | -------------------------------------------- | -------------------------- |
| **CHOP Export**     | Signal-driven        | ✅ Yes          | ⚡ Fast (per-frame) | Live data like LFOs, sensors, and animations | LFO drives rotation        |
| **Parameter Bind**  | Two-way relationship | ✅ Yes          | ⚙️ Moderate        | Global settings shared across COMPs          | Master “Opacity” slider    |
| **Expression**      | One-off logic        | ✅ Yes          | ⚙️ Moderate        | Simple maths, toggles, small conditions      | “if button on → visible”   |
| **Python Callback** | Event-driven         | ❌ (code only)  | 🧠 Logical timing  | Triggers, automation, side effects           | “Play video on event”      |
| **Direct Script**   | Manual override      | ❌ (code only)  | 🧠 Contextual      | Setup, initialisation, scene changes         | “op('base1').par.active=1” |

---

### 💡 Example Scenarios

| Scenario                                | Best Link           | Why                                            |
| --------------------------------------- | ------------------- | ---------------------------------------------- |
| Audio-reactive visualiser               | **CHOP Export**     | Smooth, continuous response to audio amplitude |
| Shared brightness control for 10 lights | **Bind**            | One master slider controls all others          |
| Light toggles on a button               | **Expression**      | Small conditional is cleanest                  |
| Play/pause with MQTT or OSC event       | **Python Callback** | Trigger-based logic                            |
| Scene loading setup                     | **Direct Script**   | One-time configuration on load                 |

---

### ⚠️ Common Mistakes

| Pitfall                                       | Problem                                 | Fix                                     |
| --------------------------------------------- | --------------------------------------- | --------------------------------------- |
| Using expressions for everything              | Slows evaluation, hard to debug         | Use CHOP exports for live signals       |
| Manually setting `.par` values in loops       | Overwrites links/binds                  | Use `.pulse()` or CHOP control          |
| Binding to unstable sources (temporary nodes) | Causes greyed-out parms or broken links | Bind only to persistent or master nodes |

---

### 🧩 Rule of Thumb

> **If it changes every frame → use CHOP.**
> **If it changes occasionally → use Bind or Expression.**
> **If it changes on events → use Python.**
