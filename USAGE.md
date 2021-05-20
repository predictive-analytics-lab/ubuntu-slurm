# Usage

> How to use SLURM.

- [Submitting jobs](#submitting-jobs)
  - [Using a job script](#using-a-job-script)
  - [`sbatch` configuration](#sbatch-configuration)
  - [Starting template](#starting-template)
    - [Full GPU](#full-gpu)
    - [1/3 of a GPU](#13-of-a-gpu)
    - [Running `ray` inside of a SLURM job](#running-ray-inside-of-a-slurm-job)
  - [Using a library to submit jobs](#using-a-library-to-submit-jobs)
  - [Specifying fractional GPUs](#specifying-fractional-gpus)
  - [Submitting multiple jobs in a bash loop](#submitting-multiple-jobs-in-a-bash-loop)
- [Monitoring SLURM](#monitoring-slurm)
  - [Monitoring the queue](#monitoring-the-queue)
  - [Displaying more information about a specific job](#displaying-more-information-about-a-specific-job)
- [Canceling a job](#canceling-a-job)


## Submitting jobs

The basic action when using SLURM is submitting jobs so that they get added to the job queue and eventually run. Broadly, there are two ways to do this: either write a job script or use some library that submits the jobs for you.

### Using a job script

The simplest job script looks something like this:

```sh
#!/bin/bash
#SBATCH --job-name=my_job

source ./my_env/bin/activate
python main.py --lr 1e-3 --layers 4
```

This needs to be put in the same directory as `main.py` in this example. The filename of the job file doesn’t really matter, but traditionally the file ending `.job` is used. So, let’s say we save the above job script as `python_main.job`. We can then submit the job with

```sh
sbatch python_main.job
```

`sbatch` is always the command to submit jobs from job files. `sbatch` takes many (optional) arguments. One of them is `--job-name`, as we saw above. Arguments for `sbatch` can either be specified on the commandline or in the job file on a line that starts with `#SBATCH`. So, for example, `--output` is used to specify a file where the stdout of the job is saved (i.e., the logging). We can either add this to the job file:

```sh
#!/bin/bash
#SBATCH --job-name=my_job
#SBATCH --output=./myjob.out

source ./my_env/bin/activate
python main.py --lr 1e-3 --layers 4
```

Or, we specify it on the commandline:

```sh
sbatch --job-name=my_job --output=./myjob.out python_main.job
```

If the arguments from the job file and the commandline conflict, then the ones from the commandline are used. The following explains more of the arguments.

### `sbatch` configuration

There are over 70 arguments for `sbatch` that can’t all be listed here. This is a selection of the most relevant ones. You can find the full list here: https://slurm.schedmd.com/sbatch.html

- `--begin=[MM/DD-]HH:MM`: begin the job at the specified time (or later, if resources aren’t free)
- `--chdir=<directory>`: set the working directory for the script (by default it’s the directory where `sbatch` was run)
- `--cpus-per-task=<ncpus>`: how many CPUs to allocate for the job
- `--deadline=[MM/DD-]HH:MM`: if the job hasn’t been started by the specified date, don’t start it at all
- `--dependency=afterok:<job_id>`: start the job only after the other job with ID `<job_id>` has finished successfully. For other kinds of dependency, see the documentation.
- `--export=ALL` or `--export=NONE`: specify whether environment variables from the current context are exported to the job script (default is `ALL`)
- `--get-user-env`: set the enviroment variables the same as the submitting user’s login environment
- `--gpu-bind=<binding>`: bind the job to a specific GPU. For example, this should restrict your jobs to run on GPUs 0, 3 and 4: `--gpu-bind=map_gpu:0,3,4` (note the additional `map_gpu:` string). For more details see the documentation.
- `--gpus-per-task=<ngpus>`: how many GPUs to allocate for the job. This is less fine grained than `--gres` below
- `--gres=<list of resources>`:
  
  `gres` is for specifying any required resource other than CPUs and main memory (RAM). In particular, this includes GPUs. To request a GPU, do `--gres=gpu:1`. If you know the type of GPU, you can be even more specific: `--gres=gpu:rtx_3090:1`. This can also be used to request a fraction of a GPU: `--gres=mps:6`; for more details on that, [see below](#specifying-fractional-gpus).
- `--job-name=<name>`: name of the job (this name shows up in the queue)
- `--mem=<size[units]>`: specify the required memory (RAM) required
- `--output=<filename pattern>`: file where the standard output (and by default also standard error) is saved. You can use `%j` in the file name and it will be replaced by the job ID, and `%x` will be replaced by the job name. So, for example `--output=./logs/%x-%j.out`
- `--partition=<partition name>`: specify the partition for the job. Partitions are kind of like different queues in SLURM. The partition affects where the job may run, what the time limit is, and what the default values for the resources are. In our case, there is a default partition called `all`, and each node has its own additional partition: `--partition=goedel`.
- `--time=[D-][H:]M`: (`D`: days, `H`: hours, `M`: minutes, `[]` marks optional parts) set a run time limit for the job. If the job is still running by the time the limit runs out, the job is killed.

All these arguments can also be set with environment variables. See [the documentation](https://slurm.schedmd.com/sbatch.html) for details on that.

### Starting template

#### Full GPU

```sh
#!/bin/bash
# --- slurm settings ---
#SBATCH --partition=goedel
#SBATCH --gpus=1
#SBATCH --job-name=example
#SBATCH --output=./myjob-%j.out
# ----------------------

# set up conda
eval "$(conda shell.bash hook)"

conda activate my_env
python -u experiment.py --some flag
```

#### 1/3 of a GPU

In this case we have to specify the number of CPUs and the amount of RAM manually.

```sh
#!/bin/bash
# --- slurm settings ---
#SBATCH --partition=goedel
# 8 GB of GPU memory:
#SBATCH --gres=mps:8
# 10 GB of (CPU) memory:
#SBATCH --mem=10G
#SBATCH --cpus-per-task=1
#SBATCH --job-name=example
#SBATCH --output=./logs/myjob-%j.out
# ----------------------

# set up conda
eval "$(conda shell.bash hook)"

conda activate my_env
python -u experiment.py --some flag
```

#### Running `ray` inside of a SLURM job

```sh
#!/bin/bash
# --- slurm settings ---
#SBATCH --partition=goedel
#SBATCH --ntasks=1
#SBATCH --gpus-per-task=3
#SBATCH --cpus-per-task=9
#SBATCH --mem=90G
#SBATCH --job-name=ray-wrapper
#SBATCH --output=./rayjob-%j.out
# ----------------------

ray start --head --num-cpus "${SLURM_CPUS_PER_TASK}" --num-gpus "${SLURM_GPUS_PER_TASK}"

# we just run here whatever is passed as arguments to this job script
python -u "$@"
```

Usage (if the above is saved to a file called `ray.job`):
```sh
sbatch ray.job experiment.py -m misc.seed=range(0,10)  hydra/launcher=ray
```

### Using a library to submit jobs

Instead of writing job files, you can use libraries to submit jobs for you. One such library is [submitit](https://github.com/facebookincubator/submitit) by Facebook. See [their documentation](https://github.com/facebookincubator/submitit/blob/master/docs/examples.md) to learn more.

### Specifying fractional GPUs

Nvidia has written a plugin for SLURM that allows specifying fractional (nvidia) GPUs as resources.
However, the user experience is not the sleekest.
First, the name: it's called [CUDA Multi-Process Service (MPS)](https://slurm.schedmd.com/gres.html#MPS_Management).
Nothing in the name hints at fractional GPUs, but that's what they chose.
In fact, they were seemingly so proud of the name, that the resource is also called `mps`.
In order to request a fractional GPU, you specify something like `--gres=mps:4`.
But what does the number ("4" in this case) mean?
Well, it has no fixed meaning.
When configuring one's SLURM installation, one can assign "MPS points" to each of the GPUs.
It's completely arbitrary how many points are assigned to a given GPU.

One way to do it (that's also mentioned in the documentation) is to assign 100 MPS points to each GPU.
In this case, you can treat the points as percentages and do `--gres=mps:33` to request 33% of a GPU.
However, that only really makes sense if all your GPUs are the same.
If you have different GPUs in your cluster, you might not want to give them all the same number of MPS points.

So, the solution I came up with is to assign an MPS point for every GB of GRAM.
So, if a GPU has 12GB of memory, it gets 12 MPS points and so on.
Then, you can request `--gres=mps:6` and know that you will always get 6GB of GPU memory, no matter on which GPU your job ends up running.
However, as MPS points are integers, under this setting it is not possible to request, for example, 6.5GB of memory.

### Submitting multiple jobs in a bash loop

When you do
```sh
sbatch --gpus=1 --time=1:00 script.job arg another-arg yet-another-arg
```
then everything after `script.job` gets passed into the job script as commandline arguments.
You can make use of that in order to submit several jobs in a queue.
You would have a _submitter_ bash script that looks like this:
```bash
#!/bin/bash
for seed in 3 4 5 6 7
do
    sbatch celeba.job data_split_seed=$seed
done
```
and a job file that looks something like this:
```sh
#!/bin/bash
#SBATCH --partition=goedel
#SBATCH --gpus=1
#SBATCH --job-name=george-celeba
#SBATCH --output=./george-celeba-%j.out
python -u run.py configs/celeba_george_config.json "$@"
```
You can see there the `"$@"` which passes on all arguments that the job script got to the python script.

## Monitoring SLURM
### Monitoring the queue

After you have submitted a job, you can check the job queue with
```
squeue
```
The output looks something like this:

```
JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
   16      PAL1    myjob    mb715 PD       0:00      1 (Resources)
   15      PAL1    myjob    tk324  R       1:05      1 goedel
```

This means job #15 is currently running on node `goedel` and has been for 1 minutes and 5 seconds. Job #16 is still in the queue, because the required resources aren’t free yet.

SLURM has a lot of job state codes. For example, `R` means “running”, and `PD` means “pending”. Here are some other job state codes:

- `CA`: canceled
- `CD`: completed
- `CG`: completing
- `F`: failed
- `OOM`: out of memory
- `PD`: pending
- `R`: running

For the full list, see [the documentation for squeue](https://slurm.schedmd.com/squeue.html).

Often you want to continuously monitor the queue. You can use

```
watch squeue
```

for that. It refreshes every 2 seconds. You can exit it with ctrl-C.

### Displaying more information about a specific job

```
scontrol show job <job id>
```

## Canceling a job

```
scancel <job id>
```
