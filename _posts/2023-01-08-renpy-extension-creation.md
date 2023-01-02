---
layout: post
title: Creation of an extension for Ren'Py
tags: renpy
description: How to package changes to Ren'Py for re-use and sharing
---
## Introduction

### What is Ren'Py

Ren'Py[^1] is a specialized game creation tool that focuses on visual novel games, a create-your-own-story game genre where the player follows the adventure of characters and gets to make decisions that influence the flow of the story (such as *Ace Attorney*[^2] and *Doki Doki Literature Club!*[^3]). To grant as much freedom as possible to the game creators, Ren'Py allows the use of Python[^4] to heavily customize a game's behavior ; all game information and code is usually gathered within files with the extension `.rpy`.

[^1]: Official website: https://www.renpy.org
[^2]: A visual novel franchise where the player tries to solve mysteries to prove the innocence of his clients. Most known for *Phoenix Wright: Ace Attorney* released in 2001.
[^3]: DDLC for short, a well-known horror game in the visual novel genre published in 2017 by Serenity Forge.
[^4]: https://www.python.org/. Python is a programming language. Ren'Py version 7.X.X and lower use Python 2.X, later versions use Python 3.X.X.

### What are Ren'Py extensions

Ren'Py Extensions[^5] are an undocumented way to extend Ren'Py's behavior beyond what is normally possible without having to maintain a custom Ren'Py repository and building its binaries. It is most useful when trying to extend the default behavior[^6] of Ren'Py with the intent of sharing those modifications with others.

Ren'Py Extensions have the extension `.rpe` and consist of a zip file with a mandatory `autorun.py` file. They may contain all types of files, however a specificity of note is that Python files will become available to `rpy` files as python modules[^7]. By default, Ren'Py Extensions have to be located directly under the `game` directory of the game that should use them and will be loaded in alphabetical order[^8].

```bash
# The files in the extension at https://ayowel.itch.io/renpy-extension-loader
$ unzip -l extension_loader.rpe
Archive:  extension_loader.rpe
  Length      Date    Time    Name
---------  ---------- -----   ----
    10751  1980-01-01 00:00   LICENSE.txt
       51  1980-01-01 00:00   CREDITS.txt
      348  1980-01-01 00:00   autorun.py
---------                     -------
    11150                     3 files
```

[^5]: Which should not be mistaken with the documented extension systems for Ren'Py.
[^6]: As extension files are read early on, they may be used to change how Ren'Py will parse the `.rpy` files or add custom file archive formats for example.
[^7]: This is due to the fact that Ren'Py internally adds the archive to the `PATH`, which Python then leverages to detect new files/modules.
[^8]: Prior to Ren'Py 7.5.4/8.0.4, the load order was not deterministic. In case of inter-dependency between extensions, using an intermediate loader extension such as https://github.com/Ayowel/renpy-extension-loader provides this guarantee in a set subdirectory.

## Creating a new extension

The following was tested with Ren'Py 8.0.3 and is heavily based on the pseudo-game [Renpy extensions demo](https://ayowel.itch.io/renpy-extensions-demo) that I published on Itch, check it out!

### Say hello to the world

The most simple extension one can create is the null extension, an extension that is loaded but performs no action as it only contains an empty `autorun.py` file. Let's do something slightly more interessant and add a single log line at load time in the autorun.

```python
# autorun.py
print("Hello world")
```

The `autorun.py` should be zipped, and the zip added at the root of a game's `game` directory, and its extension changed to `.rpe`. The log line will appear in the game's `logs.txt` file after running it once.

### Access Ren'Py and the game store

By default, none of the Ren'Py-specific classes, methods, and variables are available in an extension's context, and setting a variable in `autorun.py` will not make it available. To access Ren'Py from an extension, you will have to import it (with `import renpy`) and access the desired items in the `renpy.store` submodule[^9]. One simple way to test this is to set a variable in `autorun.py` and ensure that it exists in the game by typing its name in the developer console[^10] and verifying the printed value.

```python
# autorun.py
import renpy
renpy.store.hello = "Hello world!"
# input 'hello' and press `ENTER` in the developer console to check the value
```

If you set a variable like this, it will become part of the save files, which you might not want. What I usually do to reproduce the behavior of the Ren'Py `default` statement is to `setdefault` on the module's dict instead:

```python
# autorun.py
import renpy
renpy.store.__dict__.setdefault('hello', "Hello world!")
# input 'hello' and press `ENTER` in the developer console to check the value
```

[^9]: Extensions run before internal Ren'Py setup files, so some attributes may not be set/exist yet when the script runs. Circumventing this limitation usually relies on the definition of a function that should be called from a `rpy` file.
[^10]: To open the developer console after running the game, press `SHIFT + O` to open the console and `ESCAPE` to exit it.

### Extend Ren'Py

The previous example shows the basics required to start creating extensions, but does not show what extensions can do that Ren'Py `.rpy` files can't. One such thing is the customization of screen layouts to add new components[^11].

To show how to do this, let's use the [Creator-Defined Displayable](https://renpy.org/doc/html/cdd.html) example implementation from the official Ren'Py documentation, which creates a new Displayable component and uses it in screens. First off, let's compare how it is used in the example and how we'd like to be able to use it in a Ren'Py-native-like pattern:

```py
# Documentation example
screen alpha_magic:
    add Appearing("logo.png", 100, 200):
        xalign 0.5
        yalign 0.5
# Desired usage example
screen alpha_magic:
    appearing "logo.png":
        opaque_distance 100
        transparent_distance 200
        xalign 0.5
        yalign 0.5
```

After the change, the explicit object instantiation in the documentation is replaced with a declaration that will internally be mapped to the desired class by Ren'Py, while the class parameters move to the declarative attributes list. This is most usefull when distributing Creator-Defined Displayables with complex parameters or that may contain multiple other objects as it makes their interfaces more readable and maintainable in screens[^12].

First things first, let's move the Creator-Defined Displayable to an extension and ensure the first documentation example still works. To do that, we need to `import renpy` in the autorun and update any call to a store-bound variable to be preceded by `renpy.store`. Once this is done, we need to inject the class back into the store by setting a store variable `Appearing` in the same way we set `hello` in the previous part.


```py
# autorun.py
import renpy # Precede all store objects calls with renpy.store
import math

class Appearing(renpy.store.renpy.Displayable):

    def __init__(self, child, opaque_distance, transparent_distance, **kwargs):

        # Pass additional properties on to the renpy.Displayable
        # constructor.
        super(Appearing, self).__init__(**kwargs)

        # The child.
        self.child = renpy.store.renpy.displayable(child)

        # The distance at which the child will become fully opaque, and
        # where it will become fully transparent. The former must be less
        # than the latter.
        self.opaque_distance = opaque_distance
        self.transparent_distance = transparent_distance

        # The alpha channel of the child.
        self.alpha = 0.0

        # The width and height of us, and our child.
        self.width = 0
        self.height = 0

    def render(self, width, height, st, at):

        # Create a transform, that can adjust the alpha channel of the
        # child.
        t = renpy.store.Transform(child=self.child, alpha=self.alpha)

        # Create a render from the child.
        child_render = renpy.store.renpy.render(t, width, height, st, at)

        # Get the size of the child.
        self.width, self.height = child_render.get_size()

        # Create the render we will return.
        render = renpy.store.renpy.Render(self.width, self.height)

        # Blit (draw) the child's render to our render.
        render.blit(child_render, (0, 0))

        # Return the render.
        return render

    def event(self, ev, x, y, st):

        # Compute the distance between the center of this displayable and
        # the mouse pointer. The mouse pointer is supplied in x and y,
        # relative to the upper-left corner of the displayable.
        distance = math.hypot(x - (self.width / 2), y - (self.height / 2))

        # Base on the distance, figure out an alpha.
        if distance <= self.opaque_distance:
            alpha = 1.0
        elif distance >= self.transparent_distance:
            alpha = 0.0
        else:
            alpha = 1.0 - 1.0 * (distance - self.opaque_distance) / (self.transparent_distance - self.opaque_distance)

        # If the alpha has changed, trigger a redraw event.
        if alpha != self.alpha:
            self.alpha = alpha
            renpy.store.renpy.redraw(self, 0)

        # Pass the event to our child.
        return self.child.event(ev, x, y, st)

    def visit(self):
        return [ self.child ]

renpy.store.__dict__.setdefault('Appearing', Appearing)
```

What should be noted before continuing is the fact that the `Appearing` class' `__init__` method accepts three arguments[^13]: `child`, `opaque_distance`, and `transparent_distance`. Once we've ensured that this works as intended, we can get into adding parser support.

The Ren'Py screen language is handled via the `renpy.sl2` submodule. The first thing we need to do is to declare the parser object (and store it in a variable as we will need to re-use it later), which needs at least three parameters:

* The keyword to use in text (here, we want `"appearing"`)
* The underlying object that will be instanciated (The `Appearing` class we just created)
* The style to use for the instanciated object (set it to `"default"` if you don't want a specific value)

```py
appearing_parser = renpy.sl2.sldisplayables.DisplayableParser("appearing", Appearing, "default")
```

Once we have created the parser object, we can start configuring it. The `renpy.sl2` submodule keeps track of the last instantiated `DisplayableParser` and applies changes made by subsequent calls to it, so ensure that you do not try to configure multiple parsers in parallel if you want to instantiate more than one.

First, let's add the arguments that `Appearing` supports:

* One positional argument (the `child` image/Displayable that should appear)
* Two keyword-based arguments (the `opaque_distance` and `transparent_distance` to use when calculating whether the `child` should be visible)
* The generic position arguments all displayables should support

```py
renpy.sl2.slparser.Positional("child")
renpy.sl2.slparser.Keyword('opaque_distance')
renpy.sl2.slparser.Keyword('transparent_distance')
renpy.sl2.slparser.add(renpy.sl2.slproperties.position_properties)
```

It feels like we're done configuring the parser, yet one step remains before we can say we're done and cleanup: we have to indicate in which objects `Appearing` may be used, which is every `Container` objects (`childbearing_statements` hereafter) as well as each screen:

```py
for i in renpy.sl2.slparser.childbearing_statements:
    i.add(appearing_parser)
# Add as valid child for screens
renpy.sl2.slparser.screen_parser.add(appearing_parser)
```

And NOW, we're almost done and can test that the usage pattern we desired works as intended before cleaning-up after ourselves by ensuring that our parser object does not end up updated by code that would improperly make changes in the `renpy.sl2` scope:

```py
renpy.sl2.slparser.parser = None
```

Finally. We are done adding support for `Appearing` to the screen parser. What we just did can easily be replicated for other displayables, enjoy!

## Afterword

All code showcased above is available in [this GitHub repository](https://github.com/Ayowel/blog-2023-01-08) and is mostly a striped-down version of [this pseudo-game](https://ayowel.itch.io/renpy-extensions-demo).
Extensions have other possibilities, such as supporting custom archive formats, hooking into internal Ren'Py functions, loading binary files, maintaining a connection to a server, ... Your imagination is the limit.

[^11]: The APIs used to add this kind of feature are not documented for the most part.
[^12]: This is a personal belief, not an objective truth as readability and its evaluation criteria varies wildly depending on your environment and needs.
[^13]: In addition to the required `self` parameter and the generic `**kwargs` parameter passed to the parent object
