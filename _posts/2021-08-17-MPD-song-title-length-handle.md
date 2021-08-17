---
layout: post
---

# Fixes to the song title crop problem in current sidebar configuration

Before this update to my AwesomeWM configuration (which is a clone from [@elenapan's config](https://github.com/elenapan/dotfiles)),
the song title which the widget shows will crop.

I want to see the whole song title, to be honest.

So, let's get started.

## The fix

Initially, it was quite hard to build a method that updates the widget container to scroll the text because
I hadn't find a method that can get the rendered size of a `textbox` widget. After some digging in the API docs,
I found that `textbox:get_preferred_size(s)` will actually give rendered size of a `textbox` then it's string length.

## Code change

First, we'll change the original `mpd_title` child widget as an layouted container widget:

```lua
local mpd_title = wibox.widget{
    halign = "center",
    valign = "center",
    fill_horizontal = true,
    layout = wibox.container.place
}
```

Then add two widgets: a `textbox` widget, which contains the actual title text, and a `scroll` container contains the `textbox` widget:

```lua
local title_widget = wibox.widget{
    text = "---------",
    align = "center",
    valign = "center",
    widget = wibox.widget.textbox,
    font = "Noto Sans CJK TC Medium 14"
}
local rolling = wibox.widget{
    title_widget,
    step_function = wibox.container.scroll.step_functions.nonlinear_back_and_forth,
    speed = 50,
    fps = 60,
    layout = wibox.container.scroll.horizontal
}
```

Next, to the signal handler, we need to dynamically set what `mpd_title` will contain depending on the `title_widget`'s rendered size:

```lua
title_widget.markup =
    "<span foreground='" .. title_fg .."'>"
    .. title .. "</span>" -- Set title
local title_size = title_widget:get_preferred_size(1) -- Get rendered size
-- Dynamic widget setting, remember to put the title widget at
-- the bottom of mpd_song or it'll get complex
if title_size < dpi(250) then
    mpd_title:set_widget(title_widget)
else
    mpd_title:set_widget(rolling)
end
```

Also mention that `mpd_title` need to be put at the last of its mother container because `w:get_all_children()` will get all children widget
recursively, it'll be more convenient to simply put it at the bottom to avoid tedious task when extracting it.
