---
title: "Modern Neovim in <50 lines"
layout: post
category: programming
date: 2025-03-14
<!--published: false-->
---

Neovim configs have a tendency to go off the rails. I think some amount of this has existed for a long time with Vim, but with Neovim it feels particularly exaggerated. Maybe it's the built-in LSP, or how open the core team is to new features. Maybe it's that the newest recruits are jumping ship from IDEs with features they don't want to lose out on. Whatever it is, the user base seems to be consistently wanting more and more powerful plugins. Some of these Neovim plugins are a special breed of bloated compared to some of the Vim counterparts. Could be due in part to the advent of the Neovim floating window, or that Lua is generally considered a nicer language to write. All I know is the minute configs start requiring nested file structures and lazy-loaded plugins it feels like we've strayed pretty far from the minimalist philosophy that got us here.

One side-effect of using more plugins is with each one you add, you get further abstracted away from using the base tool. When I started out on nvim I was pretty plugin happy, and there were things I would download plugins for that I didn't even realize you could do natively. There's something powerful about having command of as much of the editor as possible. You can take that with you to literally any machine. Open up a terminal or an ssh session and there isn't much different from your regular setup. Even if you don't do it much it does feel good to be that portable.

I think the right amount of minimalism is not taking it so far that you're actually sacrificing productivity. To me there are a few modern IDE features that I think are huge enhancements to the development experience:

- Inline linting
- LSP completions & jump to definition
- Auto-formatting
- Fuzzy searching (both for file names and file contents)

...and maybe a couple of more QOL improvements. I'm going to walk through how you can get each of these with the minimal possible config. I think we can do it in less than 50 lines. I'll use Python as the language for my example setup, and macOS for my operating system, though any *nix system will use a very similar setup. Let's get to it.

## Getting a package manager

This is a prerequisite to the reset of our setup. Neovim does have a built in packages system that you can explore with `:h packages` if you want to go that route. What I don't like about going this route is the packages you're using aren't specified anywhere in your configs, they end up only in your data directory. This makes them a bit harder to use with version control, so we'll go with something 3rd party.

The simplest package manager I've found is [paq-nvim](https://github.com/savq/paq-nvim). Install it like so:

```
$ git clone --depth=1 https://github.com/savq/paq-nvim.git \
    "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/pack/paqs/start/paq-nvim
```

Then we just need to make sure we have a neovim config folder with

```
$ mkdir -p ~/.config/nvim
```

Then create a file called `init.lua` in that directory and put the following lines at the top:

```lua
require("paq")({
    "savq/paq-nvim"
})
```

We have a package manager! Now close and reopen neovim and you should have a variety of `:Paq*` commands available to you. All of your future packages will go inside that initial setup call. The format is "\<GitHub Org>/\<GitHub Repo>". Using `:PaqList` will show you what packages you have installed, and `:PaqSync` will both clean packages that have been removed and install new ones.

## Linting & LSP features

This one is a lot easier than you'd think with [`nvim-lspconfig`](https://github.com/neovim/nvim-lspconfig). The full list of linters and formatters it supports can be found [here](https://github.com/neovim/nvim-lspconfig/blob/master/doc/configs.md).

First let's add nvim-lspconfig to our packages:

```lua
require("paq")({
    "savq/paq-nvim",
    "neovim/nvim-lspconfig"
})
```

We're going to use pyright here for LSP features and completions, and ruff for linting. We'll set it up like this:

```lua
local lspconfig = require("lspconfig")
local lsps = { "pyright", "ruff" }
for _, lsp in pairs(lsps) do
    local setup = {}
    if lsp == "pyright" then
        setup = {
            settings = {
                [lsp] = {
                    analysis = {
                        diagnosticMode = "workspace",
                        typeCheckingMode = "off"
                    }
                }
            }
        }
    end
    lspconfig[lsp].setup(setup)
end
```

All we're doing here is defining the LSPs we want to look for, and then setting them up for use in Neovim. The for loop loops over our list of LSPs and adds any configuration options that we want to add. These can be found in the documentation for the respective LSP. We're mostly using pyright here for LSP features like jump to definition and completions. I don't find in-editor type checking for Python strictly necessary. Ruff is a great linter and can handle all the inline linting we need, and it has good defaults.

To be clear here, nvim-lspconfig does *not install the LSPs for you*. You need to do that yourself. This part is heavily dependent on what LSPs & linters you're using, but most I've used have been quite simple to install. For pyright & ruff, there are a few ways you can handle this. The simplest way would just be to run:

```
$ brew install pyright
$ brew install ruff
```

...and the servers should be automatically added to your $PATH and detected by Neovim. If you have Node installed on your system already I prefer to install pyright with `npm install -g pyright`. The brew install will install node as a dependency which was overriding the default node version I had set with `nvm`. For ruff, I prefer to install this into the venv of the specific Python project I'm working on, but a global install works too. If you install into a venv, you just have to make sure the virtual environment is activated before you open Neovim.

## LSP Completions

Surprise, this is already done! Neovim has a built in completion type called "omni completion" that the LSP will hook into by default. You can read about it at `:h compl-omni`. When in insert mode, you can trigger it with `<C-X><C-O>`. To see the full list of completions available with `<C-X>`, check out `:h ins-completion`. The Neovim docs even give you a way you could upgrade this to autocomplete as you type, which you can see at `:h compl-autocomplete`.

## Formatting

For formatting we'll use a plugin called Conform. Formatting works very similarly to the LSP in that you just need your formatter in your `$PATH` and Conform will pick it up.

First, of course, add `stevearc/conform.nvim` to your paq setup. I won't run you through that one again.

Next we need to setup Conform. We can use the snippet below at the bottom of our `init.lua` file.

```lua
require("conform").setup({
	formatters_by_ft = {
		python = { "ruff_organize_imports", "ruff_format" },
	},
	format_after_save = {},
})
```

Take a look at all the valid formatting commands [here](https://github.com/stevearc/conform.nvim?tab=readme-ov-file#formatters). Luckily, ruff has a built-in formatter, so we don't need any extra packages. You'll notice two separate ruff commands -- one organizes imports and one runs the ruff formatter. It's also set up to automatically format on save. Now run a PaqInstall and on to the next...

## Fuzzy Finding

Fuzzy searching is the most efficient way to jump between files in Neovim. Instead of navigating through a traditional file tree, you'll find it's usually faster to search your working directory for the file name. The fuzzy finder will live update as you type, and usually within a few keystrokes you'll get to the file you're looking for. The plugin we'll use for this not only allows search file names, but also the contents of files.

We'll use a plugin called fzf-vim for this one. To start, we need to add these two packages to paq: `junegunn/fzf` and `junegunn/fzf.vim`, in that order.

Fzf is actually a command line application that neovim interfaces with from terminal mode. So we will need a few applications installed to our system:

```
$ brew install fzf # the main fzf app
$ brew install ripgrep # to allow us to search the contents of files
$ brew install bat # for syntax highlighting in fzf
```

Fzf is a tool you will use enough that it's worth having a keymap for the main commands. Put the following right underneath your paq setup:

```lua
vim.g.mapleader = " "
vim.keymap.set("n", "<leader>f", ":Files!<cr>")
vim.keymap.set("n", "<leader>g", ":RG!<cr>")
```

In vim, "leader" is a user defined key that is commonly used as a prefix for custom keymaps. It's there so we have a way to easily set keymaps without worrying about overwriting the default ones. We've set it to the space bar here, which is a common leader key. Now, <space>-f will allow us to fuzzy search file names, and <space>-g searches file contents!

## Bringing it all Together

You've made it to the end. Let's take a look at our init.lua file after all this:

```lua
require("paq")({
    "savq/paq-nvim",
    "neovim/nvim-lspconfig",
    "stevearc/conform.nvim",
    "junegunn/fzf",
    "junegunn/fzf.vim"
})

vim.g.mapleader = " "
vim.keymap.set("n", "<leader>f", ":Files!<cr>")
vim.keymap.set("n", "<leader>g", ":RG!<cr>")

local lspconfig = require("lspconfig")
local lsps = { "pyright", "ruff" }
for _, lsp in pairs(lsps) do
    local setup = {}
    if lsp == "pyright" then
        setup = {
            settings = {
                [lsp] = {
                    analysis = {
                        diagnosticMode = "workspace",
                        typeCheckingMode = "off"
                    }
                }
            }
        }
    end
    lspconfig[lsp].setup(setup)
end

require("conform").setup({
	formatters_by_ft = {
		python = { "ruff_organize_imports", "ruff_format" },
	},
	format_after_save = {},
})
```

Less than 50 lines, just like we said. That's pretty good. Don't forget the system packages we installed! Below is every command we ran in the terminal:

```
$ git clone --depth=1 https://github.com/savq/paq-nvim.git \
    "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/pack/paqs/start/paq-nvim
$ mkdir -p ~/.config/nvim
$ brew install pyright
$ brew install ruff
$ brew install fzf
$ brew install ripgrep
$ brew install bat
```

## This is Not the End

I am under no illusion that after this you won't touch your config again. My personal config is ~100 lines where I have some additional keymaps set up, a few of the default vim options changed and a couple more plugins installed. The point here is just to show how powerful Neovim can be out-of-the-box. This is a very functional setup that you can easily build on. It doesn't require 15 files and 40 plugins to set up a great development environment for Neovim. It's already there for you!

