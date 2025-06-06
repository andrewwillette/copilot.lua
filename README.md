# copilot.lua

This plugin is the pure lua replacement for [github/copilot.vim](https://github.com/github/copilot.vim).

<details>
<summary>Motivation behind `copilot.lua`</summary>

While using `copilot.vim`, for the first time since I started using neovim my laptop began to overheat. Additionally,
I found the large chunks of ghost text moving around my code, and interfering with my existing cmp ghost text disturbing.
As lua is far more efficient and makes things easier to integrate with modern plugins, this repository was created.

</details>

## Requirements

- Curl
- NeoVim 0.10.0 or higher

## Install

Install the plugin with your preferred plugin manager.
For example, with [packer.nvim](https://github.com/wbthomason/packer.nvim):

```lua
use { "zbirenbaum/copilot.lua" }
```

### Authentication

You can authenticate using one of the following methods:

#### Permanent sign-in (Recommended)

Once copilot is running, run `:Copilot auth` to start the authentication process.

#### Token

Get a token from the github cli using:

```sh
gh auth token
```

Set either the environment variable `GITHUB_COPILOT_TOKEN` or `GH_COPILOT_TOKEN` to that token.
Note that if you have the variable set, even empty, the LSP will attempt to use it to log in.

## Setup and Configuration

You have to run the `require("copilot").setup(options)` function in order to start Copilot.
If no options are provided, the defaults are used.

Because the copilot server takes some time to start up, it is recommended that you lazy load copilot.
For example:

```lua
use {
  "zbirenbaum/copilot.lua",
  cmd = "Copilot",
  event = "InsertEnter",
  config = function()
    require("copilot").setup({})
  end,
}
```

The following is the default configuration:

```lua
require('copilot').setup({
  panel = {
    enabled = true,
    auto_refresh = false,
    keymap = {
      jump_prev = "[[",
      jump_next = "]]",
      accept = "<CR>",
      refresh = "gr",
      open = "<M-CR>"
    },
    layout = {
      position = "bottom", -- | top | left | right | horizontal | vertical
      ratio = 0.4
    },
  },
  suggestion = {
    enabled = true,
    auto_trigger = false,
    hide_during_completion = true,
    debounce = 75,
    keymap = {
      accept = "<M-l>",
      accept_word = false,
      accept_line = false,
      next = "<M-]>",
      prev = "<M-[>",
      dismiss = "<C-]>",
    },
  },
  filetypes = {
    yaml = false,
    markdown = false,
    help = false,
    gitcommit = false,
    gitrebase = false,
    hgcommit = false,
    svn = false,
    cvs = false,
    ["."] = false,
  },
  logger = {
    file = vim.fn.stdpath("log") .. "/copilot-lua.log",
    file_log_level = vim.log.levels.OFF,
    print_log_level = vim.log.levels.WARN,
    trace_lsp = "off", -- "off" | "messages" | "verbose"
    trace_lsp_progress = false,
    log_lsp_messages = false,
  },
  copilot_node_command = 'node', -- Node.js version must be > 20
  workspace_folders = {},
  copilot_model = "",  -- Current LSP default is gpt-35-turbo, supports gpt-4o-copilot
  root_dir = function()
    return vim.fs.dirname(vim.fs.find(".git", { upward = true })[1])
  end,
  should_attach = function(_, _)
    if not vim.bo.buflisted then
      logger.debug("not attaching, buffer is not 'buflisted'")
      return false
    end

    if vim.bo.buftype ~= "" then
      logger.debug("not attaching, buffer 'buftype' is " .. vim.bo.buftype)
      return false
    end

    return true
  end,
  server = {
    type = "nodejs", -- "nodejs" | "binary"
    custom_server_filepath = nil,
  },
  server_opts_overrides = {},
})
```

### panel

Panel can be used to preview suggestions in a split window. You can run the
`:Copilot panel` command to open it.

If `auto_refresh` is `true`, the suggestions are refreshed as you type in the buffer.

The `copilot.panel` module exposes the following functions:

```lua
require("copilot.panel").accept()
require("copilot.panel").jump_next()
require("copilot.panel").jump_prev()
require("copilot.panel").open({position, ratio})
require("copilot.panel").toggle()
require("copilot.panel").refresh()
```

### suggestion

When `auto_trigger` is `true`, copilot starts suggesting as soon as you enter insert mode.
When `auto_trigger` is `false`, use the `next`, `prev` or `accept` keymap to trigger copilot suggestion.

To toggle auto trigger for the current buffer, use `require("copilot.suggestion").toggle_auto_trigger()`.

Copilot suggestion is automatically hidden when `popupmenu-completion` is open. In case you use a custom
menu for completion, you can set the `copilot_suggestion_hidden` buffer variable to `true` to have the
same behavior.

<details>
<summary>Example using nvim-cmp</summary>

```lua
cmp.event:on("menu_opened", function()
  vim.b.copilot_suggestion_hidden = true
end)

cmp.event:on("menu_closed", function()
  vim.b.copilot_suggestion_hidden = false
end)
```

</details>

<details>
<summary>Example using blink.cmp</summary>

```lua
vim.api.nvim_create_autocmd("User", {
  pattern = "BlinkCmpMenuOpen",
  callback = function()
    vim.b.copilot_suggestion_hidden = true
  end,
})

vim.api.nvim_create_autocmd("User", {
  pattern = "BlinkCmpMenuClose",
  callback = function()
    vim.b.copilot_suggestion_hidden = false
  end,
})

```

</details>

The `copilot.suggestion` module exposes the following functions:

```lua
require("copilot.suggestion").is_visible()
require("copilot.suggestion").accept(modifier)
require("copilot.suggestion").accept_word()
require("copilot.suggestion").accept_line()
require("copilot.suggestion").next()
require("copilot.suggestion").prev()
require("copilot.suggestion").dismiss()
require("copilot.suggestion").toggle_auto_trigger()
```

### filetypes

Specify filetypes for attaching copilot.

Example:

```lua
require("copilot").setup {
  filetypes = {
    markdown = true, -- overrides default
    terraform = false, -- disallow specific filetype
    sh = function ()
      if string.match(vim.fs.basename(vim.api.nvim_buf_get_name(0)), '^%.env.*') then
        -- disable for .env files
        return false
      end
      return true
    end,
  },
}
```

If you add `"*"` as a filetype, the default configuration for `filetypes` won't be used anymore. e.g.

```lua
require("copilot").setup {
  filetypes = {
    javascript = true, -- allow specific filetype
    typescript = true, -- allow specific filetype
    ["*"] = false, -- disable for all other filetypes and ignore default `filetypes`
  },
}
```

### logger

Logs will be written to the `file` for anything of `file_log_level` or higher.
Logs will be printed to NeoVim (using `notify`) for anything of `print_log_level` or higher.
To turn either off, simply set its level to `vim.log.levels.OFF`.
File logging is done asynchronously to minimize performance impacts, however there is still some overhead.

Log levels used are the ones defined in `vim.log`:

```lua
vim.log = {
  levels = {
    TRACE = 0,
    DEBUG = 1,
    INFO = 2,
    WARN = 3,
    ERROR = 4,
    OFF = 5,
  },
}
```

`trace_lsp` controls logging of LSP trace messages (`$/logTrace`) can either be:

- `off`
- `messages` which will output the LSP messages
- `verbose` which adds additonal information to the message.

When `trace_lsp_progress` is true, LSP progress messages (`$/progress`) will also be logged.
When `log_lsp_messages` is true, LSP log messages (`window/logMessage`) events will be logged.

Careful turning on all logging features as the log files may get very large over time, and are not pruned by the application.

### copilot_node_command

Use this field to provide the path to a specific node version such as one installed by nvm. Node.js version must be 20 or newer.

Example:

```lua
copilot_node_command = vim.fn.expand("$HOME") .. "/.config/nvm/versions/node/v20.0.1/bin/node", -- Node.js version must be > 20
```

### server_opts_overrides

Override copilot lsp client settings. The `settings` field is where you can set the values of the options defined in [SettingsOpts.md](./SettingsOpts.md).
These options are specific to the copilot lsp and can be used to customize its behavior. Ensure that the name field is not overridden as is is used for
efficiency reasons in numerous checks to verify copilot is actually running. See `:h vim.lsp.start_client` for list of options.

Example:

```lua
require("copilot").setup {
  server_opts_overrides = {
    trace = "verbose",
    settings = {
      advanced = {
        listCount = 10, -- #completions for panel
        inlineSuggestCount = 3, -- #completions for getCompletions
      }
    },
  }
}
```

### workspace_folders

Workspace folders improve Copilot's suggestions.
By default, the root_dir is used as a wokspace_folder.

Additional folders can be added through the configuration as such:

```lua
workspace_folders = {
  "/home/user/gits",
  "/home/user/projects",
}
```

They can also be added runtime, using the command `:Copilot workspace add [folderpath]` where `[folderpath]` is the workspace folder.

### root_dir

This allows changing the function that gets the root folder, the default looks for a parent folder that contains the folder `.git`.
If none is found, it will use the current working directory.

### should_attach

This function is called to determine if copilot should attach to the buffer or not.
It is useful if you would like to go beyond the filetypes and have more control over when copilot should attach.
You can also use it to attach to buflisted buffers by simply omitting that portion from the function.
Since this happens before attaching to the buffer, it is good to prevent Copilot from reading sensitive files.

An example of this would be:

```lua
require("copilot").setup {
  should_attach = function(_, bufname)
    if string.match(bufname, "env") then
      return false
    end

    return true
  end
}
```

### server

> [!CAUTION] > `"binary"` mode is still very much experimental, please report any issues you encounter.

`type` can be either `"nodejs"` or `"binary"`. The binary version will be downloaded if used.

`custom_server_filepath` is used to specify the path of either the path (filename included) of the `js` file if using `"nodejs"` or the path to the binary if using `"binary"`.
When using `"binary"`, the download process will be disabled and the binary will be used directly.
example:

```lua
require("copilot").setup {
  server = {
    type = "nodejs",
    custom_server_filepath = "/home/user/copilot-lsp/language-server.js",,
  },
}
```

## Commands

`copilot.lua` defines the `:Copilot` command that can perform various actions. It has completion support, so try it out.

## Integrations

The `copilot.api` module can be used to build integrations on top of `copilot.lua`.

- [zbirenbaum/copilot-cmp](https://github.com/zbirenbaum/copilot-cmp): Integration with [`nvim-cmp`](https://github.com/hrsh7th/nvim-cmp).
- [giuxtaposition/blink-cmp-copilot](https://github.com/giuxtaposition/blink-cmp-copilot): Integration with [`blink.cmp`](https://github.com/Saghen/blink.cmp).
- [fang2hou/blink-copilot](https://github.com/fang2hou/blink-copilot): Integration with [`blink.cmp`](https://github.com/Saghen/blink.cmp), with some differences.
- [AndreM222/copilot-lualine](https://github.com/AndreM222/copilot-lualine): Integration with [`lualine.nvim`](https://github.com/nvim-lualine/lualine.nvim).
