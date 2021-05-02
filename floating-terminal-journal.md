# how to compose your own dmenu without having to write it yourself
## the Pledge
### dmenu is amazing
[dmenu](https://wiki.archlinux.org/title/Dmenu) is a super useful tool. <explain>

### But unfortunately I can't use it
* it runs on X. The whole reason I'm on Wayland is to get away from X, so I need a solution that isn't tied to Xserver I could use it as an Xwayland script, but my goal is eventually to not have any more Xserver programs, so I figure I'd rather fix this before I write more scripts using dmenu
* even if I concede that I can use `dmenu` as an Xwayland script, it has some weird behavior
    - it's transparent even though I don't want it to be. <add screenshot>
    - it isn't selectable via the keyboard because it doesn't play by wayland's rules. so it becomes super annoying if I focus a different window then try to go back to dmenu. I have to use the mouse. it should either be focusable from the keyboard, or it should close itself if it loses focus
    - I have to do a LOT of patching to get it to work the way I want. This is super cool, but every time I run into merge conflicts, it makes me super miserable and I have to crawl through this "super readable" codebase that has very few comments/documentation and every variable is 1-3 chars long.

### Alternatives don't behave the way I want them to
There are a lot of alternatives to `dmenu` in Wayland, particularly because everyone else also feels similarly to me. I think that's a beautiful aspect of open-source software. Everyone gets a place in the community's destiny and if there's something that doesn't work the way that you would prefer, you can change it yourself. You could fork the repo and make small changes, or you could just start an entirely new project.
I considered many many alternatives and pored over their (often lacking) documentation and nothing behaved the way I wanted:
* [gmenu](https://code.rocketnine.space/tslocum/gmenu) doesn't seem to have fuzzy-finding and it seems like it's purely a program launcher
* [bemenu](https://github.com/Cloudef/bemenu) doesn't have fuzzy-finding
* [yofi](https://github.com/l4l/yofi) is still in active development and lacks documentation. It also pointed out the issue that fuzzy search [is really hard to find](https://www.reddit.com/r/swaywm/comments/kgdc8r/yofi_a_minimalistic_application_launcher_for/) in `dmenu` look-alikes
* [wofi](https://hg.sr.ht/~scoopta/wofi) documentation is lacking and it must have a different understanding of "fuzzy-matching" because it doesn't actually do [substring matching](google.com) like I expect it to. <screenshot here>
* [ulauncher](https://github.com/Ulauncher/Ulauncher/). it's unclear whether or not this is a pure launcher or it supports "dmenu mode"

### I could build my own, but i don't _wanna_
I _know_ `fzf` behaves the way I want it to. I don't want to write my own version of `dmenu` and I don't want to learn a new system that someone else implemented differently. Some alternatives make you style using CSS or they only expose certain configuration options. 
I would have to add new dotfiles.
I would have to learn new keybindings
I would have to learn a whole new system and maybe even learn the writer's language of choice just to make the `dmenu` clone work the way I want it to
Why should I have to do so much learning and file-writing and add more configurations to my dotfiles when I have a perfectly good program that will look and behave consistently with all of my in-terminal `fzf` commands. 

### my motivation and goal
So, I will compose my own `dmenu` using nothing but a terminal, fzf, and shell commands.

## building the floating-terminal script
### tools I use
If you'd like to follow along, these are the programs you'll need to have installed. If not, that's ok, but it's helpful to have a basic one-line understanding of what they each do.
* sway
* grim
* slurp
* jq
* alacritty
* notifylib
* mako
* cat
#todo: include descriptions and links here

### the driving script
I want to use `dmenu` to select from a few options so that I can take the right screenshot. I have 3 types of screenshots that I want to take: the whole screen, a whole window, or a user-defined rectangular selection.

The actual commands in the `case` statement are unimportant; all you need to know is that they'll run the appropriate commands for the chosen screenshot mode. However, they're cool examples of minimal and composable utilities to read about if you're interested: [grim](https://github.com/emersion/grim), [slurp](https://github.com/emersion/slurp), [jq](https://stedolan.github.io/jq/)
Also I use [notify-send](https://github.com/GNOME/libnotify) (with [mako](https://github.com/emersion/mako) in the background) to give the user notifications so that there's some visual feedback that things are working.

Here's the script:
```sh
# ~/bin/screenshot-cmd

# prompt the user to select a screenshot mode then perform the screenshot in the given mode

# these are the different modes for taking a screenshot
screenshot_options=(screen window selection)
# prompt user to pick a mode
screenshot_type=$(echo ${screenshot_options[*]} | tr "[:space:]" '\n' | fzf)
notify-send "Screenshot" "Screenshotting ${screenshot_type}" --expire-time=1500

case "$screenshot_type" in
    "screen")
        # capture the whole screen
        grim -c
        ;;
    "selection")
        # use slurp to let the user manually draw a rectangle
        grim -g "$(slurp)" -c
        ;;
    "window")
        # Use slurp to get the borders of the window under the cursor
        window_borders=$(\
            swaymsg -t get_tree \
                | jq -r '.. | select(.pid? and .visible?) | .rect | "\(.x),\(.y) \(.width)x\(.height)"' \
                | slurp
        )
        grim -g "${window_borders}" -c
        ;;
    *)
        exit 1
        ;;
esac

notify-send --expire-time=2500 "grim" "💻 Screenshot captured!"
```

This script works well, but it requires an interactive terminal for `fzf` to work. This is pretty unwieldy because that means I'll have to open a new terminal every time I want to take a screenshot. Also, I'll have an extra terminal hanging around that may cover or shrink the windows that I want to capture. <insert picture>

### sway configuration
I have a rule set for sway that will automatically set any window to "floating" mode if its name starts with "!". This means that it will be displayed on top of other windows and won't be manipulated by any of the tiling constraints.

```sh
# ~/.config/sway/config

...

# float any windows whose titles start with !
for_window [title="^!.*$"] floating enable, border pixel 5
```

### first draft
```sh
# ~/bin/floating-terminal

# open a floating window and run the script given as an argument
alacritty --title '!terminal' -e sh -c "${1}"
```

```sh
# ~/bin/screenshot

# use a floating terminal to prompt the user to take a screenshot
floating-terminal screenshot-cmd
```

### problems
This isn't a big deal, but knowing the `dmenu` way of doing things, it feels like I should have 1 screenshot script, not 2. The bigger problem is that the floating window stays open during the screenshot and clutters/covers the resulting capture. <add picture> I want the `dmenu` behavior where it gets out of the way. the dmenu stage of the process should be decoupled from the screenshot stage
Right now it looks like this:
1. use fzf to pick a screenshot mode
1. allow user to select a window or custom rectangular selection (if necessary)
1. take a screenshot
1. close the floating window

It should look like this:
1. use fzf to pick a screenshot mode
1. close the floating window
1. allow user to select a window or custom rectangular selection (if necessary)
1. take a screenshot

### idea
If I can truly emulate `dmenu` by returning result of fzf, my problems would be solved. Assume I wrote `floating-fzf` that behaves exactly the same as `dmenu`. 
* I can consolidate both screenshot scripts into a single script
* The floating window will be closed by the time I take the screenshot
* I can use `floating-fzf` as a drop-in replacement for `dmenu` everywhere

This would be the final script (only line 10 changed):

```sh
#!/bin/sh

# prompt the user to select a screenshot mode then perform the screenshot in the given mode

# these are the different modes for taking a screenshot
screenshot_options=(screen window selection)
# prompt user to pick a mode
screenshot_type=$(echo ${screenshot_options[*]} | tr "[:space:]" '\n' | floating-fzf)
notify-send "Screenshot" "Screenshotting ${screenshot_type}" --expire-time=1500

case "$screenshot_type" in
    "screen")
        # capture the whole screen
        grim -c
        ;;
    "selection")
        # use slurp to let the user manually draw a rectangle
        grim -g "$(slurp)" -c
        ;;
    "window")
        # Use slurp to get the borders of the window under the cursor
        window_borders=$(\
            swaymsg -t get_tree \
                | jq -r '.. | select(.pid? and .visible?) | .rect | "\(.x),\(.y) \(.width)x\(.height)"' \
                | slurp
        )
        grim -g "${window_borders}" -c
        ;;
    *)
        exit 1
        ;;
esac

notify-send --expire-time=2500 "grim" "💻 Screenshot captured!"
```

Ultimately, we need to get the output of `fzf` and somehow write it to `Alacritty's` parent process on `stdout`. But how do we do that?? `Alacritty` doesn't seem to have any options for that. I mean, why would they? It's a terminal emulator and its `stdout` and `stderr` should only contain messages that are relevant to `Alacritty`'s runtime state, not aritrary messages from users. And who would ever pipe from `Alacritty` to another process? They'd have to be a _really_ picky person, like the kind of person that refuses to use out-of-the-box solutions and writes a guide about it.

### research and available tools
#### process substitution
[Process substitution](https://www.gnu.org/software/bash/manual/html_node/Process-Substitution.html) (AKA "anonymous pipe") seems promising. It writes a process's output as a filename and returns it to the parent process for reading. So in my head, this should be what we're looking for:
```sh
# ~/bin/floating-terminal

# open a floating window and run the script given as an argument
alacritty --title '!terminal' -e sh -c "cat <(${1})"
```
Instead of just running the command, we run the command inside a `process substitution` statement and `cat` the file to write it to `stdout`. For some reason, this doesn't work and I can't figure out how to debug it.

#### FIFO 
A First-In-First-Out file (AKA "named pipe") is a special file that is similar to a pipe. It has [FIFO behavior](https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)), so it has "first-come, first-served" behavior like a line at a bank or a grocery store. Whoever arrives first should get served first. Basically, it means that when we write messages to the pipe, the first messages written will be the first messages read. In our case, we use such little data (1 string of text that will be "selection", "window", or "screen") that the pipe won't fill up and cause a delay. But if there _was_ delay, we would know that the messages would come out in FIFO order.

This was my first time learning about FIFO files, so my initial reaction was
pretty hesitant. What if we have multiple `floating-fzf` processes and run into
a [race condition](https://www.baeldung.com/cs/race-conditions)? Perhaps we
could make each `floating-fzf` process use a unique id for the FIFO so that
they don't share FIFOs, but that
feels like it's much more complicated than it should be. For our purposes of
screenshotting, the FIFO lives for such a short time that I think we'll be OK.
Besides, the only reason a user _should_ open a `floating-fzf` instance is for
immediate contextual user-driven input, so I don't think we should _ever_ have
multiple instances living at the same time. However, knowing that this problem
could come up is a cornerstone of good engineering, and it's why experienced
battle-hardened engineers are so darn great and wise, so I still think it's good 
to consider this issue for the future.


## Footnotes
for more reading on stdout and stderr, read this
for more info on pipes, read this
