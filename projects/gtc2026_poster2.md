# Spatial Intelligence Architecture for Physical AI: Generation, Understanding, and Reasoning of Spatial Information

---

## Motivation

### ■ Challenge

- Physical AI struggles in complex and dynamic real-world environments.

### ■ Solution

- We propose Spatial Intelligence, an architecture for Al to **"See"** (3D Reconstruction), **"Understand"** (Semantic Comprehension), and **"Decide"** (Autonomous Operation).

### ■ Impact

- This architecture enables two key breakthroughs:
  1.  True autonomous operation in the real world
  2.  Safe and scalable Al learning in simulation

> "Spatial intelligence focuses on seeing, thinking, and deciding like humans"

---

## Method / Proposed Architecture

### ■ Stage 1: Generation

- **Goal:** High-fidelity 3D reconstruction of real environments.
- **Method:** SLAM&3DGS-based 3D reconstruction of the real world.
- **Output:** Realistic virtual simulations (e.g., Isaac Sim) for Al training.

#### _Stage 1 흐름도 요소_

- **Inputs:**
  - calibration
  - RGB-Point Cloud
  - Fused multimodal sensor data
- **Processes:**
  - Accurate Pose and Geometric Information (SLAM)
  - Photorealistic Rendering (3DGS)
- **Output:**
  - 3D Realistic Environment

### ■ Stage 2: Understanding

- **Goal:** Enriches 3D maps with semantic context (what objects are and where).
- **Method:** 2D-3D projection of object detection (e.g., YOLOv8) and multi-object tracking.
- **Output:** A semantic information for situational awareness.

#### _Stage 2 흐름도 요소_

- **Inputs:**
  - 3D Realistic Environment
- **Processes:**
  - High-Precision Multi-object Detection (YOLOVB, SAM2, MediaPipe)
  - Multi-Object Tracking in 30 (Kalman fitter, Hungarian algorithm)
  - Semantic Mapping (20-30 Projection, Object categorization)
- **Output:**
  - Semantic Information

### ■ Stage 3: Reasoning

- **Goal:** Transforms spatial understanding into autonomous action.
- **Method:** LLM-based multi-agent reasoning for dynamic action planning.
- **Output:** Autonomous action planning and operation.

#### _Stage 3 흐름도 요소_

- **Inputs:**
  - Human Interaction (Request)
  - Semantic Information
- **Processes:**
  - Dynamic Context Generation (RAG, Vector OB, Role Assignment)
  - LLM-based Multi-Agent Reasoning
  - Cognitive Architectures, Tool/Function Calling
  - (Autonomous Driving Tech Stack)
- **Output:**
  - Autonomous action planning Autonomous operation

---

## Results

### ■ Integrated Architecture

- A unified 3-stage architecture (Generation, Understanding, Reasoning) serving as the "head" for Physical AI.

### ■ Real-Time Spatial Intelligence on Edge Device

- The spatial intelligence architecture was deployed and verified for real-time operation on an NVIDIA Jetson Thor edge device.

### ■ Software-in-the-Loop (SIL) Testing

- Demonstrated high success in SIL testing (Isaac Sim/ROS2) for autonomous navigation, planning, and obstacle avoidance.

#### _SIL 테스트 흐름도_

- **In simulation:**
  - Realistic virtual simulations (Isaac Sim)
  - -> Synthetic sensors data
- **Software-in-the-loop Testing:**
  - (In real world on Jetson Thor)
  - -> Robot action commands (ROS2 Topic)
- **In simulation:**
  - -> Robot in simulation

---

## Future Work

- [Scaling to Large-Scale, Multi-Story Environments]
- [Application to real robot and scaling to multi-robot systems]
