# Rotations and Translations in 3D Space

A comprehensive guide to the common mathematical representations used to describe orientation and position in three dimensions, including a thorough explanation of why the **order of operations** is critical when composing rotations.

## Table of Contents
- [Coordinate System Handedness](#coordinate-system-handedness)
- [Euler Angles](#euler-angles)
- [Quaternions](#quaternions)
- [Homogeneous Transformation Matrices](#homogeneous-transformation-matrices)
- [Axis-Angle Representation](#axis-angle-representation)
- [Rotation Vectors (Rodrigues Vectors)](#rotation-vectors-rodrigues-vectors)
- [Direction Cosine Matrices (DCM)](#direction-cosine-matrices-dcm)
- [Why Order of Operations Matters](#why-order-of-operations-matters)
- [Choosing a Representation](#choosing-a-representation)
- [Quick Reference Summary](#quick-reference-summary)

---

## Coordinate System Handedness

Before working with any rotation or translation representation, it is essential to know which **handedness** the coordinate system uses. Mixing conventions without accounting for handedness is a very common source of bugs.

### Right-Handed Coordinate System

In a right-handed coordinate system, the axes satisfy:

```
X × Y = Z
```

You can verify this with the **right-hand rule**: point your right hand's fingers along the +X axis, curl them toward +Y, and your thumb points in the +Z direction.

```
         +Z
          |
          |
          +-------> +Y
         /
        /
      +X
```

Common right-handed conventions:
- **OpenGL**, **Vulkan** (world space), **ROS** (Robot Operating System): +X forward or right, +Y left or up, +Z up or out-of-screen
- **Aerospace (NED)**: +X North, +Y East, +Z Down
- **Mathematics / Physics**: standard Cartesian axes

### Left-Handed Coordinate System

In a left-handed coordinate system:

```
X × Y = -Z   (the Z axis points the opposite direction compared to right-handed)
```

You can verify with the **left-hand rule**: point your left hand's fingers along +X, curl toward +Y, and your thumb points in the +Z direction.

```
         +Z (into screen)
          |
          |
          +-------> +X
         /
        /
      +Y
```

Common left-handed conventions:
- **DirectX**, **Unity 3D**: +X right, +Y up, +Z forward (into the screen)
- **Unreal Engine** (default): +X forward, +Y right, +Z up (also left-handed in some contexts)

### Positive Rotation Direction

Handedness also determines which direction is a **positive rotation** about an axis:

- **Right-handed system**: positive rotation follows the **right-hand rule** — curl the right hand's fingers in the direction of positive rotation; the thumb points along the positive axis. A 90° positive rotation about +Z rotates +X toward +Y.
- **Left-handed system**: positive rotation follows the **left-hand rule** — the same angle produces the opposite spin direction compared to a right-handed system.

```
Right-handed +Z rotation:       Left-handed +Z rotation:
  +Y                              +Y
   ^                               ^
   |  ↺ positive                   |  ↻ positive
   +-----> +X                      +-----> +X
```

### Why This Matters in Practice

| Situation | Impact |
|-----------|--------|
| Passing rotation matrices between OpenGL and DirectX | Row/column order AND handedness differ |
| Importing 3D assets between Unity and Blender | Axis mapping and rotation sign must both be converted |
| Combining quaternions from different SDKs | A quaternion from a right-handed SDK represents a mirrored rotation in a left-handed SDK |
| Cross-platform robotics software | ROS (right-handed) vs. some game engines (left-handed) |

### Converting Between Handedness

To convert a right-handed coordinate system to a left-handed one (or vice versa), negate one axis. For example, to convert from right-handed (ROS/OpenGL-style) to Unity's left-handed system, negate the Z axis:

```
p_LH = (px, py, -pz)
```

For rotation matrices, converting handedness is equivalent to a **reflection**, which changes the sign of the determinant from +1 to -1. A rotation matrix in one handedness cannot be directly used in the other without a basis change transformation:

```
R_LH = M · R_RH · M⁻¹

where M is the axis-negation matrix, e.g. diag(1, 1, -1) to negate Z
```

For quaternions, negating the axis component that corresponds to the flipped axis achieves the same effect:

```
q_RH = (w, x, y, z)  →  q_LH = (w, x, y, -z)   [when negating Z]
```

> **Best practice**: always document the handedness of your coordinate system alongside any rotation data. Never assume handedness — always confirm it from the source library or hardware documentation.

---

## Euler Angles

Euler angles describe an orientation as a sequence of three rotations about coordinate axes. Despite their intuitive appeal, they come with important pitfalls.

### The Three Angles

| Angle | Common Name | Typical Axis |
|-------|-------------|--------------|
| φ (phi) | Roll | X |
| θ (theta) | Pitch | Y |
| ψ (psi) | Yaw | Z |

### Intrinsic vs. Extrinsic Rotations

- **Intrinsic** (body-fixed): each rotation is applied relative to the *moving* frame after the previous rotation. The axes travel with the object.
- **Extrinsic** (space-fixed): each rotation is applied relative to the *fixed* world frame. Applying intrinsic ZYX is identical to applying extrinsic XYZ in reverse order.

### Common Conventions

Different fields use different axis sequences. The convention **must** always be stated explicitly because the same three numbers produce different orientations under different conventions.

| Convention | Sequence | Common Domain |
|------------|----------|---------------|
| Tait-Bryan ZYX | Yaw → Pitch → Roll | Aerospace, robotics |
| Tait-Bryan XYZ | Roll → Pitch → Yaw | Some robotics frameworks |
| Proper Euler ZXZ | Z → X → Z | Classical mechanics |
| Proper Euler ZYZ | Z → Y → Z | Quantum mechanics |

### Rotation Matrices from Euler Angles (ZYX / Aerospace Convention)

Individual rotation matrices for rotation by angle θ about each axis:

```
Rx(φ) = | 1    0       0    |
        | 0   cos φ  -sin φ |
        | 0   sin φ   cos φ |

Ry(θ) = |  cos θ  0  sin θ |
        |   0     1    0   |
        | -sin θ  0  cos θ |

Rz(ψ) = | cos ψ  -sin ψ  0 |
        | sin ψ   cos ψ  0 |
        |   0       0    1 |
```

The combined rotation matrix for ZYX (yaw ψ, pitch θ, roll φ) is:

```
R = Rz(ψ) · Ry(θ) · Rx(φ)
```

### Gimbal Lock

Gimbal lock is the most significant limitation of Euler angles. It occurs when the middle rotation aligns two axes, collapsing a degree of freedom and making it impossible to represent certain orientations or produce certain angular velocities.

**Example (ZYX, pitch = ±90°):** when θ = ±90°, the roll and yaw axes become parallel. The system loses one degree of freedom, and infinitely many (ψ, θ, φ) triples map to the same orientation.

Gimbal lock is not just a mathematical artifact — it causes real problems in sensors, animations, and control systems. It is one of the main reasons quaternions are preferred in practice.

### Pros and Cons

| ✅ Advantages | ❌ Disadvantages |
|--------------|-----------------|
| Intuitive and human-readable | Subject to gimbal lock |
| Only 3 numbers | Non-unique representations |
| Easy to specify manually | Interpolation produces unexpected paths |
| Widely used in UI and sensors | Order convention ambiguity |

---

## Quaternions

A quaternion is a four-component number of the form:

```
q = w + xi + yj + zk
```

where `w` is the scalar (real) part and `(x, y, z)` is the vector (imaginary) part. The imaginary units satisfy:

```
i² = j² = k² = ijk = -1
ij = k,  jk = i,  ki = j
ji = -k, kj = -i, ik = -j
```

### Unit Quaternions for Rotation

A **unit quaternion** (`|q| = 1`) encodes a rotation of angle θ about a unit axis **n̂** = (nx, ny, nz) as:

```
q = cos(θ/2) + sin(θ/2)·(nx·i + ny·j + nz·k)

  w = cos(θ/2)
  x = nx · sin(θ/2)
  y = ny · sin(θ/2)
  z = nz · sin(θ/2)
```

Note the half-angle — this is fundamental to the double-cover property (both `q` and `-q` represent the same rotation).

### Rotating a Vector

To rotate a 3D point **p** by quaternion **q**:

1. Represent **p** as a pure quaternion: `p = 0 + px·i + py·j + pz·k`
2. Compute: `p' = q · p · q*`  (where `q*` is the conjugate: `w - xi - yj - zk`)

### Quaternion Multiplication (Composition)

Applying rotation `q1` followed by rotation `q2`:

```
q_total = q2 · q1
```

Given `q1 = (w1, x1, y1, z1)` and `q2 = (w2, x2, y2, z2)`:

```
w = w2·w1 - x2·x1 - y2·y1 - z2·z1
x = w2·x1 + x2·w1 + y2·z1 - z2·y1
y = w2·y1 - x2·z1 + y2·w1 + z2·x1
z = w2·z1 + x2·y1 - y2·x1 + z2·w1
```

Quaternion multiplication is **not commutative**: `q2 · q1 ≠ q1 · q2` in general.

### Converting Quaternion to Rotation Matrix

```
R = | 1-2(y²+z²)   2(xy-wz)    2(xz+wy)  |
    | 2(xy+wz)    1-2(x²+z²)   2(yz-wx)  |
    | 2(xz-wy)    2(yz+wx)    1-2(x²+y²) |
```

### SLERP — Spherical Linear Interpolation

One of the major advantages of quaternions is smooth, shortest-path interpolation between orientations:

```
SLERP(q1, q2, t) = q1 · (q1* · q2)^t    for t ∈ [0, 1]
```

or equivalently:

```
SLERP(q1, q2, t) = ( sin((1-t)Ω) · q1 + sin(tΩ) · q2 ) / sin(Ω)

where  Ω = arccos(q1 · q2)   (dot product of quaternion coefficients)
```

This produces constant-speed rotation along the geodesic on the 4D unit hypersphere, which translates to natural-looking motion.

### Pros and Cons

| ✅ Advantages | ❌ Disadvantages |
|--------------|-----------------|
| No gimbal lock | Less intuitive than Euler angles |
| Compact (4 numbers) | Double-cover ambiguity (q and -q same rotation) |
| Numerically stable | Harder to reason about directly |
| Excellent for interpolation (SLERP) | Requires normalization to stay unit |
| Fast composition and inversion | Not natively supported in all tools |

---

## Homogeneous Transformation Matrices

A **homogeneous transformation matrix** (HTM) is a 4×4 matrix that encodes both rotation and translation in a single structure, enabling both operations to be combined and composed with simple matrix multiplication.

### Structure

```
T = | R  t |  =  | r00  r01  r02  tx |
    | 0  1 |     | r10  r11  r12  ty |
                 | r20  r21  r22  tz |
                 |  0    0    0    1 |
```

Where:
- **R** (upper-left 3×3) is a rotation matrix (orthogonal, det(R) = 1)
- **t** (upper-right 3×1) is the translation vector
- The bottom row `[0, 0, 0, 1]` is a mathematical convention that enables composition

### Transforming a Point

A 3D point `p = (px, py, pz)` is extended to homogeneous coordinates `p̃ = (px, py, pz, 1)`:

```
p̃' = T · p̃

| r00  r01  r02  tx |   | px |   | r00·px + r01·py + r02·pz + tx |
| r10  r11  r12  ty | × | py | = | r10·px + r11·py + r12·pz + ty |
| r20  r21  r22  tz |   | pz |   | r20·px + r21·py + r22·pz + tz |
|  0    0    0    1 |   |  1 |   |               1               |
```

The result's first three components give the transformed 3D point.

### Composing Transformations

The power of HTMs is that a chain of transforms reduces to a single matrix multiplication:

```
T_total = T_n · T_(n-1) · ... · T_2 · T_1
```

Applying T_total to a point is equivalent to applying T_1 first, then T_2, etc.

### Inverse of a Transformation Matrix

Inverting an HTM is efficient because of the structure of R (orthogonal ⟹ R⁻¹ = Rᵀ):

```
T⁻¹ = | Rᵀ  -Rᵀt |
      |  0     1  |
```

### Example: Robot Arm Link

In robotics, each link in a kinematic chain is described by its own HTM. The pose of the end-effector in the world frame is:

```
T_world_to_end = T_01 · T_12 · T_23 · ... · T_(n-1)n
```

### Pros and Cons

| ✅ Advantages | ❌ Disadvantages |
|--------------|-----------------|
| Rotation + translation in one object | 16 elements (redundant, 6 DOF needs only 6 numbers) |
| Composition via matrix multiply | Can drift from orthogonality (floating point) |
| Standard in robotics (DH parameters) | More memory and compute than quaternions |
| Easy inversion | Not ideal for interpolation |
| Widely supported in libraries | |

---

## Axis-Angle Representation

Any rotation in 3D can be expressed as a single rotation by angle **θ** about a unit axis **n̂**:

```
(n̂, θ)  where  n̂ = (nx, ny, nz),  |n̂| = 1,  θ ∈ [0, π]
```

### Rodrigues' Rotation Formula

Given axis **n̂** and angle **θ**, rotating vector **v**:

```
v' = v·cos θ + (n̂ × v)·sin θ + n̂·(n̂ · v)·(1 - cos θ)
```

### Converting Axis-Angle to Rotation Matrix

Using `c = cos θ`, `s = sin θ`, `t = 1 - cos θ`:

```
R = | t·nx² + c      t·nx·ny - s·nz   t·nx·nz + s·ny |
    | t·nx·ny + s·nz  t·ny² + c       t·ny·nz - s·nx |
    | t·nx·nz - s·ny  t·ny·nz + s·nx   t·nz² + c     |
```

### Pros and Cons

| ✅ Advantages | ❌ Disadvantages |
|--------------|-----------------|
| Geometrically intuitive | Singularity at θ = 0 (axis undefined) |
| Minimum 4 numbers | Not ideal for composition |
| Natural for physics simulations | Wrapping issues near θ = ±π |

---

## Rotation Vectors (Rodrigues Vectors)

A rotation vector (also called an **exponential map** or **axis-angle vector**) folds the axis and angle into one compact vector:

```
r = θ · n̂   (a 3-element vector)

|r| = θ  (the magnitude is the rotation angle)
r/|r| = n̂  (the direction is the rotation axis)
```

### Relationship to SO(3) and the Lie Algebra

Rotation vectors are elements of **so(3)**, the Lie algebra of the rotation group SO(3). The mapping from so(3) to SO(3) is the **matrix exponential**:

```
R = exp([r]×)
```

where `[r]×` is the skew-symmetric matrix:

```
[r]× = |  0   -rz   ry |
       |  rz   0   -rx |
       | -ry   rx   0  |
```

### Pros and Cons

| ✅ Advantages | ❌ Disadvantages |
|--------------|-----------------|
| Minimal (only 3 numbers) | Singular at zero rotation |
| Used in optimization (e.g., SLAM) | Composition requires exp/log maps |
| No redundancy or constraint | Wrapping at 2π |

---

## Direction Cosine Matrices (DCM)

A **Direction Cosine Matrix** is simply the explicit name for a 3×3 rotation matrix when the entries are understood as the cosines of the angles between the new and old coordinate axes.

Each element `R_ij` is the cosine of the angle between the i-th axis of the new frame and the j-th axis of the original frame:

```
R = | cos(x', x)  cos(x', y)  cos(x', z) |
    | cos(y', x)  cos(y', y)  cos(y', z) |
    | cos(z', x)  cos(z', y)  cos(z', z) |
```

**Properties** of a valid rotation matrix (DCM):
- Orthogonal: `R · Rᵀ = I`
- Unit determinant: `det(R) = +1`
- Each row and column is a unit vector
- Inverse equals transpose: `R⁻¹ = Rᵀ`

DCM is widely used in aerospace guidance and navigation systems. It is numerically straightforward but suffers from **orthogonality drift** over time in floating-point arithmetic, which requires periodic re-orthogonalization (e.g., via Gram-Schmidt or SVD).

---

## Why Order of Operations Matters

This is one of the most important and frequently misunderstood aspects of working with rotations. **Rotation is not commutative.**

### Rotations Do Not Commute

For most pairs of rotations R1 and R2:

```
R2 · R1  ≠  R1 · R2
```

Applying R1 first and then R2 produces a **different result** than applying R2 first and then R1.

### Concrete Example

Start with a vector pointing along the +X axis: **v = (1, 0, 0)**

**Sequence A:** Rotate 90° around Z, then 90° around X
```
Step 1: Rotate (1,0,0) by 90° around Z  →  (0, 1, 0)   [points along +Y]
Step 2: Rotate (0,1,0) by 90° around X  →  (0, 0, 1)   [points along +Z]
Result: (0, 0, 1)
```

**Sequence B:** Rotate 90° around X, then 90° around Z
```
Step 1: Rotate (1,0,0) by 90° around X  →  (1, 0, 0)   [unchanged, on rotation axis]
Step 2: Rotate (1,0,0) by 90° around Z  →  (0, 1, 0)   [points along +Y]
Result: (0, 1, 0)
```

The two sequences produce **completely different final orientations** even though the same individual rotations were applied.

### Matrix Multiplication Order

In matrix notation, the rightmost matrix is applied **first**. If you want to rotate a point **p** by R1 first, then R2:

```
p' = R2 · (R1 · p) = (R2 · R1) · p
```

So the composed matrix `R2 · R1` has R1 on the right (applied first) and R2 on the left (applied second). Getting this order wrong silently produces incorrect results.

### Translation Order Also Matters

When combining rotation and translation, order matters even more. Consider:

**Rotate then Translate:**
```
p' = R·p + t
```
The object is rotated in place first, then moved.

**Translate then Rotate:**
```
p' = R·(p + t) = R·p + R·t
```
The object is moved first, then rotated around the origin — producing a different position.

This is why homogeneous transformation matrices are so useful: the 4×4 matrix encodes the correct ordering, and composition via matrix multiplication automatically handles it correctly.

### Intrinsic vs. Extrinsic and Order

For Euler angles, the distinction between intrinsic and extrinsic directly affects application order:

- **Extrinsic ZYX**: apply Z rotation first (in world frame), then Y, then X — matrix form: `Rx · Ry · Rz`
- **Intrinsic ZYX**: apply Z rotation in body frame, then Y in new body frame, then X — matrix form: `Rz · Ry · Rx`

These are numerically identical when the axis sequence is reversed, but the conceptual model differs and it is critical to match the convention used in your system.

### Why This Matters in Practice

| Domain | Consequence of Wrong Order |
|--------|---------------------------|
| Robotics | End-effector reaches the wrong position |
| Computer graphics | Objects rotate to unexpected orientations |
| Aerospace | Navigation errors, potentially safety-critical |
| Game development | Characters or cameras behave unexpectedly |
| SLAM / Visual Odometry | Accumulated pose errors |
| 3D printing / CNC | Incorrect toolpaths |

### The Group Theory Perspective

Rotations form a **non-abelian group** (specifically SO(3) for 3D rotations). The "non-abelian" property is precisely the mathematical statement that the group operation (composition) is not commutative. This is not a quirk or limitation — it is a fundamental property of 3D space. Any correct implementation of 3D rotations must respect this.

---

## Choosing a Representation

| Use Case | Recommended Representation |
|----------|---------------------------|
| Storing / transmitting orientation | Quaternion |
| Human input / sensor output | Euler angles |
| Smooth animation interpolation | Quaternion (SLERP) |
| Rigid body pose (position + orientation) | Homogeneous transformation matrix |
| Optimization / gradient-based methods | Rotation vector (so(3)) |
| Aerospace navigation systems | DCM |
| Geometric intuition / visualization | Axis-angle |
| Chaining multiple transforms | Homogeneous transformation matrix |

In many systems, you store orientations as quaternions, display them as Euler angles, and compose chains of transforms as HTMs.

---

## Quick Reference Summary

| Representation | Size | Gimbal Lock | Interpolation | Composition |
|----------------|------|-------------|---------------|-------------|
| Euler Angles | 3 | Yes | Poor | Indirect (convert) |
| Quaternion | 4 | No | Excellent (SLERP) | q2·q1 |
| Rotation Matrix (DCM) | 9 | No | Moderate | R2·R1 |
| Homogeneous Matrix | 16 | No | Moderate | T2·T1 |
| Axis-Angle | 4 | Singular at 0 | Moderate | Indirect |
| Rotation Vector | 3 | Singular at 0 | Via exp/log | Via exp/log |

> **Golden rule**: Always document which rotation convention you are using, and always double-check the order when composing rotations. Silent ordering bugs are among the hardest to track down in any spatial computing system.
