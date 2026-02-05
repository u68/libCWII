# libcwii.h/libcwx.h Guide 

## Scope and inclusion

This document describes the declarations present in `libcwii.h`/`libcwx.h`

To use this API on cwii, include the header:

```c
#include "libcwii.h"
```

and for cwx:

```c
#include "libcwx.h"
```

No additional includes are required for the examples in this document.

Note:

Due to limitations of the cwx system, the default configuration of libcwx specifies that there is only 2kb of heap, and 3kb of fs (64 nodes), and the screen is only 1bbp so therefore only black and white is present)

## Conventions

- Screen coordinate system:
  - $x \in [0, 191]$ (left to right)
  - $y \in [0, 63]$ (top to bottom)
- Angles are expressed in degrees and are indexed by a 360-entry sine/cosine lookup table. Valid range is typically $[0, 359]$.
- Colour values are 2-bit (0–3) for the framebuffer. In typical usage, the following symbols (defined elsewhere in the header) are used:
  - `TUI_COLOUR_WHITE` (0)
  - `TUI_COLOUR_LIGHT_GREY` (1)
  - `TUI_COLOUR_DARK_GREY` (2)
  - `TUI_COLOUR_BLACK` (3)
  - `TUI_COLOUR_IMAGE` (special handling for two-bitplane images)

Note: light grey and dark grey colours are not available for cwx, and on cwx, colour image and colour black are the same)

## Forward declarations

### `typedef struct fs_node_t;`

Opaque forward declaration for filesystem nodes.

- You may use `fs_node_t *` in APIs without needing the struct definition at the point of declaration.
- The full definition (fields, layout) exists later in the header; user code should treat `fs_node_t *` as an owning/borrowing handle and avoid accessing fields directly unless explicitly intended.

## TUI / Graphics functions

### `void tui_rotate_point(byte ax, byte ay, byte px, byte py, word angle, byte *out_x, byte *out_y);`

Rotate a point around an anchor.

- **Parameters**
  - `ax`, `ay`: Anchor (pivot) position.
  - `px`, `py`: Input point to rotate.
  - `angle`: Rotation angle in degrees.
  - `out_x`, `out_y`: Output pointers receiving the rotated coordinates.
- **Behaviour and constraints**
  - If `angle == 0`, the function returns `(px, py)` unchanged.
  - Angles are used as indices into a 360-entry lookup table; pass values in the range 0–359.
  - Results are returned as `byte`; intermediate arithmetic uses signed types.

**Example: rotate a point around the centre of the screen**

```c
void example_rotate(void) {
    byte x_out, y_out;

    /* Rotate point (140, 20) around anchor (96, 32) by 90 degrees. */
    tui_rotate_point(96, 32, 140, 20, 90, &x_out, &y_out);

    /* Visualise the result. */
    tui_set_pixel(x_out, y_out, TUI_COLOUR_BLACK, 1);
}
```

---

### `void tui_simple_line(byte x0, byte y0, byte x1, byte y1, byte colour);`

Draw a 1-pixel-thick line using a simple Bresenham rasterisation.

- **Parameters**
  - `x0`, `y0`: Start coordinate.
  - `x1`, `y1`: End coordinate.
  - `colour`: Pixel colour (0–3).
- **Notes**
  - This function draws with thickness 1 and no pattern/styling.
  - For styled/thick lines, prefer `tui_draw_line`, `tui_pattern_draw_line`, or `tui_advanced_draw_line`.

**Example: crosshair**

```c
void example_crosshair(void) {
    tui_simple_line(96, 0, 96, 63, TUI_COLOUR_DARK_GREY);
    tui_simple_line(0, 32, 191, 32, TUI_COLOUR_DARK_GREY);
}
```

---

### `void tui_advanced_draw_line(byte* data, byte bit_length, byte x0, byte y0, byte x1, byte y1, byte colour, byte thickness);`

Draw a line where its on/off pattern is stretched to the full line length.

- **Parameters**
  - `data`: Pointer to a bit-array defining the pattern.
  - `bit_length`: Number of bits in the pattern.
  - `x0`, `y0`, `x1`, `y1`: Line endpoints.
  - `colour`: Colour used when the pattern bit is 1.
  - `thickness`: Thickness (interpreted by `tui_set_pixel`).
- **Pattern format**
  - Bit index 0 is taken from `data[0]` MSB (mask `0x80`), then `0x40`, etc.
  - The pattern is *stretched* across the line length: the i-th pixel samples bit `floor((i * bit_length) / line_len)`

**Example: draw a “grow-to-fit” dashed line**

```c
void example_advanced_line(void) {
    /* 16-bit pattern: 1111 0000 1111 0000 */
    byte pattern_bits[2] = { 0xF0, 0xF0 };

    tui_advanced_draw_line(pattern_bits, 16, 10, 10, 180, 50, TUI_COLOUR_BLACK, 1);
}
```

---

### `void tui_pattern_draw_line(byte pattern, byte x0, byte y0, byte x1, byte y1, byte colour, byte thickness);`

Draw a thick line with a repeating 8-bit pattern.

- **Parameters**
  - `pattern`: 8-bit pattern; bit 7 is sampled first.
  - `x0`, `y0`, `x1`, `y1`: Line endpoints.
  - `colour`: Colour used when the pattern bit is 1.
  - `thickness`: Thickness (as interpreted by `tui_set_pixel`).
- **Behaviour**
  - If `pattern == 0xFF`, the line is solid.
  - If a pattern bit is 0, the implementation draws `TUI_COLOUR_WHITE` at that pixel position.

**Example: dotted and dashed**

```c
void example_pattern_lines(void) {
    tui_pattern_draw_line(0xAA, 10, 20, 180, 20, TUI_COLOUR_BLACK, 1); /* dotted */
    tui_pattern_draw_line(0xF8, 10, 30, 180, 30, TUI_COLOUR_BLACK, 1); /* dashed */
}
```

---

### `void tui_render_buffer(void);`

Render (copy) the software VRAM buffer to the physical screen VRAM.

- **When to call**
  - If drawing is done to the buffer (i.e., when `Write2RealScreen` is 0), you must call `tui_render_buffer()` for the user to see the result.
  - If drawing is configured to write directly to the physical screen, this may be unnecessary.

**Example: buffered frame render**

```c
void example_buffered_frame(void) {
    Write2RealScreen = 0;

    tui_clear_screen();
    tui_draw_text(10, 10, "Hello", TUI_FONT_SIZE_6x8, 0, 0, 0, TUI_COLOUR_BLACK);

    tui_render_buffer();
}
```

---

### `void tui_clear_screen(void);`

Clear the screen.

- **Behaviour**
  - If `Write2RealScreen` is non-zero, clears the physical screen VRAM.
  - Otherwise, clears the software VRAM buffer.

**Example: clear before drawing**

```c
void example_clear(void) {
    Write2RealScreen = 0;
    tui_clear_screen();
}
```

---

### `byte tui_get_pixel_b(byte x, byte y, byte buf);`

Read a pixel from a specified buffer/bitplane selection.

- **Parameters**
  - `x`, `y`: Screen coordinate.
  - `buf`:
    - If `buf == 2`, reads composite colour using `tui_get_pixel`.
    - Otherwise selects one of the underlying bitplanes / buffers.
- **Returns**
  - For out-of-bounds coordinates, returns 0.
  - Otherwise returns a value representing the queried plane/colour contribution.

**Example: probe a pixel before drawing**

```c
void example_probe(void) {
    byte col = tui_get_pixel(50, 20);
    if (col == TUI_COLOUR_WHITE) {
        tui_set_pixel(50, 20, TUI_COLOUR_BLACK, 1);
    }
}
```

---

### `byte tui_get_pixel(byte x, byte y);`

Read the composite 2-bit colour at `(x, y)`.

- **Returns**
  - 0 for out-of-bounds.
  - Otherwise a 0–3 colour value.

**Example: simple collision test**

```c
byte is_occupied(byte x, byte y) {
    return (tui_get_pixel(x, y) != TUI_COLOUR_WHITE);
}
```

---

### `void tui_set_pixel(byte x, byte y, byte colour, byte size);`

Set a pixel with a “size” parameter.

- **Parameters**
  - `x`, `y`: Centre coordinate.
  - `colour`: Colour value.
  - `size`:
    - 0: no-op
    - 1: draw a single pixel
    - 2: draw a small plus-shaped footprint
    - $\ge 3$: draws a filled circle-like stamp via `tui_circle(x, y, size >> 1, colour)`

**Example: draw three different point sizes**

```c
void example_points(void) {
    tui_set_pixel(20, 20, TUI_COLOUR_BLACK, 1);
    tui_set_pixel(40, 20, TUI_COLOUR_BLACK, 2);
    tui_set_pixel(60, 20, TUI_COLOUR_BLACK, 6);
}
```

---

### `void tui_circle(byte centerX, byte centerY, byte radius, byte c);`

Draw a filled circle (implemented as a filled disk).

- **Parameters**
  - `centerX`, `centerY`: Centre.
  - `radius`: Radius in pixels.
  - `c`: Colour.

**Example: filled disk**

```c
void example_filled_circle(void) {
    tui_circle(96, 32, 10, TUI_COLOUR_DARK_GREY);
}
```

---

### `void tui_draw_line(byte x0, byte y0, byte x1, byte y1, byte colour, byte thickness, byte style);`

Draw a styled line.

- **Parameters**
  - `thickness`: Passed to `tui_set_pixel`.
  - `style`: A style selector. Typical values are from `enum tui_line_style` (defined elsewhere):
    - `TUI_LINE_STYLE_SOLID`, `TUI_LINE_STYLE_DOTTED`, `TUI_LINE_STYLE_DASHED`
  - If `style` is not one of the predefined styles, it is treated as a custom 8-bit pattern.

**Example: thick dashed line**

```c
void example_styled_line(void) {
    tui_draw_line(5, 5, 190, 60, TUI_COLOUR_BLACK, 2, TUI_LINE_STYLE_DASHED);
}
```

---

### `void tui_draw_rectangle(byte x, byte y, byte width, byte height, sbyte ax, sbyte ay, word rotation, byte colour, byte thickness, byte style);`

Draw a rectangle outline, optionally rotated.

- **Parameters**
  - `x`, `y`: Reference position.
  - `width`, `height`: Dimensions.
  - `ax`, `ay`: Anchor offset, relative to the rectangle reference; used as the rotation pivot.
  - `rotation`: Degrees (0–359). Values outside may be normalised.
  - `colour`, `thickness`, `style`: Passed to `tui_draw_line`.

**Example: rotate around the rectangle centre**

```c
void example_rotated_rect(void) {
    byte w = 40;
    byte h = 20;

    /* Anchor at centre: (w/2, h/2). */
    tui_draw_rectangle(96, 32, w, h, (sbyte)(w/2), (sbyte)(h/2), 30,
                       TUI_COLOUR_BLACK, 1, TUI_STYLE_SOLID);
}
```

---

### `void tui_draw_points(byte cx, byte cy, byte x, byte y, byte thickness, byte colour);`

Helper used for circle rasterisation: plots the eight symmetric points of a circle.

- **Intended usage**
  - Typically called from `tui_draw_circle`.
  - May be used in custom circle/arc code when you want direct control of the midpoint algorithm.

**Example: plot a single symmetry set**

```c
void example_circle_points(void) {
    tui_draw_points(96, 32, 10, 3, 1, TUI_COLOUR_BLACK);
}
```

---

### `void tui_draw_circle(byte cx, byte cy, byte radius, sbyte ax, sbyte ay, byte thickness, byte colour);`

Draw a circle outline using a midpoint circle algorithm.

- **Parameters**
  - `cx`, `cy`: Reference centre.
  - `ax`, `ay`: Anchor offset applied as `(cx - ax, cy - ay)`.
  - `radius`: Radius.
  - `thickness`, `colour`: Passed to `tui_set_pixel`.

**Example: outline circle centred at (96, 32)**

```c
void example_outline_circle(void) {
    tui_draw_circle(96, 32, 15, 0, 0, 1, TUI_COLOUR_BLACK);
}
```

---

### `void tui_get_font_size(byte font_size, byte* width, byte* height);`

Return the pixel width and height of the given font size identifier.

- **Parameters**
  - `font_size`: Use values from `enum tui_font_size` (defined elsewhere), e.g. `TUI_FONT_SIZE_6x8`.
  - `width`, `height`: Output pointers.

**Example: compute a baseline offset**

```c
byte font_h(void) {
    byte w, h;
    tui_get_font_size(TUI_FONT_SIZE_6x8, &w, &h);
    return h;
}
```

---

### `void tui_get_text_size(byte font_size, const char* text, byte* width, byte* height);`

Return the pixel extent for a string rendered using `tui_draw_text`.

- **Behaviour**
  - Width is computed as `char_width * strlen(text)`.
  - Height equals the font height.

**Example: centre a label horizontally**

```c
void example_center_text(void) {
    byte tw, th;
    tui_get_text_size(TUI_FONT_SIZE_6x8, "MENU", &tw, &th);

    /* Center at x = 96. */
    tui_draw_text((byte)(96 - (tw/2)), 5, "MENU", TUI_FONT_SIZE_6x8, 0, 0, 0, TUI_COLOUR_BLACK);
}
```

---

### `void tui_draw_text(byte x, byte y, const char* text, byte font_size, sbyte ax, sbyte ay, word rotation, byte colour);`

Draw a null-terminated ASCII string.

- **Parameters**
  - `x`, `y`: Reference position.
  - `text`: Null-terminated string.
  - `font_size`: Font selection.
  - `ax`, `ay`: Anchor offset for rotation.
  - `rotation`: Rotation in degrees.
  - `colour`: Colour.
- **Notes**
  - Each glyph is drawn via `tui_draw_char`, which in turn uses `tui_draw_image`.

**Example: rotated label around its first character**

```c
void example_rotated_text(void) {
    tui_draw_text(20, 40, "ROT", TUI_FONT_SIZE_6x8, 0, 0, 330, TUI_COLOUR_BLACK);
}
```

---

### `void tui_draw_byte(byte x, byte y, byte data, byte data2, byte mask);`

Low-level primitive: draw an 8-pixel-wide run into the framebuffer with masking and explicit bitplanes.

- **Parameters**
  - `x`, `y`: Target position.
  - `data`: Bits for bitplane 0.
  - `data2`: Bits for bitplane 1.
  - `mask`: Per-bit write enable mask.
- **Usage**
  - Use this when implementing custom blitters or when you already have byte-packed bitmap data.

**Example: draw an 8-pixel horizontal stripe**

```c
void example_draw_byte(void) {
    /* Draw 8 black pixels starting at x=10 (both bitplanes set). */
    tui_draw_byte(10, 10, 0xFF, 0xFF, 0xFF);
}
```

---

### `void tui_draw_image(byte x, byte y, byte width, byte height, const byte* bitmap, sbyte ax, sbyte ay, word rotation, byte colour);`

Draw a bitmap.

- **Parameters**
  - `width`, `height`: Image dimensions in pixels.
  - `bitmap`: Pointer to bitmap data.
  - `ax`, `ay`: Anchor offset for placement/rotation.
  - `rotation`: Degrees. If 0, a fast byte-oriented blit is used; otherwise a per-pixel sampling path is used.
  - `colour`:
    - If `colour` is one of the standard monochrome colours, the 1-bit `bitmap` acts as a mask.
    - If `colour == TUI_COLOUR_IMAGE`, the bitmap is treated as two concatenated 1-bit planes of size `ceil(width/8) * height` each.

**Example: draw a 16×8 monochrome icon (mask) in black**

```c
static const byte icon_16x8[] = {
    0x3C, 0x00,
    0x42, 0x00,
    0xA5, 0x00,
    0x81, 0x00,
    0xA5, 0x00,
    0x99, 0x00,
    0x42, 0x00,
    0x3C, 0x00,
};

void example_draw_icon(void) {
    tui_draw_image(10, 10, 16, 8, icon_16x8, 0, 0, 0, TUI_COLOUR_BLACK);
}
```

**Example: rotate an image around its centre**

```c
void example_rotate_icon(void) {
    tui_draw_image(80, 30, 16, 8, icon_16x8, 8, 4, 45, TUI_COLOUR_BLACK);
}
```

---

### `void tui_draw_char(byte x, byte y, char c, byte font_size, sbyte ax, sbyte ay, word rotation, byte colour);`

Draw a single glyph `c` from the built-in font tables.

- **Notes**
  - Font glyphs are addressed from ASCII 32 (`' '`). Passing characters below 32 is undefined.

**Example: draw a single character**

```c
void example_draw_char(void) {
    tui_draw_char(10, 50, 'A', TUI_FONT_SIZE_8x12, 0, 0, 0, TUI_COLOUR_BLACK);
}
```

---

### `void tui_draw_full_image(const word* bitmap, byte colour);`

Fast full-screen image transfer to the physical screen VRAM.

- **Parameters**
  - `bitmap`: Pointer to a word-packed framebuffer image.
  - `colour`:
    - `TUI_COLOUR_IMAGE`: copies both bitplanes.
    - `TUI_COLOUR_LIGHT_GREY`: copies only the lower bitplane region.
    - `TUI_COLOUR_DARK_GREY`: copies only the upper bitplane region.
- **Constraints**
  - This function targets the real screen memory map; it is intended for fast full-frame presentation.

**Example: present a precomputed frame**

```c
static const word frame_words[0x600] = { 0 };

void example_present_frame(void) {
    tui_draw_full_image(frame_words, TUI_COLOUR_IMAGE);
}
```

---

## Heap management functions

The heap in this API uses a simple free-list allocator stored in the memory region described by the heap configuration macros (see “Configuration macros”).

### `void hinit(void);`

Initialise the heap allocator.

- **Required**: call once during startup before `halloc`, `hcalloc`, `hrealloc`, or `hfree`.

**Example: initialise at boot**

```c
void system_init(void) {
    hinit();
}
```

---

### `void hmerge(void);`

Coalesce adjacent free blocks.

- **When to call**
  - Usually not required explicitly; `halloc` calls it when it fails to find space.
  - May be used as a manual defragmentation pass after many frees.

**Example: manual coalesce**

```c
void example_heap_merge(void) {
    hmerge();
}
```

---

### `void *halloc(word size);`

Allocate `size` bytes.

- **Returns**
  - Pointer to allocated memory, or `0` on failure.

**Example: allocate a buffer**

```c
void example_alloc(void) {
    byte *buf = (byte*)halloc(128);
    if (!buf) {
        return;
    }

    /* Use buf... */
    hfree(buf);
}
```

---

### `void *hcalloc(word num, word size);`

Allocate and zero-initialise `num * size` bytes.

- **Returns**
  - Pointer to allocated memory, or `0` on failure.

**Example: allocate an array of 32 bytes, cleared**

```c
void example_calloc(void) {
    byte *arr = (byte*)hcalloc(32, 1);
    if (!arr) return;

    /* arr is initialised to 0. */
    hfree(arr);
}
```

---

### `void *hrealloc(void *ptr, word size);`

Reallocate a block to a new size.

- **Parameters**
  - `ptr`: Existing allocation, or `0` to behave like `halloc`.
  - `size`: New size.
- **Returns**
  - New pointer on success, or `0` on failure.
- **Notes**
  - If the allocation must move, contents are copied up to the old size.
  - The old block is marked free when a new block is allocated.

**Example: grow a buffer**

```c
void example_realloc(void) {
    byte *buf = (byte*)halloc(16);
    if (!buf) return;

    buf = (byte*)hrealloc(buf, 64);
    if (!buf) {
        return;
    }

    hfree(buf);
}
```

---

### `void hfree(void *ptr);`

Free an allocation.

- Passing `0` is a no-op.

**Example**

```c
void example_free(void) {
    void *p = halloc(8);
    hfree(p);
}
```

## Filesystem functions

This filesystem is an in-memory hierarchical node store rooted at `FS_ROOT`. Nodes are stored in a fixed table, with file data allocated from a separate pool.

### Initialisation

#### `void fs_init(void);`

Initialise the filesystem structures, create the root directory, and initialise the data pool free list.

- **Required**: call once before using any other `fs_*` function.

**Example**

```c
void example_fs_boot(void) {
    fs_init();
}
```

---

### Node creation and lookup

#### `fs_node_t *fs_create_file(fs_node_t *parent, const char *name, fs_perms_t perms);`

Create a file node as the first child of `parent`.

- **Preconditions**
  - `parent` must be a directory with write permission.
  - `name` must be non-empty and unique among siblings.
- **Returns**
  - Pointer to the new file node, or `0` on failure.

**Example: create a file in root**

```c
void example_fs_create_file(void) {
    fs_init();

    fs_node_t *f = fs_create_file(FS_ROOT, "note", PERMS_RW);
    if (!f) return;

    fs_write_file(f, "abc", 3);
}
```

#### `fs_node_t *fs_create_directory(fs_node_t *parent, const char *name, fs_perms_t perms);`

Create a directory node as the first child of `parent`.

- Sets `perms.field.is_directory = 1` on success.

**Example: create `/cfg`**

```c
void example_fs_create_dir(void) {
    fs_init();
    (void)fs_create_directory(FS_ROOT, "cfg", PERMS_RWX);
}
```

#### `fs_node_t *fs_get_node_from_path(const char *path, fs_node_t *start);`

Resolve a path to a node.

- If `path` begins with `/`, resolution starts at `FS_ROOT`.
- Otherwise, resolution starts at `start`.
- Supports `.` and `..` path components.

**Example: resolve a node**

```c
void example_fs_resolve(void) {
    fs_init();
    fs_mkdir(FS_ROOT, "/cfg", PERMS_RWX);
    fs_touch(FS_ROOT, "/cfg/a", PERMS_RW);

    fs_node_t *n = fs_get_node_from_path("/cfg/a", FS_ROOT);
    (void)n;
}
```

---

### Node mutation

#### `byte fs_delete_node(fs_node_t *node);`

Delete a node and its subtree.

- **Returns**
  - `1` on success, `0` on failure.
- **Notes**
  - Deleting `FS_ROOT` is not permitted.

**Example: delete a directory tree**

```c
void example_fs_delete(void) {
    fs_init();
    fs_node_t *d = fs_mkdir(FS_ROOT, "/tmp/sub", PERMS_RWX);
    (void)d;

    fs_node_t *tmp = fs_dir_lookup(FS_ROOT, "/tmp");
    if (tmp) {
        (void)fs_delete_node(tmp);
    }
}
```

#### `byte fs_move_node(fs_node_t *node, fs_node_t *new_parent);`

Move `node` to become a child of `new_parent`.

- **Returns** `1` on success, `0` on failure.
- **Constraints**
  - `new_parent` must be a directory with write permission.
  - Moving a node under itself or under one of its descendants is rejected.

**Example: move a file**

```c
void example_fs_move(void) {
    fs_init();

    fs_node_t *a = fs_mkdir(FS_ROOT, "/a", PERMS_RWX);
    fs_node_t *b = fs_mkdir(FS_ROOT, "/b", PERMS_RWX);

    fs_node_t *f = fs_create_file(a, "x", PERMS_RW);
    if (!f) return;

    (void)fs_move_node(f, b);
}
```

#### `byte fs_rename_node(fs_node_t *node, const char *new_name);`

Rename a node.

- **Returns** `1` on success, `0` on failure.
- Enforces sibling uniqueness.

**Example: rename a file**

```c
void example_fs_rename(void) {
    fs_init();
    fs_node_t *f = fs_create_file(FS_ROOT, "old", PERMS_RW);
    if (!f) return;

    (void)fs_rename_node(f, "new");
}
```

#### `byte fs_chmod_node(fs_node_t *node, fs_perms_t new_perms);`

Replace a node’s permissions.

- **Returns** `1` on success, `0` on failure.

**Example**

```c
void example_fs_chmod(void) {
    fs_init();
    fs_node_t *f = fs_create_file(FS_ROOT, "run", PERMS_R);
    if (!f) return;

    (void)fs_chmod_node(f, PERMS_RWX);
}
```

---

### File I/O

#### `byte fs_write_file(fs_node_t *file, const void *data, word size);`

Write `size` bytes to a file node.

- **Returns** `1` on success, `0` on failure.
- Fails if `file` is a directory or lacks write permission.
- Replaces existing file contents.

**Example: write bytes**

```c
void example_fs_write_file(void) {
    fs_init();

    fs_node_t *f = fs_create_file(FS_ROOT, "bin", PERMS_RW);
    if (!f) return;

    byte payload[4] = { 1, 2, 3, 4 };
    (void)fs_write_file(f, payload, 4);
}
```

#### `word fs_read_file(fs_node_t *file, void *buffer, word buffer_size);`

Read file contents into `buffer`.

- **Returns**
  - Number of bytes copied (0 indicates failure or empty/uninitialised file).

**Example: read into a fixed buffer**

```c
void example_fs_read_file(void) {
    fs_init();

    fs_node_t *f = fs_create_file(FS_ROOT, "msg", PERMS_RW);
    if (!f) return;

    fs_write_file(f, "OK", 2);

    char buf[8];
    word n = fs_read_file(f, buf, sizeof(buf));
    (void)n;
}
```

---

### Convenience path APIs

#### `fs_node_t *fs_mkdir(fs_node_t *parent, const char *path, fs_perms_t perms);`

Create a full directory path relative to `parent`, creating missing directories.

- Accepts absolute paths (`/x/y`) and relative paths (`x/y`).
- Returns the final directory node, or `0` on failure.

**Example: ensure `/cfg/ui` exists**

```c
void example_fs_mkdir(void) {
    fs_init();
    (void)fs_mkdir(FS_ROOT, "/cfg/ui", PERMS_RWX);
}
```

#### `fs_node_t *fs_touch(fs_node_t *parent, const char *path, fs_perms_t perms);`

Create a file at `path`, creating missing directories along the way.

- Rejects paths ending with `/`.
- Returns the created node, or `0` on failure.

**Example: create `/cfg/ui/theme`**

```c
void example_fs_touch(void) {
    fs_init();

    fs_node_t *f = fs_touch(FS_ROOT, "/cfg/ui/theme", PERMS_RW);
    if (!f) return;

    fs_write_file(f, "dark", 4);
}
```

#### `fs_node_t *fs_lookup(fs_node_t *parent, const char *path);`

Lookup a file relative to `parent`.

- Returns a file node if it exists and is a file; otherwise returns `0`.

#### `fs_node_t *fs_dir_lookup(fs_node_t *parent, const char *path);`

Lookup a directory relative to `parent`.

- Returns a directory node if it exists and is a directory; otherwise returns `0`.

**Example: distinguish file vs directory**

```c
void example_fs_lookup(void) {
    fs_init();
    fs_mkdir(FS_ROOT, "/cfg", PERMS_RWX);
    fs_touch(FS_ROOT, "/cfg/a", PERMS_RW);

    fs_node_t *d = fs_dir_lookup(FS_ROOT, "/cfg");
    fs_node_t *f = fs_lookup(FS_ROOT, "/cfg/a");

    (void)d;
    (void)f;
}
```

#### `word fs_read(fs_node_t *parent, const char *path, void *buffer, word buffer_size);`

Convenience wrapper: resolve `path` and read file contents.

#### `byte fs_write(fs_node_t *parent, const char *path, const void *data, word size);`

Convenience wrapper: resolve `path` and write file contents.

**Example: write and read by path**

```c
void example_fs_read_write_by_path(void) {
    fs_init();
    fs_touch(FS_ROOT, "/log", PERMS_RW);

    fs_write(FS_ROOT, "/log", "x", 1);

    char b[2];
    (void)fs_read(FS_ROOT, "/log", b, sizeof(b));
}
```

## Time and RTC functions

### `void get_time_string(format_t format, char* out);`

Format the current RTC time into a string.

- **Parameters**
  - `format`: One of the `format_t` values (defined later in the header):
    - `TIME_FORMAT_24H`
    - `TIME_FORMAT_12H`
    - `TIME_FORMAT_24H_WITH_SECONDS`
    - `TIME_FORMAT_12H_WITH_SECONDS`
  - `out`: Output buffer.
- **Important**
  - The RTC registers are treated as BCD-encoded values when formatting. For consistent results, write BCD values to the RTC time registers.

**Example: show current time**

```c
void example_time_string(void) {
    char out[16];
    get_time_string(TIME_FORMAT_24H_WITH_SECONDS, out);

    tui_draw_text(5, 5, out, TUI_FONT_SIZE_6x8, 0, 0, 0, TUI_COLOUR_BLACK);
}
```

---

### RTC control

#### `void rtc_reset(void);`

Reset RTC registers to a default state.

#### `void rtc_enable(void);`

Enable RTC.

#### `void rtc_disable(void);`

Disable RTC.

**Example: reset and enable**

```c
void example_rtc_start(void) {
    rtc_reset();
    rtc_enable();
}
```

---

### RTC time setters/getters

#### `void rtc_set_time(byte hours, byte minutes, byte seconds);`
#### `void rtc_set_seconds(byte seconds);`
#### `void rtc_set_minutes(byte minutes);`
#### `void rtc_set_hours(byte hours);`

Write the time registers.

- The values are written directly to hardware registers; if the hardware expects BCD, provide BCD.

#### `void rtc_get_time(byte* hours, byte* minutes, byte* seconds);`
#### `byte rtc_get_seconds(void);`
#### `byte rtc_get_minutes(void);`
#### `byte rtc_get_hours(void);`

Read time registers.

**Example: set 12:34:56 (BCD) and read back**

```c
void example_rtc_set_get(void) {
    rtc_enable();

    /* BCD encoding: 0x12 = 12, 0x34 = 34, 0x56 = 56. */
    rtc_set_time(0x12, 0x34, 0x56);

    byte h, m, s;
    rtc_get_time(&h, &m, &s);
    (void)h; (void)m; (void)s;
}
```

---

### RTC date setters/getters

#### `void rtc_set_date(word year, byte month, byte day, byte week);`
#### `void rtc_set_day(byte day);`
#### `void rtc_set_month(byte month);`
#### `void rtc_set_year(word year);`

Set the date.

- The implementation stores year as an offset from 2026 in the year register.

#### `void rtc_get_date(word* year, byte* month, byte* day, byte* week);`
#### `byte rtc_get_day(void);`
#### `byte rtc_get_month(void);`
#### `word rtc_get_year(void);`

Get the date.

**Example: set and query date**

```c
void example_rtc_date(void) {
    rtc_set_date(2026, 2, 5, 4);

    word y;
    byte mo, d, w;
    rtc_get_date(&y, &mo, &d, &w);
    (void)y; (void)mo; (void)d; (void)w;
}
```

## Delay and tick conversion

### `void delay_ms(word ms);`

Delay for `ms` milliseconds.

- Uses Timer0 and a low-level sleep/stop sequence.

### `void delay_s(word s);`

Delay for `s` seconds.

**Example: simple animation pacing**

```c
void example_delay(void) {
    tui_clear_screen();
    tui_draw_text(10, 10, "Wait", TUI_FONT_SIZE_6x8, 0, 0, 0, TUI_COLOUR_BLACK);
    tui_render_buffer();

    delay_ms(250);
}
```

### Tick conversion helpers

#### `word ms_to_ticks(word ms);`
#### `word s_to_ticks(word s);`
#### `word ticks_to_ms(word ticks);`
#### `word ticks_to_s(word ticks);`

Convert between milliseconds/seconds and timer ticks using the constants:

- `TICKS_PER_MS` (8)
- `TICKS_PER_SECOND` (8000)

**Example**

```c
void example_ticks(void) {
    word t = ms_to_ticks(125);
    word ms = ticks_to_ms(t);
    (void)ms;
}
```

## Configuration macros

The following macros are declared within the documented range and configure memory layout and hardware access.

### Graphics

#### `#define VRAM 0x9000`

Base address for the software VRAM buffer.

#### `#define Write2RealScreen *((volatile __near byte *)(VRAM + 0xC02))`

Boolean-like flag controlling whether drawing targets the physical screen (`non-zero`) or the VRAM buffer (`0`).

**Example: select buffered rendering**

```c
void example_set_buffered(void) {
    Write2RealScreen = 0;
}
```

### Heap

#### `#define HEAP_START_ADDR (VRAM + 0xC04)`
#### `#define HEAP_MAX_SIZE 0x2000`
#### `#define HEAP_BLOCK_ALIGN 2`

Heap allocator configuration.

- `HEAP_START_ADDR`: where the heap begins.
- `HEAP_MAX_SIZE`: total heap bytes managed.
- `HEAP_BLOCK_ALIGN`: allocation alignment.

### Filesystem

#### `#define FS_START_ADDR (HEAP_START_ADDR + HEAP_MAX_SIZE)`
#### `#define FS_MAX_SIZE 0x3800`
#### `#define FS_NAME_MAX_LEN 12`
#### `#define FS_INVALID_IDX 0xFF`
#### `#define FS_NULL_OFFSET 0xFFFF`
#### `#define FS_MAX_NODES 255`

Filesystem configuration:

- Names are limited to `FS_NAME_MAX_LEN - 1` visible characters due to null termination.
- Index fields use `FS_INVALID_IDX` to indicate “none”.
- File data offsets use `FS_NULL_OFFSET` to indicate “no data”.

### Permission presets

The following macros construct `fs_perms_t` values:

- `PERMS_RWX`, `PERMS_RW`, `PERMS_R`, `PERMS_WX`, `PERMS_W`, `PERMS_X`, `PERMS_RX`, `PERMS_NONE`

**Example: create a read-only file**

```c
void example_perms(void) {
    fs_init();
    (void)fs_create_file(FS_ROOT, "ro", PERMS_R);
}
```

### Root node

#### `#define FS_ROOT (&FS_NODES[0])`

Pointer to the root filesystem node.

### RTC registers

These macros provide direct volatile access to RTC registers:

- `RTC_ENABLE`
- `RTC_SECONDS`, `RTC_MINUTES`, `RTC_HOURS`
- `RTC_DAY`, `RTC_WEEK`, `RTC_MONTH`, `RTC_YEAR`

**Example: read raw registers**

```c
byte example_read_seconds(void) {
    return RTC_SECONDS;
}
```

### Time constants

- `SECONDS_PER_MINUTE` (60)
- `MINUTES_PER_HOUR` (60)
- `HOURS_PER_DAY` (24)
- `DAYS_PER_WEEK` (7)
- `MONTHS_PER_YEAR` (12)

### Tick constants

- `TICKS_PER_MS` (8)
- `TICKS_PER_SECOND` (8000)

If you made it to the end, don't forget to like and subscribe, and hit that notification bell!
