# CAMP Hybrid Assembly


<!-- [![Documentation Status](https://img.shields.io/readthedocs/camp_hybrid-assembly)](https://camp-documentation.readthedocs.io/en/latest/hybrid_assembly.html) -->

![Version](https://img.shields.io/badge/version-0.1.0-brightgreen)

## Overview

This module is designed to function as both a standalone MAG hybrid assembly pipeline as well as a component of the larger CAMP metagenomics analysis pipeline. As such, it is both self-contained (ex. instructions included for the setup of a versioned environment, etc.), and seamlessly compatible with other CAMP modules (ex. ingests and spawns standardized input/output config files, etc.). 

The CAMP hybrid assembly module allows users to select from one or both of two hybrid short- and long-read _de novo_ assembly strategies: 1) short-read-first with hybridmetaSPAdes and 2) long-read-first with MetaFlye following by short- and long-read assembly polishing. The assemblies created from the specific strategies are QC-ed afterwards with QUAST.  

## Installation

> [!TIP]
> All databases used in CAMP modules will also be available for download on Zenodo (link TBD).

### Install `conda`

If you don't already have `conda` handy, we recommend installing `miniforge`, which is a minimal conda installer that, by default, installs packages from open-source community-driven channels such as `conda-forge`.
```Bash
# If you don't already have conda on your system...
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
bash Miniforge3-Linux-x86_64.sh # --> Template <--
```

Run the following command to initialize Conda for your shell. This will configure your shell to recognize conda activate. 
```Bash
conda init
```

Restart your terminal or run:
```Bash
source ~/.bashrc  # For bash users
source ~/.zshrc   # For zsh users
```
### Setting up the Hybrid Assembly Module

1. Clone repo from [Github](<https://github.com/Meta-CAMP/camp_hybrid-assembly). 
```Bash
git clone https://github.com/Meta-CAMP/camp_hybrid-assembly
```

2. Set up the rest of the module interactively by running `setup.sh`. This step downloads databases and installs the other conda environments needed for running the module. This is done interactively by running `setup.sh`. `setup.sh` also generates `parameters.yaml` based on user input paths for running this module.
```Bash
cd camp_hybrid-assembly/
source setup.sh

# If you encounter issues where conda activate is not recognized, follow these steps to properly initialize Conda
conda init
source ~/.bashrc # or source ~/.zshrc
```

3. Make sure the installed pipeline works correctly. With 10 threads and a maximum of 40 GB allocated, the dataset should finish in under 11 minutes.
--->
```Bash
# Run tests on the included sample dataset
conda activate hybrid_assembly # --> Template <--
python /path/to/camp_hybrid-assembly/workflow/hybrid-assembly.py test 
```

## Using the Module

**Input**: `/path/to/samples.csv` provided by the user.

**Output**: 1) An output config file summarizing 2) the module's outputs. 

- `/path/to/work/dir/hybrid_assembly/final_reports/samples.csv` for ingestion by the next module (MAG binning)
- `/path/to/work/dir/hybrid_assembly/final_reports/quast.tar.gz` for the collated QUAST reports
- `/path/to/work/dir/hybrid_assembly/final_reports/ctg_lens.csv` for a list of the lengths of all contigs at all steps
- `/path/to/work/dir/hybrid_assembly/final_reports/ctg_stats.csv` for each assembly's summary statistics

### Module Structure
```
└── workflow
    ├── Snakefile
    ├── hybrid_assembly.py
    ├── utils.py
    ├── __init__.py
    └── ext/
        └── scripts/
```
- `workflow/hybrid_assembly.py`: Click-based CLI that wraps the `snakemake` and other commands for clean management of parameters, resources, and environment variables.
- `workflow/Snakefile`: The `snakemake` pipeline. 
- `workflow/utils.py`: Sample ingestion and work directory setup functions, and other utility functions used in the pipeline and the CLI.
- `ext/`: External programs, scripts, and small auxiliary files that are not conda-compatible but used in the workflow.

### Running the Workflow

1. Make your own `samples.csv` based on the template in `configs/samples.csv`. Sample test data can be found in `test_data/`. 
    - For example, `ingest_samples` in `workflow/utils.py` expects Illumina reads in FastQ (may be gzipped) form and de novo assembled contigs in FastA form
    - `samples.csv` requires either absolute paths or paths relative to the directory that the module is being run in.

2. (Optional) Update the relevant parameters in `configs/parameters.yaml`.

3. (Optional) Update the computational resources available to the pipeline in `configs/resources.yaml`. 

#### Command Line Deployment

To run CAMP on the command line, use the following, where `/path/to/work/dir` is replaced with the absolute path of your chosen working directory, and `/path/to/samples.csv` is replaced with your copy of `samples.csv`. 
    - The default number of cores available to Snakemake is 1 which is enough for test data, but should probably be adjusted to 10+ for a real dataset.
    - Relative or absolute paths to the Snakefile and/or the working directory (if you're running elsewhere) are accepted!
    - The parameters and resource config YAMLs can also be customized.
```Bash
python /path/to/camp_hybrid-assembly/workflow/hybrid_assembly.py \
    (-c number_of_cores_allocated) \
    (-p /path/to/parameters.yaml) \
    (-r /path/to/resources.yaml) \
    -d /path/to/work/dir \
    -s /path/to/samples.csv
```

#### Slurm Cluster Deployment

To run CAMP on a job submission cluster (for now, only Slurm is supported), use the following.
    - `--slurm` is an optional flag that submits all rules in the Snakemake pipeline as `sbatch` jobs. 
    - In Slurm mode, the `-c` flag refers to the maximum number of `sbatch` jobs submitted in parallel, **not** the pool of cores available to run the jobs. Each job will request the number of cores specified by threads in `configs/resources/slurm.yaml`.
```Bash
sbatch -J jobname -o jobname.log << "EOF"
#!/bin/bash
python /path/to/camp_hybrid-assembly/workflow/hybrid_assembly.py --slurm \
    (-c max_number_of_parallel_jobs_submitted) \
    (-p /path/to/parameters.yaml) \
    (-r /path/to/resources.yaml) \
    -d /path/to/work/dir \
    -s /path/to/samples.csv
EOF
```

#### Finishing Up

1. To quality-check the hybrid assemblies, download and compare the collated QUAST reports, which can be found at `/path/to/work/dir/hybrid_assembly/final_reports/quast.tar.gz`. 
    - You may have to manually rerun Medaka and Polypolish, as multiple rounds of assembly polishing may be needed to improve the contiguity and minimize misassemblies. 

2. To easily visualize assembly summary metrics such as the number of contigs, total assembly size, and average contig size across samples, follow the instructions in the Jupyter notebook:
```Bash
jupyter notebook &
```

3. After checking over `final_reports/` and making sure you have everything you need, you can delete all intermediate files to save space. 
```Bash
python3 /path/to/camp_hybrid-assembly/workflow/hybrid_assembly.py cleanup \
    -d /path/to/work/dir \
    -s /path/to/samples.csv
```

4. If for some reason the module keeps failing to finish, CAMP can print a script containing all of the remaining commands that can be run manually. 
```Bash
python3 /path/to/camp_hybrid-assembly/workflow/hybrid_assembly.py --dry_run \
    -d /path/to/work/dir \
    -s /path/to/samples.csv
```

## Credits

- This package was created with [Cookiecutter](https://github.com/cookiecutter/cookiecutter>) as a simplified version of the [project template](https://github.com/audreyr/cookiecutter-pypackage>).
 
- Free software: MIT License
- Documentation: https://camp-documentation.readthedocs.io/en/latest/hybrid_assembly.html



