imui
====

An immediate-mode user interface for sysl — no retained tree, no reconciler, and **no allocation at
all**. Written for a panel on a microcontroller, and it runs on a desktop because nothing in it
knows the difference.

```hocon
dependencies {
  imui { git = "github.com/sysl-lang/imui", version = "0.1.0" }
}
```

```sysl
import sh.sysl.imui.*

var page = layout(Rect(0, 0, 320, 480).inset(th.pad), th.pad)

ui.begin(pointer(touch_x, touch_y, touching))

toggle(&ui, &painter, th, page.cut_top(th.min_touch), "Backlight", &app.backlight)
slider(&ui, &painter, th, page.cut_top(th.min_touch), &app.brightness, 0, 100)

if button(&ui, &painter, th, page.cut_bottom(th.min_touch), "Apply")
    apply(app)

ui.finish()
```

There is no `requires` block in the manifest and every file declares `@no_alloc`, which the compiler
holds the package to — including its tests. It asks the machine for no allocator, no operating
system and no C library.

The application owns the state
------------------------------

`toggle` is handed `&app.backlight` and writes through it. There is no copy of that flag inside this
library, nothing to keep synchronised with it, and no callback that fires later and finds it gone.
The class of bug where a widget and its model disagree cannot be written.

What the library keeps between frames is **three integers**: which widget the pointer captured,
which one it is over, and which one has the focus. That is the whole of its state.

A widget's identity is where it was called from
-----------------------------------------------

Immediate mode has no object to hang an identity on, and sysl answers it with an ordinary defaulted
argument:

```sysl
button(ui: *Ui, p: *P, th: Theme, r: Rect, s: string, site_id: u32 = site()) -> bool
```

`site()` defaults to `__FILE__` and `__LINE__`, and a default is evaluated where the *call* is
written — so two `button` calls on two lines are two widgets and the caller writes nothing. Two on
one line are one widget, which is the case this cannot see; pass `site_id`, or set `Ui.seed`, which
is also how a widget inside a loop gets an identity per turn:

```sysl
for i in 0..<rows.len
    ui.seed = u32(i) + 1
    if button(&ui, &p, th, row, "delete") then remove(i)

ui.seed = 0
```

Drawing is a trait bound, so it costs nothing
---------------------------------------------

A backend implements six members — `fill_rect`, `draw_text`, `set_clip`, `clear_clip`, `text_width`,
`line_height` — and gets `stroke_rect`, `text_in`, `hline` and `vline` free, because a trait member
may carry a body.

Every widget takes `p: *P` for some `[P: Painter]`, never a `*Painter` object. The calls
monomorphize to direct calls: a built program has no method table for this trait and nothing
dynamically dispatched. That is what keeps the promise on the manifest.

Three backends come with the package
------------------------------------

| backend | what it is for |
|---|---|
| `FrameBuffer` | RGB565 straight into storage the caller owns, text blitted from a bitmap font. The board, and the reference implementation |
| `Recorder[M]` | writes the frame down instead of drawing it, and measures through the real painter. What the damage pass compares |
| `Metrics` | measures and counts and draws nothing. Lays a screen out before there is a screen, and is what these tests assert against |

`FrameBuffer` is the argument against reaching for a vector rasterizer on a board. Cairo and PlutoVG
draw paths, curves, gradients and antialiased glyphs, and pay for them with an allocator and a glyph
cache — and a panel of buttons and sliders uses none of it. A filled rectangle is a store loop and
text is a bit-blit.

**No font ships here**, because a font is data: which glyphs, at what size, is the program's own
business. `Font` takes `h` bytes per glyph, one per row, most significant bit leftmost.

Only what moved is repainted
----------------------------

A 320×480 panel is 300 KB of pixels over SPI. Pushing all of it because one button lit up is the
difference between a user interface and a slideshow, so a frame is recorded, compared with the last
one, and only the horizontal bands that differ are drawn and sent:

```sysl
rec.reset()
ui.begin(ev)
draw_the_screen(&ui, &rec, th, &app)      -- generic over the painter, so this is the only copy
ui.finish()

val mask = if rec.overflowed then all_bands(16) else damage(prev, rec.recorded(), screen, 16)
var band = next_band(mask, 16, screen, 0)

while band.is_some()
    val b = band.unwrap()

    replay(rec.recorded(), &panel, b.rect)     -- and push b.rect to the display
    band = next_band(mask, 16, screen, b.next)
```

Adjacent dirty bands come back as **one** rectangle, so two bands cost one window-set and one stream
rather than two. The demo's scripted run prints what this is worth: an idle frame repaints 30 rows
of 480, and moving a slider repaints 60.

Two things about it are worth knowing before relying on it. The comparison is **by position in the
list**, so a widget added or removed part-way through shifts everything after it and dirties the rest
of the screen — the right trade for a panel, where the same widgets are emitted in the same order
every frame. And a frame that **overflows** its command list has no usable record of itself, so
`overflowed` must be checked and answered with `all_bands`; the library will not silently guess.

What immediate mode costs
-------------------------

Stated here rather than discovered later.

| cost | what it means |
|---|---|
| No layout engine | `Layout` cuts rectangles off an area. Constraint solving is a retained-tree idea and there is no tree |
| Animation is awkward | there is no persistent object to animate, so a program drives it from its own clock |
| Text entry is fiddlier | the editing state has to live somewhere, and there is nowhere yet — see below |

The widgets
-----------

`panel`, `label`, `muted_label`, `separator`, `button`, `accent_button`, `checkbox`, `toggle`,
`slider`, `progress`, `tab`.

**They are free functions rather than methods on `Ui`, on purpose.** sysl declares a type's methods
with the type, so a widget written by a program using this library could never be a method — and a
library whose own widgets read `ui.button(...)` while everybody else's read `my_gauge(&ui, ...)`
teaches that its extension point is second class. Written this way there is no difference. A widget
is four steps:

```sysl
gauge[P: Painter](ui: *Ui, p: *P, th: Theme, r: Rect, v: int, site_id: u32 = site()) -> bool
    val id = ui.id_at(site_id)             -- an identity for the call site
    val hit = ui.interact(id, r)           -- what the pointer and the D-pad did to that rectangle

    p.fill_rect(r, ui.control_color(th, id))
    hit                                    -- the answer; the application owns the state
```

A D-pad works, not just a finger
--------------------------------

`Input` carries a pointer *and* a four-way `nav` with an `activate`, because a small board often has
buttons and no touchscreen. Focus walks the frame in the order the widgets were drawn and wraps at
both ends; a focus left on a widget that stopped being drawn falls back rather than stranding.

Not built yet
-------------

- **No text entry**, which is the one widget that needs state this library does not keep. That is
  where the fixed-size table keyed by widget id goes when it is written — it is not there now because
  nothing else needed it.
- **No scrolling**, no list virtualization.
- **No font**, as above.

Tests
-----

Thirty-one, and they drive the widgets the way a finger does — press, drag off, drag back, release —
then read the recorded frame back and assert on it.

```
sysl test .
```

License
-------

[ISC](LICENSE)
