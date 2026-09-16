# Distributed IMERG Precipitation Tracking with Dask

This repository contains a reproducible notebook workflow for identifying and tracking precipitation systems in time-ordered IMERG-style NetCDF data. The workflow uses Dask workers on AWS Fargate to parallelize independent per-timestamp operations while preserving the chronological execution required for temporal tracking.

## Scientific Workflow

The experiment transforms a sequence of precipitation snapshots into labeled, temporally tracked precipitation systems and rendered maps.

```text
NetCDF precipitation files
          |
          v
Chronological input selection
          |
          v
Precipitation masks
          |
          v
Connected-component labeling       [distributed by timestamp]
          |
          v
Temporal feature tracking           [sequential by timestamp]
          |
          v
Tracked map generation              [distributed by timestamp]
          |
          v
NumPy arrays, serialized maps, and PNG visualizations
```

### 1. Input Preparation

The notebook searches for NetCDF (`.nc4`) files under the configured local data directories and uses the requested number of files in chronological order. When the required local inputs are unavailable, it can stage them from the configured Amazon S3 prefix while preserving the source directory structure.

The default demonstration processes four consecutive precipitation snapshots. The input count can be changed with `IMERG_INPUT_FILE_COUNT`.

### 2. Precipitation Masking

Each NetCDF file is read on the notebook coordinator and converted into a precipitation mask. The masks form a time-indexed array that provides a consistent input representation for connected-component analysis.

### 3. Connected-Component Detection

Each timestamp is independent at this stage. The coordinator submits one mask at a time to the Dask cluster, where workers:

- identify spatially connected precipitation regions using eight-neighbor connectivity;
- remove regions below the configured minimum size, which defaults to 625 grid cells; and
- return the labeled component array to the coordinator.

The resulting arrays are saved per timestamp and combined into an intermediate serialized component map.

### 4. Temporal Tracking

The component maps are processed chronologically to maintain consistent labels as precipitation systems evolve. This stage compares the current timestamp with the preceding timestamp to identify continuing, newly formed, merged, split, and dissipated systems.

Temporal tracking runs sequentially because the labels at time `t` depend on the tracked state at time `t-1`.

### 5. Map Generation

After tracking is complete, rendering is distributed across the Dask workers. Each task receives one tracked timestamp and creates a PNG map with its corresponding observation time. The coordinator gathers completion results and retains the generated files in the experiment output directory.

## Distributed Execution Model

The notebook creates an on-demand Dask cluster in AWS ECS Fargate:

```text
Jupyter notebook coordinator
          |
          +---- Dask scheduler task
          |
          +---- Dask worker task 1
          +---- Dask worker task 2
          +---- Dask worker task 3
          +---- Dask worker task 4
```

The default cluster configuration is:

| Component | Count | CPU | Memory | Threads |
|---|---:|---:|---:|---:|
| Scheduler | 1 | 1 vCPU | 2 GiB | N/A |
| Worker | 4 | 1 vCPU each | 4 GiB each | 1 each |

The worker count and task sizes are configurable through environment variables. Increasing the worker count can accelerate connected-component detection and plotting when enough timestamps are available. It does not parallelize temporal tracking, whose dependency chain is inherently ordered in the current implementation.

## Runtime Configuration

The portal or runtime environment supplies the AWS cluster, identity, network, and container settings used by the notebook.

| Variable | Purpose | Default |
|---|---|---:|
| `IMERG_INPUT_FILE_COUNT` | Number of chronological NetCDF files to process | `4` |
| `DASK_WORKER_COUNT` | Number of Fargate worker tasks | `4` |
| `DASK_WORKER_CPU` | CPU units allocated to each worker | `1024` |
| `DASK_WORKER_MEMORY` | Memory in MiB allocated to each worker | `4096` |
| `DASK_WORKER_THREADS` | Dask threads per worker | `1` |
| `DASK_SCHEDULER_CPU` | CPU units allocated to the scheduler | `1024` |
| `DASK_SCHEDULER_MEMORY` | Memory in MiB allocated to the scheduler | `2048` |
| `INPUT_S3_PREFIX` | S3 prefix containing the input data | Runtime supplied |
| `DASK_WORKER_IMAGE` | Container image used by scheduler and workers | Runtime supplied |
| `DASK_ECS_CLUSTER_ARN` | ECS cluster used for temporary Dask tasks | Runtime supplied |
| `DASK_EXECUTION_ROLE_ARN` | ECS task execution role | Runtime supplied |
| `DASK_TASK_ROLE_ARN` | Application task role | Runtime supplied |
| `DASK_SUBNET_IDS` | Subnets used by the Dask tasks | Runtime supplied |
| `DASK_SECURITY_GROUP_IDS` | Security groups used by the Dask tasks | Runtime supplied |

## Outputs

Results are written under `/workspace/output/` and include:

- a time-indexed precipitation mask array;
- one connected-component array per timestamp;
- an intermediate serialized component map;
- one temporally tracked component array per timestamp; and
- one tracked PNG map per timestamp.

For the default four-file run, the workflow produces four per-timestamp component arrays, four tracked arrays, and four tracked maps.

## Repository Notebooks

| Notebook | Purpose |
|---|---|
| `1_IMERGDASK_FLINC_NASA_ESTIM_DEMO_PARALLEL.ipynb` | Implements input staging, scientific processing, distributed execution, and cleanup. |
| `2_AFTER_CAPTURE_AUDIT_VERIFY.ipynb` | Verifies the captured inputs and generated workflow artifacts. |

Run the main workflow notebook from top to bottom so that inputs are prepared before cluster startup and the temporary Dask resources are shut down after processing. The verification notebook can then be used to inspect the resulting capture and workflow artifacts.

## Resource Lifecycle

The Dask scheduler and workers are temporary Fargate tasks. The final cleanup cell closes the Dask client and cluster, and it should be run after successful completion or after any processing failure. This cleanup is separate from stopping the surrounding Jupyter environment.
