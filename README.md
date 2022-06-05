# Slurm Cheat Sheet

*Adapted from Santiago's Slurm Cheatsheet: https://github.com/bryant1410/slurm-cheatsheet*

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

## Check the overall GPU utilization
```
join \
  -a 1 \
  -o auto \
  <(sinfo \
    --partition gpu \
    --Node \
    --noheader \
    -O NodeList:7,CPUsState:12,AllocMem:7,Memory:7,Gres:11,StateCompact:6,GresUsed:.10
  ) \
  <(squeue \
    -t RUNNING \
    --partition gpu \
    --noheader \
    -o "%N %u %a" \
  | python -c "
import fileinput
import subprocess
from collections import defaultdict

# From https://github.com/NERSC/pytokio/blob/fdb8237/tokio/connectors/slurm.py#L81
def node_spec_to_list(node_spec):
  return subprocess.check_output(['scontrol', 'show', 'hostname', node_spec]).decode().strip().split()

users_by_node = defaultdict(list)

for line in fileinput.input():
  line = line.strip()

  node_spec, u, a = line.split()
  
  for node in node_spec_to_list(node_spec):
    users_by_node[node].append('{}({})'.format(u, a))

for node, node_info in sorted(users_by_node.items(), key=lambda t: t[0]):
  print('{} {}'.format(node, ','.join(node_info)))
  ") \
| awk \
  'BEGIN{
    total_cpus_alloc = 0;
    total_cpus = 0;
    total_mem_alloc = 0;
    total_mem = 0;
    total_gpus_alloc = 0;
    total_gpus = 0;

    printf("%6s %5s %10s %9s %10s %s\n", "NODE", "STATE", "ALLOC_CPUS", "ALLOC_MEM", "ALLOC_GPUS", "USERS")
  };
  {
    split($2, cpu, "/");
    split($5, gres, ":");
    split($7, gres_used, ":");

    node = $1;

    state = $6;

    cpus_alloc = cpu[1];
    total_cpus_alloc += cpus_alloc;

    cpus = cpu[4];
    total_cpus += cpus;

    mem_alloc = $3 / 1024;
    total_mem_alloc += mem_alloc;

    mem = $4 / 1024;
    total_mem += mem;

    gpus_alloc = gres_used[3];
    total_gpus_alloc += gpus_alloc;

    gpus = gres[3];
    total_gpus += gpus;

    users = $8;

    printf("%6s %5s %6d/%3d %4d/%4d %7d/%2d %s\n", node, state, cpus_alloc, cpus, mem_alloc, mem, gpus_alloc, gpus, users)
  };
  END{
    printf("%6s %5s %6d/%3d %4d/%4d %7d/%2d\n", "TOTAL", "", total_cpus_alloc, total_cpus, total_mem_alloc, total_mem, total_gpus_alloc, total_gpus)
  }'
pending_jobs=$(squeue --partition gpu -t PENDING --noheader)
if [ ! -z "$pending_jobs" ]; then
  echo
  echo Pending jobs:
  echo "$pending_jobs"
fi
```

## Check overall spgpu utilization
```
join \
  -a 1 \
  -o auto \
  <(sinfo \
    --partition spgpu \
    --Node \
    --noheader \
    -O NodeList:7,CPUsState:12,AllocMem:7,Memory:7,Gres:11,StateCompact:6,GresUsed:.10
  ) \
  <(squeue \
    -t RUNNING \
    --partition spgpu \
    --noheader \
    -o "%N %u %a" \
  | python -c "
import fileinput
import subprocess
from collections import defaultdict

# From https://github.com/NERSC/pytokio/blob/fdb8237/tokio/connectors/slurm.py#L81
def node_spec_to_list(node_spec):
  return subprocess.check_output(['scontrol', 'show', 'hostname', node_spec]).decode().strip().split()

users_by_node = defaultdict(list)

for line in fileinput.input():
  line = line.strip()

  node_spec, u, a = line.split()
  
  for node in node_spec_to_list(node_spec):
    users_by_node[node].append('{}({})'.format(u, a))

for node, node_info in sorted(users_by_node.items(), key=lambda t: t[0]):
  print('{} {}'.format(node, ','.join(node_info)))
  ") \
| awk \
  'BEGIN{
    total_cpus_alloc = 0;
    total_cpus = 0;
    total_mem_alloc = 0;
    total_mem = 0;
    total_gpus_alloc = 0;
    total_gpus = 0;

    printf("%6s %5s %10s %9s %10s %s\n", "NODE", "STATE", "ALLOC_CPUS", "ALLOC_MEM", "ALLOC_GPUS", "USERS")
  };
  {
    split($2, cpu, "/");
    split($5, gres, ":");
    split($7, gres_used, ":");

    node = $1;

    state = $6;

    cpus_alloc = cpu[1];
    total_cpus_alloc += cpus_alloc;

    cpus = cpu[4];
    total_cpus += cpus;

    mem_alloc = $3 / 1024;
    total_mem_alloc += mem_alloc;

    mem = $4 / 1024;
    total_mem += mem;

    gpus_alloc = gres_used[3];
    total_gpus_alloc += gpus_alloc;

    gpus = gres[3];
    total_gpus += gpus;

    users = $8;

    printf("%6s %5s %6d/%3d %4d/%4d %7d/%2d %s\n", node, state, cpus_alloc, cpus, mem_alloc, mem, gpus_alloc, gpus, users)
  };
  END{
    printf("%6s %5s %6d/%3d %4d/%4d %7d/%2d\n", "TOTAL", "", total_cpus_alloc, total_cpus, total_mem_alloc, total_mem, total_gpus_alloc, total_gpus)
  }'
pending_jobs=$(squeue --partition spgpu -t PENDING --noheader)
if [ ! -z "$pending_jobs" ]; then
  echo
  echo Pending jobs:
  echo "$pending_jobs"
fi
```

## Check overall cpu utilization
```
join \
  -a 1 \
  -o auto \
  <(sinfo \
    --partition standard \
    --Node \
    --noheader \
    -O NodeList:7,CPUsState:12,AllocMem:7,Memory:7,Gres:11,StateCompact:6,GresUsed:.10
  ) \
  <(squeue \
    -t RUNNING \
    --partition standard \
    --noheader \
    -o "%N %u %a" \
  | python -c "
import fileinput
import subprocess
from collections import defaultdict

# From https://github.com/NERSC/pytokio/blob/fdb8237/tokio/connectors/slurm.py#L81
def node_spec_to_list(node_spec):
  return subprocess.check_output(['scontrol', 'show', 'hostname', node_spec]).decode().strip().split()

users_by_node = defaultdict(list)

for line in fileinput.input():
  line = line.strip()

  node_spec, u, a = line.split()
  
  for node in node_spec_to_list(node_spec):
    users_by_node[node].append('{}({})'.format(u, a))

for node, node_info in sorted(users_by_node.items(), key=lambda t: t[0]):
  print('{} {}'.format(node, ','.join(node_info)))
  ") \
| awk \
  'BEGIN{
    total_cpus_alloc = 0;
    total_cpus = 0;
    total_mem_alloc = 0;
    total_mem = 0;
    total_gpus_alloc = 0;
    total_gpus = 0;

    printf("%6s %5s %10s %9s %10s %s\n", "NODE", "STATE", "ALLOC_CPUS", "ALLOC_MEM", "ALLOC_GPUS", "USERS")
  };
  {
    split($2, cpu, "/");
    split($5, gres, ":");
    split($7, gres_used, ":");

    node = $1;

    state = $6;

    cpus_alloc = cpu[1];
    total_cpus_alloc += cpus_alloc;

    cpus = cpu[4];
    total_cpus += cpus;

    mem_alloc = $3 / 1024;
    total_mem_alloc += mem_alloc;

    mem = $4 / 1024;
    total_mem += mem;

    gpus_alloc = gres_used[3];
    total_gpus_alloc += gpus_alloc;

    gpus = gres[3];
    total_gpus += gpus;

    users = $8;

    printf("%6s %5s %6d/%3d %4d/%4d %7d/%2d %s\n", node, state, cpus_alloc, cpus, mem_alloc, mem, gpus_alloc, gpus, users)
  };
  END{
    printf("%6s %5s %6d/%3d %4d/%4d %7d/%2d\n", "TOTAL", "", total_cpus_alloc, total_cpus, total_mem_alloc, total_mem, total_gpus_alloc, total_gpus)
  }'
pending_jobs=$(squeue --partition spgpu -t PENDING --noheader)
if [ ! -z "$pending_jobs" ]; then
  echo
  echo Pending jobs:
  echo "$pending_jobs"
fi
```
