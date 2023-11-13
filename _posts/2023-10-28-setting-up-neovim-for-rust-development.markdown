---
layout: post
title:  "Setting up Neovim for Rust development"
date:   2023-10-28 18:00:00 +0100
tags: ["Rust", "Neovim"]
---

Before starting on my Rust UO server implementation I needed to set up my editor to allow me to be as productive as possible. I've been using Neovim for a couple of years now but hadn't bothered setting it up properly for Rust until now as I'd only been playing around with noddy example code while reading The Rust Book.

I started from a fresh install of Neovim, completely vanilla. First step was to install a plugin manager:

### Plugin & package managers
Previously I'd used [packer.nvim](https://github.com/wbthomason/packer.nvim) and [vim-plug](https://github.com/junegunn/vim-plug) to manage Neovim plugins but the latest and greatest now seems to be [lazy.nvim](https://github.com/folke/lazy.nvim) so I decided to try it.

I took the installation snippet from the readme and added it to a *plugins* module that I required from my init file:

```lua
-- ~/.config/nvim/init.lua

require('plugins')
```

```lua
-- ~/.config/nvim/lua/plugins.lua

local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({
    "git",
    "clone",
    "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable", -- latest stable release
    lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)

require("lazy").setup(require('plugins.specs'))
```

The `require` on the last line above calls Lazy's setup function, passing it an empty plugin specs table required from a nested *specs* module I created:

```lua
-- ~/.config/nvim/lua/plugins/specs.lua

return {}
```

Next I installed [mason.nvim](https://github.com/williamboman/mason.nvim), a plugin that makes managing and installing LSP servers easier. To install it I added it to the table in the newly created *specs* module"

```lua
-- ~/.config/nvim/lua/plugins/specs.lua

return {
  { "williamboman/mason.nvim" },
  { "williamboman/mason-lspconfig.nvim" },
  { "neovim/nvim-lspconfig" },
}
```
[mason-lspconfig.nvim](https://github.com/williamboman/mason-lspconfig.nvim) and [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) were recommended to be installed alongside Mason.

I added the setup function calls needed to make Mason work to a new nested setup module.

```lua
-- ~/.config/nvim/lua/plugins/setup/mason.lua

require("mason").setup()
require("mason-lspconfig").setup()
```

And required it from the *plugins* module:

```lua
-- ~/.config/nvim/lua/plugins.lua

require('plugins.setup.mason')
```

After installing the plugin and package managers it was time to get an LSP server installed and configured:

### LSP server

The LSP server I chose to install was [rust-analyzer](https://github.com/rust-lang/rust-analyzer). Installation was easy with Mason, I just needed to run the following command in Neovim and the LSP server and dependencies were installed:

```vim
:MasonInstall rust-analyzer
```

Installing the LSP server makes the following possible:
  - Code completion
  - Refactoring
  - Linting
  - Go to definition and references
  - Code actions
  - Access to documentation
  - Snippets
  - Improved syntax highlighting
  - Formatting

Next I needed to configure rust-analyzer to work with lspconfig. The easiest way to do that was to install [rust-tools.nvim](https://github.com/simrat39/rust-tools.nvim):

```lua
-- ~/.config/nvim/lua/plugins/specs.lua

return {
  -- snip
  { "simrat39/rust-tools.nvim" },
}
```

And then call the setup function and configure the keymaps:

```lua
-- ~/.config/nvim/lua/plugins.lua

require('plugins.setup.rust-tools')
```

```lua
-- ~/.config/nvim/lua/plugins/setup/rust-tools.lua

local rt = require("rust-tools")

rt.setup({
  server = {
    on_attach = function(_, bufnr)
      vim.keymap.set("n", "<C-space>", rt.hover_actions.hover_actions, { buffer = bufnr })
      vim.keymap.set("n", "<Leader>a", rt.code_action_group.code_action_group, { buffer = bufnr })
    end,
  },
})
```

The keymaps above set up Ctrl-Space to display a floating window above the symbol under the cursor with information on it, and <Leader>-a to display a code actions floating window for the symbol under the cursor.

Some configuration was also needed. I added this to a new *lsp-opts* module, required from *init.lua*:

```lua
-- ~/.config/nvim/init.lua

require('lsp-opts')
```

```lua
-- ~/.config/nvim/lua/lsp-opts.lua

vim.diagnostic.config({
    virtual_text = false,
    signs = true,
    update_in_insert = true,
    underline = true,
    severity_sort = false,
    float = {
        border = 'rounded',
        source = 'always',
        header = '',
        prefix = '',
    },
})

vim.cmd([[
set signcolumn=yes
autocmd CursorHold * lua vim.diagnostic.open_float(nil, { focusable = false })
]])
```

The above configures the LSP floating diagnostic window and sets it to appear immediately when the cursor is stationary over a symbol.

Next I tackled getting a debugger working with Rust:

### Debugger

I opted to install the [codelldb](https://github.com/vadimcn/codelldb) debugger. Installation was via Mason:
```vim
:MasonInstall codelldb
```

To be able to use the debugger, I installed the [vimspector](https://github.com/puremourning/vimspector) plugin which provides a debugging UI:

```lua
-- ~/.config/nvim/lua/plugins/specs.lua

return {
  -- snip
  { "puremourning/vimspector" },
}
```

And then configured the UI and keymaps:

```lua
-- ~/.config/nvim/init.lua

require('debugging-opts')
```

```lua
-- ~/.config/nvim/lua/debugging-opts.lua

vim.cmd([[
let g:vimspector_sidebar_width = 85
let g:vimspector_bottombar_height = 15
let g:vimspector_terminal_maxwidth = 70
]])

vim.cmd([[
nnoremap <Leader>dd :call vimspector#Launch()<CR>
nnoremap <Leader>de :call vimspector#Reset()<CR>
nnoremap <Leader>dc :call vimspector#Continue()<CR>

nnoremap <Leader>dt :call vimspector#ToggleBreakpoint()<CR>
nnoremap <Leader>dT :call vimspector#ClearBreakpoints()<CR>

nmap <Leader>dk <Plug>VimspectorRestart
nmap <Leader>dh <Plug>VimspectorStepOut
nmap <Leader>dl <Plug>VimspectorStepInto
nmap <Leader>dj <Plug>VimspectorStepOver
]])
```

To use the debugger in a Rust project I also needed to create a Vimspector config file:

*APPNAME/.vimspector.json*:
```json
{
  "configurations": {
    "launch": {
      "adapter": "CodeLLDB",
      "filetypes": [ "rust" ],
      "configuration": {
        "request": "launch",
        "program": "${workspaceRoot}/target/debug/APPNAME"
      }
    }
  }
}
```

`APPNAME` would be changed to the name of the project e.g. `foo_project` if `cargo new foo_project` had been run.

Next up was a code completion framework and sources:

### Completion framework & sources

I installed the [nvim-cmp](https://github.com/hrsh7th/nvim-cmp) completion framework:

```lua
-- ~/.config/nvim/lua/plugins/specs.lua

return {
  -- snip
  { "hrsh7th/nvim-cmp" },
}
```

And the following sources:

```lua
-- ~/.config/nvim/lua/plugins/specs.lua

return {
  -- snip
  { "hrsh7th/cmp-nvim-lsp" },
  { "hrsh7th/cmp-nvim-lua" },
  { "hrsh7th/cmp-nvim-lsp-signature-help" },
  { "hrsh7th/cmp-vsnip" },
  { "hrsh7th/cmp-path" },
  { "hrsh7th/cmp-buffer" },
  { "hrsh7th/vim-vsnip" },
}
```

I added the call to cmp's setup function (and the configuration passed to it) in a new *setup* module for cmp, required from the *plugins* module:

```lua
-- ~/.config/nvim/lua/plugins.lua

require('plugins.setup.cmp')
```

```lua
-- ~/.config/nvim/lua/plugins/setup/cmp.lua

local cmp = require'cmp'
cmp.setup({
  snippet = {
    expand = function(args)
        vim.fn["vsnip#anonymous"](args.body)
    end,
  },
  sources = {
    { name = 'path' },
    { name = 'nvim_lsp', keyword_length = 3 },
    { name = 'nvim_lsp_signature_help'},
    { name = 'nvim_lua', keyword_length = 2},
    { name = 'buffer', keyword_length = 2 },
    { name = 'vsnip', keyword_length = 2 },
    { name = 'calc'},
  },
  window = {
      completion = cmp.config.window.bordered(),
      documentation = cmp.config.window.bordered(),
  },
  formatting = {
      fields = {'menu', 'abbr', 'kind'},
      format = function(entry, item)
          local menu_icon ={
              nvim_lsp = 'λ',
              vsnip = '⋗',
              buffer = 'Ω',
              path = '🖫',
          }
          item.menu = menu_icon[entry.source.name]
          return item
      end,
  },
  mapping = {
    ['<C-p>'] = cmp.mapping.select_prev_item(),
    ['<C-n>'] = cmp.mapping.select_next_item(),
    ['<S-Tab>'] = cmp.mapping.select_prev_item(),
    ['<Tab>'] = cmp.mapping.select_next_item(),
    ['<C-S-f>'] = cmp.mapping.scroll_docs(-4),
    ['<C-f>'] = cmp.mapping.scroll_docs(4),
    ['<C-Space>'] = cmp.mapping.complete(),
    ['<C-e>'] = cmp.mapping.close(),
    ['<CR>'] = cmp.mapping.confirm({
      behavior = cmp.ConfirmBehavior.Insert,
      select = true,
    })
  },
})
```

I set additional options in a new options module, required from *init.lua*:
```lua
-- ~/.config/nvim/init.lua

require('completion-opts')
```

```lua
-- ~/.config/nvim/lua/completion-opts.lua

vim.opt.completeopt = {'menuone', 'noselect', 'noinsert'}
vim.opt.shortmess = vim.opt.shortmess + { c = true}
vim.api.nvim_set_option('updatetime', 300) 
```

Next I configured a parser:

### Parser & syntax highlighting

The parser I installed was [nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter). Treesitter is an incremental parser so the improved syntax highlighting it gives you even shows up when there is invalid code elsewhere in the buffer.

I installed it with Lazy by adding it to the *specs* module:"

```lua
-- ~/.config/nvim/lua/plugins/specs.lua

return {
  -- snip
  { "nvim-treesitter/nvim-treesitter" },
}
```

And called its setup function in a new setup module, required from the *plugins* module:

```lua
-- ~/.config/nvim/lua/plugins.lua

require('plugins.setup.treesitter')
```

```lua
-- ~/.config/nvim/lua/plugins/setup/treesitter.lua

require('nvim-treesitter.configs').setup {
  ensure_installed = { "lua", "rust", "toml" },
  auto_install = true,
  highlight = {
    enable = true,
    additional_vim_regex_highlighting=false,
  },
  ident = { enable = true }, 
  rainbow = {
    enable = true,
    extended_mode = true,
    max_file_lines = nil,
  }
}
```

After this I needed to run the following command in Neovim to verify that the parsers for Rust, TOML and Lua had been installed:
```vim
:TSInstallInfo
```

I then configured Neovim to perform folding with treesitter, adding the settings to a new *opts* module required in *init.lua*:

```lua
-- ~/.config/nvim/init.lua

require('folding-opts')
```

```lua
-- ~/.config/nvim/lua/folding-opts.lua

vim.wo.foldmethod = 'expr'
vim.wo.foldexpr = 'nvim_treesitter#foldexpr()'
```

After finishing installing and configuring the parser, the next task on my list was to set up project search functionality:

### Fuzzy finder

To allow fuzzy finding files, buffers and more when working on Rust projects I installed [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim):

```lua
-- ~/.config/nvim/lua/plugins/specs.lua

return {
  -- snip
  { "nvim-lua/plenary.nvim" },
  { "nvim-telescope/telescope.nvim" },
  { "nvim-telescope/telescope-fzf-native.nvim", build = "make" },
}
```

[plenary.nvim](https://github.com/nvim-lua/plenary.nvim) is a requirement for telescope, and [telescope-fzf-native.nvim](https://github.com/nvim-telescope/telescope-fzf-native.nvim) is a recommended optional dependency that improves search speed.

Keymaps were configured in a new *setup* module, again required from the *plugins* module:

```lua
-- ~/.config/nvim/lua/plugins.lua

require('plugins.setup.telescope')
```

```lua
-- ~/.config/nvim/lua/plugins/setup/telescope.lua

local builtin = require('telescope.builtin')
vim.keymap.set('n', '<leader>ff', builtin.find_files, {})
vim.keymap.set('n', '<leader>fg', builtin.live_grep, {})
vim.keymap.set('n', '<leader>fb', builtin.buffers, {})
vim.keymap.set('n', '<leader>fh', builtin.help_tags, {})
```

Next task was to get a terminal working inside Neovim, so I didn't need to leave it to run Cargo or Git commands:

### Terminal

I installed [vim-floaterm](https://github.com/voldikss/vim-floaterm) to allow me to bring up a floating terminal window inside Neovim:

```lua
-- ~/.config/nvim/lua/plugins/specs.lua

return {
  -- snip
  { "voldikss/vim-floaterm" },
}
```

And configured keymaps for it in a new *opts* module, required from *init.lua*:

```lua
-- ~/.config/nvim/init.lua

require('terminal-opts')
```

```lua
-- ~/.config/nvim/lua/terminal-opts.lua

vim.cmd([[
nnoremap <Leader>ft :FloatermNew --name=myfloat --height=0.8 --width=0.7 --autoclose=2 fish <CR> 
nnoremap t :FloatermToggle myfloat<CR>
]])
vim.keymap.set('t', "<Esc>", "<C-\\><C-n>:q<CR>")
```

The last plugin I wanted to install was one to allow EasyMotion style buffer navigation:

### EasyMotion style navigation

To enable quick navigation around the code in a buffer I installed [hop.nvim](https://github.com/smoka7/hop.nvim):

```lua
-- ~/.config/nvim/lua/plugins/specs.lua

return {
  -- snip
  { "smoka7/hop.nvim" },
}
```

And configured keymaps for it:

```lua
-- ~/.config/nvim/lua/plugins.lua

require('plugins.setup.hop')
```

```lua
-- ~/.config/nvim/lua/plugins/setup/hop.lua

local hop = require('hop')
hop.setup()
vim.keymap.set('n', '<Leader>hw', hop.hint_words)
vim.keymap.set('n', '<Leader>hc', hop.hint_char2)
vim.keymap.set('n', '<Leader>hl', hop.hint_lines)
vim.keymap.set('n', '<Leader>hp', hop.hint_patterns)
```
