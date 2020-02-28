---
title:  "Set an animated GIF image as a dwm wallpaper"
category: unix, linux
---

Not only in dwm, really. The following instructions can be applied to any window manager if adjusted correctly.

The idea is to use Xlib to display an external gif player's window below all others independently from the active window manager.

We need a bit of knowledge of C scripting to accomplish this task. For the gif part, you can grab [this]( { site.github_url }/gifdec) player that doesn't drain battery or burst your CPU.

Edit the `manage(Window, XWindowAttributes*)` function in the dwm.c file as in the example.
```c
void
manage(Window w, XWindowAttributes *wa)
{
    /* this code is to be inserted here */
    char windownamegif[256];
    gettextprop(w, XA_WM_NAME, windownamegif, sizeof windownamegif);
    if (!strcmp(windownamegif, "gifwallpaper")) {
        selmon->gifwallpaper = w;
        XMoveResizeWindow(dpy, w, 0, 0, selmon->mw, selmon->mh);
        XSetWindowBorderWidth(dpy, w, 0);
        XLowerWindow(dpy, w);
        XMapWindow(dpy, w);
        return;
    }
    /* ... */
```

And add another Window field to `struct Monitor`.
```c
struct Monitor {
    /* ... */
	Client *clients;
	Client *sel;
	Client *stack;
	Monitor *next;
	Window barwin;
    Window gifwallpaper; /* <--- this */
	const Layout *lt[2];
};
```

And not forget to clean up after yourself.
```c
void
cleanupmon(Monitor *mon)
{
    /* ... */
	XUnmapWindow(dpy, mon->barwin);
	XDestroyWindow(dpy, mon->barwin);
    XUnmapWindow(dpy, mon->gifwallpaper);   /* < */
    XDestroyWindow(dpy, mon->gifwallpaper); /* < */
	free(mon);
}
```

Recompile and that's it.

Run `gifplay wallpaper.gif` to play a gif in the background.
