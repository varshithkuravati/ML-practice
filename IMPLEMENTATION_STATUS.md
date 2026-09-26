# Implementation and Pipeline Status

This document records what is implemented, what is connected to the active pipeline and evaluation, and what is currently represented by fixed assumptions. It is based on the executable call paths in `src/idr`, not only on architecture descriptions or report labels.

## Status Summary

| Component | Code exists | Connected to active pipeline | Used by current evaluation | Status |
|---|---:|---:|---:|---|
| EKF prediction and covariance propagation | Yes | Yes | Yes | Active core filter |
| GNSS position, heading, and velocity updates | Yes | Yes | Yes when GNSS is available | Active |
| AI velocity estimator | Yes | Yes in `eval/blackout.py` | Yes in the blackout evaluator | Active for selected evaluation branches |
| NHC update | Yes | Yes during GNSS denial | Yes | Active |
| ZUPT/ZARU | Yes | Yes when 3-axis IMU inputs are supplied | Not exercised by the main blackout evaluator | Conditional |
| HMM/OSM map matching | Yes | Yes in map-matching evaluation branches | Yes | Optional post-filter stage |
| Allan-variance-derived noise tuning | No estimator | No | No | Manual fixed-value tuning only |
| KalmanNet gain estimator | Yes | No | No, despite an evaluation label | Disconnected |
| KalmanNet classical fallback | Yes | No | No | Implemented but unreachable in normal evaluation |
| UKF | Yes | No | No | Standalone unused filter |
| IMU denoising network | Yes | No | No | Trained/exported, not part of fusion input path |
| `FilterConfig` noise settings | Yes | No | No | Configuration scaffold; EKF uses local constants |

## 1. Code Connected to the Active Pipeline

### EKF and fusion orchestration

`GNSSINSFusion` constructs `ExtendedKalmanFilter` in `src/idr/filters/fusion.py`. Each `step()` call performs:

1. EKF prediction from forward acceleration and yaw rate.
2. Optional stationary detection, followed by ZUPT and ZARU.
3. GNSS position, heading, and velocity updates when GNSS is available.
4. NHC and AI-velocity updates during GNSS denial.
5. Return of the EKF state.

Relevant files:

- `src/idr/filters/fusion.py`
- `src/idr/filters/ekf.py`
- `src/idr/filters/nhc.py`
- `src/idr/filters/zupt.py`

### AI velocity estimator

`eval/blackout.py` loads `VelocityEstimatorNet`, computes sliding-window velocity estimates, and passes them into `GNSSINSFusion.step()` as `ai_velocity`. This is a real connected path in the single blackout evaluation flow.

The Monte Carlo evaluator also passes a velocity estimate into the EKF during blackout. In that path, the estimate is currently derived from the EKF state magnitude rather than from a loaded velocity model.

### NHC, ZUPT, and ZARU

NHC is directly called by `GNSSINSFusion` during GNSS denial. ZUPT and ZARU are directly called when both `acc_3d` and `gyro_3d` are provided and the stationary detector reports a stop.

Therefore, NHC is active in the main blackout path, while ZUPT/ZARU are implemented but input-dependent.

### Map matching

The HMM matcher is used after the EKF trajectory is generated in the map-matching evaluation branches. It is a post-processing correction and does not feed corrections back into the EKF covariance or state during the blackout loop.

## 2. Code Implemented but Not Connected

### KalmanNet runtime integration

`src/idr/filters/kalmannet.py` contains:

- `KalmanNetGainEstimator`, a GRU-based learned gain model.
- `KalmanNetFilter.compute_gain()`, which predicts a learned gain and applies safeguards.
- Classical-gain fallback behavior.

`src/idr/models/train_kalmannet.py` generates training data and saves `kalmannet.pt`. The Monte Carlo evaluator also loads the checkpoint in `src/idr/eval/monte_carlo.py`.

However, the loaded `kalman_filter` is never passed into the actual fusion update path, and `use_kalmannet` is not used inside `run_blackout_branch()`. The branch labelled `KalmanNet Neural Fusion + NHC` currently calls the same `GNSSINSFusion` EKF path as the ordinary EKF branches.

Consequences:

- The reported KalmanNet result is currently an EKF result.
- `KalmanNetFilter.compute_gain()` is not called by the repository's evaluation path.
- The fallback is not exercised by the evaluation.
- No comparison between learned-gain fusion and classical EKF fusion is currently valid.

### UKF

`src/idr/filters/ukf.py` implements an `UnscentedKalmanFilter` and it is exported from `filters/__init__.py`. The active `GNSSINSFusion` class does not accept a filter object or filter type; it constructs an EKF directly. Therefore, the UKF is available as a standalone class but is not part of the active runtime or evaluation pipeline.

### IMU denoising network

`IMUDenoiseNet` is implemented, trained, and exported to ONNX. The current fusion path does not load its checkpoint, apply its residual to IMU data, or pass denoised IMU data into the EKF. It is therefore not part of the evaluated fused trajectory.

## 3. Not Coded as Data-Driven Logic: Fixed Assumptions and Constants

### Allan variance

There is no Allan variance or Allan deviation calculator in the repository. No stationary IMU recording is analyzed to estimate angle random walk, velocity random walk, bias instability, or bias random walk.

The EKF contains manually selected values described as smartphone-MEMS/Allan-variance tuning:

```python
self.P[8, 8] = 0.02**2
self.Q[7, 7] = (5e-3)**2
self.Q[8, 8] = (1e-4)**2
```

These values are fixed at EKF construction time. They should be described as Allan-variance-inspired or manually calibrated values, not as runtime Allan-variance-derived tuning.

### EKF noise and initial covariance

The following values are fixed in `ExtendedKalmanFilter`:

- Position, velocity, and heading initial covariance.
- Accelerometer and gyro bias initial covariance.
- Position, velocity, heading, and bias process-noise covariance.

`FilterConfig` defines `accel_noise` and `gyro_noise`, but `ExtendedKalmanFilter` does not read those fields. Changing those configuration values currently does not change the EKF behavior.

### Measurement noise and vehicle assumptions

The following are also fixed in code unless a caller overrides them:

- GNSS position covariance.
- GNSS velocity covariance.
- GNSS heading measurement noise.
- AI velocity measurement noise.
- NHC lateral and vertical noise, normally `0.05`.
- ZUPT velocity noise, normally `0.01`.
- ZARU gyro-bias measurement noise, normally `0.005`.
- Stationary detector window and acceleration variance threshold.
- Nominal sample interval, normally `0.1` seconds.
- Planar vehicle assumption: vertical velocity is forced to zero.
- NHC assumption: lateral and vertical body-frame velocity are approximately zero.

These assumptions make the pipeline runnable and reproducible, but they are not learned or automatically calibrated from each sensor/device.

## 4. What Changes When the Disconnected Code Is Connected

### Connecting KalmanNet

The fusion layer would need an explicit KalmanNet mode or filter adapter. For each supported measurement update, it would need to:

1. Compute the normal innovation, measurement Jacobian, and classical Riccati gain.
2. Call `KalmanNetFilter.compute_gain(innovation, state_pred, fallback_K)`.
3. Apply the returned gain to update the state and covariance consistently.
4. Reset the GRU hidden state at the start of each drive or evaluation scenario.
5. Record whether the learned gain or fallback gain was used.

Expected behavior changes:

- The neural branch could produce a different trajectory and different drift metrics.
- Valid learned gains would be blended with the classical gain using the current 80/20 rule.
- Invalid, extreme, NaN, Inf, or exception-producing neural outputs would use the classical gain.
- The evaluation label would become truthful only after the KalmanNet branch actually uses those gains.
- Results would need to be regenerated; existing KalmanNet-labelled results must not be treated as neural-fusion results.

A design decision is also required: KalmanNet currently predicts a `9 x 3` gain for a 3D measurement, while the fusion engine has position, heading, velocity, NHC, and AI-velocity updates with different measurement dimensions. The integration must define whether KalmanNet handles only GNSS position updates or separate gain heads/adapters are added for each measurement type.

### Connecting the UKF

`GNSSINSFusion` would need dependency injection or a filter-selection parameter instead of always constructing `ExtendedKalmanFilter`. The UKF also needs measurement-update methods matching the fusion engine's GNSS, NHC, and AI-velocity interfaces.

Expected behavior changes:

- Nonlinear state propagation would use sigma points instead of the EKF Jacobian.
- Covariance evolution and measurement corrections could differ from EKF results.
- The current generic UKF `Q` would need sensor-specific tuning before making a fair comparison.

### Connecting the IMU denoising network

The runtime path would load the denoising checkpoint, infer a residual for each IMU window, subtract or otherwise apply that residual according to the trained target definition, and then pass the corrected IMU values to the fusion engine.

Expected behavior changes:

- The EKF input acceleration and yaw rate could change before prediction.
- Drift and bias convergence could improve or degrade depending on checkpoint quality and scaling compatibility.
- Evaluation would need a separate `raw IMU` versus `denoised IMU` comparison.

### Implementing actual Allan-variance calibration

A calibration module would need to accept stationary IMU data, compute Allan deviation over averaging times, fit the relevant noise coefficients, and convert those coefficients into discrete-time `Q` and initial `P` values. The EKF should then receive those values through its constructor or a typed configuration object.

Expected behavior changes:

- Noise values could vary by phone, sensor, sampling rate, and mounting condition.
- `Q` and `P` would no longer be silently tied to smartphone constants.
- Evaluation would need to record the calibration dataset and parameters used for reproducibility.

## 5. Interpretation of Existing Results

The current EKF, AI-velocity, NHC, and optional map-matching results are meaningful for the code paths that are actually executed, subject to their fixed assumptions.

The following claims should not currently be presented as measured runtime results:

- KalmanNet neural-fusion performance.
- KalmanNet fallback frequency or fallback robustness.
- UKF performance in the main pipeline.
- Runtime benefit of the IMU denoising network.
- Automatically Allan-variance-derived process-noise performance.

The evaluation report should label these as implemented components, planned integrations, or unmeasured capabilities until their runtime paths are connected and rerun.
