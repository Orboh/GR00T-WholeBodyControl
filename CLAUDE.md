# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **GR00T Whole-Body Control (WBC)** codebase for training, evaluating, and deploying whole-body controllers for humanoid robots. The repository contains two main control systems:

1. **Decoupled WBC**: A decoupled controller (RL for lower body, IK for upper body) used in NVIDIA GR00T N1.5 and N1.6
2. **GEAR-SONIC**: A humanoid behavior foundation model that learns motor skills from large-scale human motion data using motion tracking as a training task

## Repository Structure

- **[decoupled_wbc/](decoupled_wbc/)**: Python package for decoupled whole-body control
  - [control/](decoupled_wbc/control/): Core control logic (base, policies, robot models, sensors, teleoperation, utils)
  - [data/](decoupled_wbc/data/): Robot configuration and motion data
  - [scripts/](decoupled_wbc/scripts/): Deployment and utility scripts
  - [tests/](decoupled_wbc/tests/): Test suite
  - [sim2mujoco/](decoupled_wbc/sim2mujoco/): Simulation conversion tools

- **[gear_sonic/](gear_sonic/)**: Python package for GEAR-SONIC teleoperation
  - [scripts/](gear_sonic/scripts/): Main scripts including MuJoCo sim loop and VR teleoperation server
  - [utils/](gear_sonic/utils/): Utility functions
  - [isaac_utils/](gear_sonic/isaac_utils/): IsaacLab integration utilities

- **[gear_sonic_deploy/](gear_sonic_deploy/)**: C++ inference and deployment stack
  - [src/](gear_sonic_deploy/src/): C++ source (TensorRT inference, audio, G1 robot interface)
  - [g1/](gear_sonic_deploy/g1/): G1 robot model files (MuJoCo XML, meshes)
  - [policy/](gear_sonic_deploy/policy/): ONNX policy files
  - [reference/](gear_sonic_deploy/reference/): Reference motion data
  - [scripts/](gear_sonic_deploy/scripts/): Setup and deployment scripts

## Common Development Commands

### Installation

**Python packages:**
```bash
# Decoupled WBC (with all dependencies)
pip install -e decoupled_wbc[full]

# For development
pip install -e decoupled_wbc[dev]

# GEAR-SONIC teleoperation
pip install -e gear_sonic[teleop]

# GEAR-SONIC simulation
pip install -e gear_sonic[sim]
```

**C++ Deployment (gear_sonic_deploy):**
```bash
cd gear_sonic_deploy

# Install system dependencies
./scripts/install_deps.sh

# Set up environment (adds TensorRT paths)
source scripts/setup_env.sh

# Build using just (recommended)
just build

# Or build with CMake directly
mkdir -p build
cd build
cmake -S .. -B . -DCMAKE_BUILD_TYPE=Release
cmake --build . -j$(nproc)
```

**MuJoCo Simulation Environment:**
```bash
# From repository root
bash install_scripts/install_mujoco_sim.sh
```

### Linting and Code Quality

```bash
# Check code style (from repo root)
make run-checks           # Run isort, black, ruff checks
./lint.sh                 # Run ruff and black checks only

# Auto-fix issues
make format               # Run isort and black formatters
./lint.sh --fix          # Fix with ruff and black

# Build Python packages
make build
```

### Testing

```bash
# Run tests for decoupled_wbc
pytest decoupled_wbc/tests/

# Run specific test
pytest decoupled_wbc/tests/test_specific.py
```

### Running Simulations

**Sim2Sim with MuJoCo:**
```bash
# Terminal 1: MuJoCo simulator (on host, not in Docker)
source .venv_sim/bin/activate
python gear_sonic/scripts/run_sim_loop.py

# Terminal 2: Deployment (in gear_sonic_deploy/)
cd gear_sonic_deploy
bash deploy.sh sim
```

**Decoupled WBC Control:**
```bash
# Simulation
python decoupled_wbc/control/main/teleop/run_g1_control_loop.py

# Real robot (requires network setup)
python decoupled_wbc/control/main/teleop/run_g1_control_loop.py --interface real
```

**VR Teleoperation:**
```bash
# Keep run_g1_control_loop.py running, then start teleoperation
python decoupled_wbc/control/main/teleop/run_teleop_policy_loop.py \
  --hand_control_device=pico \
  --body_control_device=pico
```

### Deployment

```bash
cd gear_sonic_deploy

# Deploy to simulation
./deploy.sh sim

# Deploy to real G1 robot
./deploy.sh real
```

## Architecture Overview

### Decoupled WBC Design

The decoupled controller splits control into two subsystems:
- **Lower body**: Uses RL policy trained in Isaac Gym/IsaacLab for locomotion
- **Upper body**: Uses inverse kinematics (IK) for manipulation tasks
- Communication between Python control loop and real robot via Unitree SDK2

Key components:
- [control/policy/](decoupled_wbc/control/policy/): Policy implementations (lower body RL, upper body IK)
- [control/robot_model/](decoupled_wbc/control/robot_model/): Robot kinematics and dynamics models using Pinocchio
- [control/teleop/](decoupled_wbc/control/teleop/): Teleoperation devices (Pico VR, LeapMotion, gamepad)
- [control/envs/](decoupled_wbc/control/envs/): Environment interfaces (simulation and real robot)

### GEAR-SONIC Design

SONIC is a foundation model for humanoid whole-body control:
- Trained on large-scale human motion data using motion tracking
- Single unified policy produces natural whole-body movement
- Supports multiple control modes: motion playback, teleoperation, kinematic planning
- Real-time inference via TensorRT on C++ deployment stack

Key components:
- **Training**: Python-based (training code to be released)
- **Deployment**: C++ with TensorRT for low-latency inference
- **Communication**: ZMQ for streaming control inputs (keyboard, gamepad, VR, planner)
- **Visualization**: PyVista-based real-time visualization via ZMQ debug stream

### C++ Deployment Stack (gear_sonic_deploy)

- **TensorRT Inference**: ONNX model → TensorRT engine → real-time policy execution
- **Unitree SDK2**: Robot communication and control commands
- **ZMQ Interface**: Receives commands from external controllers (keyboard, VR, planner)
- **Multi-threaded**: Separate threads for robot I/O, inference, and audio feedback

Critical dependencies:
- TensorRT 10.13 (x86_64) or 10.7 (Jetson/Orin with JetPack 6)
- CUDA Toolkit (12.4+)
- ONNX Runtime
- Unitree SDK2 (included in thirdparty/)

### Communication Flow

```
┌─────────────────┐     ZMQ      ┌──────────────────┐
│ VR/Keyboard/    │ ──────────▶  │ g1_deploy (C++)  │
│ Gamepad/Planner │              │ TensorRT Policy  │
└─────────────────┘              └──────────────────┘
                                          │
                                          │ Unitree SDK2
                                          ▼
                                  ┌──────────────┐
                                  │  G1 Robot    │
                                  │  (Real/Sim)  │
                                  └──────────────┘
```

## Key Configuration Files

- **[pyproject.toml](pyproject.toml)**: Root-level Python tooling (black, isort, ruff, mypy, pytest)
- **[decoupled_wbc/pyproject.toml](decoupled_wbc/pyproject.toml)**: Decoupled WBC package dependencies
- **[gear_sonic/pyproject.toml](gear_sonic/pyproject.toml)**: GEAR-SONIC package dependencies
- **[gear_sonic_deploy/CMakeLists.txt](gear_sonic_deploy/CMakeLists.txt)**: C++ build configuration
- **[.github/workflows/](https://github.com/NVlabs/GR00T-WholeBodyControl/tree/main/.github/workflows)**: CI/CD pipelines

## Environment Setup Notes

### TensorRT Setup (Critical for Deployment)

```bash
# Download TensorRT and extract to ~/TensorRT
export TensorRT_ROOT=$HOME/TensorRT

# Add to ~/.bashrc for persistence
echo "export TensorRT_ROOT=$HOME/TensorRT" >> ~/.bashrc
```

### Robot Network Configuration

For real G1 robot deployment:
- Set static IP to `192.168.123.222`
- Subnet mask: `255.255.255.0`
- Robot communicates on `192.168.123.x` network

### Docker Environments

Two Docker environments are available:
1. **decoupled_wbc**: Full control and teleoperation stack ([docker/run_docker.sh](decoupled_wbc/docker/run_docker.sh))
2. **gear_sonic_deploy**: ROS2 development environment ([docker/run-ros2-dev.sh](gear_sonic_deploy/docker/run-ros2-dev.sh))

## Control Keyboard Shortcuts

**gear_sonic_deploy (deploy.sh):**
- `]`: Start policy
- `O`: Stop control and exit (emergency stop)
- `9`: Drop robot to ground (in sim)
- `T`: Play current reference motion
- `N`/`P`: Next/previous motion sequence
- `R`: Restart current motion from beginning

**decoupled_wbc (run_g1_control_loop.py):**
- `]`: Activate policy
- `o`: Deactivate policy
- `9`: Release/hold robot
- `w`/`s`: Forward/backward
- `a`/`d`: Strafe left/right
- `q`/`e`: Rotate left/right
- `z`: Zero navigation commands
- `1`/`2`: Raise/lower base height

## Important Dependencies

**Python (3.10 required):**
- PyTorch
- NumPy 1.26.4, SciPy 1.15.3
- Pinocchio (pin, pin-pink): Robot kinematics and dynamics
- MuJoCo: Physics simulation
- PyZMQ, msgpack: Inter-process communication
- ONNX Runtime: Policy inference
- Unitree SDK2 (Python bindings)

**C++:**
- TensorRT: Neural network inference
- CUDA Toolkit: GPU acceleration
- ONNX Runtime: Model loading
- Unitree SDK2: Robot communication
- Eigen3, spdlog, nlohmann-json

## Documentation

Full documentation is available at: https://nvlabs.github.io/GR00T-WholeBodyControl/

Key pages:
- [Installation (Deployment)](https://nvlabs.github.io/GR00T-WholeBodyControl/getting_started/installation_deploy.html)
- [Quick Start](https://nvlabs.github.io/GR00T-WholeBodyControl/getting_started/quickstart.html)
- [VR Teleoperation Setup](https://nvlabs.github.io/GR00T-WholeBodyControl/getting_started/vr_teleop_setup.html)
- [Deployment Code Reference](https://nvlabs.github.io/GR00T-WholeBodyControl/references/deployment_code.html)