+++
title = "Lab 10: Localization (sim)"
date = "2025-04-28"
weight = 6
+++
## Introduction

Our robot lacks direct access to its absolute or "ground truth" location in the world. Therefore, it must infer their position using available sensor data - a process known as **localization**. Rather than relying on a single guess, probabilistic localization methods allow the robot to maintain and continuously update a **belief distribution** over all possible locations. In this lab, I implement **Grid Localization** using a **Bayes Filter**, a probabilistic algorithm that enables the robot to refine its belief as it receives motion and sensor information.

The robot’s state is defined as a 3-dimensional tuple (x, y, theta), representing its position (x, y) and orientation (theta). The environment is a continuous space bounded by:

- -1.6764 meters ≤ x < 1.9812 meters (approximately -5.5 feet ≤ x < 6.5 feet),
- -1.3716 meters ≤ y < 1.3716 meters (approximately -4.5 feet ≤ y < 4.5 feet),
- -180 degrees ≤ theta < 180 degrees.

Since the space is continuous with infinitely many possible poses, I discretize it into a **3D grid** with fixed cell sizes of:

- 0.3048 meters along the x-axis,
- 0.3048 meters along the y-axis,
- 20 degrees increments along the theta-axis.

The grid consists of 12 x 9 x 18 cells, each storing the probability that the robot is in that specific cell. The **belief** is the collection of these probabilities, and it must sum to 1 at all times. At any given moment, the grid cell with the highest probability represents the robot’s most probable pose.

# Bayes Filter

The **Bayes Filter** updates the belief distribution iteratively through two key steps:

1. **Prediction Step**  
   The robot uses its control input (u_t) to predict its next pose, incorporating motion uncertainty.
2. **Update Step**  
   The robot uses its sensor observation (z_t) to refine this prediction, correcting errors introduced during motion.

Formally, the Bayes Filter algorithm can be summarized as:

bel(x_t) = eta * p(z_t | x_t) * sum over x_(t-1) of (p(x_t | u_t, x_(t-1)) * bel(x_(t-1)))

Where:
- p(x_t | u_t, x_(t-1)) represents the **odometry motion model** (prediction),
- p(z_t | x_t) represents the **sensor model** (update),
- eta is a normalizing constant to ensure the belief sums to 1.

The **Prediction Step** generally spreads out the belief (increasing uncertainty), while the **Update Step** tightens the belief around likely poses (reducing uncertainty).

# Odometry Motion Model

The **Odometry Motion Model** expresses control input (u) between two poses as:

- rotation1: initial rotation to face the direction of motion,
- translation: forward movement,
- rotation2: final rotation to match the new orientation.

Since odometry measurements are noisy, I model errors in each component using **Gaussian distributions**. This allows the prediction step to realistically account for real-world uncertainty in the robot’s motion.



<br>
<br>


---

## Implementation

This section outlines how I implemented the grid localization using the Bayes filter. The implementation is divided into a few functions, each of which plays a role in the overall localization process.

<br>

# compute_control
- **Purpose:**  
  Extracts the control parameters (rotation1, translation, rotation2) from the current and previous robot poses.
- **Details:**  
  - Receives the current pose and previous pose as input.
  - Computes the required rotations and translation based on pose differences, using trig and the Pythagorean theorem.
  - Normalizes the angles to be within the range [-180, 180) degrees.

```python

def compute_control(cur_pose, prev_pose):
    """ Given the current and previous odometry poses, this function extracts
    the control information based on the odometry motion model.

    Args:
        cur_pose  ([Pose]): Current Pose
        prev_pose ([Pose]): Previous Pose 

    Returns:
        [delta_rot_1]: Rotation 1  (degrees)
        [delta_trans]: Translation (meters)
        [delta_rot_2]: Rotation 2  (degrees)
    """

    # Extract x, y, and theta from the current and previous poses
    cur_x, cur_y, cur_theta = cur_pose
    prev_x, prev_y, prev_theta = prev_pose

    # Calculate the initial rotation to face the direction of movement and normalize
    delta_rot_1 = mapper.normalize_angle(
        np.degrees(np.arctan2(cur_y - prev_y, cur_x - prev_x)) - prev_theta
    )

    # Compute the translation distance between the two poses
    delta_trans = np.hypot(cur_y - prev_y, cur_x - prev_x)

    # Calculate the final rotation after the translation and normalize
    delta_rot_2 = mapper.normalize_angle(cur_theta - prev_theta - delta_rot_1)

    return delta_rot_1, delta_trans, delta_rot_2

```

<br>

# odom_motion_model
- **Purpose:**  
  Computes the probability of transitioning from a previous pose to a current pose given the measured control input.
- **Details:**  
  - Retrieves the control parameters using `compute_control`.
  - Applies Gaussian noise models to each control parameter (rotation1, translation, rotation2).
  - Returns the product of these probabilities as the overall likelihood.

```python

def odom_motion_model(cur_pose, prev_pose, u):
    """ Odometry Motion Model

    Args:
        cur_pose  ([Pose]): Current Pose
        prev_pose ([Pose]): Previous Pose
        (rot1, trans, rot2) (float, float, float): A tuple with control data in the format 
                                                   format (rot1, trans, rot2) with units (degrees, meters, degrees)

    Returns:
        prob [float]: Probability p(x'|x, u)
    """

    # Compute motion controls based on current and previous poses
    delta_rot_1, delta_trans, delta_rot_2 = compute_control(cur_pose, prev_pose)

    # Calculate the probability of each control value using the motion noise model
    prob_rot_1 = loc.gaussian(delta_rot_1, u[0], loc.odom_rot_sigma)
    prob_trans = loc.gaussian(delta_trans, u[1], loc.odom_trans_sigma)
    prob_rot_2 = loc.gaussian(delta_rot_2, u[2], loc.odom_rot_sigma)

    # Return the combined probability of the motion
    return prob_rot_1 * prob_trans * prob_rot_2

```

<br>

# prediction_step
- **Purpose:**  
  Propagates the prior belief to generate a predicted belief (or prior) based on the odometry data.
- **Details:**  
  - Iterates over all possible previous and current states in the grid.
  - Uses `odom_motion_model` to compute transition probabilities.
  - Updates the temporary belief (bel_bar) by incorporating the weighted contributions from all possible transitions.
  - Normalizes the predicted belief to ensure the probabilities sum to 1.
  - Skips cells with a really small belief in order to save some compute time. This might sacrifice some accuracy, but it can save a lot of time.

```python
def prediction_step(cur_odom, prev_odom):
    """ Prediction step of the Bayes Filter.
    Update the probabilities in loc.bel_bar based on loc.bel from the previous time step and the odometry motion model.

    Args:
        cur_odom  ([Pose]): Current Pose
        prev_odom ([Pose]): Previous Pose
    """

    # Derive the control inputs from the odometry readings
    u = compute_control(cur_odom, prev_odom)

    # Reset loc.bel_bar to all zeros before prediction
    loc.bel_bar = np.zeros((mapper.MAX_CELLS_X, mapper.MAX_CELLS_Y, mapper.MAX_CELLS_A))

    # Loop through all previous belief states
    for prev_x in range(mapper.MAX_CELLS_X):
        for prev_y in range(mapper.MAX_CELLS_Y):
            for prev_theta in range(mapper.MAX_CELLS_A):

                # Skip cells with negligible belief to save time
                if loc.bel[prev_x, prev_y, prev_theta] < 0.0001:
                    continue

                # Loop through all possible current belief states
                for cur_x in range(mapper.MAX_CELLS_X):
                    for cur_y in range(mapper.MAX_CELLS_Y):
                        for cur_theta in range(mapper.MAX_CELLS_A):

                            # Calculate the probability of transitioning from previous to current state
                            p = odom_motion_model(
                                mapper.from_map(cur_x, cur_y, cur_theta),
                                mapper.from_map(prev_x, prev_y, prev_theta),
                                u
                            )

                            # Update loc.bel_bar with the weighted probability
                            loc.bel_bar[cur_x, cur_y, cur_theta] += p * loc.bel[prev_x, prev_y, prev_theta]

    # Normalize loc.bel_bar to make it a proper probability distribution
    loc.bel_bar /= np.sum(loc.bel_bar)
```

<br>

# sensor_model
- **Purpose:**  
  Calculates the likelihood of the observed sensor measurements at a given robot pose.
- **Details:**  
  - Receives a set of sensor observations.
  - Uses a Gaussian function for each individual measurement to model noise.
  - Returns an array with the likelihoods for each sensor measurement.

```python

def sensor_model(obs):
    """ This is the equivalent of p(z|x).

    Args:
        obs ([ndarray]): A 1D array consisting of the true observations for a specific robot pose in the map 

    Returns:
        [ndarray]: Returns a 1D array of size 18 (=loc.OBS_PER_CELL) with the likelihoods of each individual sensor measurement
    """

    # Return a list of sensor probabilities for each observation using a Gaussian model
    return [loc.gaussian(obs[i], loc.obs_range_data[i], loc.sensor_sigma)
            for i in range(mapper.OBS_PER_CELL)]

```


<br>

# update_step
- **Purpose:**  
  Corrects the predicted belief by integrating the sensor data to reduce uncertainty.
- **Details:**  
  - Iterates through all grid cells.
  - Uses the sensor model to compute the likelihood of the observations at each grid cell.
  - Multiplies the predicted belief by the sensor likelihood and normalizes the result.

```python

def update_step():
    """ Update step of the Bayes Filter.
    Update the probabilities in loc.bel based on loc.bel_bar and the sensor model.
    """

    # Iterate through the entire belief grid
    for cur_x in range(mapper.MAX_CELLS_X):
        for cur_y in range(mapper.MAX_CELLS_Y):
            for cur_theta in range(mapper.MAX_CELLS_A):

                # Use the sensor model to compute the likelihood for the current pose
                p = sensor_model(mapper.get_views(cur_x, cur_y, cur_theta))

                # Multiply the likelihood with the prior from loc.bel_bar
                loc.bel[cur_x, cur_y, cur_theta] = np.prod(p) * loc.bel_bar[cur_x, cur_y, cur_theta]

    # Normalize loc.bel to ensure it sums to 1
    loc.bel /= np.sum(loc.bel)

```

<br>
<br>

With these functions, the trajectory loop integrates the steps together in real time. This code:


  - Sets up the trajectory the robot will follow during the simulation.
  - At each time step:
    - Executes the motion step.
    - Calls the `prediction_step` based on the odometry data.
    - Acquires new sensor observations.
    - Calls the `update_step` to refine the belief.
  - Visualizes the ground truth and odometry poses.


---


## Simulation


Trial 1:
<div align = "center">

<iframe width="500" height="315" 
    src="https://www.youtube.com/embed/RSBlnlibRNY" 
    frameborder="1" 

    allowfullscreen>
</iframe>

</div>
Trial 2:
<div align = "center">

<iframe width="500" height="315" 
    src="https://www.youtube.com/embed/fP9mytv2sKk" 
    frameborder="1" 

    allowfullscreen>
</iframe>

</div>



The videos show the simulation running. The odometry model (red) can be seen taking measurements that don't really make sense. Compared to the ground truth (green), the sensor measurements look way off. However, the belief (blue) matches the ground truth much better. This shows the advantage of a Bayes filter for dealing with noisy sensor data.
Another interesting thing is that the filter seems to be a bit more accurate when the robot is near walls. This is probably because the sensor noise gets worse with measurements taken further from a wall.


<br>
<br>

## Collaboration

I worked extensively with Lucca Correia and Jack Long. I also referenced Stephan Wagner's website for help implementing the Bayes Filter.

