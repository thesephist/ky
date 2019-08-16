# Ky architecture


## Data structures

>I will, in fact, claim that the difference between a bad programmer and a good one is whether he considers his code or his data structures more important. Bad programmers worry about the code. Good programmers worry about data structures and their relationships. - The other Linus (Torvalds)

### Buffer (`KyBuffer`)

A buffer holds some string data. A buffer is usually backed by a file on disk, but can optionally also have no backing file, or be backed by standard input. A buffer holds the rope data structure that represents the string contents.

### Rope

A [rope](https://en.wikipedia.org/wiki/Rope_(data_structure)) is a way to efficiently represent string data such that insertion-deletion operations are fast. Ky uses ropes to represent files being edited, so edits are fast.

In addition to the standard rope data structure, Ky uses heuristics around line breaks and delimiters to be smarter about calculating ropes for structured text formats like code and markdown.

### Model (`KyModel`)

A model represents an instance of a buffer being edited. Alongside a reference to the file buffer, the model holds cursor position and viewport data alongside editor state like current model and action contexts.

A model also holds the information about the backing store of a buffer. This means that a buffer has no intrinsic reference to a file or a location for where it should be saved. When a save or load operation occurs, the model takes the data that lives in a buffer and takes the appropriate action.

### Split

Ky represents its user interface as a tree of splits. Each split references one and only one model. A split contains information about how its children are laid out in the editor UI.

### Delta

A delta is a set of updates to a buffer that represents an atomic edit operation in the editor model.

### Op

An Op is an atomic change dispatched from a model to a buffer, and is the only interface through which buffers and the corresponding ropes are modified.

Most user actions will cause changes and calls to functions in `ky.model`, and the model will then dispatch an Op with new bytes and positions (rope indices) to the buffer. At each edit, a Delta is constructed by collating a sequence of Ops. Ops are thus micro-edits that are later grouped into deltas.

Instead of an insert or delete op, Ky just has a single Op: splice. An Op (splice) includes a location, where to splice, what to add, and how much to delete. The buffer than applies the splice across potentially one or more rope nodes.


## Embedded Ink runtime

Ky has a minimal editor core written in Go, with a set of [Ink](https://github.com/thesephist/ink) bindings for composing more complex actions in the Ink runtime (which I'll just call the "runtime" from here). The Ink runtime runs in `--isolate` mode, which restricts the runtime's interfaces to the rest of the operating system to only be available through Ky bindings.

### `ky.core`

Functions in `ky.core` operate against the entire running editor, and modify things like editor configuration and layout.

### `ky.buffer`

Functions in `ky.buffer` operates against the text buffer. Most editing operations to files are handled in `ky.buffer`.

### `ky.model`

Functions in `ky.model` operates against editor models. Operations like saving/loading files and updating viewports and cursor positions are handled in `ky.model`.


## User interface

Ky is built for a terminal user interface and operates in [raw mode](https://viewsourcecode.org/snaptoken/kilo/02.enteringRawMode.html).

### Modes

Ky is a modal editor, and operates in one of three modes at any given time. They're modeled after Vim editor modes.

- **View** mode
- **Edit** mode
- **Select** mode

