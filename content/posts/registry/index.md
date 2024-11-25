---
title: 'DataLad-Registry: Bringing Benefits of Centrality to DataLad'
date: 2024-10-13T18:17:28-07:00
author:
  - Isaac To
  - Austin Macdonald
  - Yaroslav O Halchenko
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


[**DataLad**](https://www.datalad.org/) provides a platform for managing and uniformly accessing data resources. It also captures basic provenance information about data results within Git repository commits. However, discovering DataLad datasets or Git repositories that DataLad has operated on can be challenging. They can be shared anywhere online, including popular Git hosting platforms, such as [GitHub](https://github.com/), generic file hosting platforms such as [OSF](https://osf.io/), neuroscience platforms, such as [GIN](https://gin.g-node.org/), or they can even be available only within the internal network of an organization or just one particular server. We built [DataLad-Registry](https://registry.datalad.org) to address some of the problems in locating datasets and finding useful information about them. (For convenience, we will use the term "dataset", for the rest of this blog, to refer to a DataLad dataset or a Git repo that has been "touched" by the `datalad run` command, i.e. one that has "DATALAD RUNCMD" in a commit message.)

In this post, we will introduce you to DataLad-Registry both as a publicly available service and as software you can deploy yourself.

# Goals and Guiding Principle {#goals-and-guiding-principle}

In DataLad-Registry, we aim to achieve the following goals.

1. Discover online datasets   
2. Discover dataset reuses  
   1. Identify forks  
   2. Identify uses as a sub-dataset  
3. Make up-to-date dataset metadata available  
4. Enable search of datasets through their metadata

In design and implementation, we embrace automation as a guiding principle. DataLad-Registry automatically

1. Registers datasets discovered by [datalad-usage-dashboard](https://github.com/datalad/datalad-usage-dashboard), a separate project developed and maintained by our teammate, John T. Wodder II, at [CON](https://centerforopenneuroscience.org/) that periodically searches for datasets on GitHub, OSF, and GIN.  
2. Extracts metadata from the registered datasets  
3. Updates the state of registered datasets and the corresponding metadata[^1]

# Current State {#current-state}

DataLad-Registry is a publicly available service accessible at [https://registry.datalad.org/](https://registry.datalad.org/). It maintains up-to-date information on an expanding collection of datasets, currently numbering over thirteen thousand, including those on [datasets.datalad.org](http://datasets.datalad.org) and those on GitHub and GIN[^2] discovered by the [datalad-usage-dashboard](https://github.com/datalad/datalad-usage-dashboard). It makes metadata extracted by the following five extractors available.

1. `"metalad_core"`  
2. `"metalad_studyminimeta"`  
3. `"datacite_gin"`  
4. `"bids_dataset"`  
5. `"dandi"` 

The first four extractors are provided through the [datalad-metalad](https://github.com/datalad/datalad-metalad) extension, and the last is a builtin extractor which provides metadata for datasets in the DANDI archive.

## Overview of Registered Datasets

When you visit the DataLad Registry at [https://registry.datalad.org/](https://registry.datalad.org/), you'll be greeted by the web UI of the registry.

{{< figure
src="landing-page.png"
alt="The landing page of the public instance of DataLad-Registry"
    >}}

By default, this page shows you a list of all registered datasets in descending order of when they were last updated, and in this order, the list tells you the latest activities in the registered DataLad datasets.

The lower left corner of the web UI displays **statistics** of the datasets matching the current search criteria. Since there is no search query provided at this point, the datasets are all the datasets registered. If you click the "Show details" button at the bottom of the web UI, you'll see more information.

{{< figure
src="stats-of-entire-registry.png"
alt="An embedded page showing the detailed stats of the entire registry"
    >}}

In particular, you will see information such as how many unique DataLad datasets the registry is currently tracking, and the total number and size of the annexed files in these datasets.

Every DataLad dataset has a UUID attached to it, and every clone shares the same UUID regardless where it is located. Even though a platform such as GitHub allows the tracking of “explicit clones” of a repository, the identity of the original dataset could be lost. DataLad-Registry uses this ID to allow you to find clones of a dataset and datasets using the dataset as a subdataset across all platforms (see demos in the following section).

## Search functionality {#search-functionality}

A description of the search syntax in DataLad-Registry is available by clicking the "Show search query syntax" button.

{{< figure
src="search-query-syntax.png"
alt="An embedded page showing the search query syntax"
    >}}

### Example: Simple singular word search {#example:-simple-singular-word-search}

Let's do a search with a query of a singular word, `haxby`.

{{< figure
src="haxby-search-result.png"
alt="The page with the single-word search result of `haxby` "
    >}}

This searches for the substring `haxby` across all the searchable fields, `"url"`, `"ds_id"`, `"head"`, `"head_describe"`, `"branches"`, `"tags"`, and `"metadata"`. As you can see in the search result, the statistics at the left corner have been adjusted. Click the "Show details" button, and you will see that the result consists of 20 unique DataLad datasets with 23,278 annexed files of 448.5 GB.  


{{< figure
src="haxby-search-result-stats.png"
alt="An embedded page with stats of the single-word search result of `haxby` "
    >}}

### Example: Finding clones of a dataset {#example:-finding-clones-of-a-dataset}

From the search result of the search for `haxby`, click on the OpenNeuro dataset ds001297 by its DataLad dataset ID, `2e429bfe-8862-11e8-98ed-0242ac120010`.  


{{< figure
src="haxby-search-result-pick-dataset.png"
alt="Figure directing to click on the dataset with DataLad dataset ID, `2e429bfe-8862-11e8-98ed-0242ac120010`, in the single-word search result page"
    >}}

This will cause a search for datasets with DataLad dataset ID, `2e429bfe-8862-11e8-98ed-0242ac120010`, generating a field-restricted search query of `ds_id:2e429bfe-8862-11e8-98ed-0242ac120010`.  


{{< figure
src="locate-forks.png"
alt="A page showing the forks of DataLad dataset with ID, `2e429bfe-8862-11e8-98ed-0242ac120010`"
    >}}

Thanks to the persistence of DataLad dataset ID, we have just located all the forks of a particular DataLad datasets in DataLad-Registry.

### Example: Find where dataset was used (included as a subdataset) {#example:-find-where-dataset-was-used}

Let's do a search with a query of another singular word, `container`.  


{{< figure
src="container-search-result.png"
alt="A page showing the result of the single-word search of `container`"
    >}}

### 

This search locates the ReproNim/containers dataset which provides "a collection of popular computational tools provided within ready to use containerized environments". We can, of course, find all the forks of this dataset by clicking on its DataLad dataset ID, `b02e63c2-62c1-11e9-82b0-52540040489c`, as demonstrated in the last example. Using logical operators and restricting searchable metadata, we can locate datasets that use the ReproNim/containers dataset as a subdataset. We can do this using a search query of `metadata[metalad_core]:b02e63c2-62c1-11e9-82b0-52540040489c AND NOT ds_id:b02e63c2-62c1-11e9-82b0-52540040489c`.  
The query, `metadata[metalad_core]:b02e63c2-62c1-11e9-82b0-52540040489c AND NOT ds_id:b02e63c2-62c1-11e9-82b0-52540040489c`, searches for datasets with metadata extracted by the [metalad\_core](https://docs.datalad.org/projects/metalad/en/latest/user_guide/metalad-first-steps.html?highlight=metalad_core#extract-dataset-level-metadata) extractor that contains the DataLad dataset ID `b02e63c2-62c1-11e9-82b0-52540040489c` and filters out those that possesses DataLad dataset ID of `b02e63c2-62c1-11e9-82b0-52540040489c`.  


{{< figure
src="metalad_core_metadata.png"
caption="The `metalad_core` metadata extracted from the dataset at https://github.com/spatialtopology/ds005256.git."
alt="A page showing the `metalad_core` metadata extracted from the dataset at https://github.com/spatialtopology/ds005256.git"
    >}}


{{< figure
src="repronim-as-subdataset.png"
caption="Search with query `metadata[metalad_core]:b02e63c2-62c1-11e9-82b0-52540040489c AND NOT ds_id:b02e63c2-62c1-11e9-82b0-52540040489c`"
alt="A page showing the result of a search with the query `metadata[metalad_core]:b02e63c2-62c1-11e9-82b0-52540040489c AND NOT ds_id:b02e63c2-62c1-11e9-82b0-52540040489c`"
    >}}

### Example: Find BIDS datasets not in OpenNeuro {#example:-find-bids-datasets-not-in-openneuro}

Similarly, by leveraging metadata extracted by the `bids_dataset` metadata extractor, we can find BIDS datasets that are not currently available on [OpenNeuro](https://openneuro.org/) using the query `metadata[bids_dataset]:"" AND NOT url:openneuro`.

`metadata[bids_dataset]:""` finds all the BIDS datasets and `NOT url:openneuro` filters out those that are currently on [OpenNeuro](https://openneuro.org/).  


{{< figure
src="bids-not-on-openneuro.png"
caption="Search with query `metadata[bids_dataset]:\"\" AND NOT url:openneuro`"
alt="A page showing the result of a search with the query `metadata[bids_dataset]:\"\" AND NOT url:openneuro`"
    >}}

## RESTful API {#restful-api}

Besides the web UI, Datalad-Registry has a public facing RESTful API, and its interactive documentation is available at [https://registry.datalad.org/openapi/](https://registry.datalad.org/openapi/) with three different interfaces to choose from, Swagger, ReDoc, and RapiDoc[^3].  


{{< figure
src="API-doc-interfaces.png"
caption="Interactive API documentation interfaces."
alt="A page showing the interactive API documentation interfaces"
    >}}

This API has an OpenAPI V3 schema (available at [https://registry.datalad.org/openapi/openapi.json](https://registry.datalad.org/openapi/openapi.json)). Developers can generate a client from it using [OpenAPI Generator](https://openapi-generator.tech/).

### Supported API operations {#supported-api-operations}

The API currently supports the following operations.

1. Registering new datasets  
2. Querying for datasets   
3. Fetching datasets and metadata by internal IDs


{{< figure
src="API-endpoints.png"
caption="API endpoints shown in Swagger"
alt="A page showing Swagger, an interactive documentation interface, displaying the supported API endpoints"
    >}}

Registration of new datasets is currently disabled in our public-facing instance of DataLad-Registry located at [https://registry.datalad.org](https://registry.datalad.org). However, you can choose to make that available in your own instance of DataLad-Registry should you launch one. The querying operation provided through the API supports all the features provided by the search function in the web UI. Additionally, it allows further restrictions in the query and provides an option for including metadata in search results.  


{{< figure
src="API-dataset-urls-endpoint.png"
caption="A detailed view of the `dataset-urls` endpoint."
alt="A page showing detailed view of the `dataset-urls` endpoint"
    >}}

## Deploying your own instance of DataLad Registry {#deploying-your-own-instance-of-datalad-registry}

If you want to benefit from DataLad-Registry for your datasets without making them publicly available on [registry.datalad.org](http://registry.datalad.org), you can deploy your own instance. A fully featured DataLad-Registry instance consists of the following service components:

1. **Web ([Flask](https://flask.palletsprojects.com/en/stable/)):** Serves the web UI and API endpoints.  
2. **DB ([PostgreSQL](https://www.postgresql.org/)):** Stores all metadata.  
3. **Worker ([Celery worker](https://docs.celeryq.dev/en/stable/userguide/workers.html)):** Executes automated tasks.  
4. **Scheduler ([Celery beat](https://docs.celeryq.dev/en/stable/userguide/periodic-tasks.html)):** Issues periodic tasks, such as syncing with the Datalad-Usage-Dashboard for new datasets and updating registered datasets, for the worker to execute.  
5. **Broker ([RabbitMQ](https://www.rabbitmq.com/)):** Manages task queues for Celery.  
6. **Monitor ([Flower](https://flower.readthedocs.io/en/latest/)):** Provides a real-time, web-based interface to display the progress, status, and results of Celery tasks.  
7. **Backend ([Redis](https://redis.io/)):** Acts as a backend to store task results.

All service components are specified in a Docker Compose file, making it easy to [bring up](https://github.com/datalad/datalad-registry?tab=readme-ov-file#testing-and-development-setup) an instance of DataLad-Registry using the `podman-compose` command.

## Read only instances {#read-only-instances}

Multiple [read-only](https://github.com/datalad/datalad-registry?tab=readme-ov-file#read-only-mode) DataLad-Registry instances can be set up alongside a fully featured one. Such instances consist of only the web and DB services, and they are always in sync with the fully featured instance. With read-only instances, the fully featured instance is protected from potential abuse and the workload is split. In fact, the instance you interact with at [registry.datalad.org](http://registry.datalad.org) is a read-only instance.  


{{< figure
src="public-instance-setup.png"
caption="The public instance of DataLad-Registry is available through a read-only instance."
alt="A figure depicting that the public instance of DataLad-Registry is available through a read-only instance"
    >}}

# Join/Contribute {#join-or-contribute}

Do you have a collection of datasets that you need to run periodic tasks on? Or do you have interesting metadata to extract? Please file an issue at [https://github.com/datalad/datalad-registry/issues](https://github.com/datalad/datalad-registry/issues), and we can help you make those datasets and their metadata available through DataLad-Registry.

# References

- 2024 Distribits talk:  
  - [📺 YouTube](https://youtu.be/_McJ1BtLsiQ?si=2nGc_szgLxW8dZeV)  
  - [🎥 hub.datalad.org](https://hub.datalad.org/distribits/recordings/src/branch/master/distribits2024/2024-04-05_06_to_registry.webm)   
- [💻 GitHub](https://github.com/datalad/datalad-registry)  
- [🌐 Live instance](https://registry.datalad.org/)

[^1]:  If a registered dataset is updated, DataLad-Registry will automatically update the local copy of the dataset and re-extract the corresponding metadata.

[^2]:   DataLad-Registry currently registers all datasets found by [https://github.com/datalad/datalad-usage-dashboard](https://github.com/datalad/datalad-usage-dashboard) that reside on GitHub and GIN. OSF datasets found by [https://github.com/datalad/datalad-usage-dashboard](https://github.com/datalad/datalad-usage-dashboard) are to be added soon as well. We welcome suggestions of other archives.

[^3]:  Swagger, ReDoc, and RapiDoc are tools that provide interactive documentation for the API, allowing developers to explore available endpoints easily.
