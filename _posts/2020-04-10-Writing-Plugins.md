---
layout: post
title: "Writing Wayfire plugins (Part 1)"
---

If someone were to ask me what is the greatest aspect of Wayfire, then this would undoubtedly be its customizability and modularity.
There are a ton of options you can tweak to your liking.
However, options are rarely enough if you want to make your desktop truly your own.
And this is why Wayfire uses a <span class="highlight">plugin architecture</span>.
<!--more-->

Plugins in Wayfire are very powerful, and in fact, they provide most of the user interface.
This means that by changing them, you can change almost every aspect of the compositor.
Just an example: if you click and drag a window to move it across the screen, this behavior is controlled by the `move` plugin.
If you use `<alt> KEY_TAB`, then you see the switcher plugin - it has full control over where to draw windows, how to interpret input, etc.


Unfortunately, writing a plugin can be daunting in the beginning.
Wayfire's core exports many different structures, and usually to create even a simple plugin you need to understand the basic principles of Wayfire.
To make this easier, I have decided to write a series of blog posts where I will explain these basics by using examples - either by going through an
existing plugin, or by writing a new one, step by step.

## Enter primary-monitor-switch

In this first part of the blog series, I will explain how I wrote a plugin which serves a very specific function.
This plugin is intended mainly for my private usage, however others could find it useful as well.

#### Motivation

So, what do I want this plugin to do?
My main machine is a laptop. When I work, I use the 27' monitor on my desk.
I usually move all of my windows from the integrated display to the external monitor.

I would like to create a plugin which automates this process.
When the monitor is plugged in, this plugin would go through all workspaces on my integrated display and move the windows to the external one,
preserving which workspace each window is on.
When the monitor is unplugged, the plugin should simply reverse this operation.
Note that I do not want the integrated display to be completely turned off, because sometimes I still need it for temporary windows.
Seems simple, right? Let's start.

## Build setup

Before we write the actual code, we need to set up the build system.
Plugins can be developed both in-tree (as part of the Wayfire repository or your own fork), or out-of-tree.
For out-of-tree plugins, Wayfire provides a `wayfire.pc` file which contains the necessary information about where Wayfire and its headers are installed.

For building the actual source files, Wayfire uses [meson](https://mesonbuild.com/),
although you could use any other build system with pkg-config integration like CMake.

The quickest way to get started is to just get the structure from an existing project, for example from [this commit](https://github.com/ammen99/wayfire-plugins/commit/4894fe7c1e323eeb0c4f31b29686974b35198f40).
It contains the toplevel `meson.build` file, which declares the project name, version, languages, C++ standard (for Wayfire, use C++17), dependencies and so on.
This file also sets a few compiler constants.
There is also the `metadata` folder which is used for configuration options,
and `src/meson.build` which tells meson which source files to use to create plugins.

## Basic plugin

A plugin in Wayfire is represented by a class deriving from `wf::plugin_interface_t`. So, almost all plugins contain something like this:

```cpp
#include <wayfire/plugin.hpp>

class primary_monitor_switch_t : public wf::plugin_interface_t
{
  public:
    void init() override
    {
        /* Create plugin */
    }

    void fini() override
    {
        /* Destroy plugin */
    }
};

DECLARE_WAYFIRE_PLUGIN(primary_monitor_switch_t)
```

We create a class which implements the plugin interface. The only mandatory method to override is `init()`, however `fini()` is usually needed as well.
When loading a plugin, core will `dlopen` the plugin's `.so` file and look for particular symbols.
[`DECLARE_WAYFIRE_PLUGIN`](https://github.com/WayfireWM/wayfire/blob/3cbf1508a8cb95722ebcf55d3e5a0b3f766cbd55/src/api/wayfire/plugin.hpp#L183) is a macro which defines these.
Currently, core needs a function for allocating the plugin, and another one for checking versions.

## Actual logic

Outputs in Wayfire are completely independent (this includes plugins and workspaces). This allows you to have Expo active on one output, while
doing something completely different on the second one.

To facilitate this, a separate plugin instance is created for each output. Thus, we can easily detect when the external monitor is plugged in:

```cpp
#include <wayfire/output.hpp>

...

const std::string external_monitor = "HDMI-A-2";

void init()
{
    if (external_monitor == this->output->to_string())
    {
        /* External monitor is here! */
    }
}
```

First, we define the name of the external monitor, which we will make configurable later.
Second, we check `this->output->to_string()`. `this->output` is a pointer to the `wf::output_t` this plugin instance is created on, and is initialized
by core before `init()` is called.
`wf::output_t::to_string()` returns the name of the output.

So now, the actual logic of moving the views. Let's create a method for this:

```cpp
#include <wayfire/workspace-manager.hpp>

...

void move_views_to_output(wf::output_t *from, wf::output_t *to)
{
    assert(from && to);
    wf::dimensions_t grid_size = from->workspace->get_workspace_grid_size();
    static constexpr uint32_t interesting_layers =
        wf::LAYER_WORKSPACE | wf::LAYER_MINIMIZED | wf::LAYER_FULLSCREEN;

    for (int x = 0; x < grid_size.width; x++)
    {
        for (int y = 0; y < grid_size.height; y++)
        {
            auto views = from->workspace->get_views_on_workspace(
                {x, y}, interesting_layers, true);
            std::reverse(views.begin(), views.end());

            for (wayfire_view v : views)
                move_view(v, to, {x, y});
        }
    }

    to->workspace->request_workspace(
            from->workspace->get_current_workspace());
}
```

There are a few more things here.
Workspaces in Wayfire are arranged in a two dimensional grid.
With `from->workspace->get_workspace_grid_size()` we get this grid's size.
Views (another name for windows) are also arranged in layers.
There is a background layer, workspace layer, panel layer, etc.
Here we care about workspace, minimized and fullscreen layers because these are the layers that contain regular views.

For each workspace, we query the views on it in the aforementioned layers, and then move each view to the new output.
Note we first reverse views, because `get_views_on_workspace()` returns views from the top to bottom. However, when moving them,
we first want to put those in the bottom so that they are again below the others.

In the end, we switch the workspace on `to` to the workspace of `from`, so things look exactly the same as on `from`.

All of these functions are part of the workspace manager class. It is documentated
[here](https://github.com/WayfireWM/wayfire/blob/master/src/api/wayfire/workspace-manager.hpp).

Now comes the crux. How do we implement `move_view()`? Like this:

```cpp
#include <wayfire/core.hpp>
#include <wayfire/view.hpp>
#include <wayfire/plugins/common/view-change-viewport-signal.hpp>

...

void move_view(wayfire_view view, wf::output_t *to, wf::point_t target_ws)
{
    wf::get_core().move_view_to_output(view, to);
    wf::point_t current_ws = to->workspace->get_current_workspace();

    to->workspace->move_to_workspace(view, target_ws);
    if (current_ws != target_ws)
    {
        view_change_viewport_signal data;
        data.view = view;
        data.from = current_ws;
        data.to = target_ws;
        to->emit_signal("view-change-viewport", &data);
    }

    if (view->fullscreen)
        view->fullscreen_request(to, true);

    if (view->tiled_edges)
        view->tile_request(view->tiled_edges);
}
```

First off, we start by using a helper function from core which will re-assign the view's output.
Then, we make sure the view is visible on the target workspace.
Wayfire's core does not have the concept of views bound to a single workspace, because views can freely float between two or move workspaces.
However, some plugins (most notably `simple-tile`) can enforce this. To play nicely with this plugin, I have added a special signal with the help of which
other plugins can interact with `simple-tile`.

How does that work? `wf::output_t` is a `wf::signal_provider_t`. This means, plugins can connect to signals defined by a name on it, and other plugins can
emit these signals.

Note that this part is optional, however without it we will likely get a clash between `simple-tile` and `primary-monitor-switch` plugins.

Finally, we have to take care of fullscreened and maximized views, because the other output can have different size. Here, we just check whether the view
is fullscreen and/or tiled, and invoke the appropriate function.

## Are we done yet?

Nope, we have a few more things to do.
First off, we need to actually call our `move_views_to_output()` function:

```cpp
#include <wayfire/output-layout.hpp>

...

if (external_monitor == this->output->to_string())
{
    auto outputs = wf::get_core().output_layout->get_outputs();
    for (wf::output_t *other_output : outputs)
    {
        if (other_output != this->output)
            move_views_to_output(other_output, this->output);
    }
}
```

We need to extend our check we did in the beginning in the `init()` method so that we actually transfer views between the outputs.
There is nothing special here - we get the list of outputs from `wf::get_core().output_layout` which is responsible for multi-output management.
This is also the class you'd use if you want to move outputs, change resolution, rotation, etc.
After that, for every other output, we just move views from there to our output.

One last part remains, and that is restoring views back to the integrated display when the external output is changed.
We can achieve that in a similar way, by connecting to the `pre-remove` signal.
It is emitted on each output, right before the output is destroyed.

```cpp
wf::signal_connection_t on_pre_remove = {[=] (wf::signal_data_t*) {
    if (this->output->to_string() != (std::string)external_monitor)
        return;

    /* Find next output */
    auto outputs = wf::get_core().output_layout->get_outputs();
    wf::output_t *next_output = nullptr;
    for (auto other_output : outputs)
    {
        if (other_output != this->output)
            next_output = other_output;
    }

    if (next_output == nullptr)
    {
        /* Maybe we are running on a single output, or the compositor is
         * shutting down and there are no more outputs */
        return;
    }

    this->move_views_to_output(this->output, next_output);
}};
```

Nothing special here, except that we create a `wf::signal_connection_t` object and initialize it with a lambda function
which restores views to the other output.

Wayfire uses `wf::signal_connection_t` instead of the callback directly because `wf::signal_connection_t`
utilizes destructors to automatically disconnect from whichever signal(s) it is connected to.

```cpp
void init()
{
    ...

    this->output->connect_signal("pre-remove", &on_pre_remove);
}
```

And that's it, you can compile the plugin and install it, then append `primary-monitor-switch` to the end of your plugin list to see it work.
`primary-monitor-switch` comes from the name of the shared library that we specify in `src/meson.build`.

## Further improvements

There are two things that can be improved.

As mentioned in the beginning, we can (should) make the external output name configurable. To do that, we can create a file
[`metadata/primary-monitor-switch.xml`](https://github.com/ammen99/wayfire-plugins/blob/master/metadata/primary-monitor-switch.xml)
which contains descriptions of the options our plugin exposes.
This file is necessary so that the configuration library `wf-config` can verify the values of options in the config file, and for
providing default values if the user has not specified anything in the config file.

Then, we can use
```cpp
 wf::option_wrapper_t<std::string> external_monitor{"primary-monitor-switch/external-monitor"};
```
to fetch the option from the configuration file, and use it later.
Note that the option type must match the one in the XML file, otherwise a runtime exception will be thrown.

Another problem right now is the order of plugins.
Remember the interaction with `simple-tile`?
Well, plugins are loaded in the order specified in the config file.
If we specify `primary-monitor-switch` first, then this interaction cannot happen because simple-tile is not loaded yet.
To circumvent this problem, we can delay our new plugin's action until all plugins are initialized using `wf::wl_idle_call`.

Both of these improvements' implementations can be found in the [repository](https://github.com/ammen99/wayfire-plugins).

## Conclusion

Congratulations, you made it to the end of this blog post!
I hope the post was informative.
Next time, I will try to go into more details of some of the graphics aspects of Wayfire, and show how they can be used in plugins.
Thank you for reading & stay tuned.
