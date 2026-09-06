# Intelligent Dead Reckoning System Architecture

This document explains the processing architecture followed by the AI-ML
Intelligent Dead Reckoning (IDR) project. The system estimates vehicle motion
continuously and limits position drift when GNSS is temporarily unavailable.

The architecture combines smartphone IMU measurements, learned speed
estimation, vehicle-motion constraints, inertial filtering, GNSS correction,
and optional road-network map matching.

## End-to-End Architecture

```text
                    +--------------------------+
                    | Smartphone Sensors       |
                    | Accelerometer            |
                    | Gyroscope                |
                    | Magnetometer (optional)  |
                    | GNSS: GPS / NavIC        |
                    +------------+-------------+
                                 |
                                 v
                    +--------------------------+
                    | Sensor Preprocessing     |
                    | - Time synchronization   |
                    | - Resampling to 10 Hz    |
                    | - Noise and bias cleanup |
                    | - Window generation     |
                    +------------+-------------+
                                 |
                                 v
                    +--------------------------+
                    | Phone-Vehicle Alignment  |
                    | - Gravity / vertical     |
                    | - Forward-motion axis    |
                    | - Phone-to-vehicle frame |
                    +------------+-------------+
                                 |
                                 v
                    +--------------------------+
                    | AI Motion Estimation     |
                    | IMU window -> speed      |
                    | IMU Denoise Net          |
                    | VelocityEstimatorNet     |
                    +------------+-------------+
                                 |
                                 v
                    +--------------------------+
                    | Vehicle Constraints      |
                    | - No lateral slip        |
                    | - No vertical velocity   |
                    | - Stationary ZUPT/ZARU   |
                    +------------+-------------+
                                 |
                                 v
                    +--------------------------+
                    | Dead Reckoning / EKF     |
                    | IMU prediction           |
                    | Speed + heading         |
                    | Position propagation     |
                    +------------+-------------+
                                 |
                                 v
                    +--------------------------+
                    | GNSS Availability Check |
                    +-----------+--------------+
                                |
                    +-----------+-----------+
                    |                       |
                  GNSS                    GNSS
                available                denied
                    |                       |
                    v                       v
             GNSS position,          Pure inertial
             velocity and heading    prediction + NHC
                    \                       /
                     +---------+-----------+
                               |
                               v
                    +--------------------------+
                    | Fused Vehicle State      |
                    | EKF estimate of position |
                    | velocity and heading     |
                    +------------+-------------+
                                 |
                                 v
                    +--------------------------+
                    | OSM Map Matching         |
                    | Road graph / HMM         |
                    | Centerline snapping      |
                    +------------+-------------+
                                 |
                                 v
                    +--------------------------+
                    | Final Navigation Output  |
                    | Position and trajectory   |
                    +--------------------------+
```

## 1. Smartphone Sensor Layer

The input layer collects measurements from the phone and the positioning
receiver:

- **Accelerometer:** measures specific force and provides information about
  forward acceleration, braking, gravity, and vibration.
- **Gyroscope:** measures angular velocity and is especially important for
  estimating turns and heading changes.
- **Magnetometer:** can provide a magnetic heading aid, but it is not part of
  the current six-channel neural-network input. Magnetic readings are often
  affected by nearby metal and electrical equipment.
- **GNSS (GPS/NavIC):** provides absolute position when satellite reception is
  available. GPS and NavIC can be treated through the same GNSS position
  interface.

The current learned motion models use the six IMU channels
`[ax, ay, az, gx, gy, gz]`. Vehicle-side speed is used as training supervision
and is not required as an inference input.

## 2. Sensor Preprocessing Layer

This layer makes measurements consistent before they reach the models or
filters. It performs the following tasks:

- Synchronizes smartphone and vehicle records.
- Resamples data to the nominal 10 Hz operating rate.
- Organizes accelerometer and gyroscope samples into fixed-size windows.
- Separates training inputs from targets such as forward speed.
- Applies the configured data cleaning and normalization steps.

The default training window contains 50 samples, which represents five seconds
at 10 Hz. Overlapping windows allow the neural network to learn temporal
patterns instead of reacting to a single noisy sample.

## 3. Phone-Vehicle Alignment Engine

A phone can be mounted at any roll, pitch, or yaw relative to the vehicle.
Using phone coordinates directly would therefore make forward acceleration and
turning measurements ambiguous.

The alignment engine estimates a rotation from the phone frame to the vehicle
frame:

- **Vertical axis:** estimated from the mean stationary gravity vector.
- **Forward axis:** estimated from the dominant direction of dynamic
  acceleration during vehicle motion.
- **Lateral axis:** constructed to produce an orthonormal right-handed frame.

After calibration, accelerometer and gyroscope vectors are rotated into a
vehicle frame where forward, lateral, and vertical motion have a physical
meaning. This is the project component that corresponds to roll, pitch, yaw,
and vehicle-heading alignment.

## 4. AI Speed and Motion Estimation Module

The learned models extract motion information that is difficult to obtain by
simply integrating noisy phone accelerometer data.

### IMU Denoise Network

`IMUDenoiseNet` is a lightweight dilated 1D convolutional network. It receives
an IMU window and predicts a six-value noise or bias residual. The residual can
be subtracted from the raw window before inertial propagation.

### Forward Velocity Network

`VelocityEstimatorNet` uses temporal convolutions followed by a GRU. It receives
a vehicle-frame IMU window and predicts non-negative forward speed in metres per
second. A stationary-motion gate suppresses false speed caused by engine
vibration, potholes, or phone movement while the vehicle is stopped.

The model is trained using vehicle speed labels. During GNSS outage evaluation,
its speed estimate becomes a measurement that can correct the inertial filter.
The project reports the measured model and system errors in its evaluation
reports; those values should be read from the current results rather than
treated as a fixed architectural constant.

## 5. Vehicle Motion Constraint Module

Ground vehicles have predictable kinematic restrictions. The project expresses
these restrictions as pseudo-measurements for the Kalman filter:

- **Non-Holonomic Constraint (NHC):** lateral body-frame velocity is assumed to
  be approximately zero, so the vehicle does not slide sideways.
- **Vertical constraint:** vertical velocity is assumed to be approximately
  zero because the vehicle remains on the road surface.
- **Zero-Velocity Update (ZUPT):** when the stationary detector identifies a
  stop, vehicle velocity is corrected toward zero.
- **Zero Angular Rate Update (ZARU):** during a detected stop, gyro bias can be
  corrected using the near-zero angular-rate condition.

NHC is particularly useful during GNSS denial because its measurement Jacobian
  couples lateral-velocity error to heading error. This gives the filter some
  heading observability even without an absolute GNSS heading.

## 6. Dead Reckoning Engine

The dead-reckoning engine propagates the vehicle state at each IMU time step.
The EKF predicts motion using forward acceleration and yaw rate, then applies
available corrections from GNSS, AI speed, NHC, ZUPT, and ZARU.

At a simplified kinematic level, position propagation is:

```text
x(k+1) = x(k) + v * dt * cos(theta)
y(k+1) = y(k) + v * dt * sin(theta)
```

In the implementation, position is represented in a local East-North-Up frame.
The filter state also contains velocity, heading, and error-related states so
that uncertainty and sensor drift can be estimated rather than hidden.

## 7. GNSS Availability Detector and Operating Modes

At every update the fusion engine checks whether a valid GNSS position is
available.

### GNSS available

The EKF uses the GNSS position as an absolute correction. When successive GNSS
positions are available, their displacement also provides velocity and course
over ground updates. These corrections prevent inertial error from growing
without bound.

### GNSS denied

The system enters blackout mode. It continues with IMU prediction and applies
NHC plus any available AI velocity estimate. The trajectory is therefore
maintained without requiring a live positioning signal.

The same logic supports simulated blackout scenarios used by the evaluation
pipeline, including tunnels and other temporary GNSS outages.

## 8. AI and Classical Fusion Engine

The fusion engine combines measurements with different strengths:

- IMU provides frequent local motion information but drifts over time.
- AI speed provides a learned forward-motion estimate from an IMU window.
- NHC provides vehicle-physics constraints.
- GNSS provides absolute position and, when moving, course and velocity.
- ZUPT and ZARU reduce drift during detected stops.

The current high-level `GNSSINSFusion` path uses an Extended Kalman Filter
(EKF). A UKF implementation also exists in the filters package for experiments
or future configuration. The neural networks do not replace the filter; they
provide additional measurements or cleaned signals that the filter weights
according to uncertainty.

## 9. Map Matching Layer

The optional map-matching stage uses a cached OpenStreetMap road graph. It
compares the estimated trajectory with nearby road geometries and applies an
HMM-style or graph-based road constraint to reduce implausible drift.

In the current implementation, the matcher projects a point onto the nearest
road centerline when it lies within a configured distance corridor. This is
road-level matching. True lane-level navigation would require lane geometry,
lane connectivity, and additional lane-selection observations, which are not
guaranteed by the current smartphone sensor set.

## 10. Final Output and Evaluation

The final output is a time-ordered estimated trajectory containing position,
velocity, and heading. It can be converted back from local East-North-Up
coordinates to latitude and longitude.

The evaluation pipeline creates GNSS blackout scenarios and compares the
estimated trajectory with reference data using measures such as:

- Final position drift.
- Drift as a percentage of outage distance.
- Position RMSE and CEP.
- Trajectory plots and scenario-level summaries.

This makes it possible to compare raw inertial mechanization, EKF plus AI
velocity and NHC, and the optional OSM map-matched configuration.

## Implementation Status Summary

| Architecture block | Project status |
|---|---|
| Smartphone IMU and GNSS ingestion | Implemented |
| Preprocessing and temporal windows | Implemented |
| Phone-to-vehicle alignment | Implemented |
| IMU denoising network | Implemented |
| AI forward-speed estimation | Implemented |
| EKF inertial fusion | Implemented |
| NHC, ZUPT, and ZARU updates | Implemented |
| GNSS blackout evaluation | Implemented |
| OSM road-level map matching | Implemented / optional |
| Magnetometer-based heading | Not a current neural input |
| Guaranteed lane-level navigation | Future extension |