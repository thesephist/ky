# Ky architecture


## Data structures

>I will, in fact, claim that the difference between a bad programmer and a good one is whether he considers his code or his data structures more important. Bad programmers worry about the code. Good programmers worry about data structures and their relationships. - The other Linus (Torvalds)

### Buffer

```go
type interface Buffer {
    Apply(Op)
    Revert(Op)
}

// zero value is init
type ArrayBuffer struct {
    Value string
}

// zero value is init
type RopeBuffer struct {
    Rope   Rope
}
```

A buffer holds some string data. A buffer is usually backed by a file on disk, but can optionally also have no backing file, or be backed by standard input. A buffer holds the rope data structure that represents the string contents.

We'll first build an MVP implementation of the buffer 

### Rope

```go
// zero value is init
type Rope struct {
    Left  Rope
    Right Rope
    Len   int
    Val   string
}

func (r Rope) Splice(int, string, int)
// Prune ensures that all subtrees of a Rope node have
// `Rope.Val` of length less than 40 charaters.bl
func (r Rope) Prune()
func (r Rope) Buffer() []byte
```

A [rope](https://en.wikipedia.org/wiki/Rope_(data_structure)) is a way to efficiently represent string data such that insertion-deletion operations are fast. Ky uses ropes to represent files being edited, so edits are fast.

In addition to the standard rope data structure, Ky uses heuristics around line breaks and delimiters to be smarter about calculating ropes for structured text formats like code and markdown.

### Model

```go
// zero value is init
type Model struct {
    Cursor int
    Buffer Buffer
    Source io.ReadWriter
}

func (m Model) SpliceAtCursor(string, int)
func (m Model) Splice(int, string, int)
func (m Model) MoveCursor(int) // int is character offset
```

A model represents an instance of a buffer being edited. Alongside a reference to the file buffer, the model holds cursor position and viewport data alongside editor state like current model and action contexts.

A model also holds the information about the backing store of a buffer. This means that a buffer has no intrinsic reference to a file or a location for where it should be saved. When a save or load operation occurs, the model takes the data that lives in a buffer and takes the appropriate action.

### Split

```go
// zero value is init
// if Left == nil || Right == nil, then Left, Right, and Model are all nil
type Split struct {
    IsVert bool
    Pct    int
    Left   Split
    Right  Split
    Model  Model
}

func (s Split) InsertLeft() Split
func (s Split) InsertRight() Split
func (s Split) InsertUp() Split
func (s Split) InsertDown() Split
func (s Split) Remove()
```

Ky represents its user interface as a tree of splits. Each split references one and only one model. A split contains information about how its children are laid out in the editor UI.

### Delta

```go
// zero value is init
type Delta []Op
```

A delta is a set of updates to a buffer that represents an atomic edit operation in the editor model.

### Op

```go
// zero value is init
type Op struct {
    Index  int
    Insert string
    Delete int
}
```

An Op is an atomic change dispatched from a model to a buffer, and is the only interface through which buffers and the corresponding ropes are modified.

Most user actions will cause changes and calls to functions in `ky.model`, and the model will then dispatch an Op with new bytes and positions (rope indices) to the buffer. At each edit, a Delta is constructed by collating a sequence of Ops. Ops are thus micro-edits that are later grouped into deltas. Instead of an insert or delete op, Ky just has a single Op: splice. An Op (splice) includes a location, where to splice, what to add, and how much to delete. The buffer than applies the splice across potentially one or more rope nodes.


## Embedded Ink runtime

Ky has a minimal editor core written in Go, with a set of [Ink](https://github.com/thesephist/ink) bindings for composing more complex actions in the Ink runtime (which I'll just call the "runtime" from here). The Ink runtime runs in `--isolate` mode, which restricts the runtime's interfaces to the rest of the operating system to only be available through Ky bindings.

### `ky.core`

Functions in `ky.core` operate against the entire running editor, and modify things like editor configuration and layout.

### `ky.buffer`

Functions in `ky.buffer` operates against the text buffer. Most editing operations to files are handled in `ky.buffer`.

### `ky.model`

Functions in `ky.model` operates against editor models. Operations like saving/loading files and updating viewports and cursor positions are handled in `ky.model`.

`ky.model` includes the following functions.

- `spliceAtCursor`: maps to `Buffer.SpliceAtCursor`
- `splice`: maps to `Buffer.Splice`
- `moveCursor`: maps to `Buffer.MoveCursor`


## User interface

Ky is built for a terminal user interface and operates in [raw mode](https://viewsourcecode.org/snaptoken/kilo/02.enteringRawMode.html).

### Modes

Ky is a modal editor, and operates in one of three modes at any given time. They're modeled after Vim editor modes.

- **View** mode (Escape to switch)
- **Edit** mode (e to switch)
- **Select** mode (s to switch)

- h, j, k, l to move by 1 unit up/down/left/right
- H, J, K, L to move by 4 units up/down/left/right
- C-u, C-d to move half-screen up or down
- `>` to enter Ink command line
