---
layout: post
title: Documenting the spirits
tags: renpy demotools spirited
description: How I handle the documentation generation for the Ren'Py extension Spirited
---

# Documenting the spirits

[Spirited](https://ayowel.github.io/spirited/) is a Ren'Py extension that provides a unique `Spirited` class and a new `spirited` keyword in the Ren'Py grammar. Its aim is to allow fast and easy configuration of advanced particle effects. This article is a rewind on how its documentation pages came to be.

[Spirited's documentation](https://ayowel.github.io/spirited) was my first attempt at providing user-facing detailed documentation for a tool.
To me, Rust's codebase/crates documentation is a source of inspiration and I wanted my documentation pages to have five qualities that I believe Rust's codebase has:

* The documentation had to be exhaustive and relevant, if someone wanted an information, it had to be part of the documentation with as little boilerplate as possible.
* The documentation had to contain examples, all the more so because the extension's target audience would contain many code-illiterate people.
* The documentation had to be up-to-date. Rust partially ensures it by systematically testing all documentation examples.
* The documentation had to be available both online (at https://docs.rs for Rust) and offline (buildable for all downloaded dependencies of a project in Rust)

## Hosting the documentation

The first limitation I hit with those requirements was the issue of hosting the documentation. As much as possible, I did not want to host it on a personnal website but preferred to use a website where I could easily update the content from a script and where the hosting actor was guaranteed to last. I did not think long before picking GitHub Pages, which I've used on a regular basis and which allows me to upload [Markdown](https://www.markdownguide.org/)-based files that will be server as web pages through [Jekyll](https://jekyllrb.com/).

## Staying up-to-date

The main issue I wanted to tackle early-on was to ensure that the documentation was up-to-date.

To do so, the first step was minimizing the cost of maintainance by ensuring that any information only needs to be updated in one place. This meant that information available in the code should not be repeated in independant documentation files targetting users but should be extracted instead.
Unfortunately, the use of two different languages (Python and Ren'Py) that should appear on the same documentation pages as well as the desire to provide documentation examples with visuals - which is not a feature of either language - made it impossible to use common documentation tools[^1].

Hence the need for a custom tool that could parse the source code and build a documentation page with the desired layout from a template. Here is a very high-level representation of the target behavior that will be updated as this post progresses:

{% capture mermaid_graph %}{% raw %}
graph LR
  subgraph Inputs
    code(Source code)
    DT(Documentation Template)
  end

  code & DT --> bb{{Processor}}
  bb -->doc(User documentation)
{% endraw %}{% endcapture %}
{% include mermaid.html source=mermaid_graph %}

[^1]: Notwithstanding the fact that Ren'Py is not supported in tools and would require specific development

## Adding technical documentation

The extension's source code is written in Python. Considering that the target user documentation format is markdown, I decided to ignore [PEP 287](https://peps.python.org/pep-0287/)'s recommandations and write the docstrings in Markdown instead of [reStructuredText](https://docutils.sourceforge.io/rst.html)[^2].

Only user-facing interfaces need to be documented, which drastically reduces the scope of what needs to be parsed as the only exposed interfaces are the classes' constructors and/or attributes. At the time of writing, both are part of the class' documentation, which looks like this:

```py
class Spirited(renpy.display.core.Displayable, renpy.defaultstore.NoRollback):
    """
    This class provides rendering for mostly-linear effects
    such as "mystic" particles, rain, or snow.

    ## Arguments

    Tuple arguments may be written as a single value and will be implicitly expanded.
    Tuple arguments with two values should be interpreted as ranges with a minimum and a maximum value.

    * `sprite_list` **<[Displayable]>**
        An array of images to use as sprites

    * ...
    """
```

To extract docstring comments from the source code, I used the [`ast` python module](https://docs.python.org/3/library/ast.html). This module handles the core python parsing components and can be used to get an abstract representation of a python file's content[^3]. The following code parses a Python file and prints the classes' docstrings[^4]:

```py
with open('file.py', 'r') as f:
  content = f.read()
content_ast = ast.parse(content)
for classdef in content_ast.body:
  if not isinstance(classdef, ast.ClassDef):
    # Filter-out any non-class value
    continue
  for dc in classdef.body:
    if not isinstance(dc, ast.Expr):
      # Filter-out any non-docstring value
      continue
    if isinstance(dc.value, ast.Constant):
      print("Class {} has docstring: {}".format(classdef.name, dc.value.value))
      break
```

Once the class' documentation is found, the script needs to normalize it to remove the space padding on the left and lower the title level wherever there is one (done by adding a `#`):

```py
def normalize(data):
  data_array = data.split("\n")
  left_strip = 0
  for s in data_array: # Calculate the padding size on the first non-empty line
    if s.strip():
      left_strip = len(s) - len(s.lstrip())
      break
  return '\n'.join([(s[left_strip:] if '#' not in s[left_strip:left_strip+1] else '#'+s[left_strip:]) for s in data_array])
```

Finally, the script can build Markdown files from the provided data. In the current version, the generated files are a dump of the doctring's content named after the class they come from. The code extract above builds the following `Spirited.md` file, which is referenced in a different file[^5]:

```md
This class provides rendering for mostly-linear effects
such as "mystic" particles, rain, or snow.

It must be instanciated as an image before usage.

## Arguments

* `sprite_list` :
    An array of images to use as sprites
```

Updated process representation:

{% capture mermaid_graph %}{% raw %}
graph LR
  subgraph Inputs
    code(Source code)
    DT(Documentation Template)
  end
  subgraph Processor
    eds{{Extract DocStrings}}
    iid{{Generate documentation}}
    unk{{???}}
  end
  doc(User documentation)

  code --> eds
  eds --> iid
  DT --> unk
  iid --> doc
  unk --> doc
{% endraw %}{% endcapture %}
{% include mermaid.html source=mermaid_graph %}

[^2]: Using [pandoc](https://pandoc.org/) or a similar tool, it is possible to convert reStructuredText to Markdown, this may be done in the future but was deemed too burdensome at the time.
[^3]: This module is usefull for specific applications but is not needed for most applications, is used internally by Python, and may change with each Python release.
[^4]: This only works because docstrings are treated as strings and attached to their parent object in Python, comments started with a `#` are not part of the AST representation of a file. `pyparsing` might be a good alternative if you need to access such comments.
[^5]: The reference is a static reference at the time of writing

## Adding examples

Adding documentation examples in Ren'Py, unlike Rust, is not natively supported (the engine does not provide any simple way to support it). What may be done, however, is to create files in a Ren'Py project that contain whole examples and parse them, and it is what I did. Those files may then be injected in the documentation, though an additional processing step was required to make them as reusable as possible.

The main advantage of this method is that the project that holds the examples may then be used as a testing project and said examples be checked as part of the regular validation process.

To be able to decide where the examples would be inserted in the resulting documentation, I created template files with a custom markup close to Jekyll's markup where examples should be inserted with the path to said examples as a parameter:

{% raw %}
```md
## Image-based usage
[...]

{{! ../game/examples/example_image_simple.rpy }}

{{! ../game/examples/example_image_multi.rpy }}
```
{% endraw %}

The processing logic became the following:

{% capture mermaid_graph %}{% raw %}
graph LR
  subgraph Inputs
    code(Source code)
    ex(Example files)
    DT(Documentation Template)
  end
  subgraph Processor
    eds{{Extract DocStrings}}
    iid{{Generate documentation}}
    umd{{Update templates}}
  end
  doc[User documentation]

  code --> eds
  eds --> iid
  DT --> umd
  ex --> umd
  umd --> doc
  iid --> doc
{% endraw %}{% endcapture %}
{% include mermaid.html source=mermaid_graph %}

As `Spirited` is an extension that provides a graphical component, I felt that it was important to provide visuals of what the code examples would look like. Additionally, each example needed to have some kind of explanation of how it worked or what should be visible when using it.

To provide example documentation and visuals, I decided to store the information in the example files[^6] and later parse them to extract it. To do so, I needed to be able to be able to create visuals based on imperative instructions that could be stored in a text file. Once this was possible, the layout would consist of a commented description followed by the generation instructions, itself followed by the code shown to users in the documentation.

{% capture mermaid_graph %}{% raw %}
graph LR
  subgraph Inputs
    code(Source code)
    ex(Example files)
    DT(Documentation Template)
  end
  subgraph Processor
    eds{{Extract DocStrings}}
    iid{{Generate documentation}}
    umd{{Update templates}}
    exproc(Example files parser)
    gg(???)
  end
  subgraph udoc [User documentation]
    doc(Markdown documentation)
    gdoc(Visuals)
  end

  code --> eds
  eds --> iid
  DT --> umd
  umd --> doc
  iid --> doc
  DT --> exproc
  ex --> exproc
  exproc -- "Description" --> umd
  exproc -- "Code" --> umd
  exproc -- "Visual's configuration" --> gg
  gg --> gdoc
{% endraw %}{% endcapture %}
{% include mermaid.html source=mermaid_graph %}

[^6]: An alternative was to use distinct files for example code, example description, and example visuals. The main issue I have with this method is the fact that having to open multiple files to update a single resource quickly becomes a pain.

### Creating visuals for the examples

The desire to guaranty that the visuals are up to date made me create a purpose-built Ren'Py extension called [Demotools](https://ayowel.itch.io/renpy-demotools) which allows the creation of visuals by providing specific command-line inputs to Ren'Py. The generated visuals are screenshots of what is visible on the screen at regular intervals and can then be used to generate a GIF with tools like `ffmpeg`.

The command used to generate visuals looks like this[^7] (the command jumps to a label and captures whatever happens on the screen for 6 seconds before automatically exiting):

```sh
renpy.sh . demotools --render --destination render_dir "jump=spirited_example_image_simple" "pause=6"
```

This command allowed me to provide the visuals as definition comments after the description in example files. The comments only need to contain scheduling instructions (when a screen should be displayed, how long to wait, ...), as render instructions are not relevant on the example level and are handled in the script instead. This results in a very short instruction identified by a preceding `# demo:` string in the example file:

```py
# demo: jump=spirited_example_image_simple pause=6
```

This instruction can later be parsed and used to run a Ren'Py command that generates images that will then be turned into gifs with `ffmpeg`[^8].

Informations on the following code extract, all gifs are generated sequentially[^9]:

* The `command` parameter is the instruction string extracted from an example file
* The `out_file` parameter is the path to the gif that should be generated
* The `RENPY_PATH` environment variable indicates where Ren'Py is located on the system.
* The image generation occurs in a temporary directory that is deleted when it ends
* The gifs and images are generated with 10 frames per second as the target speed

```py
def generate_gif(command, out_file):
  if "RENPY_PATH" not in os.environ:
    print("No RENPY_PATH available in env, skipping gif generation")
    return
  print("Building Gif file {}".format(out_file))
  renpy = os.environ["RENPY_PATH"]
  gen_dir = tempfile.mkdtemp()
  # Ren'Py Command
  command_template = [
    "{renpy} .. demotools",
    "--destination {out_dir}",
    "--render",
    "--render-size 960:540",
    "--render-fps 10",
    "{instructions}"
    ]
  command = " ".join(command_template).format(renpy = renpy, out_dir = gen_dir, instructions = command)
  if os.system(command) != 0:
    shutil.rmtree(gen_dir)
    print("An error occured during gif images generation with command: {}".format(command))
    exit(1)
  # FFMpeg command
  gif_command_template = [
    "ffmpeg",
    "-v warning",
    "-f image2",
    "-r 10",
    "-i '{in_dir}/snapshot-%*.png'",
    "-r 10",
    "-y {out_file}"
  ]
  gif_command = " ".join(gif_command_template).format(in_dir = gen_dir, out_file = out_file)
  gif_result = os.system(gif_command)
  # Cleanup
  shutil.rmtree(gen_dir)
  if gif_result != 0:
    print("An error occured during gif images conversion with command: {}".format(gif_command))
    exit(1)
```

With all this in place, the final processing logic was the following:

{% capture mermaid_graph %}{% raw %}
graph LR
  subgraph Inputs
    code(Source code)
    ex(Example files)
    DT(Documentation Template)
  end
  subgraph Processor
    eds{{Extract DocStrings}}
    iid{{Generate documentation}}
    umd{{Update templates}}
    exproc(Example files parser)
    gg(Gif generator)
  end
  subgraph udoc [User documentation]
    doc(Markdown documentation)
    gdoc(Documentation gifs)
  end

  code --> eds
  eds --> iid
  DT --> umd
  umd --> doc
  iid --> doc
  DT --> exproc
  ex --> exproc
  exproc -- "Description" --> umd
  exproc -- "Code" --> umd
  exproc -- "Gif args" --> gg
  gg --> gdoc
{% endraw %}{% endcapture %}
{% include mermaid.html source=mermaid_graph %}

[^7]: Command simplified for clarity.
[^8]: I used the command-line version of ffmpeg because that's what I'm used to, but a pythonic gif generation would use [Pillow](https://pypi.org/project/Pillow/), [python-ffmpeg](https://pypi.org/project/python-ffmpeg/), or [images2gif](https://pypi.org/project/images2gif/).
[^9]: At the moment, the main bottleneck when rendering is the disk writes. If not for it, multiple renders could theoretically run in parallel.

## Afterword

Having a maintainable documentation for this extension was important to me and required a lot of tinkering and development. I didn't talk about what I needed to do to automate the generation of the documentation and its offline archive, or how demotools works - both of which might become posts in the future - , but there were many details that were not as straghtforward as they could - or should - have been and it was a very interessant experience overall.

This project also got me thinking about whether it would be relevant to get an actual testing tool in Ren'Py, which it probably is not as Ren'Py is mostly built around graphical components that would be hard to test and that most users have no intention to test, but I believe that it could be an interessant project nevertheless.
