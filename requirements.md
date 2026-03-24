# System Requirements

## 1. Purpose

The system is a Proof of Concept (POC) alternative compute module that
uses controlled laser manipulation to solve well‑defined mathematical
equations.

The primary objectives of the system are:
- Demonstration of the underlying computation technology
- End‑to‑end execution of a defined mathematical problem
- Validation of physical feasibility, stability, and controllability

The system is not intended to optimize cost, size, performance,
or manufacturability.

---

## 2. System Scope

### 2.1 In Scope

The system includes:
- A laser cavity acting as the computational core
- Optical elements required to generate, manipulate, and observe laser beams
- Temperature stabilization and monitoring of critical optical elements
- Timing and trigger generation and coordination between subsystems
- Sensing and acquisition of computation results
- Automation and assisted control across system lifecycle stages
- Configuration, monitoring, and operational control of the system

### 2.2 Out of Scope

The system does not:
- Define or optimize the mathematical formulation of the problem
- Replace external host computation or user applications
- Perform long‑term data analytics or result interpretation
- Optimize cost, size, or manufacturability
- Enforce component availability or long‑term supply constraints
- Implement production‑grade packaging or compliance

---

### 2.3 System Classification

The system is explicitly defined as a Proof of Concept (POC) system.

The goal of the system is to demonstrate feasibility, validate physical
principles, and enable controlled experimentation.

The system is not intended to serve as a prototype, pre‑production,
or manufacturing‑ready design.

---

## 3. System Description (Informative)

The core of the system is a laser cavity, typically composed of:
- One or more IR diode sources pumping a gain crystal
- A gain crystal generating laser emission
- Sets of mirrors and additional optical elements forming the cavity
- Spatial Light Modulators (SLMs) used to manipulate laser beams
- Cameras used to capture and digitize the optical computation results

This section is descriptive only and does not prescribe a specific design.

---

## 4. System Lifecycle and Operation Stages

The system shall support the following operational stages:

### Stage 1 – Build and Assembly
- Highly manual stage
- Performed by trained engineers
- The system shall provide observability and assisted control where possible
- Full automation is not required at this stage

### Stage 2 – Calibration
- Calibration spans both initial build and ongoing operation
- Calibration may be required due to environmental changes
  (e.g. temperature, weather, alignment drift)
- The system shall support partial to full automation of calibration procedures
- The system shall support the addition of control loops based on:
  - Sensor feedback
  - Image‑processing‑based feedback

### Stage 3 – Normal Operation
- The system shall operate autonomously
- Minimal manual intervention shall be required
- Operational sequences shall be deterministic and repeatable

---

## 5. Operational Responsibilities

### OR‑1 Temperature Control – Gain Crystal
The system shall set, regulate, and maintain the gain crystal temperature
within its defined operating range.

Temperature stability shall be sufficient to ensure stable laser frequency
and repeatable system behavior.

### OR‑2 Temperature Control – SLM Devices
The system shall control and monitor the temperature of SLM devices
to ensure stable optical modulation and prevent performance degradation.

### OR‑3 Timing and Trigger Coordination
The system shall generate, receive, and coordinate timing and trigger signals
between subsystems, including but not limited to:
- SLM image update completion
- Diode activation
- Camera exposure and readout

Trigger sequences shall be configurable per operational mode.

### OR‑4 Diode Current Control
The system shall control and regulate the current supplied to the IR diode sources
used for pumping the gain crystal.

Current control shall enable:
- Controlled laser output power
- Repeatable operation across different operational modes
- Protection against over‑current conditions

---

## 6. Functional Requirements

### FR‑1 Controlled Laser Operation
The system shall support controlled operation of the laser cavity,
including startup, steady‑state operation, and shutdown.

### FR‑2 Automation
The system shall support a high degree of automation across all operational
stages, with emphasis on autonomous behavior during normal operation.

### FR‑3 Assisted Calibration
The system shall support calibration workflows that combine:
- Automated procedures
- Sensor‑based feedback
- Optional image‑processing‑based control loops

### FR‑4 Mode‑Dependent Operation
The system shall support multiple modes of operation,
each defining a specific sequence of control actions,
timing relationships, and trigger dependencies.

### FR‑5 Deterministic Behavior
For a given configuration and mode of operation,
the system shall exhibit deterministic and repeatable behavior.

---

## 7. Performance Requirements

### PR‑1 Temperature Stability
Temperature control subsystems shall meet stability and resolution
requirements defined per controlled element (TBD).

### PR‑2 Timing Accuracy
Trigger generation and synchronization shall meet defined accuracy
and jitter requirements (TBD).

### PR‑3 Repeatability
System outputs shall be repeatable across runs under equivalent conditions,
within defined tolerances (TBD).

### PR‑4 Automation Effectiveness
Automation mechanisms shall reduce the need for manual intervention
during calibration and normal operation, while preserving system stability
and repeatability.

---

## 8. Non‑Functional Requirements

### NFR‑1 Observability
The system shall provide visibility into:
- Temperatures
- Timing events
- Operational states
- Fault conditions
- Raw measurement data where applicable

### NFR‑2 Reliability and Resilience
The system shall detect abnormal operating conditions
and support controlled degradation or shutdown.

### NFR‑3 Modularity
The system architecture shall allow replacement or modification
of subsystems (optical, electronic, control) with minimal system‑level impact.

### NFR‑4 Safety
The system is a laser‑based system and shall enforce safety requirements
relevant to laser operation, including:
- Controlled startup and shutdown
- Protection against unsafe operating conditions

No FCC, EMI, or environmental compliance is required.

---

## 9. Constraints

### C‑1 Physical Constraints
The system shall operate within physical constraints related to:
- Power
- Thermal dissipation
- Mechanical stability

### C‑2 Operational Constraints
The system shall be operated by trained personnel
in controlled laboratory environments.

### C‑3 Compliance Constraints
The system is subject to laser safety requirements only.

---

## 10. Design Philosophy

### DP‑1 Proof of Concept Orientation
The system design shall prioritize:
- Functional correctness
- Physical insight
- Experimental flexibility

over cost, size, or production optimization.

