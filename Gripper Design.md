# Parallel Jaw Gripper for ROV Manipulation

A 3D-printed, servo-actuated parallel jaw gripper designed for underwater ROV manipulation tasks. The mechanism uses a four-bar linkage with integrated spur gears for synchronized jaw motion, paired with a rotational base for versatile object handling.

## Overview

This gripper is engineered for controlled underwater robotic applications, balancing compactness, load capacity, and manufacturability through 3D printing. It features a parallel-motion jaw mechanism, a secondary rotational degree of freedom, and integrated mounting points for both the ROV body and a feedback camera.

## Key Features

- Parallel jaw motion via four-bar linkage with geared synchronization
- ~115 mm gripper opening span
- Up to 175° rotational motion on each side
- TPU-95A flexible gripping surfaces for improved friction and compliance
- Integrated camera mounting for visual feedback
- Dual-clamp ROV mounting for tool-free attachment and removal
- IP20 compliance for controlled underwater operation

## Mechanical Design

### Parallel Jaw Mechanism

The gripper is built around a four-bar linkage that keeps the jaws parallel throughout the entire opening and closing range. The inner bars of the linkage are mechanically coupled using integrated spur gears, ensuring that both sides of the gripper move synchronously. One of these geared bars is connected to the primary servo motor through an additional spur gear, forming the actuation system.

### Gear Train

| Component | Teeth | Pitch Diameter |
|-----------|-------|----------------|
| Linkage Gear | 80 | 48 mm |
| Servo-Mounted Gear | 25 | 15 mm |
| **Gear Ratio** | **5:16** | — |

The 5:16 reduction was intentionally selected to utilize most of the servo's angular range while mapping it to a gripper opening span of approximately 115 mm, with additional headroom to prevent operation near mechanical limits. Gear sizes were chosen to balance compactness, load capacity, and 3D printing feasibility.

Because the servo-mounted gear is smaller, it is more susceptible to structural failure. To address this, metal screws are embedded within the printed part, increasing strength and improving load-bearing capability under operation.

### Gripping Surfaces

The jaws are fitted with **TPU-95A** flexible inserts, allowing the contact surfaces to conform slightly to gripped objects. This improves friction and holding capability while reducing instantaneous load on the servo, which operates without feedback control.

### Rotational Mechanism

The entire gripper assembly can rotate up to **175° on each side**, driven by a secondary servo directly coupled via a **carbon fiber rod**. This rod serves a dual purpose:

- **Torque transmission** from the servo to the gripper body
- **Rotational shaft** providing structural support

Carbon fiber was selected for its high stiffness-to-weight ratio and corrosion resistance in underwater conditions.

## Integration

### ROV Mounting

The gripper attaches to the ROV using a **dual-clamp mechanism**, enabling easy installation and removal for maintenance and transport. At the gripper-to-ROV interface, a **TPU-95A damping layer** improves mechanical coupling and reduces vibration transmission.

### Camera Mount

A dedicated camera mounting clamp is integrated directly into the gripper body, providing improved visual feedback during manipulation tasks.

## Materials and Manufacturing

### PETG Structural Components

All structural components, including the gears and main body, are 3D printed using **PETG**, selected for:

- Low water absorption
- Good chemical resistance
- Superior layer adhesion, particularly along the Z-axis
- Balance between flexibility and toughness, reducing brittle failure risk

### Print Settings

| Parameter | Value |
|-----------|-------|
| Perimeter Wall Loops | 5 |
| Infill Density | 80% |

These settings significantly increase part strength while minimizing internal voids, reducing effective water absorption to near zero.

### Material Summary

| Component | Material |
|-----------|----------|
| Structural body and gears | PETG |
| Gripping surfaces | TPU-95A |
| ROV interface damping layer | TPU-95A |
| Rotational shaft | Carbon fiber rod |
| Reinforcement (small gear) | Embedded metal screws |

## Specifications

| Parameter | Value |
|-----------|-------|
| Gripper opening span | ~115 mm |
| Rotational range | ±175° |
| Gear ratio | 5:16 |
| Ingress protection | IP20 |
| Actuation | 2× servo motors (open-loop) |

## Ingress Protection

The design follows **IP20 standards**, providing protection against solid object ingress greater than 12.5 mm while maintaining an open, non-sealed structure. This is suitable for controlled underwater robotic applications where full enclosure is not required.

## Design Rationale

The gripper's design philosophy emphasizes:

1. **Synchronized motion** through geared coupling, eliminating the need for dual actuators on the jaws
2. **Mechanical compliance** via TPU-95A surfaces, compensating for the absence of force feedback
3. **Manufacturability** through 3D printing, while reinforcing high-stress components with metal hardware
4. **Underwater suitability** through careful material selection (PETG, TPU-95A, carbon fiber)
5. **Modularity** via clamp-based mounting for both the ROV and the camera



## Contributing

*Add contribution guidelines, contact information, or team credits here.*
