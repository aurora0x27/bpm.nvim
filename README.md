# bpm.nvim

Buffer pool manager for Neovim — per‑tab buffer groups, configurable detach policies, unique name shortening, and session persistence.

## Features

- **Per‑tab buffer pools** – each tabpage owns a list of buffers, independent of other tabs.
- **Detach policies** – control what happens when a buffer leaves a tab: destroy windows, replace with another buffer, or show a blank placeholder.
- **Unique buffer naming** – a suffix trie resolves the shortest unambiguous path tail for each buffer, useful for tabline or statusline components.
- **Tab renaming** – assign arbitrary string labels to tabs. The name can be restored when nvim restarted.
- **Session persistence** – serialize the full buffer pool state to JSON and restore it after session load.

## Installation

Using [lazy.nvim](https://github.com/folke/lazy.nvim):

```lua
{
  "aurora0x27/bpm.nvim",
  config = function()
    require("bpm").setup()
  end,
}
```

With [packer.nvim](https://github.com/wbthomason/packer.nvim):

```lua
use {
  "aurora0x27/bpm.nvim",
  config = function()
    require("bpm").setup()
  end,
}
```

Calling `setup()` installs autocmds and user commands. No additional options are required.

## Commands

| Command         | Arguments                 | Description                                                          |
| --------------- | ------------------------- | -------------------------------------------------------------------- |
| `:BpmRenameTab` | `<name>`                  | Rename the current tab.                                              |
| `:BpmListTab`   | –                         | List buffers attached to the current tab with their shortened names. |
| `:BpmBufName`   | `[buffer_number]` (count) | Show the resolved short name of a buffer (default current buffer).   |
| `:BpmDetach`    | `[policy] [bufnr...]`     | Detach buffers from the current tab. Policy defaults to `replace`.   |
| `:BpmEvict`     | `[policy] [bufnr...]`     | Detach buffers from **all** tabs and delete them permanently.        |
| `:BpmDumpState` | –                         | Dump the current internal state for debugging.                       |

**Policies**:

- `replace` (default) – replace windows showing the buffer with another tab buffer, or an empty idle buffer.
- `destroy` – close the windows.
- `idle` – replace windows with a blank no‑file buffer.

Omit buffer numbers to act on the current buffer. Wildcards are not supported; list available buffers via `:BpmListTab` or `:ls`.

## Buffer name shortening

`bpm.nvim` builds a suffix trie from all listed buffer paths and picks the shortest suffix that is unique. For example:

```
~/projects/foo/init.lua
~/projects/bar/init.lua
~
```

may become `foo/init.lua` and `bar/init.lua`. The resolved name is updated automatically on `BufAdd`, `BufWipeout`, and `BufFilePost`.
Use `BpmBufName` or the API `M.resolve_bufname(bufnr)` to obtain the short name.

## Session persistence

You can serialize the buffer pool state with `M.to_json()` and restore it with `M.from_json(data)`.
The state includes tab names, buffer ordering, and file paths.

Example integration with a session manager, `persistence.nvim` for example:

```lua
vim.api.nvim_create_autocmd('User', {
  pattern = 'PersistenceLoadPost',
  callback = function()
    if type(vim.g.BufferPoolState) == 'string' then
      require 'bpm'.from_json(vim.g.BufferPoolState)
    end
  end,
})

vim.api.nvim_create_autocmd('User', {
  pattern = 'PersistenceSavePre',
  callback = function()
    vim.g.BufferPoolState = require 'bpm'.to_json()
  end,
})
```

**Note**: restoration relies on tab ordering; the Nth serialized tab is applied to the Nth runtime tabpage.

## API (public functions)

| Function                     | Description                                          |
| ---------------------------- | ---------------------------------------------------- |
| `M.get_attached_buf([tab])`  | Return list of buffer numbers attached to a tab.     |
| `M.resolve_bufname(bufnr)`   | Return the shortened name of a buffer.               |
| `M.resolve_tabname(tabid)`   | Return the user‑defined name or numeric id of a tab. |
| `M.rename_tab(tabid, name)`  | Assign a name to a tab and redraw the tabline.       |
| `M.detach(buf, tab, policy)` | Remove a buffer from a tab and handle its windows.   |
| `M.evict(buf, policy)`       | Detach from all tabs and delete the buffer.          |
| `M.to_json()`                | Dump current state as a JSON string.                 |
| `M.from_json(data)`          | Restore state from a JSON string.                    |

All public functions live under the module `"bpm"`.

## Debugging

`:BpmDumpState` prints the current tab → buffer mapping with resolved names and modification flags, useful for verifying the internal state.

Here’s a suggested outline for a README that includes the design philosophy. Each section describes what to cover; you can expand it into the final document.

## Design Philosophy

> This plugin is weird, why it looks like that?

- **The editor as an ephemeral workspace instance**
  Neovim is treated as a short‑lived process that “grows” from a workspace (project directory + environment).
  Each launch is like loading a saved memory image – it restores the working set and continues where you left off.
  BPM embraces this by making session persistence a first‑class “core dump” of the buffer pool state.

- **Tabs are task threads, not just viewports**
  A tabpage represents an ordered thread of work, holding a stable list of buffers.
  Moving away from a tab hides its buffers from the global list;
  returning re‑shows them. This keeps buffer lists manageable and mirrors how you switch between tasks.

- **Detach, don’t just close**
  Removing a buffer from a tab is a deliberate act.
  BPM offers detach policies to control what happens to windows – destroy, replace with another buffer, or show a neutral idle buffer.
  This prevents orphaned windows and preserves workspace integrity.

- **Suffix trie naming = minimal context with maximum clarity**
  Instead of full paths or fragile tabpage labels, buffers get the shortest suffix that is unique across all listed buffers.
  It’s a deterministic, stateless way to display meaningful names in your tabline or statusline.

- **Session is a core dump, not a rigid snapshot**
  Serialisation captures only the listed, named buffers and tab order – not every split or temporary file.
  The goal is to revive the essential working set while discarding ephemeral noise.
  Tab order is ordinal, matching the Nth serialised tab to the Nth runtime tab after restore.
