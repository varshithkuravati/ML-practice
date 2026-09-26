# Real-Time Running Pipeline

This document describes how the AI-ML Intelligent Dead Reckoning (IDR)
system runs continuously at the nominal 10 Hz sensor rate. It is the runtime
view of the system, separate from the high-level architecture in
`ARCHITECTURE_PIPELINE.md`.

## Runtime Overview

```text
+--------------------------+
| Phone Sensor Adapter     |
|--------------------------|
| - Accelerometer          |
| - Gyroscope              |
| - GNSS                   |
| - Timestamp              |
+------------+-------------+
             |
             v
+--------------------------+
| Sample Synchronization   |
|--------------------------|
| - Match sensor times     |
| - Resample to 10 Hz      |
| - Validate measurements  |
+------------+-------------+
             |
             v
+--------------------------+
| Phone-Vehicle Alignment  |
|--------------------------|
| - Rotate IMU to vehicle  |
| - Select forward axis    |
| - Remove bias / noise    |
+------------+-------------+
             |
             v
+--------------------------+
| Rolling IMU Window       |
|--------------------------|
| - Keep latest 50 samples |
| - Build [ax..gz] input   |
| - Wait until window full |
+------------+-------------+
             |
             v
+--------------------------+
| AI Motion Inference      |
|--------------------------|
| - Denoise IMU window     |
| - Estimate forward speed |
| - Return latest result   |
+------------+-------------+
             |
             v
+--------------------------+
| GNSSINSFusion.step       |
|--------------------------|
| - EKF prediction         |
| - ZUPT / ZARU            |
| - GNSS update if valid   |
| - NHC during blackout    |
| - AI velocity update     |
+------------+-------------+
             |
             v
+--------------------------+
| Navigation Output        |
|--------------------------|
| - Position ENU           |
| - Latitude / longitude   |
| - Velocity               |
| - Heading                |
| - GNSS / blackout state  |
+--------------------------+
```

## One 10 Hz Update

The runtime loop repeats the following sequence every `0.1` seconds:

1. Read the newest accelerometer, gyroscope, and GNSS records.
2. Synchronize the records by timestamp and convert units to the project
   conventions.
3. Rotate the raw IMU vector from the phone frame into the vehicle frame.
4. Add the aligned six-channel vector `[ax, ay, az, gx, gy, gz]` to the rolling
   window.
5. Run the denoising and forward-velocity models when the window is ready.
6. Extract forward acceleration and yaw rate for the inertial filter.
7. Call `GNSSINSFusion.step(...)` once.
8. Convert the returned local ENU state to latitude and longitude.
9. Publish the navigation state to the application, logger, or display.

The filter state is retained between calls. The loop must not recreate
`GNSSINSFusion` for each sample.

## Runtime Decision Flow

```text

+--------------------------+
| New synchronized sample  |
+------------+-------------+
             |
             v
+--------------------------+
| EKF prediction           |
| fwd_accel + yaw_rate     |
+------------+-------------+
             |
             v
+--------------------------+
| Stationary detector      |
+------------+-------------+
       yes  |  no
            | \
            |  +----------------------+
            |                         |
            v                         v
+------------------+       +--------------------------+
| Apply ZUPT/ZARU  |       | Check GNSS availability |
+--------+---------+       +------------+-------------+
         |                              |
         +---------------+--------------+
                         |
              +----------+----------+
              |                     |
              v                     v
+--------------------------+  +--------------------------+
| GNSS valid               |  | GNSS denied / unavailable|
| - Position update        |  | - Set blackout state     |
| - COG heading update     |  | - Apply NHC              |
| - GNSS velocity update   |  | - Apply AI velocity      |
+------------+-------------+  +------------+-------------+
             |                             |
             +--------------+--------------+
                            v
                +--------------------------+
                | Publish current EKF state|
                +--------------------------+
```

## Python Runtime Skeleton

The following skeleton shows how an application should drive the existing
fusion engine. The sensor adapter and model runner are application-specific;
they supply the values marked with comments.

```python
from collections import deque

import numpy as np

from idr.filters.fusion import GNSSINSFusion

SAMPLE_RATE_HZ = 10.0
DT = 1.0 / SAMPLE_RATE_HZ
WINDOW_SIZE = 50


def run_realtime(sensor_stream, model_runner, ref_lat, ref_lon):
    fusion = GNSSINSFusion(ref_lat=ref_lat, ref_lon=ref_lon, dt=DT)
    imu_window = deque(maxlen=WINDOW_SIZE)

    for sample in sensor_stream:
        # The adapter returns vehicle-frame values in project units.
        acc_vehicle = sample.acc_vehicle  # [ax, ay, az] in m/s^2
        gyro_vehicle = sample.gyro_vehicle  # [gx, gy, gz] in rad/s
        gnss_pos = sample.gnss_pos  # (lat, lon), or None

        imu_window.append(np.concatenate((acc_vehicle, gyro_vehicle)))

        ai_velocity = None
        if len(imu_window) == WINDOW_SIZE:
            ai_velocity = model_runner.predict(np.asarray(imu_window))

        state = fusion.step(
            fwd_accel=float(acc_vehicle[0]),
            yaw_rate=float(gyro_vehicle[2]),
            gnss_pos=gnss_pos,
            ai_velocity=ai_velocity,
            is_gnss_denied=gnss_pos is None,
            use_nhc=True,
            acc_3d=acc_vehicle,
            gyro_3d=gyro_vehicle,
        )

        latitude, longitude = fusion.enu_to_latlon(
            east=float(state[0]),
            north=float(state[1]),
        )

        yield {
            "timestamp": sample.timestamp,
            "latitude": latitude,
            "longitude": longitude,
            "velocity_east_mps": float(state[3]),
            "velocity_north_mps": float(state[4]),
            "velocity_up_mps": float(state[5]),
            "heading_rad": float(state[6]),
            "gnss_available": gnss_pos is not None,
            "in_blackout": fusion.in_blackout,
        }
```

## State and Measurement Contract

| Input | Runtime value | Required handling |
|---|---|---|
| Accelerometer | `acc_vehicle[3]` | Vehicle-frame m/s2; used for prediction and stationarity |
| Gyroscope | `gyro_vehicle[3]` | Vehicle-frame rad/s; Z component is yaw rate |
| IMU window | `50 x 6` | Latest five seconds at 10 Hz for model inference |
| GNSS | `(latitude, longitude)` or `None` | `None` starts or continues blackout handling |
| AI velocity | Scalar forward speed or 2D velocity | Used as a measurement during GNSS denial |
| Timestamp | Monotonic sample time | Used by the adapter to preserve 10 Hz timing |

`GNSSINSFusion.step` returns the EKF state vector. The current state layout is:

```text
state[0:3]  position in local East-North-Up metres
state[3:6]  velocity in local East-North-Up metres/second
state[6]    heading / yaw in radians
state[7:]   EKF error and sensor-bias states
```

## GNSS and Blackout Behavior

When GNSS is valid, the fusion engine:

- Converts latitude and longitude to local ENU coordinates.
- Corrects EKF position.
- Uses successive GNSS positions for course-over-ground heading.
- Derives a GNSS velocity update when movement exceeds the threshold.

When GNSS is missing or explicitly denied, the fusion engine:

- Keeps propagating position from the IMU prediction.
- Applies the non-holonomic constraint when enabled.
- Applies the AI forward-velocity measurement when available.
- Sets `fusion.in_blackout` to `True`.

The first 50 samples fill the model window. During this warm-up period the EKF
can still run, but `ai_velocity` remains `None` unless the application uses a
shorter startup window.

## Timing Requirements

At the 10 Hz target:

- Each completed loop should finish within 100 ms.
- Model inference, filtering, and output publication should be measured
  separately.
- Sensor timestamps should be monotonic; processing time must not be used as
  the measurement timestamp.
- If one iteration is late, process the newest sample and record the dropped
  or delayed sample rather than silently reordering measurements.
- Model inference should run in evaluation mode and avoid gradient tracking.

The existing latency profiler is available at
`src/idr/eval/profiler.py`. ONNX exports in `models/export/` can be used by a
mobile adapter when PyTorch is too heavy for the target device.

## Current Project Boundary

The repository currently provides the fusion engine, calibration utilities,
models, evaluation scripts, and export artifacts. It does not yet include a
platform-specific Android sensor adapter or a production live sensor-stream
process. To run this pipeline on a phone, implement the adapter that maps
Android sensor callbacks and location updates into the runtime contract above.
The rest of the filter loop can remain centered on `GNSSINSFusion.step`.

## Related Files

- `src/idr/filters/fusion.py`: single-step GNSS/INS fusion runtime.
- `src/idr/filters/ekf.py`: inertial prediction and EKF measurement updates.
- `src/idr/calib/alignment.py`: phone-to-vehicle frame alignment.
- `src/idr/models/imu_denoise.py`: IMU denoising model.
- `src/idr/models/velocity_net.py`: forward-velocity model.
- `src/idr/eval/profiler.py`: real-time latency and throughput profiling.
- `ARCHITECTURE_PIPELINE.md`: static end-to-end architecture view.
