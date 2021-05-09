# Usage

> How to use SLURM.

- [Submitting jobs](#submitting-jobs)
  - [Using a job script](#using-a-job-script)
  - [`sbatch` configuration](#sbatch-configuration)
  - [Using a library to submit jobs](#using-a-library-to-submit-jobs)
- [Monitoring the queue](#monitoring-the-queue)
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
- `--gpu-bind`: bind the job to a specific GPU (see documentation for details)
- `--gres=<list of resources>`:
  
  `gres` is for specifying any required resource other than CPUs and main memory (RAM). In particular, this includes GPUs. To request a GPU, do `--gres=gpu:1`. If you know the type of GPU, you can be even more specific: `--gres=gpu:nvidia_geforce_rtx_3090:1`. This can also be used to request a fraction of a GPU: `--gres=mps:3`; for more details on that, see below.
- `--job-name=<name>`: name of the job (this name shows up in the queue)
- `--nodelist=<node name list>`: a comma-separated list of hosts that are acceptable for your job; for example: `--nodelist=goedel`
- `--mem=<size[units]>`: specify the required memory (RAM) required
- `--output=<filename pattern>`: file where the standard output (and by default also standard error) is saved. You can use `%j` in the file name and it will be replaced by the job ID, and `%x` will be replaced by the job name. So, for example `--output=./logs/%x-%j.out`
- `--time=[D-][H:]M`: (`D`: days, `H`: hours, `M`: minutes, `[]` marks optional parts) set a run time limit for the job. If the job is still running by the time the limit runs out, the job is killed.

All these arguments can also be set with environment variables. See [the documentation](https://slurm.schedmd.com/sbatch.html) for details on that.

### Using a library to submit jobs

Instead of writing job files, you can use libraries to submit jobs for you. One such library is [submitit](https://github.com/facebookincubator/submitit) by Facebook. See [their documentation](https://github.com/facebookincubator/submitit/blob/master/docs/examples.md) to learn more.

## Monitoring the queue

After you have submitted a job, you can check the job queue with `squeue`. The output looks something like this:

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

## Canceling a job

```
scancel <job id>
```
