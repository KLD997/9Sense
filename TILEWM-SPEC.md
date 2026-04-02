# tilewm — A Tiling Window Manager for 9front

## Project Overview

Build a tiling window manager for 9front (Plan 9 fork) that replaces Rio. It must function as a **9P file server** that multiplexes `/dev/draw`, `/dev/mouse`, and `/dev/cons` to client windows, because on Plan 9 the window manager *is* the windowing system. No X11, no Wayland — windows are 9P file trees.

The layout engine is a binary-tree tiler inspired by bspwm/i3. Chrome is minimal: thin borders with focus color, optional FluxBox-style tab bar per frame. Written in C, targeting the Plan 9 C compiler (`8c`/`6c`).

## Critical Constraint

**Fork Rio (`/sys/src/cmd/rio/`) as the starting point.** Rio's 9P multiplexer (`fsys.c`, `xfid.c`) is the hardest part and already works. Gut the floating layout logic; keep the file server plumbing. Do not rewrite the 9P layer from scratch.

---

## Architecture

Three layers:

### 1. 9P File Server (keep from Rio)

Serve `/dev/wsys/N/` per window. Each window directory exposes:

| File       | Purpose                                  |
|------------|------------------------------------------|
| `cons`     | Read/write text I/O                      |
| `consctl`  | `rawon` / `rawoff`                       |
| `mouse`    | Mouse events, coords local to window     |
| `draw`     | Multiplexed draw device                  |
| `winname`  | Window label (read)                      |
| `wctl`     | Control messages: hide, unhide, delete, current |
| `label`    | Window title (read/write)                |

Source files to preserve and adapt: `fsys.c`, `xfid.c`, `wctl.c`.

Mouse coordinates delivered to clients must be translated into window-local space:
```c
m.xy = subpt(m.xy, win->img->r.min);
```

### 2. Layout Engine (new code)

Binary tree of splits. Every internal node is a vertical or horizontal split; every leaf holds a window.

#### Data Structures

```c
enum { Vsplit, Hsplit, Leaf };

typedef struct Tile Tile;
struct Tile {
    int       type;     /* Vsplit, Hsplit, Leaf */
    Rectangle r;        /* allocated screen region */
    double    ratio;    /* split position, 0.0–1.0 */
    Tile     *c[2];     /* children: left/top [0], right/bottom [1] */
    Tile     *parent;
    Window   *win;      /* non-nil only for Leaf */
};

/* Per-workspace */
typedef struct Workspace Workspace;
struct Workspace {
    int    id;
    Tile  *root;
    char   name[32];
};

#define NWORKSPACE 9
Workspace workspaces[NWORKSPACE];
int       curws;  /* index of active workspace */
```

#### Core Functions to Implement

```
tile(Tile *t)
```
Recursively compute `t->r` for all descendants. For `Vsplit`, split `Dx(t->r)` by `t->ratio`; for `Hsplit`, split `Dy(t->r)`. On `Leaf`, call `wresize(t->win, t->r)` to push the new geometry to the window's backing image and 9P clients.

```
split(Tile *leaf, int type)
```
Replace a leaf with an internal node. The old leaf becomes `c[0]`; a new empty leaf becomes `c[1]`. Default `ratio = 0.5`. Retile from the new internal node.

```
unsplit(Tile *leaf)
```
Remove a leaf (window closed). Its sibling replaces the parent node. Retile.

```
neighbor(Tile *from, int dir)
```
Walk the tree to find the adjacent leaf in direction `dir` (Up, Down, Left, Right). Used for focus movement and window swapping.

```
swap(Tile *a, Tile *b)
```
Exchange the `Window*` pointers of two leaves. Retile both.

```
resize_ratio(Tile *node, double delta)
```
Adjust `node->ratio` by `delta`, clamped to `[0.1, 0.9]`. Retile subtree.

#### Tiling Algorithm

```c
void
tile(Tile *t)
{
    Rectangle r0, r1;
    int gap = 2; /* border gap in pixels */

    if(t->type == Leaf){
        if(t->win != nil)
            wresize(t->win, insetrect(t->r, gap));
        return;
    }

    r0 = t->r;
    r1 = t->r;

    if(t->type == Vsplit){
        r0.max.x = t->r.min.x + (int)(Dx(t->r) * t->ratio);
        r1.min.x = r0.max.x;
    } else { /* Hsplit */
        r0.max.y = t->r.min.y + (int)(Dy(t->r) * t->ratio);
        r1.min.y = r0.max.y;
    }

    t->c[0]->r = r0;
    t->c[1]->r = r1;
    tile(t->c[0]);
    tile(t->c[1]);
}
```

### 3. Input Handling & Chrome (new code)

#### Keybindings

Read keyboard via `kbdproc` (or `/dev/kbdraw`). Use `Mod4` (windows key) as the prefix. Handle in a dedicated proc:

| Binding            | Action                              |
|--------------------|-------------------------------------|
| `Mod4+Return`      | Spawn `rc` in new window            |
| `Mod4+v`           | Split focused window vertically     |
| `Mod4+h`           | Split focused window horizontally   |
| `Mod4+w`           | Close focused window                |
| `Mod4+[hjkl]`      | Move focus (left/down/up/right)     |
| `Mod4+Shift+[hjkl]`| Swap window in direction            |
| `Mod4+[1-9]`       | Switch to workspace N               |
| `Mod4+Shift+[1-9]` | Move focused window to workspace N  |
| `Mod4+[=/-]`       | Grow/shrink split ratio by 0.05     |
| `Mod4+f`           | Toggle focused window fullscreen (monocle) |
| `Mod4+Shift+q`     | Exit WM                             |

#### Border Drawing

After each retile, draw borders on the screen image:

```c
void
drawborders(Tile *t)
{
    if(t->type == Leaf){
        Image *col;
        if(t->win == focused)
            col = focuscol;    /* e.g., 0x4488CCFF */
        else
            col = unfocuscol;  /* e.g., 0x333333FF */
        border(screen, t->r, 2, col, ZP);
        return;
    }
    drawborders(t->c[0]);
    drawborders(t->c[1]);
}
```

#### Optional: Tab Bar

For multi-window frames (future): a small bar at the top of each leaf showing the window label, drawn via `string()` from `libdraw`. Click to select tab. Not required for v1.

---

## File Layout

```
/sys/src/cmd/tilewm/
    mkfile          # Plan 9 mkfile
    main.c          # Entry, screen init, event loop
    tile.c          # Layout engine (tree ops, tiling)
    fsys.c          # 9P file server (adapted from rio)
    xfid.c          # Fid handling (adapted from rio)
    wind.c          # Window struct, create/destroy/resize
    input.c         # Keyboard/mouse dispatch, keybind table
    workspace.c     # Workspace management
    border.c        # Chrome drawing
    wctl.c          # wctl message parsing (adapted from rio)
    dat.h           # Shared data structures
    fns.h           # Function prototypes
```

### mkfile

```mk
</$objtype/mkfile

BIN=/$objtype/bin
TARG=tilewm

OFILES=\
    main.$O\
    tile.$O\
    fsys.$O\
    xfid.$O\
    wind.$O\
    input.$O\
    workspace.$O\
    border.$O\
    wctl.$O\

HFILES=dat.h fns.h

</sys/src/cmd/mkone
```

---

## Implementation Order

### Phase 1: Fork and Strip Rio
1. Copy `/sys/src/cmd/rio/` to `/sys/src/cmd/tilewm/`.
2. Rename the binary from `rio` to `tilewm` in `mkfile`.
3. Rip out the floating/overlapping window logic in `wind.c` and `rio.c`. Remove the hidden menu, sweep/resize with mouse.
4. Verify it still compiles and starts (even if windows don't render correctly yet).

### Phase 2: Binary Tree Layout
1. Implement `Tile` tree in `tile.c` with `tile()`, `split()`, `unsplit()`.
2. On `Window` create, find the focused leaf, call `split()` to subdivide it, place the new window in the new leaf.
3. On `Window` destroy, call `unsplit()`.
4. After any tree mutation, call `tile(root)` then `wresize()` each leaf's window to its computed rectangle.

### Phase 3: Input
1. Implement keybind dispatch in `input.c`. Read keyboard events, match `Mod4+key` combos, call tree operations.
2. Implement `neighbor()` for directional focus.
3. Implement `swap()` for window swapping.

### Phase 4: Chrome
1. Draw 2px borders after each retile.
2. Highlight focused window border in a distinct color.

### Phase 5: Workspaces
1. Array of `Workspace` structs, each with its own `Tile *root`.
2. Switching workspaces: unmap all windows in current tree, map all in target tree, retile.

### Phase 6: Polish
1. Ratio resize keybindings.
2. Monocle/fullscreen toggle.
3. Optional tab bar per leaf.
4. Config file for colors, gap size, keybinds (parse from `/usr/$user/lib/tilewm/config`).

---

## Key Rio Files to Study and Adapt

| Rio File   | What to Keep                          | What to Gut                  |
|------------|---------------------------------------|------------------------------|
| `fsys.c`   | All 9P handling                       | —                            |
| `xfid.c`   | Fid open/read/write/clunk for wsys    | —                            |
| `wind.c`   | `Window` struct, `wresize`, `wcreate` | Floating position logic      |
| `rio.c`    | Screen init, main event loop          | Button3 menu, sweep, drag    |
| `wctl.c`   | `wctl` message parsing                | `move` / `resize` commands   |
| `data.h`   | `Window`, `Fid` structs              | —                            |

---

## Style Guidelines

- Plan 9 C style: tabs for indentation, K&R braces, short names.
- No `malloc` where Plan 9 pool allocators exist.
- Use `Channel` and `proc` (not `pthread`) for concurrency.
- `emallocz()` / `erealloc()` for allocation with error handling.
- All coordinates use `Rectangle` and `Point` from `libdraw`.

---

## Testing

1. Boot 9front in a VM (QEMU with `-vga virtio`).
2. Build: `cd /sys/src/cmd/tilewm && mk install`
3. Start: `tilewm` (from drawterm or console).
4. Verify: open multiple `rc` shells, `acme`, `sam` — all must run unmodified.
5. Test splits, focus movement, workspace switching, window close.
6. Verify mouse coordinates are correct (test with `stats -lm` or a drawing program).

---

## References

- `/sys/src/cmd/rio/` — the existing WM source
- `rio(1)`, `draw(3)`, `mouse(3)`, `keyboard(3)`, `9p(2)` — man pages
- `/sys/include/draw.h`, `/sys/include/mouse.h` — data structure definitions
- `drawterm` source — how draw multiplexing works from the client side
- http://man.cat-v.org/9front/ — 9front man pages online
