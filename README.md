# DRAGON

This is the companion repository for the paper [*DRAGON: Predicting Decompiled Variable Data Types with Learned Confidence Estimates*](https://www.ndss-symposium.org/ndss-paper/auto-draft-629/).

# Data Artifacts
Several artifacts are available for download [here](https://drive.google.com/drive/folders/1ccE8IJiHOLn0l9_GG3hysSGxCC2tF_eo?usp=drive_link).
The folders include:

- `binaries` - the set of debug binaries used in the paper
- `datasets` - prebuilt PyG datasets used in the paper
- `dragon_evals` - dragon eval output data from which paper results were computed
- `training_artifacts` - training scripts, parameters, and outputs for the models used in the paper

# Run DRAGON on prebuilt datasets

1. Download pre-built datasets for the paper from [here](https://drive.google.com/drive/folders/1ccE8IJiHOLn0l9_GG3hysSGxCC2tF_eo?usp=drive_link)

    - Download and extract one or more desired datasets from the `datasets` folder and place the dataset folders underneath `dragon/datasets`.

    > The scripts expect individual dataset folders to exist directly inside `dragon/datasets`. These folders are the ones which include two subfolders, `processed` and `raw`.

2. Create a Python virtual environment and activate it (tested with Python 3.8.10, although newer versions should work also):

    `python -m venv <dragon_env> && <dragon_env>/bin/activate`

3. Install python dependencies from the activated virtual environment:

    `pip install -r pip_freeze_2025.01.01.txt`

    > You may need to install Pytorch separately if the command above doesn't install it properly.
    If so, see my notes [here](https://github.com/lasserre/datatype-recovery-experiments?tab=readme-ov-file#install-pytorch).

4. Run this script from the top-level folder to evaluate the DRAGON models from the paper on the benchmark datasets:

    `./scripts/run_dragon_benchmarks.py`

**These steps are all that is required to run DRAGON on the prebuilt datasets used in the paper.**
The remaining notes below are included for reference only if you need to create new datasets from debug binaries.

# (Optional) Creating Datasets
To build a PyG dataset from a set of debug binaries, the process is:

1. Process the debug binaries using an `import-dataset` [wildebeest](https://github.com/lasserre/wildebeest) experiment. This performs the following steps:
    - Create a debug & stripped pair for each binary (copy the debug binary and strip all debug info)
    - Import all binaries into a Ghidra repository
    - Decompile and export function ASTs for all binaries
2. Build a PyG dataset from the extracted AST data. The debug binary information is used
to label the dataset as described in the paper.

## Modified Ghidra Server
DRAGON exports function ASTs using a modified version of Ghidra, available here: https://github.com/lasserre/ghidra.

The Ghidra setup/build steps I used are summarized in the readme here: https://github.com/lasserre/datatype-recovery-experiments?tab=readme-ov-file#setup.
For more complete instructions, see the Ghidra readme for more details on building and running the Ghidra server.

The default DRAGON parameters try to connect to a Ghidra server at `localhost`, but this can be changed if desired.

## Included AST Export Scripts
The scripts used to build the datasets used in the paper are included, and can serve as
a reference for building new datasets if desired.

To run the scripts:

1. Ensure the binaries exist at the expected location (default is in `~/dragon_data/`). They can be downloaded from the link mentioned at the beginning.
2. Check the other parameters at the top of each script to make sure it works with your setup
(e.g. set `NJOBS` to an appropriate number for your machine).
3. Then run the scripts below to import binaries into Ghidra from scratch and export AST data to JSON files

### Benchmarks

`./scripts/export_benchmark_asts.sh`

When this completes, the PyG datasets can be built by running:

`./scripts/build_benchmark_datasets.py`

### TYDA-min training dataset (our sample of TYDA-min)

`./scripts/export_tydamin_asts.sh`

When this completes, the PyG dataset can be built by running:

`./scripts/build_tydamin_dataset.sh`

### ReSym training dataset

`./scripts/export_resym_train_asts.sh`

When this completes, the PyG dataset can be built by running:

`./scripts/build_resym_train_dataset.sh`

#### ReSym failure cases
In this version of Ghidra, these binaries fail to import successfully when I imported them:
~~~
02ab506
9f89779
2cefb8a
fc2d878
cea8f48
674ee41     # this one doesn't fail, but seems to hang forever, so is excluded
~~~

Here is what my final status looked like once I successfully reran a couple of
jobs that had hung:

~~~
$ wdb status | grep -v finished | grep Run
Run 1271 (-1271.02ab506) - FAILED during "export_asts_strip" [0:03:17]
Run 1577 (-1577.9f89779) - FAILED during "export_asts_strip" [0:03:33]
Run 2941 (-2941.2cefb8a) - FAILED during "export_asts_strip" [0:03:01]
Run 3725 (-3725.674ee41) running "ghidra_import_debug" [0:27:45] Total: [0:27:45]
Run 3945 (-3945.fc2d878) - FAILED during "export_asts_strip" [0:03:10]
Run 4845 (-4845.cea8f48) - FAILED during "export_asts_strip" [0:02:23]
~~~

## Miscellaneous tips for working with wildebeest runs
Here are a few examples of wildebeest (`wdb`) status commands you can run to
control or check the status of your experiments as they run multiple jobs in parallel.

A couple of high-level notes:
- Wildebeest works with experiments, which are by default folders ending in `.exp`
- Most wildebeest commands expect to be run within the experiment folder
- You can run `wdb -h` for more help on the available options.

### Fixing any jobs that hang
If running with many parallel jobs, sometimes a single job will get hung up and
make it appear nothing is progressing. In this case, you may need to kill it and rerun any
jobs that didn't finish.

### Check status from within a wdb experiment folder (e.g. `dragon/exps/tydamin_sample.exp`) with:

`wdb status`

or more usefully:

`wdb status | grep -v finished`

### Restart a failed run `N` with:

`wdb run --from <failed_step_name> -f N` # single run

`wdb run --from <failed_step_name> -f X,Y,Z` # multiple runs, where X, Y, and Z are run numbers (e.g. 47,48,62)

`wdb run --from <failed_step_name> -f N --debug` # useful to run a single job in-process and shows output in terminal instead of logfile

### Show logfile for run N

`wdb log N`
