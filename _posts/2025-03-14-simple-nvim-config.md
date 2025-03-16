---
title: "Modern Neovim in 50 lines"
layout: post
category: programming
date: 2025-03-14
<!--published: false-->
---

Neovim configs have a tendency to go off the rails. I think some amount of this has existed for a long time with Vim, but with Neovim it feels particularly exaggerated. Maybe it's the built-in LSP, or how open the core team is to new features. Maybe it's that the newest recruits are jumping ship from IDEs with features they don't want to lose out on. Whatever it is, the user base seems to be consistently wanting more and more powerful plugins. Some of these Neovim plugins are a special breed of bloated compared to some of the Vim counterparts. Could be due in part to the advent of the Neovim floating window, or that Lua is generally considered a nicer language to write. All I know is the minute configs start requiring nested file structures and lazy-loaded plugins it feels like we've strayed pretty far from the minimalist philosophy that got us here.

One side-effect of using more plugins is with each one you add, you get further abstracted away from using the base tool. When I started out on nvim I was pretty plugin happy, and there were things I would download plugins for that I didn't even realize you could do natively. There's something powerful about having command of as much of the editor as possible. You can take that with you to literally any machine. Open up a terminal or an ssh session and there isn't much different from your regular setup. Even if you don't do it much it does feel good to be that portable.

I think the right amount of minimalism is not taking it so far that you're actually sacrificing productivity. To me there are a few modern IDE features that I think are huge enhancements to the development experience:

- Inline linting
- LSP completions
- Formatting
- Fuzzy searching (both for file names and file contents)
- Interactive git blaming (re-blaming at parent commit for any line)

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

