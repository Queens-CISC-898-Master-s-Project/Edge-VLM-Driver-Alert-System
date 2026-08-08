# ROS 2 Design — EdgeVLM-Drive

## 1. Purpose

This document defines the ROS 2 architecture for EdgeVLM-Drive, a lightweight
Vision-Language Model (VLM) system for driver distraction and drowsiness
warning on NVIDIA Jetson Nano.

The system is designed for:

- Offline operation
- Low event-level latency
- Local/private camera processing
- Configurable inference and temporal decision parameters
- Comparison of compact VLMs before selecting the deployment model

The system is not intended to perform high-frame-rate video analytics. It
samples camera frames and uses temporal decision logic to determine whether
an unsafe driver state persists.

---

## 2. High-Level Pipeline

```text
      Camera
        |
        | sampled frames
        v
+-------------------+
|   Camera Node     |
+-------------------+
        |
        | Frame messages
        v
+-------------------+
| Inference Node    |
|                   |
| +---------------+ |
| | Latest-frame  | |
| | bounded queue | |
| +---------------+ |
|        |          |
|        v          |
|   Model Adapter   |
|   / VLM Pipeline  |
+-------------------+
        |
        | Structured prediction
        v
+-------------------+
|  Decision Node    |
|                   |
| +---------------+ |
| | Recent        | |
| | prediction    | |
| | history       | |
| +---------------+ |
|        |          |
|        v          |
| Temporal logic    |
+-------------------+
        |
        | Alert command
        v
+-------------------+
|   Alert Node      |
+-------------------+
        |
        v
 Buzzer / GPIO
```

---

## 3. ROS 2 Packages

The ROS 2 workspace is planned around the following packages:

```text
src/
├── edgevlm_interfaces/
├── edgevlm_camera/
├── edgevlm_inference/
├── edgevlm_decision/
└── edgevlm_alert/
```

### 3.1 `edgevlm_interfaces`

Owns custom ROS 2 message definitions used for communication between nodes.

Responsibilities:

- Define stable communication contracts
- Keep messages independent of a specific VLM
- Represent driver-state predictions
- Represent alert commands/status

This package should contain message definitions rather than application logic.

### 3.2 `edgevlm_camera`

Owns camera acquisition.

Responsibilities:

- Initialize the configured camera
- Capture frames
- Timestamp frames
- Apply required basic image handling
- Publish frames to the inference node

It should not:

- Run VLM inference
- Make driver-state decisions
- Control the buzzer

### 3.3 `edgevlm_inference`

Owns VLM inference.

Responsibilities:

- Receive camera frames
- Maintain a bounded latest-frame queue
- Discard stale frames when the inference system falls behind
- Select/load the configured model
- Run the final VLM/LangChain/LangGraph inference pipeline
- Convert model output into a standardized prediction
- Publish the prediction with timing/model metadata

The node must hide model-specific implementation details from the rest of the ROS system.

### 3.4 `edgevlm_decision`

Owns temporal reasoning and alert decisions.

Responsibilities:

- Receive standardized predictions
- Maintain a short configurable prediction history
- Evaluate persistence of unsafe states
- Apply configurable thresholds/rules
- Prevent unnecessary repeated alerts
- Publish an alert command when the configured conditions are satisfied

The decision node should not depend on the internal implementation of a particular VLM.

### 3.5 `edgevlm_alert`

Owns physical alert output.

Responsibilities:

- Receive alert commands
- Control the buzzer through the appropriate hardware interface
- Manage alert activation/deactivation
- Provide a clean abstraction for future alert devices such as LEDs or displays

It should not perform VLM inference or temporal driver-state reasoning.

---

## 4. Queue Design

Two different buffering mechanisms are intentionally used.

### 4.1 Inference frame queue

The inference node uses a bounded queue for camera frames.

Purpose:

> Prefer the newest available frame instead of processing stale frames.

Example:

```text
Camera:
F1 F2 F3 F4 F5 F6
                ^
                |
        inference is busy

Old frames can be discarded, the system should prefer F6 over processing an old F2.
```

The queue size is configurable.

Initial values should be treated as experimental parameters rather than permanent
design decisions. Small values such as 1–3 are expected to be evaluated.

### 4.2 Decision prediction history

The decision node stores recent predictions, not images.

Example:

```text
t1  NORMAL
t2  DROWSY
t3  DROWSY
t4  DROWSY
```

The decision logic can require a persistent unsafe state before generating an
alert.

The history/window is configurable so that different settings can be measured
for:

- Alert latency
- False alarms
- Detection reliability

This gives the project a measurable research dimension.

---

## 5. Model Integration

The ROS 2 inference node should not depend directly on one specific VLM.

The intended architecture is:

```text
                  Inference Node
                        |
                        v
                  Model Adapter
                  /     |     \     |       Qwen  SmolVLM  Other
                  \     |     /
                   \    |    /
                    Standard
                   prediction
```

The exact model may be selected after external experiments.

The model team may work independently of ROS 2. Their experimental work can
include:

- Model selection
- Prompt engineering
- LangChain/LangGraph pipelines
- Output-format experiments
- Accuracy comparisons

Once the final pipeline is selected, it is integrated behind the inference
node's model adapter/interface.

The rest of the ROS 2 system should receive the same standardized prediction
format regardless of which model is active.

---

## 6. Model Configuration

The active model should be configurable externally.

The intended concept is:

```text
config/inference/
    inference.yaml
```

The configuration may eventually specify items such as:

- Active model
- Model path
- Prompt/configuration path
- Image resolution
- Sampling interval
- Frame queue size
- Maximum inference settings

Exact parameter names will be defined when the ROS 2 inference package is
implemented.

This allows experiments without changing the decision or camera nodes.

---

## 7. Driver States

The initial standardized driver states are:

```text
NORMAL
DROWSY
LOOKING_AWAY
PHONE_USAGE
```

The system should allow the model layer to produce structured output that can
be validated before it reaches the decision logic.

The decision layer, rather than the VLM itself, is responsible for deciding
whether a state persists strongly enough to trigger an alert.

---

## 8. Communication Boundaries

The planned communication flow is:

```text
Camera Node
    |
    | image/frame message
    v
Inference Node
    |
    | driver-state prediction
    v
Decision Node
    |
    | alert command
    v
Alert Node
```

The nodes communicate through ROS 2 topics rather than direct function calls.

This keeps the system modular and allows individual components to be replaced
or tested independently.

---

## 9. Configurability and Experimentation

The following should be configurable rather than hard-coded where practical:

- Camera sampling interval
- Image resolution
- Inference frame queue size
- Active VLM
- Model path
- Prompt configuration
- Decision history/window
- Confidence thresholds
- Persistence requirements
- Alert timing

This is important because the research evaluates trade-offs between:

- Accuracy
- Latency
- Memory usage
- Power usage
- False alarms
- Alert delay

The exact default values will be selected after implementation and
benchmarking.

---

## 10. Runtime Flow

A typical runtime sequence is:

```text
1. Camera captures a frame.
2. Camera node timestamps and publishes it.
3. Inference node receives the frame.
4. Stale frames are discarded if the bounded queue is full.
5. The active VLM pipeline processes the newest useful frame.
6. The inference node validates and publishes a structured prediction.
7. Decision node adds the prediction to its recent history.
8. Decision logic evaluates temporal persistence.
9. If the unsafe condition satisfies the configured rule,
   an alert command is published.
10. Alert node activates the buzzer.
```

---

## 11. Latency Instrumentation

Latency measurement is part of the research scope.

The architecture should preserve timestamps needed to estimate:

- Frame capture time
- Frame publication/receipt time
- Inference start/end time
- Prediction publication time
- Decision time
- Alert command time
- Alert activation time where measurable

These measurements will later support benchmark results and plots.

---

## 12. Testing Strategy

ROS 2 package-level tests will be added when the ROS 2 skeleton is created.

Expected test areas include:

- Interface/message validation
- Camera node behavior
- Frame queue behavior
- Inference output validation
- Decision/temporal logic
- Alert triggering
- End-to-end pipeline behavior

The current repository-level CI does not require ROS 2 tests yet because the
ROS 2 workspace has not been implemented.

Once the ROS 2 packages exist, CI can be extended with ROS 2 build and test
commands such as `colcon build` and `colcon test`.

---

## 13. Deployment Considerations

Development and Jetson deployment environments are intentionally separated.

```text
docker/Dockerfile.dev
    |
    +-- PC development environment

docker/Dockerfile.jetson
    |
    +-- Jetson deployment environment
```

The exact ROS 2 distribution and JetPack-compatible dependencies will be
selected after the target Jetson software environment is confirmed.

The ROS application architecture should remain as independent as practical
from the underlying container implementation.

---

## 14. Design Principles

The architecture follows these principles:

1. One clear responsibility per ROS 2 node.
2. Communication through ROS 2 interfaces/topics.
3. Model-specific code is isolated inside the inference layer.
4. The newest useful frame is preferred over stale frames.
5. Temporal reasoning is separated from VLM inference.
6. Physical hardware control is isolated in the alert node.
7. Experimental parameters are configurable.
8. Measurements are collected for research evaluation.
9. The system is designed for offline edge execution.
10. The architecture should support replacing the VLM without redesigning the
    rest of the pipeline.

---

## 15. Decisions Still To Be Finalized

The following should be decided during implementation rather than guessed now:

- ROS 2 distribution
- Exact ROS 2 message fields
- Exact topic names
- Camera message type and transport strategy
- Queue implementation
- Default queue sizes
- Decision-window implementation
- VLM adapter API
- Exact model(s) used for deployment
- Jetson-specific inference backend
- GPIO implementation details
- Final Jetson Dockerfile

These decisions depend partly on the Jetson/JetPack environment and the final
model-selection experiments.
