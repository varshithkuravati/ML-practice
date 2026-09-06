# EKF, GNSS Availability, and Fusion: Detailed Working

This document explains what happens **after the Vehicle Motion Constraint
layer** in the Intelligent Dead Reckoning (IDR) system.

The central idea is simple:

> The system predicts motion continuously with the IMU, then corrects that
> prediction with whichever trustworthy measurements are currently available.

The correction source changes when GNSS is lost. With GNSS, the system is
anchored to an absolute position. Without GNSS, it uses AI speed estimates and
vehicle physics to slow down drift until GNSS returns.

## 1. Why Do We Need an EKF?

An IMU produces measurements frequently, but it does not directly report the
vehicle's position. To obtain position, acceleration must be integrated into
velocity and velocity must be integrated into position.

That creates an error problem:

```text
small acceleration bias
        -> velocity error
        -> position error
        -> increasing dead-reckoning drift
```

GNSS has the opposite behavior. It gives an absolute position, but its updates
are less smooth and may disappear in tunnels, underground roads, or urban
canyons.

The system therefore needs a method that can:

- combine measurements with different noise levels;
- predict between GNSS updates;
- estimate uncertain quantities such as sensor bias;
- continue operating when one measurement source disappears; and
- decide how strongly each new measurement should influence the result.

An **Extended Kalman Filter (EKF)** performs this job. It is the mathematical
state estimator at the center of the GNSS-INS fusion engine.

The EKF is not an AI model. It is a probabilistic estimator that combines a
motion model with measurements. The neural network supplies an additional
speed measurement; the EKF decides how that measurement should modify the
vehicle state.

## 2. What Is a Kalman Filter?

A Kalman filter maintains two things:

1. **Best current estimate:** what the system believes the vehicle state is.
2. **Uncertainty:** how confident the system is in that estimate.

Every sensor update follows two stages:

```text
PREDICT: use the motion model to forecast the next state
    |
    v
UPDATE: compare the forecast with a measurement and correct it
```

For example, suppose the predicted position is 100 m and GNSS reports 103 m.
The EKF does not blindly replace 100 with 103. It considers uncertainty:

- If the IMU prediction is very uncertain and GNSS is reliable, it moves close
  to 103 m.
- If the GNSS measurement is noisy and the prediction is trusted, it makes a
  smaller correction.
- It also updates related quantities such as velocity and sensor bias through
  the covariance relationships.

This is why the EKF is better than simply averaging GPS and IMU values.

## 3. Why Is It Called an Extended Kalman Filter?

A basic Kalman filter requires linear equations. Vehicle motion is nonlinear:

- heading is an angle;
- forward acceleration must be rotated into East and North directions;
- turning creates centripetal acceleration;
- speed and heading interact.

The EKF handles these nonlinear equations by locally approximating them with a
linear Jacobian matrix. In this project, the Jacobian describes how a small
change in position, velocity, heading, or bias changes the next predicted
state.

The EKF is appropriate here because the vehicle normally moves smoothly and
the nonlinearities are moderate over the short 0.1-second update interval.

## 4. The EKF State in This Project

The implementation stores a nine-value state vector in a local East-North-Up
(ENU) coordinate frame:

```text
x = [E, N, U, vE, vN, vU, psi, ba, bw]
```

| State | Meaning |
|---|---|
| `E` | East position in metres |
| `N` | North position in metres |
| `U` | Up position in metres |
| `vE` | East velocity in m/s |
| `vN` | North velocity in m/s |
| `vU` | Up velocity in m/s |
| `psi` | Vehicle yaw or heading in radians |
| `ba` | Estimated forward accelerometer bias |
| `bw` | Estimated gyroscope bias |

The filter also stores `P`, its covariance matrix. `P` contains uncertainty
for each state and relationships between states. For example, if heading
uncertainty affects predicted velocity, the covariance records that link.

The filter has a process-noise matrix `Q` for uncertainty introduced by noisy
phone sensors and imperfect motion modeling. Each measurement has its own
measurement-noise matrix, usually called `R` in the update functions.

## 5. EKF Prediction: What Happens at Every IMU Step?

The project runs at a nominal 10 Hz rate, so `dt` is approximately 0.1 seconds.
At each step the fusion engine first calls the EKF prediction stage.

### 5.1 Correct the raw inputs

The filter removes its current bias estimates:

```text
corrected acceleration = measured acceleration - ba
corrected yaw rate     = measured yaw rate - bw
```

The bias estimates are part of the state, so they can later be corrected by
GNSS, AI velocity, or other measurements.

### 5.2 Rotate forward acceleration into the world frame

The input acceleration is expressed along the vehicle's forward direction. The
EKF uses the current heading `psi` to split it into East and North components:

```text
aE = acceleration * cos(psi) + turning contribution
aN = acceleration * sin(psi) + turning contribution
```

The turning contribution accounts for the fact that a vehicle's velocity
direction changes while it turns.

### 5.3 Propagate position and velocity

The filter integrates the motion for one time step:

```text
E(k+1)  = E(k)  + vE(k) * dt + 0.5 * aE * dt^2
N(k+1)  = N(k)  + vN(k) * dt + 0.5 * aN * dt^2
vE(k+1) = vE(k) + aE * dt
vN(k+1) = vN(k) + aN * dt
```

The project applies the ground-vehicle assumption `vU = 0`, so vertical motion
does not accumulate as if the car were flying. Heading is advanced using the
corrected yaw rate:

```text
psi(k+1) = psi(k) + corrected yaw rate * dt
```

The heading is wrapped to the range `[-pi, pi]` so it cannot grow without
bound.

### 5.4 Propagate uncertainty

The EKF also predicts how uncertainty changes during motion:

```text
P(next) = F * P(current) * F^T + Q
```

`F` is the local motion Jacobian. `Q` adds process noise. During a long GNSS
outage, `P` generally grows because the system is predicting without an
absolute position measurement. That growing uncertainty causes later
measurements to receive more influence.

## 6. Measurement Updates: How Does the EKF Correct Itself?

When a measurement arrives, the EKF compares it with what the current state
predicts:

```text
innovation = actual measurement - predicted measurement
```

It then computes a Kalman gain. Conceptually:

```text
large measurement uncertainty -> small correction
small measurement uncertainty  -> large correction
large filter uncertainty        -> measurement gets more influence
```

The update changes both the state `x` and uncertainty `P`. This lets a position
measurement indirectly improve velocity, heading, and bias estimates when the
covariance matrix shows those quantities are related.

## 7. What Happens When GNSS Is Available?

At each fusion step, the engine checks two conditions:

```text
GNSS is usable when:
    is_gnss_denied is false
    and gnss_pos is not None
```

When both are true, the GNSS latitude and longitude are converted into local
ENU metres around the initial reference point. Using local metres makes the
filter equations easier to compute than directly using latitude and longitude.

### 7.1 GNSS position update

The EKF compares the GNSS ENU position with its predicted ENU position:

```text
GNSS position -> position innovation -> EKF position correction
```

This prevents the position estimate from drifting indefinitely.

### 7.2 GNSS velocity and heading update

The fusion engine stores the previous GNSS ENU position. When the vehicle has
moved more than a small threshold, it calculates:

```text
delta position = current GNSS position - previous GNSS position
GNSS velocity  = delta position / dt
course         = atan2(delta North, delta East)
```

The velocity update corrects `vE`, `vN`, and `vU`. The course update corrects
heading. These updates also help the EKF learn accelerometer and gyro biases
through their covariance cross-terms.

### 7.3 Normal operating flow

```text
IMU sample
   |
   v
EKF predict
   |
   v
GNSS position available?
   |
  YES
   |
   +--> GNSS position update
   +--> GNSS velocity update, if moving
   +--> GNSS course/heading update, if moving
   |
   v
Corrected vehicle state
```

## 8. What Happens When GNSS Is Unavailable?

GNSS denial occurs when either the scenario marks the sample as denied or
there is no GNSS position supplied. The fusion engine then follows this branch:

```text
IMU sample
   |
   v
EKF predict
   |
   v
GNSS unavailable
   |
   +--> Do not apply GNSS position update
   +--> Do not apply GNSS velocity update
   +--> Do not apply GNSS heading update
   +--> Apply NHC pseudo-measurement
   +--> Apply AI velocity measurement, if available
   |
   v
Dead-reckoned corrected state
```

The important point is that the system does **not** stop. It keeps producing a
position every 0.1 seconds. The estimate is now relative dead reckoning rather
than GNSS-anchored navigation.

### 8.1 NHC correction during blackout

The vehicle constraint module supplies two pseudo-measurements:

```text
lateral body-frame velocity = 0
vertical velocity            = 0
```

The EKF compares these expected zeros with its current state. If it believes
the vehicle is sliding sideways, it corrects velocity and can also correct
heading because the lateral velocity depends on heading.

This does not tell the filter the vehicle's absolute location. It only removes
physically implausible motion and slows drift.

### 8.2 AI speed correction during blackout

The velocity network estimates forward speed from the recent IMU window. The
fusion engine uses that value as a measurement of current speed.

Conceptually:

```text
AI says: estimated speed is 15 m/s
EKF says: predicted horizontal speed is 13 m/s
result: move the velocity estimate toward 15 m/s
```

The filter does not blindly replace its velocity with the neural-network
output. The configured speed uncertainty controls the strength of the update.

Together, AI speed and NHC constrain the motion while GNSS is absent:

```text
AI speed -> how fast the vehicle is moving
NHC      -> which directions it should not move
gyro     -> how its heading is changing
IMU      -> short-term acceleration and rotation
```

None of these alone gives an absolute position, which is why some drift remains
during a blackout.

## 9. Stationary Detection, ZUPT, and ZARU

Before the GNSS branch is selected, the fusion engine can check whether the
vehicle appears stationary using three-axis acceleration and gyroscope data.

If a stop is detected:

- **ZUPT** applies a near-zero velocity measurement.
- **ZARU** applies a near-zero angular-rate condition and helps correct gyro
  bias.

These updates are useful at traffic lights, parking stops, or the beginning and
end of an outage. They prevent the filter from integrating vibration and bias
as real vehicle motion.

## 10. The Complete Fusion Cycle

This is the complete order used by `GNSSINSFusion.step()`:

```text
1. Receive vehicle-frame forward acceleration and yaw rate.
2. EKF predict using IMU and the current bias estimates.
3. Check for stationary motion.
4. If stationary, apply ZUPT and ZARU.
5. Check GNSS availability.
6. If GNSS is available:
      a. Convert latitude/longitude to ENU.
      b. Update position with GNSS.
      c. If moving, calculate GNSS velocity and course.
      d. Update velocity and heading.
7. If GNSS is unavailable:
      a. Mark the system as being in blackout mode.
      b. Apply NHC if enabled.
      c. Apply AI forward-speed update if available.
8. Save and return the corrected nine-state estimate.
```

The returned state contains the current estimated position, velocity, heading,
and bias values. The fusion engine also stores each state in its trajectory
history for later plotting and evaluation.

## 11. GNSS Reacquisition: What Happens When Signal Returns?

When a GNSS fix becomes available after a blackout, the EKF immediately accepts
the GNSS position update and begins moving the state back toward the absolute
reference. This corrects accumulated drift.

If the dead-reckoned position is far from the first reacquired GNSS fix, a raw
replacement can create a visible position jump. The repository also contains a
`ReacquisitionSmoother` that can blend the dead-reckoned position toward the
GNSS position over a configurable duration, such as 1.5 seconds.

The important implementation detail is that this smoother is a separate helper;
the current `GNSSINSFusion.step()` method does not automatically call it. An
application integrating this helper must trigger it when the first valid GNSS
fix after blackout arrives.

Conceptually, the reacquisition sequence is:

```text
blackout estimate:       100 m
first GNSS measurement:  112 m

without smoothing:       output can jump toward 112 m
with smoothing:          100 -> 103 -> 107 -> 112 m over the blend period
```

The EKF correction still happens; smoothing controls how the displayed or
exported trajectory transitions to the corrected position.

## 12. Where Map Matching Fits

Map matching is normally applied after the trajectory has been estimated. It is
not a replacement for the EKF.

```text
EKF fused trajectory
        |
        v
OSM road graph
        |
        v
nearest plausible road centerline
```

The current matcher projects a point onto a nearby road centerline when it is
inside the configured distance corridor. This can reduce visible road-level
drift, but it cannot correct every error and it does not guarantee lane-level
positioning.

## 13. A Simple Tunnel Example

Imagine a car entering a 500 m tunnel.

### Before entering the tunnel

GNSS is available. The EKF repeatedly receives:

```text
IMU prediction + GNSS position + GNSS velocity + GNSS course
```

The position, velocity, heading, and bias estimates are well anchored.

### Inside the tunnel

GNSS becomes unavailable. The EKF continues with:

```text
IMU prediction + NHC + AI speed + ZUPT/ZARU when stationary
```

The position is now dead reckoned. The covariance grows, but NHC and AI speed
prevent the estimate from behaving like unconstrained raw integration.

### Leaving the tunnel

GNSS returns. The EKF uses the new position, then velocity and course once
successive fixes are available. The accumulated drift is corrected. An optional
reacquisition smoother can make the visible transition gradual.

## 14. Short Summary

```text
EKF prediction:
    What should happen according to IMU and vehicle motion?

EKF update:
    How should that prediction change according to available evidence?

GNSS available:
    Correct absolute position, velocity, and heading.

GNSS unavailable:
    Continue prediction and use AI speed plus vehicle constraints.

GNSS returns:
    Correct accumulated drift and optionally smooth the transition.
```

The purpose of the complete fusion block is therefore not to choose one sensor
as the answer. It is to maintain the best estimate possible from all current
evidence, while changing gracefully between GNSS-anchored navigation and
inertial dead reckoning.