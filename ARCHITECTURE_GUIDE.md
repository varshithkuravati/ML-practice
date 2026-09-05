# AI-ML Intelligent Dead Reckoning System
## Beginner-Friendly Architecture Guide

This document explains the complete architecture of the **AI-ML IDR System** in beginner-friendly language.

The project is an Intelligent Dead Reckoning system for vehicles. Its goal is to keep estimating a vehicle's position when GNSS/GPS becomes unavailable, such as inside a tunnel, underground parking area, or urban canyon.

The system combines:

- Smartphone IMU sensors: accelerometer and gyroscope.
- GPS/GNSS data when it is available.
- Vehicle speed information during model training.
- Neural networks that estimate speed and sensor errors.
- Classical Kalman-filter sensor fusion.
- Vehicle-motion constraints.
- Optional OpenStreetMap road matching.
- Evaluation tools that measure position drift during simulated GNSS outages.

The most important idea is this:

> GPS gives an absolute position but may disappear. IMU sensors are always available but accumulate error. This project combines both so that GPS corrects the system when available and inertial estimation continues when GPS is lost.

---

## 1. What Problem Does the Project Solve?

### 1.1 GNSS outage problem

A vehicle usually receives its position from GPS/GNSS. However, GNSS can temporarily fail because of:

- Tunnels.
- Tall buildings.
- Underground roads.
- Mountains.
- Radio interference.
- Deliberate signal obstruction.

When GNSS stops, a navigation system still needs to estimate where the vehicle is.

### 1.2 Dead reckoning

Dead reckoning estimates a new position from a previous position and motion measurements.

For example, if a vehicle was at position `(0, 0)`, then moved forward at `10 m/s` for one second, its new position is approximately `(10, 0)`.

In practice, the system uses acceleration and turning-rate measurements:

1. Read acceleration and angular velocity from the IMU.
2. Estimate velocity.
3. Integrate velocity to estimate position.
4. Correct accumulated errors whenever another trustworthy measurement is available.

The problem is that small sensor errors accumulate over time. If the accelerometer has a tiny bias, integrating it once affects velocity, and integrating velocity again affects position. This is why pure IMU dead reckoning drifts.

### 1.3 The project's solution

The project uses several layers of protection against drift:

```text
IMU sensors
    |
    v
Sensor cleaning and alignment
    |
    v
Neural velocity and denoising models
    |
    v
EKF inertial prediction
    |
    +--> GNSS correction when GNSS is available
    |
    +--> NHC/ZUPT/AI-speed corrections during GNSS outage
    |
    +--> Optional road-map snapping
    |
    v
Estimated trajectory and drift metrics
```

---

## 2. Repository Structure

The repository uses a Python `src` layout.

```text
AI-ML-IDR-System/
|
+-- data/                         Generated data directories
|   +-- raw/                      Downloaded or generated CSV files
|   +-- processed/                Windowed NumPy training files
|   +-- osm/                      Cached OpenStreetMap data
|
+-- models/                       Saved PyTorch checkpoints and exports
|
+-- reports/                      Evaluation output and generated figures
|
+-- results/                      Checked-in benchmark results and figures
|
+-- scripts/
|   +-- download_data.py          Download, generate, and validate data
|   +-- prepare_data.py           Preprocess raw data into windows
|   +-- export_onnx.py            Extended model export script
|
+-- src/idr/
|   +-- config.py                 Global paths and hyperparameters
|   +-- io/                       CSV schemas, loaders, and preprocessing
|   +-- calib/                    Phone-to-vehicle sensor alignment
|   +-- models/                   Neural networks and training programs
|   +-- filters/                  EKF, UKF, NHC, ZUPT, and fusion
|   +-- mapmatch/                 Road graph loading and trajectory snapping
|   +-- eval/                     Blackouts, scenarios, metrics, plots
|   +-- export/                   ONNX conversion and validation
|
+-- tests/                        Unit tests
+-- DECISIONS.md                  Architecture decision log
+-- Makefile                      Workflow commands
+-- pyproject.toml                Python package configuration
+-- requirements.txt              Dependencies
+-- README.md                     Project overview and quickstart
```

### 2.1 Generated directories

The `data/`, `models/`, and `reports/` directories are partly runtime directories. `src/idr/config.py` creates the expected data and report directories when it is imported.

Raw and processed data are normally kept out of version control. The checked-in `results/` and `reports/` content contains example or previously generated evaluation outputs.

---

## 3. The Main Execution Pipeline

The normal workflow is controlled by the Makefile.

```text
make setup
    |
    v
make data
    |
    +--> download/validate raw IO-VNBD CSV files
    +--> standardize smartphone and vehicle records
    +--> create sliding training windows
    |
    v
make train
    |
    +--> train VelocityEstimatorNet
    +--> train IMUDenoiseNet
    |
    v
make eval
    |
    +--> create GNSS outage scenarios
    +--> run inertial fusion
    +--> calculate drift metrics
    |
    v
make plots
    |
    +--> create trajectory and error figures
    +--> write a report
    |
    v
make export
    |
    +--> export selected networks to ONNX
    +--> validate ONNX execution
```

The commands are defined in `Makefile`:

| Command | Purpose |
|---|---|
| `make setup` | Installs Python dependencies and the package in editable mode. |
| `make data` | Downloads or creates raw data, validates it, and preprocesses it. |
| `make train` | Trains the primary denoising and velocity models. |
| `make eval` | Runs GNSS blackout evaluation. |
| `make plots` | Generates evaluation figures and report files. |
| `make export` | Exports the primary models to ONNX. |
| `make test` | Runs the tests in `tests/`. |
| `make clean` | Removes common build and cache directories. |

The documented `demo` target refers to a Streamlit file under `src/idr/demo/app.py`, but that file is not present in the inspected repository tree. The demo command therefore requires additional implementation before it can be used.

---

## 4. Configuration and Important Constants

The central configuration is in [src/idr/config.py](src/idr/config.py).

### 4.1 Sampling rate

```python
SAMPLING_RATE_HZ = 10.0
DT = 0.1
```

The target sampling rate is 10 measurements per second. `DT` is the time between measurements:

$$
DT = \frac{1}{10} = 0.1\ \text{seconds}
$$

Most window sizes and filter updates are based on this value.

### 4.2 Reproducibility

The default random seed is `42`. `set_seed()` seeds Python, NumPy, and PyTorch. This helps make training and generated experiments repeatable, although complete determinism can still depend on the hardware and backend.

### 4.3 Dataset split

The default drive-level split is:

| Split | Drives |
|---|---|
| Training | `M`, `S`, `Vta`, `Vtb`, `Vw` |
| Validation | `Y1` |
| Test | `Vf` |

A drive-level split is important. Neighboring windows from the same drive are highly similar. Splitting individual windows randomly could allow the model to see nearly identical motion in both training and testing, producing misleadingly good results.

### 4.4 Training windows

The default window contains 50 samples. At 10 Hz:

$$
50 \times 0.1 = 5\ \text{seconds}
$$

The stride is 10 samples, or one second. Therefore, consecutive windows overlap heavily.

### 4.5 Model configuration

Important defaults include:

- Six IMU input channels.
- Hidden dimension of 64 for the main models.
- Batch size of 64.
- Learning rate of `0.001`.
- 30 training epochs.
- Weight decay of `0.0001`.

### 4.6 Filter configuration

The filter configuration contains expected noise levels for:

- GNSS position.
- GNSS velocity.
- Accelerometer process noise.
- Gyroscope process noise.
- Lateral NHC measurement noise.
- Vertical NHC measurement noise.

These values are not just software settings. They express how much the filter should trust each sensor or constraint.

---

## 5. Input Dataset and Data Representation

The main dataset is IO-VNBD. It contains paired smartphone and vehicle data.

### 5.1 Smartphone CSV files

Smartphone files use names such as:

```text
S-M.csv
S-Vf.csv
```

They are expected to contain six IMU channels:

```text
acc_x, acc_y, acc_z,
gyro_x, gyro_y, gyro_z
```

They can also contain GPS latitude, longitude, speed, and heading.

### 5.2 Vehicle CSV files

Vehicle files use names such as:

```text
V-M.csv
V-Vf.csv
```

They can provide vehicle-side supervision such as:

- Wheel speed.
- Vehicle yaw rate.
- GPS position.
- Other vehicle signals depending on the source file.

The vehicle information is mainly useful during training and evaluation. The intended deployed system uses smartphone sensors rather than vehicle CAN/OBD access.

### 5.3 Canonical schema

`src/idr/io/schema.py` converts many possible source column names into a shared vocabulary.

For example, the following names may all be recognized as speed-like fields:

```text
speed
vehicle_speed
ground_speed
gps_speed
ecu_speed
```

The important objects and functions are:

- `CANONICAL_FIELDS`: list of supported standard names.
- `FIELD_ALIASES`: aliases for matching source columns.
- `SchemaMap`: stores the source-to-canonical mapping.
- `normalize_col_name()`: makes column names easier to compare.
- `detect_schema()`: detects the schema and decides whether a file is smartphone or vehicle data.

The schema detector requires all six accelerometer and gyroscope fields. GPS fields are strongly useful but produce a warning rather than always causing failure.

### 5.4 Standardized data

`src/idr/io/loader.py` performs the next step:

1. Find the smartphone and vehicle CSV pair.
2. Detect their schemas.
3. Rename source columns into canonical names.
4. Convert timestamps into relative seconds.
5. Interpolate missing values.
6. Fill missing values at the beginning or end of a series.
7. Align phone and vehicle streams.

`IOVNBDrive.get_synced_data()` exposes the aligned information as arrays such as:

- `phone_imu`: shape `(N, 6)`, ordered as three accelerometer and three gyroscope values.
- `phone_gps`: latitude, longitude, and available GPS fields.
- `vehicle_speed`: one speed value per time sample when available.
- `timestamps`: relative time in seconds.

Here, `N` means the number of synchronized measurements.

---

## 6. Data Ingestion and Preprocessing

### 6.1 `scripts/download_data.py`

This script is responsible for obtaining and checking raw data.

Its main responsibilities are:

- Attempt to download the IO-VNBD archive.
- Extract the archive.
- Generate a mock dataset if the official download is unavailable.
- Validate expected drive pairs.
- Support `--validate-only` so existing data can be checked without downloading again.

The mock-data behavior is useful for development and smoke tests, but synthetic data is not a substitute for real benchmark data.

### 6.2 `scripts/prepare_data.py`

This script calls the preprocessing package code. It also loads a drive and generates an exploratory sensor-timeseries plot.

The data path is:

```text
raw CSV files
    -> load_drive_pair()
    -> get_synced_data()
    -> create_sliding_windows()
    -> train_data.npz / val_data.npz / test_data.npz
```

### 6.3 `src/idr/io/preprocess.py`

The main preprocessing objects are:

- `create_sliding_windows()`.
- `IDRWindowDataset`.
- `preprocess_dataset()`.

A typical window has this form:

```text
IMU input:       (6, 50)
velocity target: scalar
IMU target:      (6,)
```

The six channels are arranged first, followed by the time dimension. This is the common PyTorch `Conv1d` layout:

```text
(batch, channels, time)
```

The velocity target normally represents the speed at the end of the window. The IMU target is used by the denoising training path.

---

## 7. Coordinate Frames and Calibration

Sensor data has a coordinate-system problem. A phone may be mounted at an angle. Its `x` axis may not point in the vehicle's forward direction.

### 7.1 Why alignment matters

The filter assumes that it knows which direction is forward, lateral, and vertical. If phone axes are used directly while the phone is rotated, forward acceleration can be mistaken for side acceleration, and turning behavior can be interpreted incorrectly.

### 7.2 `src/idr/calib/alignment.py`

The main class is `PhoneToVehicleAligner`.

Its approach is:

1. Use a stationary period to estimate gravity direction.
2. Use motion acceleration to estimate the forward direction.
3. Build a rotation matrix.
4. Rotate accelerometer and gyroscope measurements into vehicle coordinates.

Important functions include:

- `compute_rotation_matrix()`: creates a Z-Y-X Euler rotation matrix.
- `estimate_from_stationary_and_motion()`: estimates orientation from stationary and moving data.
- `transform_imu()`: transforms IMU arrays into vehicle coordinates.

After alignment, the rest of the pipeline can interpret channels consistently. The intended convention treats forward acceleration and vehicle yaw rate as important signals for the motion model.

---

## 8. Neural Network Models

The project contains several neural models. The primary workflow trains two of them, while the others support experiments or future extensions.

### 8.1 Forward velocity estimator

File: `src/idr/models/velocity_net.py`

Class: `VelocityEstimatorNet`

Input:

```text
(B, 6, L)
```

Where:

- `B` is the batch size.
- `6` is the number of IMU channels.
- `L` is the number of time samples in a window.

Architecture conceptually:

```text
six-channel IMU window
    -> Conv1D feature extraction
    -> dilated temporal convolutions
    -> GRU sequence processing
    -> motion/stationary gate
    -> Softplus speed output
```

The output is:

```text
(B, 1)
```

The Softplus output keeps the predicted speed non-negative. This makes physical sense because speed is normally not negative, even though signed forward velocity would need a different interpretation.

The model is trained using a Huber-style loss in `train_all.py`. Huber loss is less sensitive to extreme target errors than ordinary squared error.

### 8.2 IMU denoising network

File: `src/idr/models/imu_denoise.py`

Class: `IMUDenoiseNet`

Input:

```text
(B, 6, L)
```

Output:

```text
(B, 6)
```

The network is a residual dilated 1D CNN. It summarizes a time window and predicts a six-value bias or noise residual.

The idea is that the network can learn common sensor artifacts such as:

- Constant sensor bias.
- High-frequency vibration.
- Phone mounting noise.
- Motion-induced sensor patterns.

The primary training code currently uses the phone IMU as the default denoising target. That means the current training path does not necessarily train against a truly clean sensor reference. This is an important implementation limitation when interpreting the denoiser's output.

### 8.3 Inertial odometry network

File: `src/idr/models/inertial_odom.py`

Class: `InertialOdomNet`

This is a separate odometry model that predicts two-dimensional body-frame displacement and uncertainty:

```text
[dx_body, dy_body, log_variance_x, log_variance_y]
```

Its architecture includes:

- Multi-scale 1D convolution blocks.
- Residual blocks.
- A GRU.
- A four-value output head.

It supports both `(B, 6, W)` and `(B, W, 6)` input layouts.

The associated `gaussian_nll_loss()` uses heteroscedastic Gaussian negative log likelihood. In simple terms, the network predicts both its displacement estimate and how uncertain that estimate is.

`augment_imu_sample()` creates physics-inspired training variations such as:

- Speed scaling.
- Accelerometer bias perturbation.
- Noise scaling.
- Planar rotation.

This model is trained by `src/idr/models/train_odom.py`, separately from the main `make train` target.

### 8.4 Residual drift network

File: `src/idr/models/residual_net.py`

Class: `ResidualDriftNet`

This model predicts:

```text
[delta_x, delta_y, delta_yaw]
```

It is intended to estimate remaining position and heading drift. However, the inspected primary training and inference paths do not train or use it.

### 8.5 KalmanNet

File: `src/idr/filters/kalmannet.py`

Classes:

- `KalmanNetGainEstimator`.
- `KalmanNetFilter`.

KalmanNet uses a GRU to estimate Kalman gains from innovations and predicted state information. A Kalman gain determines how strongly a filter should adjust its state based on a measurement.

The implementation includes a classical EKF fallback. If a learned gain is invalid or too large, the filter can use the ordinary calculated gain instead.

Training support is in `src/idr/models/train_kalmannet.py`.

The model exists in the repository, but the primary Monte Carlo evaluation path does not currently make the `use_kalmannet` setting change the actual filtering behavior.

---

## 9. Training Architecture

### 9.1 Main trainer

File: `src/idr/models/train_all.py`

The main trainer contains reusable training loops:

- `train_epoch()` for a training pass.
- `eval_epoch()` for a validation pass.
- `train_pipeline()` for training the velocity and denoising networks.

The normal outputs are:

```text
models/velocity_net.pt
models/imu_denoise_net.pt
```

If processed data is missing, the script can create synthetic tensors for a development run. This allows the code path to execute, but the resulting models should not be treated as models trained on the real IO-VNBD benchmark.

### 9.2 Odometry trainer

File: `src/idr/models/train_odom.py`

This trainer:

1. Loads synchronized drives.
2. Converts GPS positions to local coordinates.
3. Computes body-frame displacement labels.
4. Creates augmented windows.
5. Trains `InertialOdomNet` with Gaussian NLL.
6. Saves `inertial_odom.pt`.

This training path uses its own drive preparation logic. Its split should be checked separately from `DatasetConfig` when comparing experiments.

### 9.3 KalmanNet trainer

File: `src/idr/models/train_kalmannet.py`

This trainer creates synthetic filter innovations and classical Riccati gains, then trains the neural gain estimator to imitate or improve the classical gain behavior.

A Riccati update is the mathematical part of a Kalman filter that updates its uncertainty covariance and calculates how much to trust a measurement.

---

## 10. Classical Sensor Fusion

The project uses a Kalman-filter family because the filter is designed for continuously combining predictions and measurements with uncertainty estimates.

### 10.1 EKF state

File: `src/idr/filters/ekf.py`

Class: `ExtendedKalmanFilter`

The main state has nine values:

```text
x = [p_e, p_n, p_u,
     v_e, v_n, v_u,
     yaw,
     accel_bias,
     gyro_bias]
```

Meaning:

- `p_e`, `p_n`, `p_u`: position in local East-North-Up coordinates.
- `v_e`, `v_n`, `v_u`: velocity in the same local frame.
- `yaw`: heading angle around the vertical axis.
- `accel_bias`: estimated accelerometer bias term.
- `gyro_bias`: estimated gyroscope bias term.

The filter stores both the state estimate and a covariance matrix. The covariance represents uncertainty in the state.

### 10.2 Prediction step

`predict()` advances the state using acceleration and yaw rate.

A simplified one-dimensional version is:

$$
 v_{t+1} = v_t + a_t \Delta t
$$

$$
 p_{t+1} = p_t + v_t \Delta t + \frac{1}{2}a_t(\Delta t)^2
$$

The real implementation handles multiple position and velocity components, heading, biases, and covariance propagation.

### 10.3 Measurement updates

The EKF provides updates for:

- `update_gnss_pos()`: corrects position using latitude/longitude converted into local coordinates.
- `update_gnss_vel()`: corrects velocity.
- `update_heading()`: corrects yaw.
- `update_velocity()`: applies a scalar AI speed measurement.
- `update_velocity_2d()`: applies a body-frame two-dimensional velocity measurement.

The general pattern is:

```text
prediction from IMU
    -> compare prediction with measurement
    -> calculate innovation
    -> calculate Kalman gain
    -> correct state and covariance
```

The innovation is the difference between what the filter expected and what the sensor observed.

---

## 11. Vehicle-Motion Constraints

A normal ground vehicle has useful physical constraints. These constraints provide information even when GNSS is unavailable.

### 11.1 Non-Holonomic Constraints

File: `src/idr/filters/nhc.py`

Classical ground vehicles usually have approximately:

- Zero lateral velocity in the vehicle body frame.
- Zero vertical velocity in the vehicle body frame.

These are called Non-Holonomic Constraints, or NHC.

The filter therefore creates a virtual measurement:

```text
lateral velocity ~= 0
vertical velocity ~= 0
```

The measurement is not from a physical sensor. It comes from the vehicle's expected motion.

The NHC update uses the vehicle orientation to transform world-frame velocity into body-frame velocity. The implementation also includes heading sensitivity in the Jacobian. This allows the constraint to help make heading observable during a GNSS outage.

`apply_nhc_update()` is the main function.

### 11.2 Zero-velocity updates

File: `src/idr/filters/zupt.py`

`StationaryDetector` looks for periods when the vehicle is stopped, using signals such as:

- Accelerometer variance.
- Gyroscope magnitude.

When the vehicle is stationary, `apply_zupt()` constrains velocity toward zero. This prevents the filter from inventing motion while the vehicle is stopped.

### 11.3 Zero angular-rate updates

`apply_zaru()` uses stationary periods to estimate gyroscope bias. Removing this bias is important because a small angular-rate error can cause a heading error, which then rotates future position estimates incorrectly.

### 11.4 Vehicle profiles

File: `src/idr/filters/vehicle_profiles.py`

The profile classes are:

- `VehicleProfile`: common interface and parameters.
- `CarProfile`: tighter ordinary-car constraints.
- `TwoWheelerProfile`: lean-aware motorcycle constraints.

A motorcycle can lean while turning. Treating its lateral motion exactly like a car would create false errors. `TwoWheelerProfile` estimates a roll angle and increases the allowed lateral tolerance during a turn.

---

## 12. Fusion Controller

File: `src/idr/filters/fusion.py`

Class: `GNSSINSFusion`

This class combines the data sources and filter updates into a time-ordered loop.

A typical cycle is:

```text
1. Convert current GPS coordinate to local ENU if needed.
2. Read current IMU measurement.
3. Predict the EKF state from acceleration and yaw rate.
4. Apply stationary corrections if the vehicle is stopped.
5. If GNSS is available:
       update position, heading, and velocity.
6. If GNSS is unavailable:
       apply NHC constraints.
       apply AI velocity if available.
       continue inertial prediction.
7. Save the state to trajectory_history.
```

The output is a trajectory containing estimated position and related state values over time.

### 12.1 Local ENU coordinates

Latitude and longitude are geographic coordinates. Filter equations are easier to use in a local Cartesian coordinate system.

The project uses ENU:

- `E`: East.
- `N`: North.
- `U`: Up.

The first reference GPS point becomes the local origin. Other GPS points are represented as offsets from that origin.

---

## 13. GNSS Blackout Evaluation

### 13.1 Scenario generation

File: `src/idr/eval/scenarios.py`

Class: `OutageScenario`

A scenario stores:

- Drive identifier.
- Start and end indices of a simulated outage.
- IMU input.
- GPS input.
- Ground-truth ENU trajectory.
- Outage length and motion metadata.

`build_scenario_library()` creates multiple outage cases with different lengths and driving conditions. This is better than judging the system using only one hand-picked outage.

### 13.2 Blackout execution

File: `src/idr/eval/blackout.py`

Important functions include:

- `run_trajectory_dead_reckoning()`: runs the fusion loop over a trajectory.
- `simulate_blackout_benchmark()`: compares system configurations.
- `main()`: command-line entry point for the evaluation target.

A blackout is simulated by making GNSS unavailable during a selected interval. The filter must continue from its last trusted state using IMU-based prediction and constraints.

### 13.3 Monte Carlo evaluation

File: `src/idr/eval/monte_carlo.py`

The evaluation compares configurations such as:

- Raw IMU baseline.
- EKF fusion.
- EKF with AI velocity.
- EKF with NHC.
- EKF with map snapping.

`run_raw_imu_baseline()` provides an open-loop acceleration/yaw integration baseline. Other evaluation functions run the fusion system.

### 13.4 Current evaluation caveat

The main blackout simulation function accepts a data directory, but the inspected implementation can generate a synthetic trajectory instead of always loading processed benchmark data. Therefore, users should verify the exact data path and evaluation mode before treating a run as a real IO-VNBD benchmark.

---

## 14. Map Matching

Map matching attempts to keep an estimated trajectory close to roads.

### 14.1 Road graph loading

File: `src/idr/mapmatch/osm_graph.py`

Class: `OSMGraphLoader`

It can:

- Load a cached GraphML road graph.
- Fetch OpenStreetMap data when configured.
- Build a synthetic fallback graph.
- Build a local graph from trajectory waypoints.

The offline design goal is to use a locally cached road graph during inference, avoiding live network calls.

### 14.2 Trajectory snapping

File: `src/idr/mapmatch/hmm_matcher.py`

Class: `HMMMapMatcher`

The matcher extracts road edges and projects estimated positions onto nearby road geometry.

Although the class and project documentation refer to HMM/Viterbi matching, the inspected implementation currently performs nearest-edge geometric snapping. It does not currently implement the full HMM sequence calculation with emission probabilities, transition probabilities, and Viterbi decoding.

This distinction matters when interpreting results: nearest-road projection is simpler than a complete temporal map-matching algorithm.

---

## 15. Navigation Metrics

File: `src/idr/eval/metrics.py`

Class: `NavigationMetrics`

The metrics include:

- Total traveled distance.
- Final position drift.
- Drift percentage.
- Position RMSE.
- CEP50.
- 95th-percentile position error.
- Mean speed.

### 15.1 Final drift

Final drift is the distance between the estimated final position and the ground-truth final position:

$$
\text{final drift} = \left\|p_{estimated,end} - p_{truth,end}\right\|
$$

### 15.2 Drift percentage

The project uses traveled distance as the denominator:

$$
\text{drift percent} =
100 \times \frac{\text{final drift}}{\text{distance traveled}}
$$

The stated acceptance target is less than 10 percent of distance traveled.

### 15.3 RMSE

Root mean square error summarizes the typical position error over all samples:

$$
RMSE = \sqrt{\frac{1}{N}\sum_{i=1}^{N} e_i^2}
$$

where `e_i` is the position error at sample `i`.

### 15.4 CEP50

CEP50 is the radius containing 50 percent of position errors. It is a useful way to describe typical rather than worst-case error.

---

## 16. Plotting and Reports

File: `src/idr/eval/plotting.py`

`generate_evaluation_plots()` creates visual outputs such as:

- Estimated versus true trajectory.
- Drift versus outage distance.
- Speed curves.
- Map overlays.
- Error CDFs.
- Scenario comparisons.
- GNSS reacquisition transition plots.

The generated outputs are normally written under:

```text
reports/
reports/figures/
```

The checked-in `results/` directory contains benchmark summaries and visual galleries intended for review.

One implementation detail is important: some plotting code creates approximate CDF curves from summary statistics rather than rebuilding the curves directly from every raw scenario error. Such plots should be interpreted as summaries, not necessarily as exact empirical distributions.

---

## 17. GNSS Reacquisition Smoothing

File: `src/idr/eval/transition.py`

Class: `ReacquisitionSmoother`

When GNSS returns after a long outage, its position may disagree with the dead-reckoned position. Applying the correction instantly can create a visible jump.

The smoother uses a cosine-bell blend over approximately 1.5 seconds. The weight increases gradually from zero to one:

$$
 w(t) = \frac{1}{2}\left(1 - \cos\left(\frac{\pi t}{T}\right)\right)
$$

This allows the estimate to move smoothly toward the recovered GNSS solution.

The smoother exists as a component, but it is not currently connected into the main `GNSSINSFusion` execution path according to the inspected implementation.

---

## 18. Model Export

### 18.1 Main export path

File: `src/idr/export/convert.py`

The main export command exports:

- `VelocityEstimatorNet`.
- `IMUDenoiseNet`.

It writes ONNX files and validates them with ONNX Runtime.

The usual command is:

```bash
make export
```

Output is placed under:

```text
models/export/
```

### 18.2 Additional export script

File: `scripts/export_onnx.py`

This script has a broader export path for:

- Velocity estimator.
- IMU denoiser.
- Inertial odometry model.
- KalmanNet.

The main Makefile export target and this script therefore do not export exactly the same set of models.

### 18.3 ONNX and TFLite note

The documentation describes ONNX and TFLite/mobile deployment goals. The inspected implementation provides ONNX conversion and validation, but no complete TFLite conversion implementation was found in the repository.

---

## 19. Tests

File: `tests/test_idr.py`

The current tests cover four focused behaviors:

1. `test_imu_denoise_net_forward()` checks the denoiser output shape.
2. `test_velocity_net_forward()` checks the velocity output shape and non-negative speed.
3. `test_ekf_prediction_and_nhc()` checks that prediction changes the state and that NHC does not create non-finite values.
4. `test_navigation_metrics()` checks distance, final drift, and drift percentage on a simple path.

Run them with:

```bash
make test
```

The current tests do not fully cover:

- CSV schema detection.
- Raw-data download and validation.
- Sliding-window preprocessing.
- Complete training.
- Complete fusion blackout transitions.
- Map matching.
- KalmanNet behavior.
- ONNX exports.
- GNSS reacquisition smoothing.
- End-to-end `make data`, `make train`, and `make eval` execution.

---

## 20. Python Package Organization

The package is configured in `pyproject.toml`.

```text
src/idr/
```

is the importable package root. The `src` layout prevents accidental imports from the repository root and encourages installation as a real package.

The package is installed in editable mode by:

```bash
python -m pip install -e .
```

Editable mode means changes in `src/idr` are immediately used without rebuilding and reinstalling the package after every edit.

The pytest configuration adds `src` to the Python path, allowing tests to import modules such as:

```python
from idr.models.velocity_net import VelocityEstimatorNet
```

---

## 21. Dependencies and Why They Exist

The main dependencies in `requirements.txt` are grouped by responsibility:

| Dependency | Role |
|---|---|
| PyTorch | Neural networks and training. |
| NumPy | Numerical arrays and vectorized math. |
| pandas | CSV loading and tabular data processing. |
| SciPy | Scientific and signal-processing utilities. |
| scikit-learn | General machine-learning utilities. |
| FilterPy | Kalman-filter support utilities. |
| OSMnx | OpenStreetMap network loading. |
| Shapely | Geometric operations and road projections. |
| NetworkX | Graph representation and graph algorithms. |
| GeoPandas | Geospatial tabular data. |
| pyproj | Coordinate transformations. |
| Matplotlib | Plots and reports. |
| ONNX | Portable model representation. |
| ONNX Runtime | Running and checking exported ONNX models. |
| tqdm | Progress bars. |
| PyYAML | YAML configuration support. |

The project expects Python 3.11 or newer according to `pyproject.toml`.

---

## 22. End-to-End Example in Plain English

Imagine a car driving into a tunnel.

### Before the tunnel

1. Smartphone IMU data is read continuously.
2. GPS provides an absolute position.
3. The EKF uses IMU data to predict motion.
4. GPS position, speed, and heading updates correct the EKF.
5. The neural velocity model estimates forward speed from recent IMU history.
6. The system records the latest trusted state at the tunnel entrance.

### Inside the tunnel

1. GPS updates are unavailable.
2. IMU data continues arriving.
3. The EKF predicts position from acceleration and turning rate.
4. The AI model provides a speed estimate.
5. NHC tells the filter that lateral and vertical vehicle velocity should be near zero.
6. ZUPT can correct the state if the car stops.
7. The estimated trajectory is stored for later analysis.
8. Optional road snapping keeps the estimate near a road segment.

### After leaving the tunnel

1. GPS becomes available again.
2. The filter compares the recovered GPS position with the dead-reckoned state.
3. The EKF corrects the position and uncertainty.
4. The optional reacquisition smoother can blend the correction gradually instead of creating a jump.
5. Evaluation compares the estimated path with the ground-truth path.

---

## 23. Important Current Implementation Differences

The architecture described by the project documentation is broader than the behavior exercised by every current executable path. The following points should be kept in mind:

- The main `make train` path trains the velocity estimator and denoiser, but not every model present in `src/idr/models`.
- `ResidualDriftNet` exists but is not integrated into the main training or inference pipeline.
- KalmanNet support exists, but the main Monte Carlo evaluator does not currently apply the neural gain estimator in the supplied path.
- The map matcher is currently nearest-edge snapping rather than a complete HMM/Viterbi implementation.
- Reacquisition smoothing exists but is not connected to the main fusion loop.
- The denoising target in the primary trainer can be the same phone signal used as input, so it is not automatically a clean ground-truth denoising task.
- The blackout evaluator may use synthetic trajectories depending on the execution path.
- The odometry trainer has different drive preparation logic from the central `DatasetConfig` split.
- The Makefile mentions TFLite and a Streamlit demo, but complete implementations for those paths are not present in the inspected tree.
- The tests are component-level tests, not a complete system verification suite.

These are not merely stylistic details. They affect how benchmark claims and generated reports should be interpreted.

---

## 24. Recommended Beginner Reading Order

A new contributor can understand the project in this order:

1. Read `README.md` for the project goal and quickstart.
2. Read `DECISIONS.md` for the design reasoning.
3. Read `src/idr/config.py` to learn the constants and dataset split.
4. Read `src/idr/io/schema.py` and `src/idr/io/loader.py` to understand the input data.
5. Read `src/idr/io/preprocess.py` to understand training windows.
6. Read `src/idr/models/velocity_net.py` and `src/idr/models/imu_denoise.py`.
7. Read `src/idr/filters/ekf.py` to understand the state and prediction.
8. Read `src/idr/filters/nhc.py` and `src/idr/filters/zupt.py` to understand physical corrections.
9. Read `src/idr/filters/fusion.py` to see how the pieces are called together.
10. Read `src/idr/eval/blackout.py`, `src/idr/eval/metrics.py`, and `src/idr/eval/scenarios.py` to understand evaluation.
11. Read `src/idr/export/convert.py` to understand deployment packaging.
12. Run `make test` before attempting the full data and training workflow.

---

## 25. One-Sentence Summary

The AI-ML IDR System is a Python pipeline that learns useful motion estimates from smartphone IMU data, combines those estimates with inertial physics and vehicle constraints in a Kalman filter, optionally keeps the trajectory near mapped roads, and measures how much position error accumulates during GNSS outages.
