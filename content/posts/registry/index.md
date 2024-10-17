---
title: 'DataLad-Registry: Bringing Benefits of Centrality to DataLad'
date: 2024-10-13T18:17:28-07:00
author:
  - Isaac To
  - Austin Macdonald
tags:
  - DataLad
  - DataLad-Registry
cover:
  # webp (or png, jpg) should be 1200x630 px for social media cards
  image: cover.webp
  alt: A screenshot of DataLad-Registry web UI
  # must be 'true' if image is in page bundle, i.e. next to the
  # markdown file with the article
  relative: true
# important bits should be within the first 110 chars of the description
description: >
  An overview of DataLad-Registry, detailing its core goals and guiding principle, showcasing current features, and exploring future development plans
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

[//]: # (todo: A light hearted introduction)
Ever wondered what datasets are out there on the web managed, or "touched", by
DataLad? Can any of those that be useful to you? DataLad-Registry is here to help.
DataLad-Registry is a service that maintains up-to-date information on an expanding
collection of datasets, currently numbering over ten thousand. It automatically
registers datasets from various online sources, extracts metadata, and keeps both the
datasets and their metadata current. DataLad-Registry offers search functionalities,
including metadata-based searches, accessible through both a web interface and a RESTful
API.

Introduce usecases
    - searching for derivatives of a dataset
    - searching for sourcedata
    (Searchable repositories include wider range than others)
        include datalad top, github, gin, osf
    - metascience usecase?
Appeal to extractor authors
extendable extractors
[//]: # (todo: Introduce core goals and guiding principles of DataLad-Registry)

[//]: # (todo: Illustrate, briefly how datasets are automatically registered and their metadata extracted by different extrator.)
list the sources
list the extractors (which can be extended or add new ones)
[//]: # (todo: Demon the search functionalities &#40;using a screenshot&#41; of the syntax page, and doing some searches.)

[//]: # (todo: Showcase the access to the RESTful API)

[//]: # (todo: point to the github repo for details about setting up a local instance.)
usecase: add private repos and extract metadata

[//]: # (todo: Discuss the future development plans of DataLad-Registry, possibly)

[//]: # (todo: Provide a brief conclusion)

Links
    - docs
    - contributor page
    - file issue
    - Talk in distribits 2024