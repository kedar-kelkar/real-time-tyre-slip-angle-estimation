# Real-Time Tyre Slip Angle Estimation

## Overview

Tyre slip angle is one of the most important quantities governing the lateral performance of a race car. It describes the difference between the direction a wheel is pointing and the direction in which the tyre contact patch is actually travelling.

The objective of this project was to develop a method for estimating the **real-time slip angle at each wheel during cornering** using measured steering geometry and vehicle motion data.

The project was developed as part of a student Formula racing program (Shaurya Racing, VIT Chennai). The work progressed through the design and validation of the steering measurement system and an initial vehicle test on a skidpad. The methodology final stage — obtaining the vehicle velocity vector using an IMU/GPS system and transforming it to each wheel — was developed as the proposed continuation of the project.

---

## Project Objective

The intended measurement chain was:

**Steering rack displacement → Wheel angle → Local wheel velocity → Tyre slip angle**

The steering rack displacement could be measured directly using a linear potentiometer (linpot). Since the vehicle's steering geometry was known, rack travel could then be converted into the corresponding wheel angle for each front wheel.

The remaining challenge was determining the actual velocity vector at each tyre contact patch.

The proposed solution was to combine an **IMU with a GPS receiver**, estimate the vehicle velocity at the centre of gravity, and then use rigid-body velocity transformation to obtain the velocity vector at each wheel.

---

# 1. Steering Geometry

The steering system was designed around a rack-and-pinion arrangement with different steering angles for the inner and outer front wheels.

For a given rack displacement, the steering mechanism produces a corresponding wheel angle. Because the two steering arms do not rotate through identical geometries, the inner and outer wheels naturally develop different steering angles during cornering.

The design geometry was reconstructed in CAD and used to establish the theoretical rack-travel-to-wheel-angle relationship.

![Steering Geometry Overview](media/Steering%20Geometry%20Overview.png)

The steering geometry was also recreated in SolidWorks and is included in the [`CAD`](./CAD/) directory.

The fixed geometric parameters — including wheelbase, track width, scrub radius, steering-arm geometry and tie-rod length — constrain the relationship between rack displacement and wheel angle.

![Theoretical Wheel Angles](media/Angles%20measured.png)

The calculated wheel-angle data is available in the [Wheel Angles Measured](./data/Wheel%20Angles%20measured.xlsx) dataset.

---

# 2. Wheel Angle Validation

Before using rack displacement as a measurement of wheel angle, the theoretical steering relationship had to be validated on the manufactured vehicle.

A laser pointer was mounted to each front wheel hub. The steering rack was moved through a series of known positions and the resulting wheel angles were measured using a physical protractor setup.

![Laser Protractor Setup](media/Laser%20Protractor%20setup.png)

The measured wheel angles were then compared against the angles predicted by the original steering geometry.

## Left Front Wheel

![Left Front Wheel Angle Validation](media/Left%20Front%20Wheel%20Angle%20Validation.png)

The measured left-wheel angle followed the predicted design relationship closely across the tested steering range.

## Right Front Wheel

![Right Front Wheel Angle Validation](media/Right%20Front%20Wheel%20Angle%20Validation.png)

The right-wheel measurements similarly showed close agreement with the design prediction.

The validation established that the manufactured steering system was sufficiently close to the original design geometry for rack displacement to be used as a reliable indicator of wheel angle.

---

# 3. Inner vs Outer Wheel Steering

The validation also demonstrated the expected difference between the inner and outer front wheel angles.

![Wheel Angle Inner vs Outer](media/Wheel%20angle%20innver%20vs%20outer.png)

For the same rack displacement, the inner wheel turns through a larger angle than the outer wheel. This is necessary because the two tyres follow different-radius paths while the vehicle is cornering.

This relationship was incorporated into the proposed real-time measurement system so that a single measured rack position could be converted into the corresponding angle of each front wheel.

---

# 4. Real-Time Steering Measurement

To obtain steering information during vehicle operation, a **linear potentiometer (linpot)** was mounted to the steering rack.

The linpot measures rack displacement directly. Since the rack displacement-to-wheel-angle relationship had already been validated experimentally, the measured rack position could be converted into the instantaneous steering angle of each front wheel.

A CAD model of the mounting arrangement was recreated as part of this project.

![Linear Potentiometer Mounted Setup](media/Linear%20Potentiometer%20Mounted%20Setup.png)

The recreated mounting geometry is included in the [`CAD`](./CAD/) directory.

The intended real-time signal chain was:

**Linear potentiometer → Rack displacement → Validated steering geometry → Left / Right wheel angle**

# 5. Skidpad Testing

The vehicle was subsequently operated on a skidpad course to observe the steering response during representative cornering.

A simplified representation of the test course is shown below.

![Skidpad Course](media/Skidpad%20course%20sketch.png)

During the run, steering measurements were used to obtain the front-wheel angle history.

![Skidpad Run](media/Skidpad%20Run-8.png)

The resulting wheel-angle time history shows the steering transitions associated with the vehicle entering, maintaining and exiting the corner.

This provided the first stage of the intended real-time tyre analysis pipeline: obtaining the actual wheel orientation during vehicle operation.

# 7. Proposed IMU + GPS Motion Estimation

The next stage of the project was to obtain the vehicle's velocity vector without relying on an external motion-tracking system.

A practical solution is to combine an onboard **IMU with a 10 Hz GPS receiver**.

The IMU measures specific force and angular velocity in the vehicle body-fixed frame. After accounting for gravity and transforming the acceleration appropriately, the acceleration can be integrated to obtain velocity.

Conceptually:

**Acceleration → Integration → Velocity → Integration → Position**

However, direct inertial integration accumulates sensor bias and measurement noise. Even small acceleration errors therefore grow with time, causing the estimated velocity and especially position to drift.

GPS provides an independent absolute position and velocity reference. The proposed system therefore uses GPS measurements to periodically correct the inertial solution.

The intended architecture is:

**IMU → High-rate inertial motion estimate**

**GPS → Absolute position / velocity reference**

**IMU + GPS → Sensor-fused vehicle motion estimate**

The original project also considered whether a lower-cost off-the-shelf IMU would provide sufficient accuracy for a race-car environment. Vehicle vibration, sensor noise and sensor bias would be important considerations, and appropriate filtering would be required.

# 8. From Vehicle Velocity to Wheel-Local Velocity

A velocity measurement at the vehicle's centre of gravity (CG) is not directly equal to the velocity experienced by each tyre.

During cornering, the vehicle is undergoing rotational motion as well as translational motion. Therefore, each point on the vehicle has a slightly different instantaneous velocity depending on its position relative to the CG.

The velocity of an arbitrary point on a rigid body is obtained using the transport theorem:

$$
\mathbf{V}_P = \mathbf{V}_{CG} + \boldsymbol{\omega} \times \mathbf{r}_{P/CG}
$$

where:

- $\mathbf{V}_P$ is the velocity of the point of interest
- $\mathbf{V}_{CG}$ is the velocity of the vehicle CG
- $\boldsymbol{\omega}$ is the vehicle angular velocity
- $\mathbf{r}_{P/CG}$ is the position vector from the CG to the point of interest

For the tyre contact patches, this relation is applied separately to each wheel.

Thus:

$$
\mathbf{V}_{FL} = \mathbf{V}_{CG} +
\boldsymbol{\omega} \times \mathbf{r}_{FL/CG}
$$

$$
\mathbf{V}_{FR} = \mathbf{V}_{CG} +
\boldsymbol{\omega} \times \mathbf{r}_{FR/CG}
$$

and similarly for the rear wheels.

This accounts for the fact that the outer and inner wheels experience different local velocity vectors while the vehicle is cornering.

# 9. Determining the Tyre Slip Angle

Once the local velocity vector at each tyre is known, the tyre slip angle can be calculated by comparing that velocity direction with the orientation of the corresponding wheel.

For a wheel with local velocity components $V_x$ and $V_y$ expressed in the wheel coordinate system, the velocity direction is:

$$
\beta = \tan^{-1}\left(\frac{V_y}{V_x}\right)
$$

The tyre slip angle is then obtained from the difference between the wheel heading and the direction of travel.

In general form:

$$
\alpha = \delta - \beta
$$

where:

- $\alpha$ = tyre slip angle
- $\delta$ = wheel steering angle
- $\beta$ = direction of the local velocity vector relative to the vehicle reference axis

The sign convention can be defined according to the vehicle coordinate system and steering direction used in the analysis.

This calculation would be performed independently for each wheel, producing a real-time estimate of:

**Front-left slip angle**

**Front-right slip angle**

**Rear-left slip angle**

**Rear-right slip angle**

The resulting slip-angle history could then be compared with vehicle speed, steering input, lateral acceleration and other vehicle parameters.
