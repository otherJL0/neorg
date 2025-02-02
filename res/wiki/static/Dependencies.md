# Understanding Neorg dependencies

Neorg depends on a number of moving parts and will only continue to accrue more dependencies to deliver on many of the features in it's ROADMAP. Many of these features, like tree-sitter support in Neovim, are still marked experimental in their parent projects. This document is intended to help Neorg users navigate these dependencies, but users are always encouraged to refer to the parent projects for appropriate documentation.


## Neovim

**Neorg is a Neovim plugin, it is not an app.** Neorg attempts to utilize Neovim features when it makes sense to do so in order to provide a familiar editing environment that corresponds to Neovim and Neovim plugin conventions. However, the project reserves the right to break from those conventions in situations where the project determines that doing so will enable better execution on the project vision.


### Terminal Support

If you are running neovim in a terminal emulator, be aware the terminal emulators are weird things. A particularly good description of terminals is provided by [https://wezfurlong.org/wezterm/what-is-a-terminal.html](https://wezfurlong.org/wezterm/what-is-a-terminal.html). A concise statement is that terminal emulators _emulate old terminal hardware, so that the kernel can pretend it is still talking to old terminal hardware_. Over the years, various emulators have wanted to offer capabilities above and beyond what those original terminals offered. Command line programs which wanted to use these extended capabilities needed to know whether or not they were on a supporting terminal, since attempting to use these features without appropriate support might cause bad behavior on terminals that did not support those features. Generally, a program can check the `TERM` environment variable and then lookup the relevant `termcap` or `terminfo` entry in the file system database of `term` files. This requires the system to have an appropriate `term` file installed in the relevant location. Alternatively, some terminal emulators support using escape sequences to query terminal capabilities via the XTVERSION and XTGETTCAP escape sequences, which directly query the emulator.

If neovim is unable to determine the correct set of terminal capabilities, it may choose conservative defaults that prevent rendering of certain text styles in the terminal, such as italic, strikethrough, undercurl, etc. **For example, `screen` and `tmux` frequently advertise a `TERM` with limited capabilities because they cannot guarantee what kind of terminal emulator will attach to them.**

*If your terminal font settings do not include appropriate rendering styles, this may also prevent you from seeing text rending in the way you expect.*

There are a couple of ways that may help determine if terminal support is limiting the display of characters in neovim.


> #### Test outside of neovim
>
> Check the output of the following:
>
> ```bash
> echo -e "\e[1mbold\e[0m"
> echo -e "\e[3mitalic\e[0m"
> echo -e "\e[3m\e[1mbold italic\e[0m"
> echo -e "\e[4munderline\e[0m"
> echo -e "\e[9mstrikethrough\e[0m"
> echo -e "\x1B[31red\e[0m"
>
> printf "\x1b[58:2::255:0:0m\x1b[4:1msingle\x1b[4:2mdouble\x1b[4:3mcurly\x1b[4:4mdotted\x1b[4:5mdashed\x1b[0m\n"
> ```
>
> If this does not produce what you expect, there is a terminal or a font problem. If it works, but does not work in Neovim, then this may either be a problem with advertising/detecting terminal capabilities or setting highlight groups.
>
>
> > ##### Windows ConPTY does not support undercurl
> >
> > Windows terminal emulators almost all use ConPTY, which strips out certain escape sequences, including those used for more advanced underlines. There is an open ticket for this feature in `microsoft/terminal` but it is unknown when it will become a priority to address given that terminal support is not generally a Microsoft business priority.
>
>
> #### Check neovim diagnostics
>
> `nvim -V3log` will produce a file named `log`. After exiting, check the `log` file for a section beginning with `--- Terminal info ---`.
>
> `:checkhealth` will run some neovim diagnostics and open them in a new `:h tabpage`. You can use `:q` to exit the tab. This will include a section on the `terminal` and in some cases identifies `TERM` settings which may cause problems and how to address them
>
>
> #### Check that neovim is able to render these characters in other contexts
>
> 1. `:highlight mytest cterm=italic gui=italic` will create a highlight group named `mytest`.
> 2. `:highlight mytest` will show a line that looks like `mytest /xxx/ cterm=italic gui=italic`. The `xxx` should be rendered _in the format specified_ (in this case italics). If not, Neovim is likely not getting the correct terminal capabilities.
> 3. Test for any other format which you are concerned is not appearing correctly.
>
>
> #### Check that highlight groups are getting assigned correctly and have an appropriate definition
>
> See [Tree-sitter](#tree-sitter) and [Colorschemes](#colorschemes) for this.


## Tree-sitter

Parsing of `.norg` documents in `neorg` is primarily handled by the `tree-sitter` library.


### Tree-sitter support in neovim

Tree-sitter functionality is provided natively by Neovim, **but native support is not the same as supported with no configuration**. Neovim is only responsible for loading a binary **\*.so** file, providing facilities for executing queries against the parse tree, and for creating highlight groups and indent rules based on those queries when they are defined in an appropriate file location. Supplying these parser and query files is the responsibility of the user or may be delegated by the user to a plugin. See `:h treesitter-parsers` for more details on how Neovim locates its tree-sitter parsers and `:h treesitter-query` and `:h treesitter-highlight` for details on the runtimepath files like `queries/*/highlights.scm`.


### Nvim-treesitter plugin

To make things easier the [`nvim-treesitter`](https://github.com/nvim-treesitter/nvim-treesitter) plugin provides best-effort configuration support for downloading tree-sitter grammars (source code) from their repositories, compiling them automatically, and placing them in the correct paths along with appropriate highlight and other queries.

`nvim-treesitter`, in turn, outsources its language-specific efforts to repositories of the form `tree-sitter-<lang>`. See the project repo for more details.

**In order for Neorg to work properly, these features must be enabled when you load and `setup()` nvim-treesitter, especially the highlight module.** See [https://github.com/nvim-treesitter/nvim-treesitter#modules](https://github.com/nvim-treesitter/nvim-treesitter#modules) for more details on how to do this. Within Neovim, you can run `:TSConfigInfo` to ensure that `modules.enable.highlight` has the expected value.


> ##### C/C++ Toolchain
>
> In order for `nvim-treesitter` to properly build the `tree-sitter-norg` parser, it requires an appropriate compiler toolchain. Ensure that the `CC` environment variable points to a compiler that has C++14 support.
> 
> ###### MacOS C/C++ Toolchain often outdated
>
> The compiler bundled with many editions of MacOS lacks the appropriate level of support. You can run Neovim like so: `CC=/path/to/newer/compiler nvim -c "TSInstallSync norg"` in your shell of choice to install the Neorg parser with a newer compiler. You may also want to export the CC variable in general: `export CC=/path/to/newer/compiler`.


### Tree-sitter-norg

As many of the features of `neorg` depend on proper `tree-sitter`-based parsing of the `.norg` document, it is important to ensure that you update `neorg` and the `tree-sitter-norg` parser at the same time.

The manual way to do this is that after your `git pull` to update the `neorg` plugin, you call `nvim -c "TSInstallSync norg"`.

The easiest way to do this is by using a [Plugin Manager](#plugin-manager).

## Plugin Manager

Plugin managers reduce the burden of maintaining a (Neo)vim configuration by providing mechanisms to automate repetitive tasks, most notably for updating groups of plugins and ensuring that plugin updates trigger additional commands, such as updating the `tree-sitter-norg` parser.

Many users also appreciate the ability to lazy-load plugins and reduce Neovim starting time; however, lazy loading complicates the loading order of plugins and this is frequently misconfigured. **Therefore, we recommend lazy loading be disabled when troubleshooting an issue to ensure that packages are loading in the order you believe.** In many plugin managers, this includes keys like `ft`, `cmd`, `event`, etc.

Importantly, plugin managers are not meant to prevent users from understanding [Neovim](#neovim). Neovim understands how to find plugin files on the basis of its `:h runtimepath`. Plugin managers add appropriate entries to that path when it is time to load a plugin.

They also make it possible to express a `Neovim` configuration which has multiple plugins (and therefore spans many files and directories) within a single file for the purposes of sharing configurations.


### Lazy.nvim

Here is an example minimal `init.lua` which utilizes ['lazy.nvim'](https://github.com/folke/lazy.nvim):

```lua
-- bootstrap lazy.nvim
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

require('lazy').setup({
    {
        "nvim-neorg/neorg",
        build = ":Neorg sync-parsers",
        opts = {
            load = {
                ["core.defaults"] = {}, -- Loads default behaviour
                ["core.concealer"] = {}, -- Adds pretty icons to your documents
                ["core.dirman"] = { -- Manages Neorg workspaces
                    config = {
                        workspaces = {
                            notes = "~/notes",
                        },
                        default_workspace = "notes",
                    },
                },
            },
        },
        dependencies = {
            { "nvim-lua/plenary.nvim", },
            {
                -- YOU ALMOST CERTAINLY WANT A MORE ROBUST nvim-treesitter SETUP
                -- see https://github.com/nvim-treesitter/nvim-treesitter
                "nvim-treesitter/nvim-treesitter",
                opts = {
                    auto_install = true,
                    highlight = {
                        enable = true,
                        additional_vim_regex_highlighting = false,
                    },
                },
                config = function(_,opts)
                    require('nvim-treesitter.configs').setup(opts)
                end
            },
            { "folke/tokyonight.nvim", config=function(_,_) vim.cmd.colorscheme "tokyonight-storm" end,},
        },
    },
})
```

### Packer.nvim

I do not use Packer.nvim and cannot provide direct help for this package manager. However, it is common to see Packer problems which stem from `:PackerCompile` not being called after an update, resulting in incorrect things being cached.


## Colorschemes

A lightweight text markup plugin like `neorg` benefits from being able to display text-decorations like **bold** and _italic_, as well as highlighting headings in different colors. Neovim users often have colorschemes configured already, which may be among the default colorschemes bundled with Neovim or be one installed from an online source like Github.

A full description of vim highlighting is best left to `:h highlight` and `:h syntax`. Suffice it to say that _syntax_ files use regular expressions to identify areas of text and mark them as belonging to some _highlight group_. Neovim extends this with `:h treesitter-highlight` to provide another mechanism of assigning _highlight groups_. `:h highlight` then allows users or colorschemes to define how highlight groups should appear.

Vim has defined a set of `:h highlight-groups` with `:h group-name` naming conventions which have been conserved for a long time. However, many plugins define their own highlight-groups to allow for more specific theming. Well-behaved plugins generally provide fallbacks which link to one of these conserved highlight-groups in case the colorscheme does not define an appearance for the plugin-specific highlight group. However, the conserved set is primarily defined with programming in mind. Thus, there is no highlight-group which is guaranteed to be **bold**. `nvim-treesitter` attempts to standardize a set of names for highlight-groups which provide expanded functionality, such as `@text.strong`, based on community consensus. See [https://github.com/nvim-treesitter/nvim-treesitter/blob/master/CONTRIBUTING.md](https://github.com/nvim-treesitter/nvim-treesitter/blob/master/CONTRIBUTING.md) for a full list of these groups.

`neorg` uses many of these new conserved tree-sitter highlight groups as fallbacks for its plugin-specific highlight groups. Because this is a relatively new development, and because these groups are being promulgated by a plugin instead of Neovim core, many colorschemes **including the ones bundled with Neovim** do not support these new highlight groups. `:highlight @text.strong` will tell you the definition that Neovim currently has for that group. `tokyonight` and `kanagawa` are two themes which are known to support the relevant tree-sitter highlight-groups.


## Concealing

Although the purpose of a lightweight markup language is to produce documents that remain readable even in plain-text, it is often nice from a readability standpoint to be able to conceal the markup from view.

Neorg, through it's `core.conceal` module, is designed to substitute many characters with more aesthetic choices, hide the multiple characters in e.g. headlines and lists, and conceal link URLs and other markup. It achieves this by setting the `:h conceal` argument on the highlight group. On its own, this only applies a `conceal` tag to the highlight group. Users must configure their `:h conceallevel` and `:h concealcursor` to actually hide the text, as the default `conceallevel` setting does not conceal any text. This is good default Neovim behavior since only people who are aware of the `conceallevel` options and functionality will have text hidden from them.


### Ugly line wraps when concealing text

There is a known bug in how concealing interacts with wrapped lines. This is a long-standing vim and Neovim behavior with no easy fix since it touches on complex rendering and UI logic to allow performant and logical editing on a file from within a window which displays contents which differ from buffer contents.

There is a Neovim effort underway known as `anticonceal` which should hopefully address this bug. However, like all things, time is limited and priorities must be balanced and this is not an easy bug to address. Discussions around this bug have been ongoing for several years - if it is so pressing to your use case, implement a fix yourself or pay someone to do so at market rates.

Having said all that, as users of Neorg, we also regularly deal with this bug. Some suggested option of workflows that accommodate for this behavior include the following:
1. Do not use conceal
2. Use hard wrap (`:h formatting`) and manually insert line breaks so that the concealed text wraps in a manner that suits your requirements
3. Use soft wrap (`:h wrap`) in combination with the anchors features described in the [norg-specs](https://github.com/nvim-neorg/norg-specs) to separate the link usage from its definition. This mitigates the rendering issues in Neovim by reducing the number of characters which must be concealed, resulting in better reflow behavior in most cases. Be cognizant that anchor names will need to be unique for linking to operate as expected.
    - For a better editing experience, you may also be interested in `:h linebreak`, `:h breakindent` and `:h breakindentopt`.
