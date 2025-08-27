---
layout: post
title: "X11 -> WLC -> libweston -> wlroots: how the current Wayfire came to be"
---

Wayfire is a project which has had <span>4 versions</span> and has been rewritten <span>3 times</span>. In this blog post I'll try to outline why those rewrites were necessary.
<!--more-->

# The "dark" X11 era

I started Wayfire as a X compositing manager, and it was called Fire. My primary motivation at that time was to learn more about C++, window management, etc. On top of that, Compiz was just being announced as discontinued (although it turns out that Compiz 0.8.x is still maintained). So I thought that the time was perfect to start a clone of it. My early attempts were more or less copy-paste-refactor parts of the Compiz code. I was frustrated by some aspects of X and how X clients worked - what is that `WM_LEADER`? There were certainly specs on the internet, but things were incomprehensible for a beginner like me. Nonetheless, I soldiered on, and eventually got things working.

# Let there be Wayland

Then I heard about Wayland. It was supposed to be so much better than X11. I read that it will be the future of the Linux desktop (and by the way it turns out that those things are true). So I jumped on the wagon, after a bit of research `WLC` was found. I thought it an amazing library, although it was in a very early stage and there was just a single person working seriously on it. I rewrote Fire, renaming it Wayfire. I implemented subsurfaces for `WLC` (yes, that wasn't supported, even though it is needed to display menus in some applications like Gedit). But then more and more issues came - a particularly frustrating one was lack of clipboard synchronization between legacy Xwayland clients and native Wayland clients. It turned out, I couldn't achieve a fully functional desktop with it, except if I invested a lot of effort adding new features. But I lacked both motivation and knowledge.

# Libweston era

So when `libweston` came out, I was ready to make the switch, again. `libweston` is written by the same people that develop Wayland, so new features should come fast. It also seemed to be full of nice abstractions for views, outputs, etc. With its simple API and relatively simple codebase it made writing a compositor a lot easier. I didn't encounter a lot of difficulties when rewriting Wayfire on top of `libweston`, and I had the opportunity to think about how to reorganize the code, APIs, etc.

It turned out, however, that `libweston` wasn't a perfect fit either. For somebody creating a simple window manager, it would be quite good. However, Wayfire needs to have full control over output rendering. Let's look into a concrete example: window animations. A simple one like fade out would be quite easy with `libweston`: we have a `weston_view` struct representing a window (actually, the combination of `weston_surface`+`weston_desktop_surface` represents a window and `weston_view` represents an instance of the window displayed on the screen). We can either use the built-in `weston_fade_run` or animate ourselves by adapting `view->alpha` each frame. Now, if we wanted to also add a zoom effect (the window shrinking or expanding) we can transform the view by adding a scaling matrix. We can also rotate the window this way.

But customizability ends here. Let's say we want to have the burn effect from Compiz. Or we want wobbly windows. This means we want to customize the way views are rendered with OpenGL. Well, we can't, not without patching `libweston`. I did patch `libweston`, but in the discussions which followed it was seen that those patches are stretching the `libweston` APIs way further than they were ever intended to be.

Another shortcoming of `libweston` is the fact that it isn't desktop-oriented. It lacks support for protocols like `wlr-protocols`, which are necessary to build a DE - you can't build portable panels/bars/whatever, you can't have a proper lockscreen and so on. And when you try to use it as a daily driver, you start to encounter some little annoying bugs or missing features. Merging fixes for them didn't go as fast as one could wish for (which is perhaps understandable, as there aren't that many people working on `weston` and those people are also responsible for developing the core `wayland` protocols and libraries)

# The Present - wlroots

And then, out of the ashes of WLC, wlroots was born. Having convinced myself that patches to allow Wayfire to run with upstream `libweston` won't be merged, I took the painful decision for yet another rewrite, because I finally saw a chance to solve my main problems with `libweston`. In `wlroots`, the compositor gets full control over everything: Instead of having the rendering loop inside the library (the case with `libweston`), a `wlroots`-based compositor can run its own loop, and use the `wlr_output.frame` events to know when to redraw. `wlroots` won't get in your way by trying to render windows by itself. Rather, you can tell it exactly when and where to render them with the `wlr_renderer_*` functions (if you want), or you can copy the window contents and then manually draw the scenegraph (the approach I use in Wayfire).

In `libweston`, damage tracking (a technique which lets the compositor repaint only those parts of the screen that actually changed) was also managed internally. This is all very good, because damage tracking isn't a trivial task. However, because of how `libweston` was designed, I couldn't use it in scenarios like the Expo plugin in Wayfire (especially in multi-monitor setups). With `wlroots`, the compositor again gets absolute freedom. There is per-window damage (or per-surface in Wayland terminology) which the compositor gets with the `wlr_surface.commit` events. We also have the `wlr_output_damage` helper, which lets you calculate the damaged region for each output on each frame. The compositor then manually combines those together, which is more work compared to `libweston`, but enables important optimizations in Wayfire - a prime example being damage tracking for blurred windows.

Last but not least, through `wlroots` I get to easily implement those crucial desktop-related wayland protocols which were missing with `libweston`. Sounds like a dream, right?

Of course, when we speak about such complex pieces of software, there will always be caveats. With `libweston`, I got a nice abstraction for different shells - `wl_shell`, `xdg-shell-v6`, `xwayland` (the abstraction is called `weston_desktop_surface` and you can create a `weston_view` from it). I could for example call `weston_desktop_close(surface)` and it would automatically do the right call for me, regardless of the actual window type. In `wlroots`, I need to manage all of them separately, which can be quite tedious.

In `libweston`, there are already helpers to create key-, button- and touch bindings. However `libweston` doesn't support touchscreen gestures, and I had to do some hacks to get them working. With `wlroots`, I can write the code much more cleanly, because I have full control over input, although I also have to manually implement key and button bindings.

# Is a next rewrite coming? a.k.a The Future

Short: No, Wayfire will stick to `wlroots` for the foreseeable future.

Long: Despite the added complexity, `wlroots` is the perfect match for Wayfire. When rewriting the codebase, I could just implement things in the most meaninful way, being free of the limitations of `libweston`. I could add new features, some of which wouldn't have been possible with `libweston`. And thanks to the amazing team that now develops `wlroots` (and Sway) updates for new protocols are coming even faster than with libweston.

So I can confidently say I found what I was looking for :)
