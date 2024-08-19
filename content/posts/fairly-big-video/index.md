---
title: "Fairly big video workflow"
author:
- Michał Szczepanik
date: "2024-08-16T18:30:00+02:00"
showtoc: true
tags:
- DataLad
- git-annex
- HTCondor
- video editing
- metadata
description: >
  TODO
cover:
  image: cover.webp
  alt: "Screenshot of a video page of the dataset described in this post as hosted at https://hub.datalad.org/distribits/recordings, and the FFmpeg, HTCondor, git-annex, and DataLad logos on top."
  relative: true
---

Two years ago, my colleagues published [FAIRly big: A framework for computationally reproducible processing of large-scale data](https://doi.org/10.1038/s41597-022-01163-2).
In this paper, they describe how to partition a large analysis (their example: processing anatomical images of 42 thousand subjects from UK Biobank), using DataLad to provision data and capture provenance, so that individual results can be reproduced on a laptop, even though a cluster is needed to run the entire group analysis.
The article is accompanied by a [workflow template](https://github.com/psychoinformatics-de/fairly-big-processing-workflow/) and a [tutorial dataset](https://github.com/psychoinformatics-de/fairly-big-processing-workflow-tutorial).

This year, we organised [Distribits](https://distribits.live/), a conference about technologies for distributed data management.
The talks were live-streamed on YouTube, and we promptly cut the long livestreams into a [playlist](https://www.youtube.com/playlist?list=PLEQHbPfpVqU6esVrgqjfYybY394XD2qf2), but given the conference theme it was clear from the beginning that we would want to publish the recordings as a [git-annex](https://git-annex.branchable.com/) repository, too.

How exactly things would be done was also important to us. In particular, we wanted to:

-   re-encode the YouTube videos using open codecs and container formats,
-   include metadata (titles, speakers, license, etc.) both in the video container and as [git-annex metadata](https://git-annex.branchable.com/metadata/),
-   record the video processing commands as [DataLad run records](https://handbook.datalad.org/en/latest/basics/101-108-run.html) for an informative and actionable git history.

The FAIRly big workflow was a good fit for the task.
Although not strictly necessary at this scale, I preferred to use my Institute's cluster, which meant using [HTCondor](https://htcondor.org) scheduler.
This was my first "real" project on the cluster, and my first "proper" implementation of the workflow.

What followed was a good deal of exploration of DataLad, git-annex, and HTCondor features, as well as video encoding and metadata details.
Below, I will break down the process and share what I learned.
The resulting dataset (videos and code to generate them) is available from [hub.datalad.org/distribits/recordings](https://hub.datalad.org/distribits/recordings).


## Video encoding decisions {#video-encoding-decisions}


### Codec and container choice {#codec-and-container-choice}

After short experimentation and discussion, we settled on using AV1 and opus codecs for video and audio respectively.
They allowed us to keep a very satisfying balance between size and quality (keeping most of the 20-minute talks under 100 MB), while staying clear of any concerns related to proprietary codecs[^fn:1].

For the container format we chose WebM, which is a limited subset of Matroska, and has a nice benefit of being playable in web browsers.

[FFmpeg](https://ffmpeg.org/) was the natural choice to run the video conversion.


### Metadata woes {#metadata-woes}

[Matroska metadata format specification](https://matroska.org/technical/tagging.html), (which also applies to webm) allows tagging different organization levels (think: track, album, ...), which can be inside or outside the file[^fn:2].
With that mechanism, it was possible to add titles for both the talk and conference, in a way which was picked up by vlc and mpv media players, by using "chapter" (30) and "movie/episode" (50) level tags, respectively.
It would have been more semantically correct to use "collection" (70) level for the conference, however vlc seems to fixate on levels 30/50 (probably because for audio files they correspond to track/album) and refuses to use others.

Another practical concession was needed for the artist tag.
The specification says: "Multiple items SHOULD never be stored as a list in a single TagString. If there is more than one tag of a certain type to be stored, then more than one SimpleTag SHOULD be used."
However, when I added multiple artist tags, vlc only showed the last one---and GNOME Video showed the _first_.

The metadata were added using mkvpropedit[^fn:3], a dedicated tool from the [mkvtoolnix](https://mkvtoolnix.download/) suite.
However, a popular tool for viewing and editing various kinds of metadata, [exiftool](https://exiftool.org/) needs to be called with an additional `-ee` (extract embedded) option to show them (in fairness, its [documentation says so](https://exiftool.org/TagNames/Matroska.html)).


## Initial dataset setup {#initial-dataset-setup}

A dataset has been set up with two YouTube videos to be sliced (one livestream per day, about 8 hours each) and a tsv file with information about individual talks (start and stop times, titles, speakers, etc.).
The videos have been added as URL keys pointing to YouTube -- and that probably requires a little bit of explanation.


### git-annex URL keys {#git-annex-url-keys}

If you got introduced to git-annex via DataLad, you probably know that annexed files are stored under keys, typically based on file size and checksum, (e.g. `MD5E-s109468445--cfeba7e9a8264ad524dab1bbdd56ac5b.webm`).
However, there are more [git-annex backends](https://git-annex.branchable.com/backends/) than those based on checksums.
One of them is `URL`, in which the key is generated from the file URL, without checksumming.
This means that there is no guarantee about the file content integrity, but it can be used when downloads are deferred for the future, the source can be trusted, or the content integrity does not matter that much.

A special case of the URL keys is `URL--yt`: if git-annex is asked to `get` such key, it will use [yt-dlp](https://github.com/yt-dlp/yt-dlp) to download the video.
This requires yt-dlp to be installed, and security rules to be relaxed (`git -c annex.security.allowed-ip-addresses="all" annex get...`) [^fn:4].

Such data can be migrated to use a more standard checksum backend with [git annex migrate](https://git-annex.branchable.com/git-annex-migrate/).
A typical use case would be adding the URL first and saving the downloaded content later.
However, git-annex will also accept transferring data saved with the URL backend to and from special remotes without changing the backend, although getting would require allowing unverified downloads from that remote.


### DataLad containers setup {#datalad-containers-setup}

The decision to use mkvpropedit to edit video metadata meant that software unavailable on our cluster was required.
To solve that, I decided to create a Singularity container with ffmpeg, mkvtoolnix, python3, and locales&nbsp;[^fn:5], which came out at a reasonable 286 MB.

After adding the image to the dataset, I had to instruct DataLad how to use it, with the `containers-add` command from the DataLad-container extension&nbsp;[^fn:6]:

```bash
datalad containers-add \
  -d . \
  --image singularity/converter.sif \
  --call-fmt "singularity run {img} {cmd}" converter
```


## Conversion code {#conversion-code}

The main conceptual challenge was to break up the processing into three related but independent layers, and figuring out how to pass inputs between them:

1.  Basic processing of a single video: a single script which would be called via DataLad (containers-)run as part of the workflow, but should also work on its own.
2.  DataLad orchestration: setting up a temporary dataset, retrieving inputs, processing a single video, capturing provenance, and transferring the output to storage.
3.  Condor submit file: scheduling the above to be repeated for each talk.


### Basic processing unit: `render_video.sh` {#basic-processing-unit-render-video-dot-sh}


#### Inputs: command line and environment variables {#inputs-command-line-and-environment-variables}

Because the long videos were accompanied by a table with information about talks (a tsv file), only two inputs were essential: relative path to the collection directory (we anticipated storing videos from more events), and the number of the desired video clip (i.e. its row number in the table).
The input file, cut points, and all required metadata would be read from the table.

At first, I wanted the arguments to include the number of threads used for encoding, too.
However, this argument is semantically different than the other two (it describes how to arrive at the result, and not what the result will be), and encoding it in the run record would cause any rerun to use the same number of threads, which may not be desired.

Then I learned that Condor supports setting environment variables in the submit file.
Not only that, it [sets some commonly used ones](https://htcondor.readthedocs.io/en/latest/users-manual/env-of-job.html#extra-environment-variables-htcondor-sets-for-jobs), such as `OMP_NUM_THREADS` by default.
Hence, I can rely on the variable being set, by Condor or the user, and provide a default behavior (no thread limit) as an alternative.

The \`set -e -u\` at the beginning tells bash to exit immediately when a pipeline returns an error, and to treat undefined variables as errors [^fn:7].

```bash { linenos=true, linenostart=1 }
set -e -u

collection_dir=$1
clip_no=$2

# set number of threads for SVT-AV1
# use n_threads or OMP_NUM_THREADS variables if defined; otherwise 0, i.e. all available
if [[ -v n_threads ]]; then
    :
elif [[ -v OMP_NUM_THREADS ]]; then
    n_threads=$OMP_NUM_THREADS
else
    n_threads=0
fi
```

**Friction point**: DataLad run records capture the command and its parameterisation, but not the environment.
In this case, this is advantageous, as the number of threads should be orthogonal to the produced result.
If, however, a parameter was crucial for generating the result, that would be a limitation to its reproducibility.


#### Inputs read from a file in dataset {#inputs-read-from-a-file-in-dataset}

The next step is to read more detailed inputs required by ffmpeg (e.g. start and end points) from the contained tsv file.
At first, I wanted to use a simple pipe, such as `head -n $clip_no | tail -n 1 | read -r ...`.
It turns out such construct would work in zsh, but not in Bash, since bash executes each pipe element in a subprocess [^fn:11] (TIL), so the variables assigned by read would be lost.
In the end, I used this solution, with awk used for getting one line of the file, and a here-string redirection to the read command:

```bash { linenos=true, linenostart=16 }
# read one line from clips.tsv
IFS=$'\t' read -r source collection license date track start end name speakers title abstract \
   <<< $(awk -v row="$clip_no" "NR == row {print}" "${collection_dir}/clips.tsv")
```


#### Encoding and metadata embedding {#encoding-and-metadata-embedding}

Next comes the crucial piece, processing the video and adding metadata:

```bash { linenos=true, linenostart=20 }
# convert
ffmpeg -y -i "${collection_dir}/${source}" \
       -ss "$start" \
       -to "$end" \
       -c:v libsvtav1 -preset 6 -svtav1-params lp="$n_threads" \
       -c:a libopus -ac 1 -ab 24k \
       "${collection_dir}/${date}_${track}_${name}.webm"

python3 code/create_xml.py "${collection_dir}/clips.tsv" "$clip_no" "/tmp/${date}_${track}_${name}.xml"

mkvpropedit "${collection_dir}/${date}_${track}_${name}.webm" \
            --tags "all:/tmp/${date}_${track}_${name}.xml"

```

The ffmpeg call (lines 21-26) is filled with the previously described parameters and some constant values.
For the video encoding (`-c:v`), preset 6 was chosen following the encoder documentation ("Presets 4-6 are commonly used by home enthusiasts as they represent a balance of efficiency and reasonable compute time" -- [Common questions](https://gitlab.com/AOMediaCodec/SVT-AV1/-/blob/master/Docs/CommonQuestions.md)) and after confirming that it can encode at roughly 0.5 ✕ realtime speed.
Clipping the number of threads with `-svtav1-params`&nbsp;[^fn:8] was done to avoid exceeding the number of threads requested from the job scheduler.
Setting the audio track (`-c:a`) to mono (`-ac 1`) and lowering the bitrate (`-ab 24k`) shaved off another few megabytes.
In our case, the audio channels were identical, so using mono simply removed redundancy -- but it is also preferred for verbal content (in my experience things become not so nice for listening on headphones if the speaker sound happens to be placed off-center).

Then (lines 28-31) comes the addition of metadata.
I wrote a Python script ([create_xml.py](https://hub.datalad.org/distribits/recordings/src/branch/master/code/create_xml.py)) to generate a required xml file on the fly (written into `/tmp` and thus not saved in the dataset)[^fn:12] and used that as an input to mkvpropedit, setting and overwriting all existing tags (it is possible to set them on a per-stream basis, but this seemed not necessary).


### DataLad orchestration: `video_job.sh` {#datalad-orchestration-video-job-dot-sh}

The script above is sufficient to produce a single talk recording.
The next challenge is to do so under the control of DataLad.
Here's the full DataLad orchestration script, explanations will follow:

```bash { linenos=true, linenostart=1 }
#! /bin/bash
set -e -u

# input parameter(s)
clip_no=$1

# parameters to be read from the environment variables:
#
# $collection_dir  - directory with video files and clips.tsv
# $dssource        - URL for DataLad clone
# $storage_name    - name of special remote to pull / push
# $dslockfile      - lock file for push, should be accessible to all jobs

# parameters read from the environment by the render_video script:
#
# $n_threads OR $OMP_NUM_THREADS  - number of threads for SVT-AV1 to use when encoding

# make a temporary clone using annex.private to avoid recording availability in git-annex branch
datalad -c annex.private=true clone $dssource /tmp/distribits-videos
cd /tmp/distribits-videos
git config annex.private true

# create and check out clip-specific branch
git branch "clip-${clip_no}"
git switch "clip-${clip_no}"

# make local storage cost lower than web, and allow unverified downloads from there (url keys)
git config --local "remote.${storage_name}.annex-cost" 150
git config --local "remote.${storage_name}.annex-security-allow-unverified-downloads" ACKTHPPT

# read input and output file from the tsv to have explicit i/o for datalad run
# columns: *source* collection license *date* *track* start end *name* speakers title abstract
input_file=$(awk -F '\t' -v row=$clip_no 'NR==row {print $1}' < "${collection_dir}/clips.tsv")
output_file=$(awk -F '\t' -v row=$clip_no 'NR==row {print $4 "_" $5 "_" $8 ".webm"}' < "${collection_dir}/clips.tsv")

datalad containers-run \
        -m "Convert ${output_file}" \
        -n "converter" \
        --explicit \
        -o "${collection_dir}/${output_file}" \
        -i "${collection_dir}/clips.tsv" \
        -i "${collection_dir}/${input_file}" \
        bash code/render_video.sh $collection_dir $clip_no

# push result file content first - does not need a lock, no interaction with Git
datalad push --to $storage_name
# and the output branch next - needs a lock to prevent concurrency issues
# the lock file should be accessible to all jobs
flock --verbose $dslockfile git push origin

```

The beginning (lines 1-16) is fairly standard.
We only accept one command line argument, the clip number (so that we can easily loop over the clips in the submit file).
The other parameters, constant for all the clips, come via environment variables, which are listed in the comment.

The most interesting parts are in the middle.

First (lines 18-21), we perform a temporary (ephemeral) clone using [annex private mode](https://git-annex.branchable.com/tips/cloning_a_repository_privately/).
This is a novelty compared to the FAIRly big workflow paper (which relied on the [git annex dead](https://git-annex.branchable.com/git-annex-dead/) mechanism instead).
With the private setting, information about the newly-cloned location will be stored in a private journal under `.git/annex`, and not in the git-annex branch.
This means we do not have to call `git annex dead here` before pushing, and we avoid cluttering the git-annex branch with information about dead clones [^fn:13].
Because `datalad clone` combines `git clone` and `git annex init`, we first pass the configuration option on the command line, and write it to the local configuration later (see [this comment](https://git-annex.branchable.com/tips/cloning_a_repository_privately/#comment-2725ecbd413dffad080eab473787c530) and the following response under the private mode documentation page for how exactly that matters).

Next (lines 24-25), we set ourselves up for an octopus merge in the future, by switching to a new branch.
The original FAIRly big workflow makes sure that the branch names are unique by including a job identifier passed from Condor, here we simply use the clip identifier.

Then (lines 28-29), we want to make sure that the full video will not be getting downloaded from YouTube by each job.
For this, we set the cost of the RIA storage remote to below that of web (200)[^fn:14].
Since we kept the URL key (to maintain the ability to download from YouTube), we need to allow unverified downloads from the RIA storage, too.

**Friction point**: the RIA store typically combines the git remote with the annex special remote.
After cloning, the git remote is named origin, but the special remote keeps its initial name.
We could discover it by looking at `datalad siblings` but it is simpler to pass it to the script from outside.

Getting the names of the video files (read and generated) from the included text file is done (on lines 33-34) with `awk` (`-F` to declare field separator, `NR` to get the desired line).
The video processing script will have to read the file yet again to get the cut points -- a slight inefficiency, but insignificant time-wise compared to actual processing.

The `render_video` script is wrapped in `datalad containers run` (lines 36-43).
Based on the given arguments, DataLad picks the previously configured container (`-n`), retrieves the required inputs from locations known to git-annex (`-i`), and saves the declared outputs (`-o`, `--explicit`).

**Friction point** When `datalad containers run` has been used, the `datalad rerun` will also try to use the container.
In other words, it is currently not possible to run in the container, and rerun natively.
It can be argued to be better for reproducibility, but it makes Singularity a dependence for reruns.

Finally (lines 46, 49) we push back to the RIA store.
Compared to FAIRly big, our setup here is simpler, in that the input and output stores are the same.
The main quirk remains: we need to use `flock` to avoid problems if two git pushes happen at the same time.
The lock file needs to be placed outside our (ephemeral) working directory, in a place available to all the jobs.


### Scheduling: `process_videos.submit` {#scheduling-process-videos-dot-submit}

The final level in the workflow is scheduling the jobs to run on the cluster, which is accomplished with the following submit file:

``` plaintext { linenos=true}
# The environment
universe       = vanilla
getenv         = True
request_cpus   = 4
request_memory = 4G
request_disk   = 4G

# tell condor that a job is self contained and the executable
# is enough to bootstrap the computation on the execute node
should_transfer_files = yes
# explicitly do not transfer anything back
# we are using datalad for everything that matters
transfer_output_files = ""

# The actual condor-independent job script
executable     = $ENV(PWD)/code/video_job.sh

environment = "\
  collection_dir=distribits2024 \
  dssource='ria+file:///data/group/psyinf/distribits/distribits.ria#~distribits-videos' \
  storage_name=juseless-storage \
  dslockfile=$ENV(PWD)/.condor_datalad_lock \
  "

# Logs
log            = $ENV(HOME)/logs/$(Cluster).$(Process).log
output         = $ENV(HOME)/logs/$(Cluster).$(Process).out
error          = $ENV(HOME)/logs/$(Cluster).$(Process).err

Queue 1 arguments from seq 1 29 |

```

Most of the content is specific to the condor scheduler.
The parameter definitions can be found in the [condor_submit manpage](https://htcondor.readthedocs.io/en/latest/man-pages/condor_submit.html), and building the file is explained in the [Submitting a job](https://htcondor.readthedocs.io/en/latest/users-manual/submitting-a-job.html) tutorial.
Let's focus on what matters in our context - and it is mostly parametrizing a job.

The CPU and memory requests (lines 1-6) are mostly just promises, and they are used to find matching machines.
However, as explained earlier, Condor will set some environment variables [^fn:9] typically obeyed by multi-threaded programs, and we made sure to use one of them, `OMP_NUM_THREADS` when running ffmpeg.

The next section (lines 8-13) describes Condor behavior when transferring files between the submit machine and the machine running the jobs, in the presence or absence of a shared file system.
It is copied from the FAIRly big workflow, together with the comments.

As the executable (line 16), we specify the bash script with DataLad orchestration, introduced above.
The script needed several inputs to be provided as environment variables, and we do that with `environment` (lines 18-22).

Finally, we wrote that script so that it takes only one input argument, the index of the clip in the list of all clips.
We benefit from it now (line 30), by writing the simplest queue command (`Queue ... from seq`) [^fn:10].

**Friction point** Condor could also run the given executable in a singularity container, and it is very easy to do so (by specifying `container_image`).
So in theory we could use the regular `datalad run` command (instead of `containers run`), and specify the container on Condor level.
We could also include DataLad in that container, if it hadn't been available on our cluster.
However, using `containers run` seems good for portability.


## Git-annex metadata {#git-annex-metadata}

It was important for us to additionally expose the metadata included in the video containers as git-annex metadata.
We did it after all videos were processed. A script to loop over all videos and properties can be found in our published dataset ([add_annex_metadata.py](https://hub.datalad.org/distribits/recordings/src/branch/master/code/add_annex_metadata.py)), but the API is very intuitive (and has no problem with multiple speakers) and comes down to:

```bash
git annex metadata \
  --set title="Talk title" \
  --set speaker+="Jane Doe" \
  --set speaker+="John Doe" \
  file.webm
```

Adding annex metadata isn't done just for the sake of it, but it enables practical operations, shown below.

Show metadata for a single file (grep is used in the example to filter out change dates):

```nil
❱ git annex metadata distribits2024/2024-04-04_01_welcome_day1.webm | grep -v lastchanged
metadata distribits2024/2024-04-04_01_welcome_day1.webm
  abstract=Welcome from the organizers
  collection=Distribits 2024
  date=2024-04-04
  license=CC-BY-3.0
  name=welcome_day1
  speaker=Joey Hess
  speaker=Michael Hanke
  speaker=Yaroslav Halchenko
  title=Welcome and overview
  track=01
ok
```

Search, e.g. for a videos listing the given speaker:

```nil
❱ git annex find --metadata speaker="Joey Hess"
distribits2024/2024-04-04_01_welcome_day1.webm
distribits2024/2024-04-04_03_hess_gitannex_complete.webm
distribits2024/2024-04-04_05_discussion_halchenko_hanke_hess.webm
distribits2024/2024-04-04_08_discussion_hardcastle_thoenniessen.webm
distribits2024/2024-04-05_12_unconference.webm
```

Restructure the dataset with [metadata driven views](https://git-annex.branchable.com/tips/metadata_driven_views/).
This is particularly interesting: it uses git branches, and because annexed files are usually symlinks, it is very fast.
The example below reorders into folders organized by date and then title:


```nil
❱ git annex view "date=*" "title=*"
view (searching...)
Switched to branch 'views/date=_;title=_'
ok
❱ tree -d
.
├── 2024-04-04
│   ├── Balancing Efficiency and Standardization for a Microscopic Image Repository on an HPC System
│   ├── DataLad beyond Git, connecting to the rest of the world
  ...
├── 2024-04-05
│   ├── A Tour of Magit
│   ├── DataLad-Registry﹕ Bringing Benefits of Centrality to DataLad
  ...
```

The `git annex vpop` command goes back to the previous view (or no view at all).
From there we can try another useful one, i.e. group by speakers:

```
❱ git annex view "speaker=*"
view  (searching...)
Switched to branch 'views/master(speaker=_)'
ok
❱ tree -d
.
├── Alex Waite
├── Felix Hoffstaedter
...
```


## Summary {#summary}

When laid out as above, the whole process seems logical -- but there is a lot of forethought in the narrative, and the actual implementation work involved some back-and-forth and moving things around between the processing levels.
Getting to the final product involved a lot of conceptualization, trial and error, and learning about DataLad and git-annex features.
However, the process was interesting, and the end product is very satisfying.

Queuing 29 jobs, each expected to last about an hour, and seeing half of them start immediately is a great feeling -- and a nice perk of working in a place with a hardware setup which allows that.

[^fn:1]: See: [An Invisible Tax on the Web: Video Codecs](https://blog.mozilla.org/en/mozilla/royalty-free-web-video-codecs/) from Mozilla's blog, dist://ed.
[^fn:2]: For example, a music album can be split into several files, and each file can store metadata about both the contained track and the album it came from; respective titles are stored as two "title" tags with different target type values assigned. However, matroska container can also hold all tracks within a single file, with the same metadata logic.
[^fn:3]: Metadata can also be added using ffmpeg, but it does not support multiple occurrences of a key, so there is no way to have multiple artists (other than concatenating them), or to have hierarchical titles (for both the talk and conference).
[^fn:4]: Otherwise, git-annex shows a message with explanation ("This url is supported by youtube-dl, but youtube-dl could potentially access any address, and the configuration of annex.security.allowed-ip-addresses does not allow that."), and searching the [git-annex man page](https://git-annex.branchable.com/git-annex/) for "yt-dlp" helps clarify that it needs to be set to "all" because yt-dlp has no way of enforcing tighter restrictions.
[^fn:5]: I discovered that mkvpropedit tries to be locale-aware, and crashes in a bare-bones container; including locales and generating the English one (matching the cluster) seemed to be the easiest way out
[^fn:6]: See [Computational reproducibility with software containers](https://handbook.datalad.org/r?containers) chapter of the DataLad handbook for a thorough introduction.
[^fn:7]: "This builtin is so complicated that it deserves its own section", starts the bash manual section about [The Set Builtin](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html)
[^fn:8]: Note that this is a parameter provided to the encoder; I could not find a way to effectively limit CPU usage with ffmpeg's own arguments. The full list of [SVT-AV1 Parameters](https://gitlab.com/AOMediaCodec/SVT-AV1/-/blob/master/Docs/Parameters.md) can be found in SVT-AV1 docs. 
[^fn:9]: [Extra Environment Variables HTCondor sets for Jobs](https://htcondor.readthedocs.io/en/latest/users-manual/submitting-a-job.html) from Condor "Environment and services for a running job" user manual.
[^fn:10]: [Submitting many similar jobs with one queue command](https://htcondor.readthedocs.io/en/latest/users-manual/submitting-a-job.html#submitting-many-similar-jobs-with-one-queue-command) from Condor "Submitting a Job" user manual
[^fn:11]: Bash manual: [Pipelines](https://www.gnu.org/software/bash/manual/bash.html#Pipelines)
[^fn:12]: In our Condor configuration, each job has its own isolated `/tmp` directory, but the file names are kept distinct to avoid conflicts when processing in a local environment.
[^fn:13]: Git-annex might not act on the repositories declared dead, but it does not easily forget. The UUIDs are still stored in `uuid.log`, and they just get an X in the `trust.log`; see the [git-annex internals](https://git-annex.branchable.com/internals/) page. The assumption is that, e.g., a lost pendrive can be found, and declared undead. This won't happen to our temporary clones. When their number starts going into thousands (like in FAIRly big), the storage cost for "wasted lines" is still small; nevertheless piling up meaningless UUIDs has always bothered me. The private mechanism, introduced in git-annex 8.20210428, seems much more appropriate for temporary, or "ephemeral", clones.
[^fn:14]: This might not be strictly necessary: DataLad sets the cost of the ORA (RIA -storage) remote to 100 if it is local, and 200 if it is remote, see [this piece of code](https://github.com/datalad/datalad/blob/d4ce9ce377bde2d59fc3a550938b51931dbfbb7b/datalad/distributed/ora_remote.py#L1607-L1613), but better safe than sorry.
