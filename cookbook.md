# Cookbook.md

## Index

- [Main concepts](#main-concepts)
- [Component fields](#component-fields)
- [Builtin conditions and utilities](#builtin-conditions-and-utilities)
- [Recipes](#recipes)
  - [Getting started](#getting-started)
  - [Colors, colors, more colors!](#colors-colors-more-colors)
  - [Crash course: the ViMode](#crash-course-the-vimode)
  - [Crash course part II: FileName and friends](#crash-course-part-ii-filename-and-friends)
  - [FileType, FileSize, FileEncoding and FileFormat](#filetype-filesize-fileencoding-and-fileformat)
  - [Ruler](#ruler)
  - [FileSize](#filesize)
  - [LSP](#lsp)
  - [Diagnostics](#diagnostics)
  - [Git](#git)
  - [Debugger](#debugger)
  - [Tests](#tests)
  - [Working Directory](#working-directory)
  - [Terminal Name](#terminal-name)
  - [Helpfile Name](#helpfil-name)
  - [Snippets Indicator](#snippets-indicator)
  - [Conditional Statuslines](#conditional-statuslines)
  - [Theming](#theming)
- [Putting it all together](#putting-it-all-together)

## Main concepts

In heirline, everything is a [`StatusLine`](lua/heirline/statusline.lua#L31)
object. There is no distinction in the way one defines the final statusline
from any of its components.

You don't need to explicitly create a `StatusLine` object, the `setup` function
will handle that. What you should do, is to create a lua table that will serve as
a blueprint for the creation of such objects.

That's it, your statusline(s) are just some nested tables.

The nested tables will be referred to as `components`. Components may contain
other components, each of which may contain others. A component within another
component is called a `child`, and will inherit the fields of its `parent`.
There is no limit in how many components can be nested into each other.

```lua
local statusline = {
{...}, {...}, {..., {...}, {...}, {..., {...}, {..., {...}}}}
}
require'heirline'.setup(statusline)
```

Writing nested tables can be tiresome, so the best approach is to define simple
components and then assemble them. For example:

```lua
local Component1 = { ... }

local Sub1 = { ... }

local Component2 = { ... }

local statusline = {
    ...,
    {Component1, Sub1},
    Component2,
}

require'heirline'.setup(statusline)
```

## Component fields

So, what should be the content of a component table? Well it's fairly simple,
don't let the detailed description discourage you! Just keep one thing in mind:
whenever you see a function, know that the function is executed in the context
of the buffer and window the statusline belongs to. (The indices of the actual
buffer and window you're in are stored in the default vim global variables
`vim.g.actual_curbuf` and `vim.g.acutal_curwin`.)

Each component may contain _any_ of the following fields:

> Note that all functions described below are actual ***methods*** of the component
> itself, which can be accessed via the `self` parameter. Because of inheritance,
> children will look for unknown attributes within their own parent fields.

**Basic fields**:

- `provider`:
  - Type: `string` or `function(self) -> string|nil`
  - Description: This is the string that gets printed in the statusline. No
    escaping is performed, so it may contain sequences that have a special
    meaning within the statusline, such as `%f` (filename), `%p` (percentage
    through file), `%-05.10(` `%)` (to control text alignment and padding),
    etc. For more, see `:h 'statusline'`. To print an actual `%`, use `%%`.
- `hl`:
  - Type: `table` or `function(self) -> table`. The table may contain any of:
    - `fg`: The foreground color. Type: `string` to hex color code or vim
      builtin color name (eg.: `"#FFFFFF"`, `"red"`).
    - `bg`: The background color. Type: as above.
    - `guisp`: The underline/undercurl color, if any. Type: as above.
    - `style`: Any of the supported comma-separated highlight styles: `italic`,
      `bold`, `underline`, `undercurl`, `reverse`, `nocombine` or `none`. (eg.:
      `"bold,italic"`)
    - `force`: Control whether the parent's `hl` fields will override child's hl.
    Type: `bool`.
  - Description: `hl` controls the colors of what is printed by the component's
    `provider`, or by any of its descendants. At evaluation time, the `hl` of
    any component gets merged with the `hl` of its parent (whether it is a
    function or table), so that, when specified, the fields in the child `hl`
    will always take precedence unless `force` is `true`.
- `condition`:
  - Type: `function(self) -> any`
  - Description: This function controls whether the component should be
    evaluated or not. It is the first function to be executed at evaluation
    time. The truth of the return value is tested, so any value besides `nil`
    and `false` will evaluate to true. Of course, this will affect all of the
    component's progeny.
- `{...}`:
  - Type: `list`
  - Description: The component progeny. Each item of the list is a component
    itself and may contain any of the basic and advanced fields.

**Advanced fields**

- `stop_at_first`:
  - Type: `bool`
  - Description: If a component has any child, their evaluation will stop at
    the first child in the succession line who does not return an empty string.
    This field is not inherited by the component's progeny. Use this in
    combination with children conditions to create buffer-specific statuslines!
    (Or do whatever you can think of!)
- `init`:
  - Type: `function(self) -> any`
  - Description: This function is called whenever a component is evaluated
    (right after `condition` but before `hl` and `provider`), and can be used
    to modify the state of the component itself via the `self` parameter. For
    example, you can compute some values that will be accessed from other
    functions within the component genealogy (even "global" statusline
    variables).
- `static`:
  - Type: `table`
  - Description: This is a container for static variables, that is, variables
    that are computed only once at component definition. This is useful to
    organize data that should be shared among children, like icons or
    dictionaries. Any keyword defined within this table can be accessed by the
    component and its children methods as a direct attribute using the `self`
    parameter. (eg: `static = { x = ... }` can be accessed as `self.x`
    somewhere else).
- `restrict`:
  - Type: `table[keyword = bool]`
  - Description: Set-like table to control which component fields can be
    inherited by the component's progeny. The supplied table gets merged with
    the defaults. By default, the following fields are private to the
    component: `stop_at_first`, `init`, `provider`, `condition` and `restrict`.
    Attention: modifying the defaults could dramatically affect the behavior of
    the component! (eg: `restrict = { my_private_var = true, provider = false
    }`)

**The StatusLine life cycle**

There are two distinct phases in the life of a StatusLine object component: its
_creation_ (instantiation) and its _evaluation_. When creating the "blueprint"
tables, the user instructs the actual constructor on the attributes and methods
of the component. The fields `static` and `restrict` will have a meaning only
during the instantiation phase, while `condition`, `init`, `hl`, `provider` and
`stop_at_first` are evaluated (in this order) every time the statusline is
refreshed.

Confused yet? Don't worry, everything will come together in the [Recipes](#recipes) examples.

## Builtin conditions and utilities

While heirline does not provide any default component, it defines a few useful
test and utility functions to aid in writing components and their conditions.
These functions are accessible via `require'heirline.conditions'` and
`require'heirline.utils'`

**Built-in conditions**:

- `is_active()`: returns true if the statusline's window is the active window.
- `buffer_matches(patterns)`: Returns true whenever a buffer attribute
  (`filetype`,`buftype` or `bufname`) matches any of the lua patterns in the
  corresponding list.
  - `patterns`: table of the form `{filetype = {...}, buftype = {...}, bufname = {...}}` where each field is a list of lua patterns.
- `width_percent_below(N, threshold)`: returns true if `(N / current_window_width) <= threshold`
  (eg.: `width_percent_below(#mystring, 0.33)`)
- `is_git_repo()`: returns true if the file is within a git repo (uses [gitsigns]())
- `has_diagnostics()`: returns true if there is any diagnostic for the buffer.
- `lsp_attached():` returns true if an LSP is attached to the buffer.

**Utility functions**:

- `get_highlight(hl_name)`: returns a table of the attributes of the provided
  highlight name. The returned table contains the same `hl` fields described
  above.
- `clone(component[, with])`: returns a new component which is a copy of the
  supplied one, updated with the fields in the optional `with` table.
- `surround(delimiters, color, component)`: returns a new component, which
  contains a copy of the supplied one, surrounded by the left and right
  delimiters supplied by the `delimiters` table. This action will override the
  `bg` field of the component `hl`.
  - `delimiters`: table of the form `{left_delimiter, right_delimiter}`.
    Because they are actually just providers, delimiters could also be
    functions!
  - `color`: `nil` or `string` of color hex code or builtin color name. This
    color will be the foreground color of the delimiters and the background
    color of the component.
  - `component`: the component to be surrounded.

## Recipes

### Getting started

Ideally, the following code snippets should go within a configuration file, say
`~/.config/nvim/lua/plugins/heirline.lua`, that can be required in your
`init.lua` (or from packer `config`) using `require'plugins.heirline'`.

Your configuration file will start like this:

```lua
local conditions = require("heirline.conditions")
local utils = require("heirline.utils")
```

### Colors, colors, more colors

You will probably want to define some colors. This is not required, you don't
even have to use them if you don't like it, but let's say you like colors.

Colors can be specified directly in components, but it is probably more
convenient to organize them in some kind of table. If you want your statusline
to blend nicely with your colorscheme, the utility function `get_highlight()`
is your friend. To create themes and have your colors updated on-demand, see
[Theming](#theming).

```lua
local colors = {
    red = utils.get_highlight("DiagnosticError").fg,
    green = utils.get_highlight("String").fg,
    blue = utils.get_highlight("Function").fg,
    gray = utils.get_highlight("NonText").fg,
    orange = utils.get_highlight("DiagnosticWarn").fg,
    purple = utils.get_highlight("Statement").fg,
    cyan = utils.get_highlight("Special").fg,
    diag = {
        warn = utils.get_highlight("DiagnosticWarn").fg,
        error = utils.get_highlight("DiagnosticError").fg,
        hint = utils.get_highlight("DiagnosticHint").fg,
        info = utils.get_highlight("DiagnosticInfo").fg,
    },
    git = {
        del = utils.get_highlight("diffDeleted").fg,
        add = utils.get_highlight("diffAdded").fg,
        change = utils.get_highlight("diffChanged").fg,
    },
}
```

Perhaps, your favourite colorscheme already provides a way to get the theme colors.

```lua
local colors = require'kanagawa.colors'.setup() -- wink
```

### Crash course: the ViMode

No statusline is worth its weight in _fancyness_ :star2: without an appropriate
mode indicator. So let's cook ours! Also, this snippet will introduce you to a
lot of heirline advanced capabilities.

```lua
local ViMode = {
    -- get vim current mode, this information will be required by the provider
    -- and the highlight functions, so we compute it only once per component
    -- evaluation and store it as a component attribute
    init = function(self)
        self.mode = vim.fn.mode(1) -- :h mode()
    end,
    -- Now we define some dictionaries to map the output of mode() to the
    -- corresponding string and color. We can put these into `static` to compute
    -- them at initialisation time.
    static = {
        mode_names = { -- change the strings if yow like it vvvvverbose!
            n = "N", no = "N?", nov = "N?", noV = "N?", ["no"] = "N?", niI =
            "Ni", niR = "Nr", niV = "Nv", nt = "Nt", v = "V", vs = "Vs", V =
            "V_", Vs = "Vs", [""] = "^V", ["s"] = "^V", s = "S", S = "S_",
            [""] = "^S", i = "I", ic = "Ic", ix = "Ix", R = "R", Rc = "Rc",
            Rx = "Rx", Rv = "Rv", Rvc = "Rv", Rvx = "Rv", c = "C", cv = "Ex", r
            = "...", rm = "M", ["r?"] = "?", ["!"] = "!", t = "T",
        },
        mode_colors = {
            n = colors.red ,
            i = colors.green,
            v = colors.cyan,
            V =  colors.cyan,
            [""] =  colors.cyan, -- this is an actual ^V, type <C-v><C-v> in insert mode
            c =  colors.orange,
            s =  colors.purple,
            S =  colors.purple,
            [""] =  colors.purple, -- this is an actual ^S, type <C-v><C-s> in insert mode
            R =  colors.orange,
            r =  colors.orange,
            ["!"] =  colors.red,
            t =  colors.red,
        }
    },
    -- We can now access the value of mode() that, by now, would have been
    -- computed by `init()` and use it to index our strings dictionary.
    -- note how `static` fields become just regular attributes once the
    -- component is instantiated.
    -- To be extra meticulous, we can also add some vim statusline syntax to
    -- control the padding and make sure our string is always at least 2
    -- characters long
    provider = function(self)
        return "%-2("..self.mode_names[self.mode].."%)"
    end,
    -- Same goes for the highlight. Now the foreground will change according to the current mode.
    hl = function(self)
        local mode = self.mode:sub(1, 1) -- get only the first mode character
        return { fg = self.mode_colors[mode], style = "bold", }
    end,
}
```

### Crash course part II: FileName and friends

Perhaps one of the most important components is the one that shows which file
you are editing. In this second recipe, we will revisit some heirline concepts
and explore new ways to assemble components. We will also learn a few useful
vim lua API functions. Because we are all crazy about icons, we'll require
[nvim-web-devicons](), but you are absolutely free to omit that if you're not
an icon person.

```lua

local FileNameBlock = {
    -- let's first set up some attributes needed by this component and it's children
    init = function(self)
        self.filename = vim.api.nvim_buf_get_name(0) 
    end,
}
-- We can now define some children separately and assemble them later

local FileIcon = {
    init = function(self) 
        local filename = self.filename
        local extension = vim.fn.fnamemodify(filename, ":e")
        self.icon, self.icon_color = require("nvim-web-devicons").get_icon_color(filename, extension, { default = true })
    end,
    provider = function(self)
        return self.icon and (self.icon .. " ")
    end,
    hl = function(self)
        return { fg = self.icon_color }
    end
}

local FileName = {
    provider = function(self)
        -- first, trim the pattern relative to the current directory. For other
        -- options, see :h filename-modifers
        local filename = vim.fn.fnamemodify(self.filename, ":.")
        -- now, if the filename would occupy more than 1/4th of the available
        -- space, we trim the file path to its initials
        if not conditions.width_percent_below(#filename, 0.25) then
            filename = vim.fn.pathshorten(filename)
        end
    end,
    hl = { fg = utils.get_highlight("Directory").fg },
}

local FileFlags = {
    {
        provider = function() if vim.bo.modified then return "[+]" end,
        hl = { fg = colors.green }
                
    }, {
        provider = function() if (not vim.bo.modifiable) or vim.bo.readonly then return "" end,
        hl = { fg = colors.orange }
    }
}

-- Now, let's say that we want the filename color to change if the buffer is
-- modified. Of course, we could do that directly using the FileName.hl field,
-- but we'll see how easy it is to alter existing components using a "modifier"
-- component

local FileNameModifer = {
    hl = function()
        if vim.bo.modified then
            -- use `force` because we need to override the child's hl foreground
            return { fg = colors.cyan, style = 'bold', force=true }
        end
    end,
}

-- let's add the children to our FileNameBlock component
FileNameBlock[1] = FileIcon
FileNameBlock[2] = utils.append(FileNameModifer, FileName) -- a new table where FileName is a child of FileNameModifier
FileNameBlock[3] = FileFlags
FileNameBlock[4] = { provider = '%<'} -- this means that the statusline is cut
                                      -- here when there's not enough space

```

## FileType, FileSize, FileEncoding and FileFormat

```lua
local FileType = {
    provider = function()
        return string.upper(vim.bo.filetype)
    end,
    hl = { fg = utils.get_highlight("Type").fg },
}
```

### Ruler

```lua
local Ruler = {
    provider = "%(%l/%3L%):%2c",
}
```

### LSP

```lua

local LSPActive = {
    condition = conditions.lsp_attached,
    provider = " [LSP]",
    hl = { fg = colors.green, style = "bold" },
}

local LSPMessages = {
    provider = require("lsp-status").status,
    hl = { fg = colors.gray },
}

local Gps = {
    condition = require("nvim-gps").is_available,
    provider = require("nvim-gps").get_location,
    hl = { fg = colors.gray },
}
```

### Diagnostics

```lua
local Diagnostics = {

    condition = conditions.has_diagnostics,

    static = {
        error_icon = vim.fn.sign_getdefined("DiagnosticSignError")[1].text,
        warn_icon = vim.fn.sign_getdefined("DiagnosticSignWarn")[1].text,
        info_icon = vim.fn.sign_getdefined("DiagnosticSignInfo")[1].text,
        hint_icon = vim.fn.sign_getdefined("DiagnosticSignHint")[1].text,
    },

    {
        provider = function(self)
            local count = #vim.diagnostic.get(0, { severity = vim.diagnostic.severity.ERROR })
            return count > 0 and (self.error_icon .. count .. " ")
        end,
        hl = { fg = colors.diag.error },
    },
    {
        provider = function(self)
            local count = #vim.diagnostic.get(0, { severity = vim.diagnostic.severity.WARN })
            return count > 0 and (self.warn_icon .. count .. " ")
        end,
        hl = { fg = colors.diag.warn },
    },
    {
        provider = function(self)
            local count = #vim.diagnostic.get(0, { severity = vim.diagnostic.severity.INFO })
            return count > 0 and (self.info_icon .. count .. " ")
        end,
        hl = { fg = colors.diag.info },
    },
    {
        provider = function(self)
            local count = #vim.diagnostic.get(0, { severity = vim.diagnostic.severity.HINT })
            return count > 0 and (self.hint_icon .. count)
        end,
        hl = { fg = colors.diag.hint },
    },
}
```

### Git

```lua
local Git = {
    condition = conditions.is_git_repo,

    init = function(self)
        self.dict = vim.b.gitsigns_status_dict
    end,

    hl = { fg = utils.get_highlight("WarningMsg").fg },

    {
        provider = function(self)
            return " " .. self.dict.head
        end,
    },
    {
        provider = ')'
    },
    {
        provider = function(self)
            local count = self.dict.added or 0
            return count > 0 and ("+" .. count)
        end,
        hl = { fg = colors.git.add },
    },
    {
        provider = function(self)
            local count = self.dict.removed or 0
            return count > 0 and ("-" .. count)
        end,
        hl = { fg = colors.git.del },
    },
    {
        provider = function(self)
            local count = self.dict.changed or 0
            return count > 0 and ("~" .. count)
        end,
        hl = { fg = colors.git.change },
    },
    {
        provider = ")",
    },
}
```

### Debugger

```lua
local DAPMessages = {
    condition = function()
        local session = require("dap").session()
        if session then
            local filename = vim.api.nvim_buf_get_name(0)
            if session.config then
                local progname = session.config.program
                return filename == progname
            end
        end
    end,
    provider = function()
        return " " .. require("dap").status()
    end,
    hl = { fg = "red" },
}
```

### Tests

```lua
local UltTest = {
    condition = function()
        return vim.api.nvim_call_function("ultest#is_test_file", {}) ~= 0
    end,
    static = {
        passed_icon = vim.fn.sign_getdefined("test_pass")[1].text,
        failed_icon = vim.fn.sign_getdefined("test_fail")[1].text,
        passed_hl = { fg = utils.get_highlight("UltestPass").fg },
        failed_hl = { fg = utils.get_highlight("UltestFail").fg },
    },
    init = function(self)
        self.status = vim.api.nvim_call_function("ultest#status", {})
    end,
    {
        provider = function(self)
            return self.passed_icon .. self.status.passed .. " "
        end,
        hl = function(self)
            return self.passed_hl
        end,
    },
    {
        provider = function(self)
            return self.failed_icon .. self.status.failed .. " "
        end,
        hl = function(self)
            return self.failed_hl
        end,
    },
    {
        provider = function(self)
            return "of " .. self.status.tests - 1
        end,
    },
}
```

### Working Directory

```lua
local WorkDir = {
    provider = function()
        local icon = (vim.fn.haslocaldir(0) == 1 and "l" or "g") .. " " .. " "
        local cwd = vim.fn.getcwd(0)
        cwd = vim.fn.fnamemodify(cwd, ":~")
        cwd = vim.fn.pathshorten(cwd)
        return icon .. cwd .. "/"
    end,
    hl = { fg = colors.blue },
}
```

### Terminal Name

```lua
local TerminalName = {
    provider = function()
        local tname, _ = vim.api.nvim_buf_get_name(0):gsub(".*:", "")
        return " " .. tname
    end,
    hl = { fg = colors.blue, style = "bold" },
}
```

### Helpfile Name

```lua
local HelpFilename = {
    condition = function()
        return vim.bo.filetype == "help"
    end,
    provider = function()
        local filename = vim.api.nvim_buf_get_name(0)
        return vim.fn.fnamemodify(filename, ":t")
    end,
    hl = { fg = colors.blue },
}
```

### Snippets Indicator

```lua
local Snippets = {
    provider = function()
        local fwd = ""
        local bwd = ""
        if vim.fn["UltiSnips#CanJumpForwards"]() == 1 then fwd = "" end
        if vim.fn["UltiSnips#CanJumpBackwards"]() == 1 then bwd = " " end
        return vim.tbl_contains({ "s", "i" }, vim.fn.mode()) and (bwd .. fwd) or ""
    end,
    hl = { fg = "red", syle = "bold" },
}
```

## Putting it all together

```lua

local DefaultStl = {
    -- hl = { bg = "blue" },
    utils.surround({ "", "" }, utils.get_highlight("NonText").fg, { ViMode, Snippets })
    {provider = ' '},
    FileName,
    {provider = ' '},
    {provider = '%<'},
    Git,
    {provider = ' '},
    Diagnostics,
    {provider = '%='},

    Gps,
    {provider = ' '},
    DAPMessages,
    {provider = '%='},

    LSPActive,
    {provider = ' '},
    LSPMessages,
    {provider = ' '},
    UltTest,
    {provider = ' '},
    FileType,
    {provider = ' '},
    Ruler,
}

local SpecialStl = {
    condition = function()
        return conditions.buffer_matches({
            buftype = { "nofile", "help", "quickfix" },
            filetype = { "^git.*", "fugitive" },
        })
    end,
    FileType,
    {provider = ' '},
    HelpFilename,
}

local TerminalStl = {
    condition = function()
        return conditions.buffer_matches({ buftype = { "terminal" } })
    end,
    hl = { bg = utils.get_highlight("DiffDelete").bg },
    {
        condition = conditions.is_active,
        ViMode,
    },
    {provider = ' '},
    FileType,
    {provider = ' '},
    TerminalName,
}

local InactiveStl = {
    condition = function()
        return not conditions.is_active()
    end,
    FileType,
    {provider = ' '},
    FileName,
}

local statuslines = {
    stop = true,
    SpecialStl,
    TerminalStl,
    InactiveStl,
    DefaultStl,
}

require("heirline").setup(statuslines)
-- we're done.
```