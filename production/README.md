# Production Files

Production files are used to keep track of different parameters, settings and environments across the entire CTA simulation pipeline. For example, references to the input parameters used in CORSIKA and `sim_telarray` are stored alongside the corresponding run. Similarly, CORSIKA and `sim_telarray` version numbers are stored as part of a build configuration. This keeps all relevant information in a central location, making it easier to keep track of runs in the future.

Each file uses [yaml](https://yaml.org/) syntax, so as to be both human- and machine-readable.


## Contents

* [Database hierarchy](#database-hierarchy)
* [Adding a new run](#adding-a-run-to-runsyml)
* [Adding a new production](#adding-a-production-to-productionsyml)
* [Adding a new set of SLURM staging parameters](#adding-SLURM-parameters-to-stagingyml)
* [Adding a new build](#adding-a-build-to-buildsyml)
* [Future work](#future-work)


## Database hierarchy

Inside the yaml database there is a hierarchy designed to encapsulate different scopes of the work, which is structured as follows:

* Low level: staging and build parameters
* Mid level: runs, containing one set of staging and build parameters each
* High level: productions, containing multiple runs

Staging parameters contain information on parameters used in the job staging process. Build parameters contain dependency versions, compilers and other build parameters used during compilation. Both are required to fully define a run, which may be seen as a combination of input and output parameters, binary executables and SLURM job information. Sitting above this are productions, which are sets of runs with a singular purpose. These can be seen as grouping runs into related projects, and exist purely as an organisational layer.


## Adding a run to `runs.yml`

Following is an example entry for a new run with run ID `example_run_id_01` added to the `runs.yml` file:

```yaml
example_run_id_01:
  summary: |
    An example of a CORSIKA run
    Using gamma, SSTOnly defaults, nshow=1000, max scatter radius = 1500m
  task: CORSIKA
  status: COMPLETED
  parameters: ~/CTA_MC/Data/corsika/run000001/input
  build: example_build_01
  staging:
    base: example_base_01
    job:
      id: 1234567
      submit: 2019-01-31 00:00:00+10:30
      start: 2019-01-31 00:01:00+10:30
      end: 2019-01-31 12:00:00+10:30
      parameters:
        mem: 512MB
  output: ~/CTA_MC/Data/corsika/run000001
```

The `example_run_id_01` is an example of a unique identifier used to name each run. This is a value chosen by the user to reflect at-a-glance what the run is.

The remaining fields are as follows:

* `summary`: a human-readable summary of the run. This may contain a brief summary of the array geometry used, number of showers simulated, etc.
* `task`: either one of `CORSIKA` or `simtel_array`
* `status`: one of `PENDING`, `RUNNING`, `COMPLETED`, `MIXED` or `FAILED`. The meaning of each value is as follows:
  * `PENDING`: either awaiting submission, or in the queue yet to start running
  * `RUNNING`: currently running on the SLURM queue
  * `COMPLETED`: successfully completed with no errors
  * `MIXED`: for array jobs, where some tasks completed sucessfully and others failed
  * `FAILED`: exited with an error
* `parameters`: path to input parameter files for the given `task` type
* `build`: the unique ID corresponding to a build entry in `builds.yml`
* `staging`: job staging information
  * `base`: the unique ID corresponding to a staging entry in `staging.yml`
  * `job`: information on the job itself. Fields may be blank depending on the value of `status`
    * `id`: SLURM job ID
    * `submit`: time of submission to SLURM queue, including timezone
    * `start`: time job began running, including timezone. Only present if job is not `PENDING`
    * `end`: time job finished running, including timezone. Only present if `COMPLETED`, `MIXED` or `FAILED`
    * `parameters`: (optional) a list of SLURM parameters that are changed relative to the ones given by the staging `base` field
* `output`: path to directory containing output files

### CORSIKA

In the above example, you may have noticed that the input file is not the same as that given to CORSIKA when invoked by one of the run scripts. This is because the one given in the run script is a generic file which is then processed, giving a set of parameters that correspond to those given to the run script. This is written as a file called `input` in each `runXXXXXX/` directory. The reason why input and output are written to the same directory is because CORSIKA uses its own numbering system to keep runs separate. It does this by reading the number stored in `$CTA_PATH/Data/corsika/run.conf` and making a directory based on that number.

### `sim_telarray`

Unlike CORSIKA, `sim_telarray` does not process a generic input file. All base input (configuration) files used in CTA are found in `$CTA_PATH/sim_telarray/multi/`. As an example, for a `sim_telarray` job simulating a PROD4 SST telescope using the ASTRI+CHEC-S camera would have the following parameter field:

```yaml
parameters: ~/CTA_MC/sim_telarray/multi/multi_cta-prod4-sst-astri+chec-s.cfg
```

Similarly, the output of a `sim_telarray` job also follows a different pattern to the CORSIKA outputs. When `sim_telarray` is called through a run script, a CORSIKA run number must be provided, which is then used to pipe the CORSIKA results into `sim_telarray`. As such, when the output is written, it uses the same run number as the corresponding CORSIKA run. For example, if a `sim_telarray` run is working on output from a CORSIKA run with run number 10, the corresponding `output` field would look like:

```yaml
output: ~/CTA_MC/Data/sim_telarray/cta-prod4-sst-astri+chec-s/0.0deg/Data/proton_20deg_180deg_run10___cta-prod4-sst-astri+chec-s_desert-2150m-Paranal-sst-astri+chec-s.simtel.zst
```

What one might also notice is that the `sim_telarray` output paths are much more messy. However, this path is simply a way of sorting outputs by their camera type. Similarly, the actual name of the output file summarises primary particle (`proton`), shower zenith angle (`20deg`) and azimuth angle (`180deg`), atmosphere profile (`desert`), elevation (`2150m`) and site location (`Paranal`), alongside telescope and camera type.

### Note on paths

Note that for array jobs that have multiple input and output files, it is suffice to use the path expansion shorthand `{a..b}`, where `a` is the start of the range, and `b` is the end. For example, if a CORSIKA array job runs five tasks each simulating primary particles from a unique subrange of the full energy range, then the `parameters` field for the run should look like this:

```yaml
parameters: ~/CTA_MC/Data/corsika/run00000{1..5}
```

The above applies for when the the current CORSIKA run number in `$CTA_PATH/Data/corsika/run.conf` is `1`. If the first run has a different CORSIKA run number, then the range should be updated accordingly.

As `sim_telarray` jobs do not use input files unique to each run, we would only use this range syntax in the `output` field:

```yaml
output: ~/CTA_MC/Data/sim_telarray/cta-prod4-sst-astri+chec-s/0.0deg/Data/proton_20deg_180deg_run{10..100}___cta-prod4-sst-astri+chec-s_desert-2150m-Paranal-sst-astri+chec-s.simtel.zst
```

The benefit of using this syntax is that the path is able to be expanded by the shell. In other words, typing `ls ~/CTA_MC/Data/corsika/run00000{1..5}` into the terminal lists all of the contents of each directory given in the range.

#### Padding the range for CORSIKA runs

Due to this ability for the shell to expand the range, special consideration has to be taken when working with CORSIKA runs that cross an order of magnitude, _i.e._ 9-11, 99-101, 600-1020. When using this range syntax, leading zeros must be included as padding to make sure the number of digits for the start and end of the range is the same. For example:

```yaml
parameters: ~/CTA_MC/Data/corsika/run000{001..100}
```

Note also that the number of zeros in the `run` section before the range is also cut, so upon expansion there will always be six digits after `run`.

This padding is not needed for `sim_telarray` runs.


## Adding a production to `productions.yml`

The `productions.yml` file groups runs given in `runs.yml` together based on their final purpose. For example, one production could be running CORSIKA to estimate the disk usage and compute time needed. As part of this, runs simulating gammas and protons would be used. Following is an example of what a production entry in `productions.yml` looks like:

```yaml
example_production:
  CORSIKA:
    runs:
      - example_corsika_run_01
      - example_corsika_run_02
      - example_corsika_run_03
  sim_telarray:
    runs:
      - example_simtel_run_01
      - example_simtel_run_02
      - example_simtel_run_03
```

Here, each ID listed under the `runs` field correspond to run entries given in `runs.yml`. Again, `example_production` is a user-defined unique identifier in the same way as with run identifiers.


## Adding SLURM parameters to `staging.yml`

The `staging.yml` file contains sets of SLURM staging parameters that may be used by different groups of jobs. For example, a set of staging parameters could be used for `sim_telarray` jobs which usually require less memory and walltime than corresponding CORSIKA jobs. Following is an example of what a staging entry looks like:

```yaml
example_base_01:
  scheduler: SLURM
  parameters:
    account: cta
    partition: cpu
    get-user-env: true
    ntasks: 1
    cpus-per-task: 1
    time: 1-00:00:00
    mem: 512MB
```

`example_base_01` is a user-defined ID which can be referred to in a run's `job` section, through the `base` field.

The remaining fields are defined as follows:

* `scheduler`: the job scheduler in use. For Phoenix, this is `SLURM`, but this may be different for other supercomputers
* `parameters`: a map of (SLURM parameter name, value) pairs as used in `#SLURM` directives in `.job` files


## Adding a build to `builds.yml`

The `build.yml` file records version information for tools like compilers used in the building process of the CTA simulation pipeline, as well as the versions of the resulting products. An example build entry looks like the following:

```yaml
example_build_01:
  modules:
    - GCCcore/6.3.0
  tools:
    gcc:
      version: 6.3.0
  outputs:
    CORSIKA:
      version: 6.990
    IACT/ATMO:
      version: 1.56
    sim_telarray:
      version: 1541504878
```

Again, `example_build_01` can be any user-defined identifier. It is used in run entries as the value of the `build` field.

The remaining fields are:

* `modules`: any modules loaded through `module load` that were used in compiling the source
* `tools`: binaries used throughout the build process, including any identifying information such as version number, git commit hash, etc.
* `outputs`: the resulting products, including any identifying information such as version number, git commit hash, etc.


## Future work

Much of the work done in maintaining these databases could be quite easily automated. Due to yaml being both human- and machine-readable, a Python script could be written that updates the appropriate databases when jobs are submitted.

It is expected that the needs of users will change in the future, and so the structure of this database may change to reflect this. For example, an automation script for submission may remove the need to use the `staging.yml` file. If any updates in this vein occur, it is important that documentation is updated and maintained to reflect this. If there is one thing worse than undocumented code, it's outdated documentation.
