---
title: "Reconfiguring Neovim"
date: 2022-02-20T14:49:02+01:00
author: Martin Kauppinen
---

I was recently inspired by a friend to nuke my Neovim config and start over from
scratch, doing things a bit more "properly" this time. But I didn't want to
entirely destroy my current config -- which I had not yet put under version
control -- before I had the new one set up in a way I was sure worked and I was
happier with than my current one. This post documents my approach for when I
feel like doing the same thing again some time in the future. Although I am now
going to put my config under version control, so _technically_ I won't have to.
You never know, though.

## Making a Neovim config sandbox
Neovim respects the `$XDG_CONFIG_HOME` and `$XDG_DATA_HOME` environment
variables, so the approach here is actually very simple: set those to paths
other than `$HOME/.config/` and `$HOME/.local/share`! In particular, I made a
directory at `$HOME/tmp/nvim-setup/` where I wanted the sandbox to be for
experimentation, but only while I was in the sandbox. In this directory, I
created two subdirectories -- one for each environment variable -- called
`config` and `data`. Neovim will take care of putting an `nvim` subdirectory in
each of these:

```sh
$HOME/tmp/nvim-setup/
├── config/
└── data/
```

I used [direnv](https://direnv.net/) to accomplish this, setting the environment
variables when the current working directory was the sandbox directory and
resetting them when I changed to somewhere else. This way I didn't have to
change my shell setup or remember to run Neovim like

```bash
XDG_CONFIG_HOME=$HOME/tmp/nvim-setup/config nvim
```

to test things. As long as the current working directory is the sandbox
directory, simply launching `nvim` will use the sandbox configs.

## Vimscript or Lua?
I previously had my Neovim config in Lua, since that's the thing I figured you
were supposed to do. However, I'm not that comfortable with Lua and I actually
had a hard time reading my old config to pick out things I wanted to keep. So
the new config will use Vimscript, because I'm more familiar with it and it
feels better as a config language. I may revisit this decision next time I feel
like redoing my setup and if I become more comfortable with Lua.

## init.vim
My old `init.lua` was one big file with all settings, mappings, plugins and
stuff just shoved in there. I didn't want this to be the way the new setup
worked, because it made it harder to find and tweak certain settings when I felt
like I needed to. Instead, `init.vim` will just dispatch the setup to other
`.vim` files with the `runtime!` command:

```vim
runtime! options.vim
runtime! mappings.vim
runtime! plugins.vim
runtime! plugin-settings/**/*.vim
runtime! plugin-settings/**/*.lua
```

These files then actually do the setup their name suggests they do.

## File structure
In the config directory, the four files `init.vim`, `mappings.vim`,
`options.vim`, and `plugins.vim` reside along with the directory
`plugin-settings/`, where every plugin has its own config file (in Vimscript or
Lua depending on the plugin). This way I get nice separation of global config
and plugin-specific config. Any other vim runtime directories -- like `ftplugin`
for filetype-specific config -- are also here. Here's a directory tree:

```
$XDG_CONFIG_HOME/nvim/
├── ftplugin/
│   └── rust/
│       ├── rust_mappings.vim
│       └── rust.vim
├── plugin-settings/
│   ├── foldmk.vim
│   ├── gruvbox-material.vim
│   ├── nvim-cmp.lua
│   ├── nvim-treesitter.lua
│   ├── rust-tools.lua
│   └── telescope.lua
├── init.vim
├── mappings.vim
├── options.vim
└── plugins.vim
```

## Plugins
I usually only use a handful of plugins if any at all. It sort of goes in cycles
of completely spartan zero plugin configs and full-blown "make-vim-into-an-ide"
configs. But I wanted to test plugins in the sandbox, so I added
[vim-plug](https://github.com/junegunn/vim-plug) into the proper
`$XDG_DATA_HOME` subdirectory:

```
$XDG_DATA_HOME/nvim/
└── site/
    └── autoload/
        └── plug.vim
```
Then I could install any plugins I wanted to try in the `plugins.vim` config
file, with configs in `plugin-settings/`.

## Editing configs when they're spread out
One might think that not having everything in one file will make it harder to
remember what the files were called exactly and take longer to find, but this is
not the case. I have a mapping in Neovim, `<Leader>e`, which opens `init.vim` in
a vertical split. Now this file only contains `runtime!` commands, but putting
the cursor on any of the filenames and hitting `gf` in normal mode will open
that config file. This even works for the `plugin-settings/` directory, opening
the directory in `netrw` so I can browse to the right plugin's config file. I
don't have to remember or type in any filenames. It's pretty neat.

## Version control
This time I've actually put my Neovim config under version control. It's
available [here](https://github.com/martinkauppinen/neovim-config).
