# ImergView-Dask

ImergView investigates how precipitation systems form, evolve, merge, split, and dissipate over time. The broader workflow works with STARE-indexed scientific products derived from NASA Global Precipitation Measurement (GPM) IMERG precipitation observations and Midlatitude Storm Area (MCMS) extra-tropical cyclone tracks derived from MERRA-2 sea-level pressure fields.

These products operate at different resolutions: GPM IMERG precipitation is approximately 10 km every 30 minutes, while the MCMS cyclone product is based on 0.625 x 0.5 degree MERRA-2 fields every three hours. Combining high-resolution gridded observations, spatial feature detection, and chronological tracking makes the experiment computationally intensive and difficult to reproduce consistently.

The notebook in this repository focuses on the precipitation-feature workflow. It reads a time-ordered sequence of IMERG-style NetCDF files containing `PRECTOT`, identifies connected precipitation regions at each timestamp, tracks those regions through time, and creates maps of the tracked systems.

## Scientific Workflow

```text
Time-ordered precipitation snapshots
                 |
                 v
      Precipitation thresholding
                 |
                 v
     Binary precipitation masks
                 |
                 v
  Connected-component identification
                 |
                 v
 Chronological feature association
                 |
                 v
     Tracked precipitation systems
                 |
                 v
       Arrays and map products
```

### 1. Select the observations

The experiment selects NetCDF precipitation snapshots in chronological order and extracts the observation time from each filename. Maintaining this order is essential because each tracking step depends on the preceding timestamp.

### 2. Create precipitation masks

For every timestamp, the workflow reads the `PRECTOT` field and applies a precipitation threshold. The result is a time-indexed binary mask that separates qualifying precipitation from the surrounding grid.

### 3. Identify precipitation systems

Connected-component labeling identifies contiguous precipitation regions using eight-neighbor spatial connectivity. Regions smaller than the configured minimum area are removed so that the remaining labels represent substantial precipitation systems rather than isolated grid cells.

### 4. Track systems through time

The workflow compares labels at each timestamp with labels from the immediately preceding timestamp. Spatial overlap is used to identify continuing systems, new formation, mergers, splits, and dissipation. Stable labels are assigned across the sequence to preserve the history of each tracked system.

### 5. Generate scientific products

The experiment writes intermediate masks and component maps, tracked label arrays, and one visualization for each processed timestamp. Together, these products expose both the spatial structure and temporal evolution of the detected precipitation systems.

## Why This Experiment Is Challenging to Reproduce

- The inputs are large, multidimensional scientific files sampled at frequent time intervals.
- Feature identification operates over every grid cell in every selected snapshot.
- Some stages can process timestamps independently, while temporal tracking must preserve strict chronological dependencies.
- The workflow combines NetCDF I/O, numerical arrays, connected-component labeling, geospatial plotting, and serialized intermediate products.
- Results depend on consistent input ordering, thresholds, connectivity rules, package versions, and filesystem layout.

## Data Context

| Scientific product | Source | Spatial resolution | Temporal resolution |
|---|---|---:|---:|
| Precipitation | GPM IMERG | Approximately 10 km | 30 minutes |
| Extra-tropical cyclones | MCMS from MERRA-2 sea-level pressure | 0.625 x 0.5 degrees | 3 hours |

The ImergView ecosystem uses STARE, pySTARE, and STAREPandas to support spatiotemporal indexing and analysis across these products. The notebook in this repository demonstrates the precipitation-system detection and tracking portion of that larger scientific problem.

## Notebook

`ImergView-Dask.ipynb` contains the complete experiment, including input selection, precipitation masking, connected-component identification, temporal tracking, output generation, and resource cleanup.

Run the notebook from top to bottom. The final outputs include:

- a time-indexed precipitation mask;
- connected-component arrays for each timestamp;
- serialized intermediate component maps;
- temporally tracked component arrays; and
- one tracked precipitation map per timestamp.
