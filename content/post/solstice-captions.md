---
title: "Broadcast open captions for pre-written scripts"
date: 2022-12-19T19:30:00-08:00
---

# Broadcast captions from pre-written script

## Overview

1. Get the script in plaintext format
2. Preprocess, i.e. manually split it up into chunks to be displayed on the screen
3. Generate slides from the chunked script
4. Add source to OBS
5. During the show, advance the slideshow captions to keep up with speech and song


## Prerequisites

This workflow requires comfort with the command line and some bash/python scripting.

Knowledge of some basic HTML/CSS is also helpful if you need to tweak the caption appearance.

- **bash**: for some basic scripting 
- **python**: to automatically turn the plaintext captions into a markdown slide format
- **npm**: to install `nodeppt`, our markdown-slides generator
- **OBS**: for the stream :)

## Preprocessing

Make a copy of the script and save it as `script-captions.txt`.
Making a separate copy of the processed text is helpful because it means you don't have to redo the entire script should parts of it get updated.
Instead, you can re-download updated versions of the script as `script-edited-2022-12-19.txt` and use the `diff` tool with the previous copy of the script to just see the differences.
Then you can update `script-captions.txt` with just the updated text.


Captions need to be readable, which means keeping a short line length and keeping only a small number of lines on the screen at a time.
Readable line length is about 80 characters long.
For speech, there should be at most 2-3 lines on the screen at a time.

For songs, I made an exception to the rule that captions should be at most 2 lines long, because some songs go quickly.
I chose to allow song lyrics a maximum of 4 lines on-screen at a time.
These lines should ideally be short or repetitive like a chorus.


## Generating slides

When it comes to markdown-based slide apps, there are a
[lot of options](https://gist.github.com/johnloy/27dd124ad40e210e91c70dd1c24ac8c8).

I tried a few of them and landed on
[`nodeppt`](https://github.com/ksky521/nodeppt#%E9%85%8D%E7%BD%AE) because it:

- has simple syntax for delimiting slides
- supports custom css (and has nice css by default)
- works as a webserver with hot-reload for local development
- can also make a static build with all of the html, css, and js needed
- doesn't rely on some hosted service
- is free and open-source
- worked


Make sure you have npm installed
(if not, [follow their docs](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm#using-a-node-version-manager-to-install-nodejs-and-npm)).

Then you can install `nodeppt`.

```bash
npm install -g nodeppt
```

Create a new file called `make-slides.py` and paste the following inside:

```python
#!/usr/bin/env python

import re

# preamble is required at the top of slide.md for nodeppt
# https://github.com/ksky521/nodeppt#%E9%85%8D%E7%BD%AE
preamble = """title: solstice
speaker: eli
css: [./css/style.css]
plugins:
    - echarts"""

# this line denotes a new slide
new_slide = '<slide :class="aligncenter slide-bottom">'

def make_slides(script: str):
    deck = [preamble]
    # assume the script has been preprocessed such that new slides are
    # delimited by two or more newlines in a row
    split = re.split("\n\n+", script)

    for l in split:
        # ensure each newline is interpreted correctly by markdown
        # without this, newlines will just turn into spaces in the slides
        l = l.replace("\n", "  \n")
        # create new slide
        deck.append(new_slide + "\n" + l + "\n\n")

    # put the whole thing together
    return "\n".join(deck)


def main(in_file, out_file):
    with open(in_file, "r") as f:
        script = f.read()
    deck = make_slides(script)
    with open(out_file, "w") as f:
        f.write(deck)


if __name__ == "__main__":
    import sys
    assert len(sys.argv) == 3, f"{__file__} in_file out_file"
    main(sys.argv[1], sys.argv[2])
```

Then to generate the markdown for slides, you can run:
```
python ./make-slides.py script-captions.txt slides.md
```

This will take your `script-captions.txt` file as input, then add the markdown
sauce for turning it into a proper slide deck called `slides.md` for use with
`nodeppt`. 


## Configuring OBS

Make sure to install OBS via one of the **official** methods, otherwise it may be outdated.
I ran into this on Fedora linux and my version didn't have Browser Source support.

https://obsproject.com/wiki/install-instructions

Now we can proceed one of two ways: either add the captions as a _browser
source_ (preferred) or use screen capture on your browser's application window
combined with layer blending to make it transparent.

Browser source is the best and you should only fallback to screen capture if
you can't get browser source to work.


### Browser source


### Application window capture 
