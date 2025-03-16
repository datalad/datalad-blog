---
title: 'Hosting really large datasets with Forgejo-aneksajo'
date: 2024-08-27T17:55:00+02:00
author:
- Michael Hanke
social:
  fediverse_creator: "@mih@mas.to"
tags:
- DataLad
- Forgejo
- git-annex
cover:
  image: cover.webp
  alt: "A screenshot of https://hub.datalad.org/hcp-openaccess, and the Forgejo, git-annex, and DataLad logos on top."
  relative: true
description: >
  Looking into the performance and demands of Forgejo when hosting thousands of DataLad datasets
---

One scenario where DataLad shines is managing datasets that are larger than
what a single Git repository can deal with. The combination of git-annex's
capabilities to separate Git hosting from data hosting in extremely flexible
ways with DataLad's approach to orchestrating collections of nested
repositories as a joint "mono repo" is the foundation for that.

One example of such a large dataset is the [WU-Minn HCP1200
Data](https://humanconnectome.org/study/hcp-young-adult/document/1200-subjects-data-release),
a collection of brain imaging data, acquired from more than a thousand
individual participants by the [Human Connectome
Project](http://www.humanconnectomeproject.org) (HCP). The [DataLad
Handbook](https://handbook.datalad.org) has an [article on how the HCP1200
DataLad dataset was
created](https://handbook.datalad.org/en/latest/usecases/HCP_dataset.html) a
few years ago. However, the focus of this blog post is not how it was created,
but rather how and where it is hosted.

The dataset comprises 15 million files with a total of 80TB size.  All files
are hosted on [AWS S3](https://aws.amazon.com/s3), in a bucket that is owned by
HCP. This is important, because who gets to access these files is decided by HCP.
Any dataset user needs to request and provide individual credentials. The DataLad
dataset only contains URL pointers to all 15 million individual files.

This separation of file content and version-controlled access pointers made it
possible to host [the dataset on
GitHub](https://github.com/datalad-datasets/human-connectome-project-openaccess),
and also to create special purpose views or slices of the dataset, such as for
the [functional magnetic resonance imaging data of the participants watching
movie clips](https://github.com/datalad-datasets/hcp_movies) during brain
imaging.

However, no single Git repository can contain all 15 million files without
digestive pains. Therefore, the whole DataLad dataset actually comprises about
4500 subdatasets, which are first-level and second-level Git submodules. Managing
such a number of repositories is not feasible without some programmatic
helpers. Moreover, these subdatasets are actually *not* hosted on GitHub, but
in a [RIA
store](https://handbook.datalad.org/en/latest/beyond_basics/101-147-riastores.html)
on a different server. Such a store is basically a collection of bare
repositories with an annex and some additional features. They can even be put
on machines that have no git-annex installed. But they are a bit of a black hole,
because they lack a built-in ability to browse or search them.

A Forgejo-aneksajo instance -- theoretically -- provides a much nicer
environment for hosting such a large dataset built from many components. Git-annex is
immediately available, no need for custom implementations.  There is a nice UI
that promises convenient browsing, and all the other goodies.  The big question
is:

How well can Forgejo handle this amount of repositories?

## Pushing the full dataset to Forgejo-aneksajo

Let's start with the result: [This organization on
hub.datalad.org](https://hub.datalad.org/hcp-openaccess) holds the full HCP1200
DataLad dataset. The [root dataset](https://hub.datalad.org/hcp-openaccess/hcp1200)
(or superdataset) is the entry point to ~4500 subdatasets. Each and every single
one of the 15 million file records is reachable via the Forgejo web UI!

Individual repositories in this organization were created by sequentially
obtaining the DataLad datasets for all individual study participants from the
original hosting.  Because I needed to touch every single repository, I decided
to do some pending git-annex metadata maintenance at the same time. Just for
posterity, here is the shell loop that got executed in the HCP1200 superdataset
clone:

```bash
for sub in $(ls -1 HCP1200); do
  # skip a participant dataset that is already around (has been processed before)
  test -d "HCP1200/${sub}/unprocessed" && continue
  # get participant dataset (with subdatasets), use annex private mode, because
  # this is a temporary clone, and we do not want to remember it
  datalad -c annex.private=true get -n -r "HCP1200/${sub}"

  # process subdatasets for individual data types
  for t in -unprocessed -T1w -MNINonLinear -MEG ''; do
    export dspath="HCP1200/${sub}/${t#-}"
    # not all participants have all components
    test ! -d "$dspath" && continue
    # declare previous temporary clones dead (just beautification)
    git -C "$dspath" annex dead \
        $(git -C "$dspath" cat-file -p git-annex:uuid.log | grep hcp_ | cut -d ' ' -f 1)
    # add hub.datalad.org as a new remote
    git -C "$dspath" remote add hub "ssh://git@hub.datalad.org/hcp-openaccess/${sub}${t@L}.git"
    # adjust the submodule URL in the superdataset to hub.datalad.org (enables browesability)
    [ -n "$t" ] && git -C "HCP1200/${sub}" submodule set-url \
        "${t#-}" "https://hub.datalad.org/hcp-openaccess/${sub}${t@L}.git"
    [ -z "$t" ] && git -C "HCP1200/${sub}" commit -m "Point submodules to hub.datalad.org" \
        .gitmodules
    # also adjust the URL in the root dataset
    [ -z "$t" ] && git submodule set-url \
        "${sub}" "https://hub.datalad.org/hcp-openaccess/${sub}.git"
    # push the updated dataset to hub.datalad.org
    git -C "$dspath" push --all hub
  done

  # drop a participant's subdatasets (leaner operation)
  datalad drop -r --what datasets HCP1200/${sub}/*
done
```

Because Forgejo supports
[push-to-create](https://forgejo.org/docs/latest/user/push-to-create/#push-to-create)
there was no API interaction necessary for establishing 4500 new projects. Just
adding a remote and git-push was sufficient.

As a side note: I used `git push` rather than `datalad push`, because there are
no annex keys to process or upload. As mentioned before, all file content is
in the HCP S3 bucket.

## Performance and load

The implementation shown above is purely serial. Minimizing the time to
completion was not the goal. Instead, I wanted to get some insight into the
resource consumption of Forgejo under these conditions to help size (virtual)
machines for future deployments. Moreover, I was interested in the growth of
the Forgejo database with a few thousand additional projects. Arguably, the
vast majority of these projects will never be visited in the web UI.  It makes
no sense to have, for example, a dedicated issue tracker for the unprocessed
MRI data of participant 100206.

So how did it go? Once completed

- the bare Git repositories of the `hcp-openaccess` organization on hub.datalad.org weighed 20.7GB
- the Forgejo SQLite database (`gitea/data/gitea.db`) grew from an initial 3.1MB to a moderate 57.5MB

This appears to be a negligible overhead.

Importantly, the web UI remained and remains snappy. Rendering the (paginated)
[repository listing](https://hub.datalad.org/hcp-openaccess) takes less than
200ms. Showing all 1113 participant subdataset handles on [a single
page](https://hub.datalad.org/hcp-openaccess/hcp1200/src/branch/main/HCP1200)
takes less than 500ms.

The following figure illustrates some key figures for resource consumption in
the virtual machine running hub.datalad.org over a six hour window near the end
of the test run.

It appears as if a deployment on a dual-core VM with 2GB RAM would have been
able to deliver the same performance as the test system -- keeping in mind that
the pushes to Forgejo were also running in the same VM, and that load would
typically not be present.

{{< figure
    src="system-stats.webp"
    caption="Rendering of some [netdata](https://www.netdata.cloud) statistics for a six hour window during the dataset creation procedure."
    alt="A summary load graph and four time-resolved plots for total and per-user CPU utilization, and total and per-user RAM usage. They key figures are a relatively constant CPU utilization by Forgejo (git) of 50% single-core, and an also near-constant RAM usage of ~320MB. Overall CPU load was 35% for this four-core VM, which includes the client processes that pushed the 4500 Git repositories to Forgejo."
    >}}

Running Forgejo maintenance operations, such a "Garbage collect all
repositories", on this instance after loading it up with this many projects
also showed a similarly lean resource footprint.

## Conclusions

Everything worked smoothly. There were no indications of performance issues,
neither during the creation procedure, nor after the instance was populated.

If anything could cause problems, it would be listing Git trees (directories)
with many entries. A preliminary test with a different dataset that contains
45k submodule records in a single directory shows a multi-minute rendering time
for the corresponding page. The system utilization remained low, however, and even
the web UI stayed (somewhat) responsive. There is a [Forgejo
issue](https://codeberg.org/forgejo/forgejo/issues/4119) that is already asking
to mitigate this problem for some scenarios.

Overall, the presentation of datasets with many directory entries is nicer than
on GitHub. For example, compare [this dataset on
Forgejo](https://hub.datalad.org/hcp-openaccess/hcp1200-functional-connectivity)
with the [same dataset on
GitHub](https://github.com/datalad-datasets/hcp-functional-connectivity).
GitHub cuts the content listing off hard. A web UI user has no chance to even
realize that there is a README explaining the purpose of the dataset -- much
nicer with Forgejo.

Bottom line: With a small VPS, in the price range of 5-10 Euro/month, it is
possible to host a Forgejo instance that can (easily) handle thousands of
repositories.  It provides a convenient and fast front end to explore such
datasets. There is no indication that an instance deployed for this purpose
would require a more sophisticated database server setup than the default
SQLite.
