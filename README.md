<div align="center">

# 🧠 `smart-splits.nvim`

</div>

🧠 Smarter and more intuitive split pane management that uses a mental model of left/right/up/down
instead of wider/narrower/taller/shorter for resizing. Supports seamless navigation between Neovim and terminal
multiplexer split panes. See [Multiplexer Integrations](#multiplexer-integrations).

![demo](https://user-images.githubusercontent.com/8648891/201928611-4338e3cb-cca9-4e15-92c6-0405b7072279.gif)

**Table of Contents**

- [Install](#install)
- [Configuration](#configuration)
  - [Hooks](#hooks)
    - [Examples:](#examples)
- [Usage](#usage)
  - [Multiplexer Integrations](#multiplexer-integrations)
    - [Tmux](#tmux)
    - [Wezterm](#wezterm)
    - [Kitty](#kitty)
      - [Credits](#credits)
    - [Multiplexer Lua API](#multiplexer-lua-api)

## Install

`smart-splits.nvim` now supports semantic versioning via git tags. See [Releases](https://github.com/mrjones2014/smart-splits.nvim/releases)
for a full list of versions and their changelogs, starting from 1.0.0.

With Packer.nvim:

```lua
use('mrjones2014/smart-splits.nvim')
-- or use a specific version
use({ 'mrjones2014/smart-splits.nvim', tag = 'v1.0.0' })
-- to use Kitty multiplexer support, run the post install hook
use({ 'mrjones2014/smart-splits.nvim', run = './kitty/install-kittens.bash' })
```

With Lazy.nvim:

```lua
{ 'mrjones2014/smart-splits.nvim' }
-- or use a specific version, or a range of versions using lazy.nvim's version API
{ 'mrjones2014/smart-splits.nvim', version = '>=1.0.0' }
-- to use Kitty multiplexer support, run the post install hook
{ 'mrjones2014/smart-splits.nvim', build = './kitty/install-kittens.bash' }
```

## Configuration

You can set ignored `buftype`s or `filetype`s which will be ignored when
figuring out if your cursor is currently at an edge split for resizing.
This is useful in order to ignore "sidebar" type buffers while resizing,
such as [nvim-tree.lua](https://github.com/kyazdani42/nvim-tree.lua)
which tries to maintain its own width unless manually resized. Note that
nothing is ignored when moving between splits, only when resizing.

> **Note** `smart-splits.nvim` does not map any keys on it's own. See [Usage](#usage).

Defaults are shown below:

```lua
require('smart-splits').setup({
  -- Ignored buffer types (only while resizing)
  ignored_buftypes = {
    'nofile',
    'quickfix',
    'prompt',
  },
  -- Ignored filetypes (only while resizing)
  ignored_filetypes = { 'NvimTree' },
  -- the default number of lines/columns to resize by at a time
  default_amount = 3,
  -- Desired behavior when your cursor is at an edge and you
  -- are moving towards that same edge:
  -- 'wrap' => Wrap to opposite side
  -- 'split' => Create a new split in the desired direction
  -- 'stop' => Do nothing
  -- function => You handle the behavior yourself
  -- NOTE: If using a function, the function will be called with
  -- a context object with the following fields:
  -- {
  --    mux = {
  --      type:'tmux'|'wezterm'|'kitty'
  --      current_pane_id():number,
  --      is_in_session(): boolean
  --      current_pane_is_zoomed():boolean,
  --      -- following methods return a boolean to indicate success or failure
  --      current_pane_at_edge(direction:'left'|'right'|'up'|'down'):boolean
  --      next_pane(direction:'left'|'right'|'up'|'down'):boolean
  --      resize_pane(direction:'left'|'right'|'up'|'down'):boolean
  --      split_pane(direction:'left'|'right'|'up'|'down',size:number|nil):boolean
  --    },
  --    direction = 'left'|'right'|'up'|'down',
  --    split(), -- utility function to split current Neovim pane in the current direction
  --    wrap(), -- utility function to wrap to opposite Neovim pane
  -- }
  -- NOTE: `at_edge = 'wrap'` is not supported on Kitty terminal
  -- multiplexer, as there is no way to determine layout via the CLI
  at_edge = 'wrap',
  -- when moving cursor between splits left or right,
  -- place the cursor on the same row of the *screen*
  -- regardless of line numbers. False by default.
  -- Can be overridden via function parameter, see Usage.
  move_cursor_same_row = false,
  -- whether the cursor should follow the buffer when swapping
  -- buffers by default; it can also be controlled by passing
  -- `{ move_cursor = true }` or `{ move_cursor = false }`
  -- when calling the Lua function.
  cursor_follows_swapped_bufs = false,
  -- resize mode options
  resize_mode = {
    -- key to exit persistent resize mode
    quit_key = '<ESC>',
    -- keys to use for moving in resize mode
    -- in order of left, down, up' right
    resize_keys = { 'h', 'j', 'k', 'l' },
    -- set to true to silence the notifications
    -- when entering/exiting persistent resize mode
    silent = false,
    -- must be functions, they will be executed when
    -- entering or exiting the resize mode
    hooks = {
      on_enter = nil,
      on_leave = nil,
    },
  },
  -- ignore these autocmd events (via :h eventignore) while processing
  -- smart-splits.nvim computations, which involve visiting different
  -- buffers and windows. These events will be ignored during processing,
  -- and un-ignored on completed. This only applies to resize events,
  -- not cursor movement events.
  ignored_events = {
    'BufEnter',
    'WinEnter',
  },
  -- enable or disable a multiplexer integration;
  -- automatically determined, unless explicitly disabled or set,
  -- by checking the $TERM_PROGRAM environment variable,
  -- and the $KITTY_LISTEN_ON environment variable for Kitty
  multiplexer_integration = nil,
  -- disable multiplexer navigation if current multiplexer pane is zoomed
  -- this functionality is only supported on tmux and Wezterm due to kitty
  -- not having a way to check if a pane is zoomed
  disable_multiplexer_nav_when_zoomed = true,
  -- Supply a Kitty remote control password if needed,
  -- or you can also set vim.g.smart_splits_kitty_password
  -- see https://sw.kovidgoyal.net/kitty/conf/#opt-kitty.remote_control_password
  kitty_password = nil,
  -- default logging level, one of: 'trace'|'debug'|'info'|'warn'|'error'|'fatal'
  log_level = 'info',
})
```

### Hooks

The hook table allows you to define callbacks for the `on_enter` and `on_leave` events of the resize mode.

##### Examples

Integration with [bufresize.nvim](https://github.com/kwkarlwang/bufresize.nvim):

```lua
require('smart-splits').setup({
  resize_mode = {
    hooks = {
      on_leave = require('bufresize').register,
    },
  },
})
```

Custom messages when using resize mode:

```lua
require('smart-splits').setup({
  resize_mode = {
    silent = true,
    hooks = {
      on_enter = function()
        vim.notify('Entering resize mode')
      end,
      on_leave = function()
        vim.notify('Exiting resize mode, bye')
      end,
    },
  },
})
```

## Usage

### Key Mappings

If you are a [legendary.nvim](https://github.com/mrjones2014/legendary.nvim) (>= v2.10.0) user, you can
quickly easily and easily create with the `legendary.nvim` extension for `smart-splits.nvim`. See more
option in the [extension documentation in `legendary.nvim`](https://github.com/mrjones2014/legendary.nvim/blob/master/doc/EXTENSIONS.md#smart-splitsnvim).

```lua
require('legendary').setup({
  extensions = {
    -- to use default settings:
    smart_splits = {},
    -- default settings shown below:
    smart_splits = {
      directions = { 'h', 'j', 'k', 'l' },
      mods = {
        -- for moving cursor between windows
        move = '<C>',
        -- for resizing windows
        resize = '<M>',
        -- for swapping window buffers
        swap = false, -- false disables creating a binding
      },
    },
    -- or, customize the mappings
    smart_splits = {
      mods = {
        -- any of the mods can also be a table of the following form
        swap = {
          -- this will create the mapping like
          -- <leader><C-h>
          -- <leader><C-j>
          -- <leader><C-k>
          -- <leader><C-l>
          mod = '<C>',
          prefix = '<leader>',
        },
      },
    },
  },
})
```

Otherwise, here are some recommended mappings.

```lua
-- recommended mappings
-- resizing splits
-- these keymaps will also accept a range,
-- for example `10<A-h>` will `resize_left` by `(10 * config.default_amount)`
vim.keymap.set('n', '<A-h>', require('smart-splits').resize_left)
vim.keymap.set('n', '<A-j>', require('smart-splits').resize_down)
vim.keymap.set('n', '<A-k>', require('smart-splits').resize_up)
vim.keymap.set('n', '<A-l>', require('smart-splits').resize_right)
-- moving between splits
vim.keymap.set('n', '<C-h>', require('smart-splits').move_cursor_left)
vim.keymap.set('n', '<C-j>', require('smart-splits').move_cursor_down)
vim.keymap.set('n', '<C-k>', require('smart-splits').move_cursor_up)
vim.keymap.set('n', '<C-l>', require('smart-splits').move_cursor_right)
-- swapping buffers between windows
vim.keymap.set('n', '<leader><leader>h', require('smart-splits').swap_buf_left)
vim.keymap.set('n', '<leader><leader>j', require('smart-splits').swap_buf_down)
vim.keymap.set('n', '<leader><leader>k', require('smart-splits').swap_buf_up)
vim.keymap.set('n', '<leader><leader>l', require('smart-splits').swap_buf_right)
```

### Lua API

```lua
-- resizing splits
-- amount defaults to 3 if not specified
-- use absolute values, no + or -
-- the functions also check for a range,
-- so for example if you bind `<A-h>` to `resize_left`,
-- then `10<A-h>` will `resize_left` by `(10 * config.default_amount)`
require('smart-splits').resize_up(amount)
require('smart-splits').resize_down(amount)
require('smart-splits').resize_left(amount)
require('smart-splits').resize_right(amount)
-- moving between splits
-- You can override config.at_edge and
-- config.move_cursor_same_row via opts
-- See Configuration.
require('smart-splits').move_cursor_up({ same_row = boolean, at_edge = 'wrap' | 'split' | 'stop' })
require('smart-splits').move_cursor_down()
require('smart-splits').move_cursor_left()
require('smart-splits').move_cursor_right()
-- Swapping buffers directionally with the window to the specified direction
require('smart-splits').swap_buf_up()
require('smart-splits').swap_buf_down()
require('smart-splits').swap_buf_left()
require('smart-splits').swap_buf_right()
-- the buffer swap functions can also take an `opts` table to override the
-- default behavior of whether or not the cursor follows the buffer
require('smart-splits').swap_buf_right({ move_cursor = true })
-- persistent resize mode
-- temporarily remap your configured resize keys to
-- smart resize left, down, up, and right, respectively,
-- press <ESC> to stop resize mode (unless you've set a different key in config)
-- resize keys also accept a range, e.e. pressing `5j` will resize down 5 times the default_amount
require('smart-splits').start_resize_mode()
```

### Multiplexer Integrations

`smart-splits.nvim` can also enable seamless navigation between Neovim splits and `tmux`, `wezterm`, or `kitty` panes.
You will need to set up keymaps in your `tmux`, `wezterm`, or `kitty` configs to match the Neovim keymaps.

#### Tmux

Add the following snippet to your `~/.tmux.conf`/`~/.config/tmux/tmux.conf` file (customizing the keys and resize amount if desired):

```tmux
# Smart pane switching with awareness of Vim splits.
"
bind-key -n C-h if -F "#{@pane-is-vim}" 'send-keys C-h'  'select-pane -L'
bind-key -n C-j if -F "#{@pane-is-vim}" 'send-keys C-j'  'select-pane -D'
bind-key -n C-k if -F "#{@pane-is-vim}" 'send-keys C-k'  'select-pane -U'
bind-key -n C-l if -F "#{@pane-is-vim}" 'send-keys C-l'  'select-pane -R'

bind-key -n M-h if -F "#{@pane-is-vim}" 'send-keys M-h' 'resize-pane -L 3'
bind-key -n M-j if -F "#{@pane-is-vim}" 'send-keys M-j' 'resize-pane -D 3'
bind-key -n M-k if -F "#{@pane-is-vim}" 'send-keys M-k' 'resize-pane -U 3'
bind-key -n M-l if -F "#{@pane-is-vim}" 'send-keys M-l' 'resize-pane -R 3'

tmux_version='$(tmux -V | sed -En "s/^tmux ([0-9]+(.[0-9]+)?).*/\1/p")'
if-shell -b '[ "$(echo "$tmux_version < 3.0" | bc)" = 1 ]' \
    "bind-key -n 'C-\\' if -F \"#{@pane-is-vim}\" 'send-keys C-\\'  'select-pane -l'"
if-shell -b '[ "$(echo "$tmux_version >= 3.0" | bc)" = 1 ]' \
    "bind-key -n 'C-\\' if -F \"#{@pane-is-vim}\" 'send-keys C-\\\\'  'select-pane -l'"

bind-key -T copy-mode-vi 'C-h' select-pane -L
bind-key -T copy-mode-vi 'C-j' select-pane -D
bind-key -T copy-mode-vi 'C-k' select-pane -U
bind-key -T copy-mode-vi 'C-l' select-pane -R
bind-key -T copy-mode-vi 'C-\' select-pane -l
```

#### Wezterm

> **Note**
> It is recommended _not to lazy load_ `smart-splits.nvim` if using the Wezterm integration.
> If you need to lazy load, you need to use a different `is_vim()` implementation below.
> The plugin is small, and smart about not loading modules unnecessarily, so it should
> have minimal impact on your startup time. It adds about 0.07ms on my setup.

> **Note**
> Pane resizing currently requires a nightly build of Wezterm.
> Check the output of `wezterm cli adjust-pane-size --help` to see if your build supports it; if not,
> you can check how to obtain a nightly build by [following the instructions here](https://wezfurlong.org/wezterm/installation.html).

First, ensure that the `wezterm` CLI is on your `$PATH`, as the CLI is used by the integration.

Then, if you're on Wezterm nightly, you can use Wezterm's [experimental plugin loader](https://github.com/wez/wezterm/commit/e4ae8a844d8feaa43e1de34c5cc8b4f07ce525dd):

```lua
local wezterm = require('wezterm')
local smart_splits = wezterm.plugin.require('https://github.com/mrjones2014/smart-splits.nvim')
local config = wezterm.config_builder()
-- you can put the rest of your Wezterm config here
smart_splits.apply_to_config(config, {
  -- the default config is here, if you'd like to use the default keys,
  -- you can omit this configuration table parameter and just use
  -- smart_splits.apply_to_config(config)

  -- directional keys to use in order of: left, down, up, right
  direction_keys = { 'h', 'j', 'k', 'l' },
  -- modifier keys to combine with direction_keys
  modifiers = {
    move = 'CTRL', -- modifier to use for pane movement, e.g. CTRL+h to move left
    resize = 'META', -- modifier to use for pane resize, e.g. META+h to resize to the left
  },
})
```

Otherwise, add the following snippet to your `~/.config/wezterm/wezterm.lua`:

```lua
local w = require('wezterm')

-- if you are *NOT* lazy-loading smart-splits.nvim (recommended)
local function is_vim(pane)
  -- this is set by the plugin, and unset on ExitPre in Neovim
  return pane:get_user_vars().IS_NVIM == 'true'
end

-- if you *ARE* lazy-loading smart-splits.nvim (not recommended)
-- you have to use this instead, but note that this will not work
-- in all cases (e.g. over an SSH connection). Also note that
-- `pane:get_foreground_process_name()` can have high and highly variable
-- latency, so the other implementation of `is_vim()` will be more
-- performant as well.
local function is_vim(pane)
  -- This gsub is equivalent to POSIX basename(3)
  -- Given "/foo/bar" returns "bar"
  -- Given "c:\\foo\\bar" returns "bar"
  local process_name = string.gsub(pane:get_foreground_process_name(), '(.*[/\\])(.*)', '%2')
  return process_name == 'nvim' or process_name == 'vim'
end

local direction_keys = {
  h = 'Left',
  j = 'Down',
  k = 'Up',
  l = 'Right',
}

local function split_nav(resize_or_move, key)
  return {
    key = key,
    mods = resize_or_move == 'resize' and 'META' or 'CTRL',
    action = w.action_callback(function(win, pane)
      if is_vim(pane) then
        -- pass the keys through to vim/nvim
        win:perform_action({
          SendKey = { key = key, mods = resize_or_move == 'resize' and 'META' or 'CTRL' },
        }, pane)
      else
        if resize_or_move == 'resize' then
          win:perform_action({ AdjustPaneSize = { direction_keys[key], 3 } }, pane)
        else
          win:perform_action({ ActivatePaneDirection = direction_keys[key] }, pane)
        end
      end
    end),
  }
end

return {
  keys = {
    -- move between split panes
    split_nav('move', 'h'),
    split_nav('move', 'j'),
    split_nav('move', 'k'),
    split_nav('move', 'l'),
    -- resize panes
    split_nav('resize', 'h'),
    split_nav('resize', 'j'),
    split_nav('resize', 'k'),
    split_nav('resize', 'l'),
  },
}
```

#### Kitty

> **Note** `config.at_edge = 'wrap'` is not supoprted in Kitty terminal multiplexer due to inability to determine
> pane layout from CLI.

> **Note**
> This won't work if the pane is connected over SSH, as the pane will not properly report the foreground process name.

Add the following snippet to `~/.config/kitty/kitty.conf`, adjusting the keymaps and resize amount as desired.

```
map ctrl+j kitten pass_keys.py neighboring_window bottom ctrl+j
map ctrl+k kitten pass_keys.py neighboring_window top    ctrl+k
map ctrl+h kitten pass_keys.py neighboring_window left   ctrl+h
map ctrl+l kitten pass_keys.py neighboring_window right  ctrl+l

# the 3 here is the resize amount, adjust as needed
map alt+j kitten pass_keys.py relative_resize down  3 alt+j
map alt+k kitten pass_keys.py relative_resize up    3 alt+k
map alt+h kitten pass_keys.py relative_resize left  3 alt+h
map alt+l kitten pass_keys.py relative_resize right 3 alt+l
```

By default, it matches against the name of the current foreground process to detect if `vim`/`nvim` is running.
If that doesn't work for you, or you want to include other CLI/TUI programs in the exclusion, you can provide
an additional regex argument:

```
map ctrl+j kitten pass_keys.py neighboring_window bottom ctrl+j "^.* - nvim$"
map ctrl+k kitten pass_keys.py neighboring_window top    ctrl+k "^.* - nvim$"
map ctrl+h kitten pass_keys.py neighboring_window left   ctrl+h "^.* - nvim$"
map ctrl+l kitten pass_keys.py neighboring_window right  ctrl+l "^.* - nvim$"

# the 3 here is the resize amount, adjust as needed
map alt+j kitten pass_keys.py relative_resize down  3 alt+j "^.* - nvim$"
map alt+k kitten pass_keys.py relative_resize up    3 alt+k "^.* - nvim$"
map alt+h kitten pass_keys.py relative_resize left  3 alt+h "^.* - nvim$"
map alt+l kitten pass_keys.py relative_resize right 3 alt+l "^.* - nvim$"
```

Then, you must allow Kitty to listen for remote commands on a socket. You can do this
either by running Kitty with the following command:

```bash
# For linux only:
kitty -o allow_remote_control=yes --single-instance --listen-on unix:@mykitty

# Other unix systems:
kitty -o allow_remote_control=yes --single-instance --listen-on unix:/tmp/mykitty
```

Or, by adding the following to `~/.config/kitty/kitty.conf`:

```
# For linux only:
allow_remote_control yes
listen_on unix:@mykitty

# Other unix systems:
allow_remote_control yes
listen_on unix:/tmp/mykitty
```

##### Credits

Thanks @knubie for inspiration for the Kitty implementation from [vim-kitty-navigator](https://github.com/knubie/vim-kitty-navigator).

Thanks to @chancez for the relative resize [Python kitten](https://github.com/chancez/dotfiles/blob/badc69d3895a6a942285126b8c372a55d77533e1/kitty/.config/kitty/relative_resize.py).

### Multiplexer Lua API

You can directly access the multiplexer API for scripting purposes as well.
To get a handle to the current multiplexer backend, you can do:

```lua
local mux = require('smart-splits.mux').get()
```

This returns the currently enabled multiplexer backend, or `nil` if none is currently in use.
The API offers the following methods:

```lua
local mux = require('smart-splits.mux').get()
-- mux matches the following type annotations
---@class SmartSplitsMultiplexer
---@field current_pane_id fun():number|nil
---@field current_pane_at_edge fun(direction:'left'|'right'|'up'|'down'):boolean
---@field is_in_session fun():boolean
---@field current_pane_is_zoomed fun():boolean
---@field next_pane fun(direction:'left'|'right'|'up'|'down'):boolean
---@field resize_pane fun(direction:'left'|'right'|'up'|'down', amount:number):boolean
---@field split_pane fun(direction:'left'|'right'|'up'|'down',size:number|nil):boolean
---@field type 'tmux'|'wezterm'|'kitty'
```
