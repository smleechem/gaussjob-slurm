# gaussjob-slurm

Submit **Gaussian 16** jobs to **SLURM** with per-job scratch directories and automatic cleanup.

* Single command: `gaussjob NODE [INPUT.(com|gjf)] [CPUS] [TIME]`
* Per-job scratch at `$SCR_BASE/$SLURM_JOB_ID` via `GAUSS_SCRDIR`
* Robust cleanup on normal exit and common signals
* Uses OpenMP threads from `SLURM_CPUS_PER_TASK`
* Preserves the generated batch script for traceability

> **Note**
> This project does **not** include Gaussian. You must have a valid Gaussian 16 installation and license.

---

## Repository layout

```
gaussjob-slurm/
├─ bin/
│  └─ gaussjob            # the script (chmod +x)
├─ examples/
│  └─ opt.com       # small placeholder Gaussian input
├─ LICENSE
└─ README.md
```

---

## Why this script?

Running Gaussian on SLURM typically requires the same boilerplate: safe scratch handling, consistent thread counts, logs, and tidy cleanup. **`gaussjob`** wraps that into a single, auditable command your group can standardize on.

---

## Requirements

* SLURM (`sbatch` available in `PATH`)
* Gaussian 16 installed on compute nodes
* POSIX shell environment
* A shared or node-local filesystem for scratch (configurable)

---

## Installation

```bash
# Inside the cloned repo
chmod +x bin/gaussjob

# Option 1: add to PATH
echo 'export PATH="$HOME/src/gaussjob-slurm/bin:$PATH"' >> ~/.bashrc

# Option 2: install system-wide (admin)
sudo install -m 0755 bin/gaussjob /usr/local/bin/gaussjob
```

---

## Quick start

From a directory containing `input.com`:

```bash
gaussjob n1
```

Defaults used:

* `NODE = n1`
* `INPUT = input.com`
* `CPUS = 10`
* `TIME  = 168:00:00` (7 days)

Outputs:

* SLURM stdout: `g16.<JOBID>.out`
* Gaussian log: `input.log`

---

## Usage

```
gaussjob NODE [INPUT.(com|gjf)] [CPUS] [TIME]
```

**Arguments**

* `NODE`  (required): SLURM node name or list, e.g., `n1` or `n1,n2`
  (passed to `#SBATCH --nodelist=...`)
* `INPUT` (optional): Gaussian input file. Default: `input.com`
* `CPUS`  (optional): OpenMP threads / SLURM `--cpus-per-task`. Default: `10`
* `TIME`  (optional): SLURM walltime. Default: `168:00:00` (hh:mm:ss)

**Examples**

```bash
gaussjob n1                              # uses ./input.com, 10 CPUs, 168:00:00
gaussjob n1 test.com 14                  # 14 threads
gaussjob n3 job.gjf 16 72:00:00          # 16 threads, 72 hours
```

---

## Configuration

Edit the **USER EDIT AREA** at the top of `bin/gaussjob`:

```sh
G16_EXEC="/home/smlee99/program/g16/g16"   # Path to Gaussian16 executable
GAUSS_EXEDIR="/home/smlee99/program/g16"   # Gaussian EXEDIR
SCR_BASE="/home/smlee99/scratch"           # Base path for per-job scratch
JOBNAME="g16"                              # SLURM job name prefix
DEFAULT_TIME="168:00:00"                   # Default walltime
```

These values are exported into the generated `sbatch` script and echoed in the job banner at runtime.

---

## How it works

1. **Validates input** (argument count, `sbatch` availability, input file existence).
2. **Resolves absolute input path** using `readlink -f` or `realpath`.
3. **Generates a temporary batch file** (kept in the submit directory, e.g., `gaussjob_AB12CD.sbatch`).
4. **Sets up per-job scratch** at `$SCR_BASE/$SLURM_JOB_ID` and exports `GAUSS_SCRDIR`.
5. **Sets OpenMP threads** from `SLURM_CPUS_PER_TASK`.
6. **Runs Gaussian**:

   ```
   g16 <input> > <basename>.log 2>&1
   ```
7. **Cleans scratch** on `EXIT` and common signals (`INT`, `TERM`, `HUP`, `QUIT`, `USR1`, `USR2`).

---

## Logs & outputs

* **SLURM output**: `${JOBNAME}.${JOBID}.out` (e.g., `g16.1234567.out`)
* **Gaussian log**: `${BASENAME}.log` (same basename as your `.com`/`.gjf`)

The batch script used for submission is preserved for reproducibility.

---

## Best practices

* Put large scratch on a **fast local disk** (node-local SSD/NVMe) if available; point `SCR_BASE` to that mount.
  If node-local paths differ by node, consider per-partition wrappers or environment modules.
* Match `CPUS` to your Gaussian route (e.g., `%NProcShared=16`) or rely on `OMP_NUM_THREADS`.
* Remove or adjust `#SBATCH --gres=gpu:0` if your cluster/partition doesn’t require it.

---

## Troubleshooting

**`ERROR: sbatch not found in PATH`**
SLURM client not loaded. Load your scheduler module or update your `PATH`.

**`ERROR: input file not found: ...`**
Provide a valid `.com` or `.gjf` path.

**`No space left on device`** (in SLURM or Gaussian logs)
Scratch/base filesystem is full or out of inodes.

* Check: `df -h "$SCR_BASE"`
* Prune old job dirs under `$SCR_BASE`
* Ensure `$GAUSS_SCRDIR` points to a filesystem with sufficient free space

**Gaussian aborts with scratch errors**
Confirm `$GAUSS_EXEDIR` and `$GAUSS_SCRDIR` are correct and writable on the compute node.

**Threading looks wrong**
The script sets `OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK`.
Verify your partition grants the requested CPUs, and your Gaussian route isn’t overriding with a conflicting `%NProcShared`.

---

## Security & quotas

* The script removes scratch on completion and common signals, but **abrupt node failures** can leave residual scratch.
  Periodically prune `$SCR_BASE` (e.g., delete dirs older than *N* days).
* Ensure permissions on `$SCR_BASE` are appropriate for a multi-tenant cluster.

---

## Development

* Keep the script POSIX-sh friendly for portability.
* Keep SLURM directives minimal; site-specific flags can be toggled via small patches or environment variables.

---

## Roadmap

* Optional `--partition` / `--qos`
* Optional GPU toggle
* Auto-inject `%NProcShared` if absent
* Email notifications (`--mail-type/--mail-user`)

---

## Contributing

PRs and issues are welcome. When reporting bugs, please include your SLURM version and a brief description of your cluster environment.

---

## License

MIT © Contributors. “Gaussian” and “Gaussian 16” are trademarks of **Gaussian, Inc.** This project is not affiliated with or endorsed by Gaussian, Inc.

---

## Citation

If this tool helps your work, cite the repository URL in your computational methods or acknowledgements as a utility for SLURM submission and scratch management.

---

## Example session

```bash
$ ls
input.com

$ gaussjob n3 input.com 16 72:00:00
Submitted Gaussian job: 1234567
  Batch script: ./gaussjob_Z9K2VY.sbatch
  SLURM log   : g16.1234567.out

# Later
$ tail -n 6 g16.1234567.out
== Gaussian16 submit ==
Host           : n3
JobID          : 1234567
Submit dir     : /home/user/calc
GAUSS_SCRDIR   : /scratch/1234567
Time limit     : 72:00:00
```
