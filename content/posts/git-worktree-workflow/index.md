---
title: "Git Worktrees + DataLad: The Missing Link Between Daily Development and Reproducible Analysis"
date: 2025-12-08T19:42:00+02:00
author:
- Jiameng Wu
tags:
- Datalad
- git-worktree
- YODA
- Workflow
- data-analysis
cover:
  image: cover.jpeg
  alt: 
  relative: true
description: >
  Bridging fast iteration during development and reproducible batch processing with git worktrees and DataLad. 
  A practical workflow that is suitable for the 90% of work that lies between the initial exploration and the final publication.
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

In neuroscience research, we often find ourselves caught between two worlds: the messy, exploratory world of data analysis where we iterate rapidly on code, and the structured, reproducible world of scientific publication where every figure must be exactly reproducible. Between these worlds lies a labyrinth of processing pipelines, half-written scripts, and the ever-present risk of breaking your working analysis while trying to improve it.

Today's solutions often fall into two extremes: heavyweight reproducible workflows like [fairly-big-processing-workflow](https://github.com/psychoinformatics-de/fairly-big-processing-workflow) that are beautiful for final analysis but feel like overkill for daily development, or ad-hoc scripts that get the job done but leave no trace of how they did it.

At the recent [Distribits](https://www.distribits.live/events/2025-distribits/) meeting, I shared my struggle with [Yarik](https://github.com/yarikoptic), and confessed that I never fully understood the [YODA principle](https://handbook.datalad.org/en/latest/basics/101-127-yoda.html) - I was on the way, but I couldn't see clearly where I was heading. Together, we've found a sweet spot that gives me both **fast iteration during development** and **clean reproducibility for batch processing** - all without duplicating data or rebuilding containers. The secret? [Git worktrees](https://git-scm.com/docs/git-worktree) combined with [DataLad](https://handbook.datalad.org/en/latest/index.html)'s nested datasets.

### When YODA's wisdom grows on trees
```
Meng_Psychedelics/ (superdataset)
├── raw/
│ ├── L5b_2p/ (subdataset)
│ └── L5b_bpod/ (subdataset)
├── derived/
│ └── L5b/ (subdataset)
│   ├── inputs/
│   │ ├── L5b_2p/ (subdataset → ../../raw/L5b_2p)
│   │ └── L5b_bpod/ (subdataset → ../../raw/L5b_bpod)
│   ├── code/ (subdataset)
│   ├── containers/ (subdataset)
│   ├── 01_suite2p/
│   ├── ...
│   └── 04_dataframes/
└── ...
```
The active work in my project happens in the 'derived/L5b' subdataset that consumes raw data as inputs and produces multiple intermediate outputs, e.g. '01_suite2p' ... '04_dataframes'. The subdataset 'code' is a pure git repo and I use [Jujutsu](https://docs.jj-vcs.dev/latest/) for active development (because it's such a beautiful tool). After a rather nerve-wracking experience of mixing datalad with `jj` - it was like Schrödinger's cat, everything was simultaneously staged and unstaged 😱 - I restrict `jj` usage to 'code', and continue using `datalad` for managing all subdatasets with annexed content, orchestrating across nested datasets as a whole, and for capturing provenance of all derived data and figures with the [datalad (containers-)](https://docs.datalad.org/projects/container/en/latest/generated/man/datalad-containers-run.html) [run](https://docs.datalad.org/en/latest/generated/man/datalad-run.html#datalad-run) command.

### Neither ready to go big nor go home
Usually, I craft one [`datalad run`](https://docs.datalad.org/en/latest/generated/man/datalad-run.html) command, test it, and then write a bash script to run it for multiple subjects and experiments. Here is when it becomes painful: once it's running you can't carry on working, because any modification will interrupt the next `datalad run` command. Ideally, I can run things in an isolated environment that is clean and stays untouched until the job finishes. Git worktrees come in handy, because they essentially access the same annexed content (so no time wasted in getting content for another clone), but live in a different directory on the disk. Worktrees share the annex and repo history, but not the state the repo is in. This allows me to checkout different branches simultaneously since they live on different worktrees.

Since there is no automatic way to create a nested worktree (described in [this issue](https://github.com/datalad/datalad/issues/7764)), I manually create a worktree for each subdataset inside 'derived/L5b':

```bash
# add a worktree on a new branch with the name 'runs' for the superds
M/d/L5b (master)> git worktree add /mnt/Data/L5b-runs -b runs
Preparing worktree (new branch 'runs')
HEAD is now at fde26554f [DATALAD RUNCMD] plot rois, traces and F displots for mouse J, Lisuride

# repeat for each subdataset
M/d/L5b (master)> cd inputs/L5b_bpod
M/d/L/i/L5b_bpod (master)> git worktree add /mnt/Data/L5b-runs/inputs/L5b_bpod -b runs
Preparing worktree (new branch 'runs')
HEAD is now at e32a9bbf2c [DATALAD RUNCMD] Rename bpod files of mouse J, LSD with the missing date.

M/d/L/i/L5b_bpod (master)> cd ../L5b_2p
M/d/L/i/L5b_2p (master)> git worktree add /mnt/Data/L5b-runs/inputs/L5b_2p -b runs
Preparing worktree (new branch 'runs')
HEAD is now at 3bd0b284b2 rename interrupted recording

M/d/L/i/L5b_2p (master)> cd ../../code
M/d/L/code> git worktree add /mnt/Data/L5b-runs/code runs # here I skip -b because the branch already exists
Preparing worktree (checking out 'runs')
HEAD is now at ef5dafb generate_plots: plot rois, traces, and F displots
```

The branch 'runs' in `code/` has already been created, when I got ready with the code:
```bash
# the new datalad run command is ready
M/d/L/code> git switch -c runs
Switched to a new branch 'runs'

# 'runs' points to the commit ef5dafb4: generate_plots
M/d/L/code> jj log
Done importing changes from the underlying Git repo.
@  turmruqs jiameng.wu@gmail.com 2025-12-03 13:38:37 f8743c11
│  (empty) (no description set) # <- the clean state we need
│ ○  tnxopmxz jiameng.wu@gmail.com 2025-12-03 13:38:37 2cd4912a
├─╯  bar plot of mice IJK # <- this is just another dev branch
○  vlwmszww jiameng.wu@gmail.com 2025-12-03 13:21:37 master?? runs git_head() ef5dafb4
│  generate_plots: plot rois, traces, and F displots # <- target commit
```
For all perspective jj users, having the 'empty' change on top of the target commit is a necessary condition for a clean working tree and thus for datalad run to happen.

So the plan is to park the worktree 'runs' at this clean state on top of the commit 'ef5dafb4', run the script 'generate_plots.py' on multiple subjects and experiments, while continuing developing on dirty branches. Let's see if that works out!

In the newly created worktree 'runs', I verify that it is in the right state that records the successful `datalad run` command:
```bash
# in worktree 'runs'
/m/D/L5b-runs (runs)> git log --oneline
fde26554f (HEAD -> runs, master) [DATALAD RUNCMD] plot rois, traces and F displots for mouse J, Lisuride
```

Nice, I will first attempt a rerun! Oh wait, not so fast, I need to set up my [Python virtual environment](https://docs.python.org/3/library/venv.html) using a wonderful and fast [`uv`](https://docs.astral.sh/uv/) tool. So I go:
```bash
cd code
uv sync
source .venv/bin/activate.fish
cd ..
```

Now, RERUN!
```bash
/m/D/L5b-runs (runs)> datalad rerun
[INFO   ] run commit fde2655; (plot rois, traces...) 
[INFO   ] Making sure inputs are available (this may take some time) 
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_01.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_02.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_03.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_04.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_06.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_07.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_08.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_somata.png (file)    
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_tuft.png (file)      
unlock(ok): 05_visualization/240227J_Sim-Cre/rois/all_cells.png (file)          
  [1 similar message has been suppressed; disable with datalad.ui.suppress-simil[INFO   ] == Command start (output follows) =====                               
Stacking the dataframe into trial format ...
Finished in 0.1674485206604004 second.
Plotting ROIs on top of the mean image for each cell ...
Plotting fluorescence traces for each cell across multiple ROIs ...
Plotting the distribution of the fluorescence values for each cell ...
[INFO   ] == Command exit (modification check follows) ===== 
  [18 similar messages have been suppressed; disable with datalad.ui.suppress-similar-results=off]
run(ok): /mnt/Data/L5b-runs (dataset) [python code/src/process2p/generate_plots...]
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_01.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_02.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_03.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_04.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_06.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_07.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_08.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_somata.png (file)       
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_tuft.png (file)         
add(ok): 05_visualization/240227J_Sim-Cre/rois/all_cells.png (file)             
  [1 similar message has been suppressed; disable with datalad.ui.suppress-simil  [18 similar messages have been suppressed; disable with datalad.ui.suppress-similar-results=off]
action summary:                                                                 
  add (ok: 28)
  get (notneeded: 3)
  run (ok: 1)
  save (notneeded: 4)
  unlock (ok: 28)

/m/D/L5b-runs (runs)> git log
fde26554f (HEAD -> runs, master) [DATALAD RUNCMD] plot rois, traces and F displots for mouse J, Lisuride
```
It completes without creating a new commit - the outputs are byte-for-byte identical to the previous run. This is where the magic happens: I'm running from a completely different directory, yet I don't need to `datalad get` anything or `datalad save` results. The annex's clever symlinks make the same data available in both worktrees, eliminating redundancy entirely! 🎩✨

Now it's time to write that bash script, and to test how going forth and back between worktrees works. To not mix things up, I will never develop in the worktree 'runs', but only update it with what ever has been developed and become available on 'master' - well, actually we don't care about branches in jj's universe but anyway you get the gist.

```bash
#!/usr/bin/env bash
# This is the batch-processing script
set -u

# -------------------------------
# Target script
# -------------------------------
script="generate_plots.py"

# -------------------------------
# Inputs
# -------------------------------
subjects=("240226I" "240227J" "240229K")
drugs=("Ketamine" "LSD" "Lisuride" "Saline" "Saline2")

# -------------------------------
# Datalad RUNCMD
# -------------------------------
run_one() {
  local subid="$1" drug="$2"

  echo "=== ${subid} / ${drug} ==="

  datalad run \
    -m "Plot rois, traces and distribution of F_corrected for mouse  ${subid}, ${drug}." \
    -i "04_dataframes/${subid}_Sim-Cre/${drug}/plane0/roiset_suite2p/F.pkl" \
    -i "01_suite2p/${subid}_Sim-Cre/${drug}/plane0/ops.npy" \
    -i "01_suite2p/${subid}_Sim-Cre/${drug}/plane0/stat.npy" \
    -o "05_visualization/${subid}_Sim-Cre/${drug}" \
    "python code/src/process2p/${script} {inputs} {outputs}"  || {
        echo "ERROR: datalad run failed for ${subid} / ${drug}, but continuing..."
	datalad save -m "run ${subid} / ${drug} failed, save and continue with the next one"
  	return 0  # Return success to continue the loop
    }
}

# -------------------------------
# Main loop
# -------------------------------
failed_jobs=()

for subid in "${subjects[@]}"; do
  for drug in "${drugs[@]}"; do
    if ! run_one "${subid}" "${drug}"; then
      failed_jobs+=("${subid}/${drug}")
      echo "Recorded failur:  ${subid} ${drug}"
    fi
  done
done

echo "All done."
```
I have added this bash script under `pipelines/run_generate_plots.sh` path and updated the 'README.md', but the git branch 'runs' is still lagging behind and I'm in a 'HEAD detached' state:
```bash
# back in the main worktree
/M/d/L/code> jj log
@  mxvrsvoq jiameng.wu@gmail.com 2025-12-04 19:57:42 708ab665
│  (empty) (no description set)
○  yknyuprl jiameng.wu@gmail.com 2025-12-04 19:56:49 git_head() 84312427
│  update README.md # <- another new commit
○  turmruqs jiameng.wu@gmail.com 2025-12-04 19:56:29 08065905
│  add bash script to run generate_plots.py # <- the new commit
│ ○  tnxopmxz jiameng.wu@gmail.com 2025-12-03 13:38:37 2cd4912a
├─╯  bar plot of mice IJK
○  vlwmszww jiameng.wu@gmail.com 2025-12-03 13:21:37 master?? runs ef5dafb4
│  generate_plots: plot rois, traces, and F displots # <- 'runs' is still here

/M/d/L/code> git status
HEAD detached from refs/heads/runs
nothing to commit, working tree clean
```
Let's update that branch and sew the 'HEAD' back to 'runs'. There are multiple ways to do that, I use jj:
```bash
/M/d/L/code [2]> jj bookmark set runs -r 'git_head()'
Moved 1 bookmarks to yknyuprl 84312427 runs* | update README.md

# now both 'runs' and 'git_head()' point to the right commit
/M/d/L/code> jj log
@  mxvrsvoq jiameng.wu@gmail.com 2025-12-04 19:57:42 708ab665
│  (empty) (no description set)
○  yknyuprl jiameng.wu@gmail.com 2025-12-04 19:56:49 runs git_head() 84312427
│  update README.md # <- the right commit
○  turmruqs jiameng.wu@gmail.com 2025-12-04 19:56:29 08065905
│  add bash script to run generate_plots.py
│ ○  tnxopmxz jiameng.wu@gmail.com 2025-12-03 13:38:37 2cd4912a
├─╯  bar plot of mice IJK
○  vlwmszww jiameng.wu@gmail.com 2025-12-03 13:21:37 master?? ef5dafb4
│  generate_plots: plot rois, traces, and F displots
```

Back in worktree 'runs', I confirm the branch has been updated for 'code' and the superdataset also registers the update:
```bash
# in worktree 'runs'
/m/D/L/code (runs)> git log --oneline
8431242 (HEAD -> runs, refs/jj/keep/843124271281fc8e8a7a3ceff202a4fa6f0a08a8) update README.md
0806590 (refs/jj/keep/0806590504da65021ec465ddfac2fb3c7bc2c977) add bash script to run generate_plots.py
ef5dafb (refs/jj/keep/ef5dafb4ac4b7e9e2c6854cff726bede0c47ac53, master) generate_plots: plot rois, traces, and F displots

/m/D/L/code (runs)> cd ..

# in the superds of the same worktree
m/D/L5b-runs (runs)> datalad status
 modified: code (dataset) # need to save this modification
```
Should I also save this update in my main worktree? No, I will only save it in this worktree and then update the main worktree when the plots are ready! (What's datalad's behavior with branches anyway?)

### Seasoning with a pinch of Jujutsu magic
I'm almost ready to go, except that I have to amend another modification to the bash script. Out of curiosity I decide to use `jj squash` to add this modification to the previous change instead of creating a new commit. I observe that the commit ID changes, and that 'runs' still points to the correct commit:
```bash
# in main worktree
/M/d/L/code> jj squash --into turmruqs
Rebased 1 descendant commits
Working copy  (@) now at: oqwzryko 8af9e583 (empty) (no description set)
Parent commit (@-)      : yknyuprl 25eff5ce runs | update README.md

# the commit ID associated with the change ID 'turmruqs' has changed from '08065905' to '42c0112b'
/M/d/L/code> jj log
@  oqwzryko jiameng.wu@gmail.com 2025-12-04 20:18:57 8af9e583
│  (empty) (no description set)
○  yknyuprl jiameng.wu@gmail.com 2025-12-04 20:18:57 runs git_head() 25eff5ce
│  update README.md # <- the child commit changed automatically
○  turmruqs jiameng.wu@gmail.com 2025-12-04 20:18:57 42c0112b
│  add bash script to run generate_plots.py # <- modification squashed into this commit
```
How does this change of history affect the other worktree?
```bash
# in worktree 'runs'
/m/D/L/code (runs)> git log --oneline
25eff5c (HEAD -> runs, refs/jj/keep/25eff5cef05040cd0d54e4cfb36107c8f7708d0b) update README.md
42c0112 (refs/jj/keep/42c0112b40b45d58ce5d86bb57061ab5d0387d82) add bash script to run generate_plots.py
ef5dafb (refs/jj/keep/ef5dafb4ac4b7e9e2c6854cff726bede0c47ac53, master) generate_plots: plot rois, traces, and F displots
```
That's fantastic! I sneaked in a new modification without running any `datalad save` command and get the clean worktree for free! Hurrraaayy!!! But wait, git and datalad does detect the change:
```bash
# still in worktree 'runs'
/m/D/L/code (runs)> datalad status
 modified: pipelines/run_generate_plots.sh (file) # <- that's the batch-processing script

/m/D/L/code (runs)> git status
On branch runs
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   pipelines/run_generate_plots.sh
```
It is because the file has in fact changed due to the `jj squash` operation. I changed the history and there is a mismatch with the presence. All I need to do is to reset it to 'runs' with `git reset --hard runs`.

### The trade-off between efficiency and reproducibility: You win some you loose some

The script fails, and I try to rerun this one command that worked ealier, it fails as well:
```bash
/m/D/L5b-runs (runs)> datalad rerun fde26554f
[INFO   ] run commit fde2655; (plot rois, traces...) 
[INFO   ] Making sure inputs are available (this may take some time) 
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_01.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_02.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_03.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_04.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_06.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_07.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_08.png (file)        
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_somata.png (file)    
unlock(ok): 05_visualization/240227J_Sim-Cre/displots/cell_tuft.png (file)      
unlock(ok): 05_visualization/240227J_Sim-Cre/rois/all_cells.png (file)          
  [1 similar message has been suppressed; disable with datalad.ui.suppress-simil[INFO   ] == Command start (output follows) =====                               
Traceback (most recent call last):
  File "/mnt/Data/L5b-runs/code/src/process2p/generate_plots.py", line 9, in <module>
    from process2p.utils import *
ModuleNotFoundError: No module named 'process2p'
[INFO   ] == Command exit (modification check follows) ===== 
  [18 similar messages have been suppressed; disable with datalad.ui.suppress-similar-results=off]
run(error): /mnt/Data/L5b-runs (dataset) [python code/src/process2p/generate_plots...]
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_01.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_02.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_03.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_04.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_06.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_07.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_08.png (file)           
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_somata.png (file)       
add(ok): 05_visualization/240227J_Sim-Cre/displots/cell_tuft.png (file)         
add(ok): 05_visualization/240227J_Sim-Cre/rois/all_cells.png (file)             
  [1 similar message has been suppressed; disable with datalad.ui.suppress-simil  [18 similar messages have been suppressed; disable with datalad.ui.suppress-similar-results=off]
action summary:                                                                 
  add (ok: 28)
  get (notneeded: 3)
  run (error: 1)
  save (notneeded: 5)
  unlock (ok: 28)
```
What happened? - Apparently, I've lost the '.venv' directory during the hard reset which causes "ModuleNotFoundError: No module named 'process2p'". määäh! This actually speaks for the use of a container. The problem with the container is that I have to rebuild it everytime I update my code ... annoying! I guess that's the trade-off between efficiency and reproducibility. To illustrate this 'highly complex' dilemma with Deepseek's smart-ass comment in a graph:
```
        Fast Iteration ←──────────→ Reproducibility
              ↑                          ↑
          Convenience           Containerized Certainty
```
The attentive reader might now also want to blame my decision to rewrite the history instead of creating a new commit which would have avoided the `git reset` operation and the loss of the uv environment. I guess also here there is a trade-off between maintaining a clean history and keeping development momentum:
```
         Clean History ←──────────→ Development Flow
              ↑                          ↑
         Atomic Commits             Rapid Iteration
```

For now I will just repeat the `uv sync` operation from above and add this code snippet to all my future scripts:
```bash
# Always ensure .venv exists
if [ ! -f "code/.venv/bin/python" ]; then
    echo "WARNING: .venv missing! Recreating..."
    cd code
    uv sync
    source .venv/bin/activate.fish
    cd ..
fi
```

### Run, Datalad, Run!

Meanwhile, the script has finished and left me with a set of successful `[DATALAD RUNCMD]` and an ocean of plots that I can swim in for the next few days :D Let's push the results back to the main worktree! In my main worktree, I simply run `git merge runs`, and bahm, all the plots are there!
```bash
# back in the main worktree
/M/d/L5b (master)> git merge runs
Updating fde26554f..7f9f524f1
Fast-forward
 05_visualization/240226I_Sim-Cre/Ketamine/displots/cell_01.png     | 1 +
 05_visualization/240226I_Sim-Cre/Ketamine/displots/cell_02.png     | 1 +
 05_visualization/240226I_Sim-Cre/Ketamine/displots/cell_03.png     | 1 +
 05_visualization/240226I_Sim-Cre/Ketamine/displots/cell_05.png     | 1 +
 05_visualization/240226I_Sim-Cre/Ketamine/displots/cell_06.png     | 1 +
...

/M/d/L5b (master)> git log --oneline
7f9f524f1 (HEAD -> master, runs) [DATALAD RUNCMD] Plot rois, traces and distribution of F_corrected for mouse  240229K, Lisuride.
c2682b545 [DATALAD RUNCMD] Plot rois, traces and distribution of F_corrected for mouse  240229K, LSD.
923d34037 [DATALAD RUNCMD] Plot rois, traces and distribution of F_corrected for mouse  240229K, Ketamine.
80250575c [DATALAD RUNCMD] Plot rois, traces and distribution of F_corrected for mouse  240227J, Saline2.
...
```
I'm very happy! :D Also, meanwhile I did update my code repo with the code snippet above:
```bash
# still in the main worktree
/M/d/L/code> jj log
@  kxqpponr jiameng.wu@gmail.com 2025-12-04 21:18:05 fba5bff6
│  recreate venv inside the bash script # <- new commit
○  oqwzryko jiameng.wu@gmail.com 2025-12-04 20:18:57 git_head() 8af9e583
│  (empty) (no description set)
○  yknyuprl jiameng.wu@gmail.com 2025-12-04 20:18:57 runs 25eff5ce
│  update README.md
```
The change 'kxqpponr' has been created in the main worktree during the plotting that happened in the 'runs' worktree on top of the 'empty' change from where I sent off the plotting jobs. Now I can simply clean up with `jj abandon`.
```bash
/M/d/L/code> jj abandon oq
Abandoned 1 commits:
  oqwzryko 8af9e583 (empty) (no description set)
Rebased 1 descendant commits onto parents of abandoned commits
Working copy  (@) now at: kxqpponr 7e551486 recreate venv inside the bash script
Parent commit (@-)      : yknyuprl 25eff5ce runs | update README.md

/M/d/L/code> jj log
@  kxqpponr jiameng.wu@gmail.com 2025-12-04 21:21:45 7e551486
│  recreate venv inside the bash script
○  yknyuprl jiameng.wu@gmail.com 2025-12-04 20:18:57 runs git_head() 25eff5ce
│  update README.md
```
Beaauuutiful!

### All the magic in a nutshell
That was a very pleasant experience and quite a breakthrough in my personal data management journey. Between **daily development chaos** and **production-ready reproducibility** I found the sweetspot for a data analysis workflow that costs absolutely nothing in terms of time and disk space.

The beauty of this approach is its simplicity and efficiency:

- No data duplication: Worktrees share annexed files, so you don't need to `datalad get` everything again
- Parallel workflows: Develop in main worktree with Jujutsu or your favourite VCS, process in clean worktree with DataLad
- Seamless integration: Merge results back with a simple `git merge`
- Low overhead: Environment setup happens once, then persists for all runs (unless you are enthusiastic time-traveler like me :P)

This isn't a replacement for full reproducibility frameworks - for final publication runs, I still reach for containers and fully specified environments, and adapt something like [the fairly big processing workflow](https://github.com/psychoinformatics-de/fairly-big-processing-workflow). But for the 90% of my work that happens between initial exploration and final publication, this workflow hits the perfect balance.

Here is the destilled recipe for the workflow:
1. Create a nested git worktree for your nested dataset on a new branch, here 'runs'
2. Set up the computational environment for this worktree
3. Craft and test the `datalad run` command invocation
4. Write a script that executes the datalad run command over multiple instances, here over subjects and experiments
5. Make the update available on the branch/worktree 'runs'
6. Run the script in the worktree 'runs'
7. Merge the results into the main worktree
8. Repeat 3-7 for the next iteration

Happy analyzing! 🧠
