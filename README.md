# Slurm Cheat Sheet

## Show the GPU partition queue for an account
```bash
squeue \
  -A test \
  --partition gpu \
  -o "%.18i %.9P %.40j %.8u %.8a %.8T %.6D %.4C %.5D %.6m %.13b %.10M %.10l %.28R"
```

## Account billing

```bash
sreport \
  -T billing \
  cluster AccountUtilizationByUser \
    start=mm/dd/yy \
    end=mm/dd/yy \
    account=test \
| sed -r 's/(.*billing\s+)([0-9]+)\b(.*)/echo "\1 \\$$(echo scale=2\\; \2\/100000 \| bc)\3"/ge'
```

## Account GPU-hours utilization

```bash
sreport \
  -t hours \
  -T gres/gpu \
  cluster AccountUtilizationByUser \
    start=mm/dd/yy \
    end=mm/dd/yy \
    account=test
```

## Allocate bash with GPU

```bash
srun \
    --account test \
    --partition gpu \
    --cpus-per-task 20 \
    --mem 32GB \
    --gpus 1 \
    --time 5:00:00 \
    --pty bash
```

## Shell script to run a job

```
    #!/bin/bash

    # The interpreter used to execute the script

    #“#SBATCH” directives that convey submission options:

    #SBATCH --job-name=name
    #SBATCH --mail-user=email
    #SBATCH --mail-type=BEGIN,END
    #SBATCH --cpus-per-task=32
    #SBATCH --nodes=1
    #SBATCH --ntasks-per-node=1
    #SBATCH --mem-per-cpu=10000m 
    #SBATCH --time=24:00
    #SBATCH --account=test
    #SBATCH --partition=gpu
    #SBATCH --output=/home/%u/%x-%j.log

    # The application(s) to execute along with its input arguments and options: 
```