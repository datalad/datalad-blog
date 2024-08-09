# Sources for the DataLad blog

This is a [DataLad](https://www.datalad.org) dataset containing a
[Hugo](https://gohugo.io) website.

Visit the blog at: https://blog.datalad.org


## Contribute an article

The contribute an article, clone this DataLad dataset. If you you have Hugo
installed, you could follow [their instruction to create new
content](https://gohugo.io/commands/hugo_new_content/). Otherwise, create a new
directory under `content/posts/`. The name of the directory will be the
[slug](https://developer.mozilla.org/en-US/docs/Glossary/Slug) of the new
article. Inside that directory, create an `index.md` file.

Include front matter at the top of the file. Here is a starting point:

```yaml
title: 'A blog on data management and DataLad'
date: 2024-07-11T20:28:47+02:00
author:
- Michael Hanke
- ...
tags:
- DataLad
- ...
cover:
  # webp (or png, jpg) should be 1200x630 px for social media cards
  image: cover.webp
  alt: DataLad minions reading and climbing books to write something.
  # must be 'true' if image is in page bundle, i.e. next to the
  # markdown file with the article
  relative: true
# important bits should be within the first 110 chars of the description
description: >
  On starting a blog after more than a decade of working on DataLad.
# optional choices, can be deleted
# within-article table of contents shown
showToc: true
# hide top-level metadata (date, length, author)?
hidemeta: false
# no highlighting of "code"
disableHLJS: true # to disable highlightjs
# no sharing buttons
disableShare: false
# no summary shown at the top
hideSummary: false
---
```

Now compose the article underneath the front matter.

Add as many figures and illustrations as desired. Do add a cover image (see
front matter).  There is a template `cover_template.svg` with the right sizes
and shapes. Lastly convert to `webp` format using your favorite workflow (e.g.,
`inkscape > png > gimp > webp`).

Commit your changes when done. A `datalad save -m "<message>"` will do the
right thing. Otherwise, commit the text to Git, and annex any image or media
file, regardless of its size.

If you have Hugo, use the [server](https://gohugo.io/commands/hugo_server/) to
get to see what you are writing, while you write. This will need the subdataset
with the theme, installed. Get it via `datalad get --recursive .`.
