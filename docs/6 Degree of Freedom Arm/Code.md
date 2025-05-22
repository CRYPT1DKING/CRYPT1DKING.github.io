---
title: Code
---

## Code

```python
from machine import Pin, PWM
import math
from ulab import numpy as np
from time import sleep_ms

# --- Servo Setup ---
servo1 = PWM(Pin(5), freq=50)
servo2 = PWM(Pin(18), freq=50)
servo3 = PWM(Pin(19), freq=50)
servo4 = PWM(Pin(21), freq=50)
servo5 = PWM(Pin(22), freq=50)
servo6 = PWM(Pin(23), freq=50)

def angle_to_us_servo1(angle):
    return 600 + (angle / 180) * (2500 - 600)

def angle_to_us_servo2(angle):
    return 500 + ((180 - angle) / 270) * (2500 - 500)

def angle_to_us_servo3(angle):
    return 500 + ((angle + 135) / 270) * (2500 - 500)

def angle_to_us_servo4(angle):
    return 500 + ((angle + 90) / 180) * (2400 - 500)

def angle_to_us_servo5(angle):
    return 500 + ((angle + 90) / 180) * (2380 - 500)

def angle_to_us_servo6(angle):
    return 590 + ((angle + 90) / 180) * (2410 - 590)

def us_to_duty(us):
    return int((us / 20000) * 65535)

def move_servo(servo, angle_fn, angle):
    duty = us_to_duty(angle_fn(angle))
    servo.duty_u16(duty)

# --- Robot Arm Geometry ---
a = [9.5, 17.0, 17.0, 6.0, 6.0, 3.0]  # Arm segment lengths

# --- Inverse Kinematics Function ---
def inverse_kinematics(target_position):
    x, y, z = target_position
    
    # --- Solve for the first three joints (theta1, theta2, theta3) ---
    r = math.sqrt(x**2 + y**2)  # Planar distance (ignore z for now)
    d = math.sqrt((r - a[3])**2 + z**2)
    
    # Solving for theta1 (base rotation angle)
    theta1 = math.atan2(y, x)

    # Solve for theta2 (shoulder joint)
    cos_theta2 = (d**2 - a[1]**2 - a[2]**2) / (2 * a[1] * a[2])
    sin_theta2 = math.sqrt(1 - cos_theta2**2)
    theta2 = math.atan2(sin_theta2, cos_theta2)

    # Solve for theta3 (elbow joint)
    theta3 = math.atan2(z, r - a[3])

    # For simplicity, we'll assume no end-effector orientation (theta4, theta5, theta6 are zeros for now)
    theta4 = 0
    theta5 = 0
    theta6 = 0

    # Return calculated angles
    return [math.degrees(theta1), math.degrees(theta2), math.degrees(theta3), theta4, theta5, theta6]

# --- Robot Arm Kinematics Functions ---
def dh_matrix(theta, alpha, r, d):
    theta = math.radians(theta)
    alpha = math.radians(alpha)
    return np.array([
        [math.cos(theta), -math.sin(theta)*math.cos(alpha), math.sin(theta)*math.sin(alpha), r*math.cos(theta)],
        [math.sin(theta),  math.cos(theta)*math.cos(alpha), -math.cos(theta)*math.sin(alpha), r*math.sin(theta)],
        [0,               math.sin(alpha),                 math.cos(alpha),                 d],
        [0,               0,                               0,                               1]
    ])

def forward_kinematics(thetas):
    alpha = [90, 0, 0, 90, 90, 0]
    r = [0, a[1], a[2], 0, 0, 0]
    d = [a[0], 0, 0, -a[3], a[4], a[5]]
    T = np.eye(4)
    for i in range(6):
        T = np.dot(T, dh_matrix(thetas[i], alpha[i], r[i], d[i]))
    return T

# --- Jacobian Calculation ---
def compute_jacobian(thetas):
    alpha = [90, 0, 0, 90, 90, 0]
    r = [0, a[1], a[2], 0, 0, 0]
    d = [a[0], 0, 0, -a[3], a[4], a[5]]

    T = np.eye(4)
    positions = [np.array([0.0, 0.0, 0.0])]
    z_vectors = [np.array([0.0, 0.0, 1.0])]
    for i in range(6):
        T = np.dot(T, dh_matrix(thetas[i], alpha[i], r[i], d[i]))
        positions.append(np.array([T[0, 3], T[1, 3], T[2, 3]]))
        z_vectors.append(np.array([T[0, 2], T[1, 2], T[2, 2]]))

    J = np.zeros((6, 6))
    pe = positions[-1]
    for i in range(6):
        z = z_vectors[i]
        pi = positions[i]
        J[0:3, i] = np.cross(z, pe - pi)
        J[3:6, i] = z
    return J

# --- Inverse Kinematics Step ---
def ik_velocity_step(thetas, v_linear, v_angular, step_size=0.5):
    J = compute_jacobian(thetas)
    try:
        # Pseudo-inverse using (JᵀJ)⁻¹Jᵀ
        JT = J.T
        JTJ = np.dot(JT, J)
        JTJ_inv = np.linalg.inv(JTJ)
        J_pinv = np.dot(JTJ_inv, JT)
    except Exception:
        print("Jacobian inversion failed!")
        return thetas

    # Combine velocity vectors
    v = np.array(list(v_linear) + list(v_angular))
    delta_theta = np.dot(J_pinv, v)

    new_thetas = []
    for i in range(6):
        theta = thetas[i] + math.degrees(step_size * delta_theta[i])

        # Apply range limits
        if i == 0:  # Servo 1: 0–180°
            if not (0 <= theta <= 180):
                print(f"Servo {i+1} angle {theta:.1f}° out of range [0, 180]! Stopping.")
                return thetas
        elif i == 1:  # Servo 2: -90–180°
            if not (-90 <= theta <= 180):
                print(f"Servo {i+1} angle {theta:.1f}° out of range [-90, 180]! Stopping.")
                return thetas
        elif i == 2:  # Servo 3: -135–135°
            if not (-135 <= theta <= 135):
                print(f"Servo {i+1} angle {theta:.1f}° out of range [-135, 135]! Stopping.")
                return thetas
        else:  # Servo 4–6: -90–90°
            if not (-90 <= theta <= 90):
                print(f"Servo {i+1} angle {theta:.1f}° out of range [-90, 90]! Stopping.")
                return thetas
        new_thetas.append(theta)

    return new_thetas


# --- Main Loop ---
dt_ms = 100
step_size = dt_ms / 1000  # = 0.1 seconds

# Desired velocities (per second)
v_linear_per_sec = np.array([5, -2, 1])       
v_angular_per_sec = np.array([0, 0, 0])      

target_position = np.array([10.0, 1.0, 15.0])  # Example target position for the end-effector

# Get initial joint angles for the target position using inverse kinematics
thetas = inverse_kinematics(target_position)#[0.0,0.0,0.0,0.0,0.0,0.0]

# Print initial angles
print("Initial joint angles from inverse kinematics:", thetas)

# Move servos to initial position
move_servo(servo1, angle_to_us_servo1, thetas[0])
move_servo(servo2, angle_to_us_servo2, thetas[1])
move_servo(servo3, angle_to_us_servo3, thetas[2])
move_servo(servo4, angle_to_us_servo4, thetas[3])
move_servo(servo5, angle_to_us_servo5, thetas[4])
move_servo(servo6, angle_to_us_servo6, thetas[5])

while True:
    # Scale velocities for current time step
    v_linear = v_linear_per_sec * step_size
    v_angular = v_angular_per_sec * step_size

    # Perform IK velocity step
    new_thetas = ik_velocity_step(thetas, v_linear, v_angular, step_size)

    # Stop if no change
    if new_thetas == thetas:
        print("Motion stopped due to servo limit or Jacobian issue.")
        break

    thetas = new_thetas

    # Move servos to new angles
    move_servo(servo1, angle_to_us_servo1, thetas[0])
    move_servo(servo2, angle_to_us_servo2, thetas[1])
    move_servo(servo3, angle_to_us_servo3, thetas[2])
    move_servo(servo4, angle_to_us_servo4, thetas[3])
    move_servo(servo5, angle_to_us_servo5, thetas[4])
    move_servo(servo6, angle_to_us_servo6, thetas[5])

    # Print current end-effector position
    T = forward_kinematics(thetas)
    position = T[0:3, 3]
    print("End-effector position:", position)

    sleep_ms(dt_ms)
```