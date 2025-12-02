# **SliceRT-Farm**

### *End-to-End µTVM Auto-Scheduler + Zephyr Slice Runtime + QEMU Microbenchmark Farm*

**Deterministic, reproducible multi-worker infrastructure for real-time ML scheduling on Cortex-M**

**SliceRT-Farm** is an end-to-end, fully reproducible research infrastructure for exploring real-time deep neural network (DNN) execution on Cortex-M microcontrollers. It integrates the TVM auto-scheduler, μTVM C runtime, a slice-based preemptive execution model for Zephyr RTOS, and an instrumented QEMU Cortex-M simulator that emits cycle-accurate kernel and interrupt traces. A scalable microbenchmark farm—built on Docker and Redis—enables rapid evaluation of thousands of candidate schedules across many parallel workers without requiring physical hardware.

SliceRT-Farm emphasizes *determinism*, *automation*, and *structured artifact generation*. Every experiment produces version-pinned builds, feature vectors, runtime measurements, and slice-level logs suitable for cost-model training and real-time analysis. Through hierarchical tiling, tile-boundary preemption, and precise instrumentation, SliceRT-Farm enables controlled study of latency, interrupt overheads, and preemption behavior under simulated real-time conditions.

The system provides a practical foundation for research at the intersection of **real-time systems, ML compilation, and microcontroller scheduling**, supporting design-space exploration, WCET-aware optimization, and early-stage evaluation before hardware deployment.

---

# **1. Overview**

SliceRT-Farm automates the entire pipeline for exploring real-time ML execution on Cortex-M devices:

* **TVM Auto-Scheduler**
* **μTVM code generation + C runtime**
* **Zephyr RTOS slice-based preemptive runtime**
* **Cycle-instrumented QEMU Cortex-M simulation**
* **Distributed worker farm for large-scale experiments**
* **Structured artifact storage for reproducibility and model training**

The system allows you to:

* Evaluate thousands of schedules quickly and deterministically
* Study latency under fine-grained slice preemption
* Collect cycle-level traces of kernels and interrupts
* Train cost models (XGBoost/LightGBM) using structured logs
* Scale horizontally with tens of workers
* Reproduce any experiment from fixed Docker images

---

# **2. Architecture**

```
              +------------------------+
              |       Controller       |
              |    (task dispatcher)   |
              +-----------+------------+
                          |
                     Redis Queue
                          |
   ---------------------------------------------------------
   |                         |                            |
   v                         v                            v
+---------------+      +---------------+        +---------------+
|   Worker 1    |      |   Worker 2    |  ...   |   Worker N    |
|    Docker     |      |    Docker     |        |    Docker     |
+-------+-------+      +-------+-------+        +-------+-------+
        |                      |                        |
   μTVM Codegen         μTVM Codegen              μTVM Codegen
   Zephyr Build         Zephyr Build              Zephyr Build
      QEMU                 QEMU                     QEMU
```

Each worker independently builds the app, runs QEMU, logs cycles, and uploads artifacts.

---

# **3. Components**

### **Controller (`farm/controller`)**

* Loads experiment manifests
* Generates tasks
* Dispatches tasks via Redis
* Collects worker artifacts and organizes results

### **Worker (`farm/worker`)**

* Performs μTVM codegen
* Builds Zephyr slice-runtime app
* Runs QEMU with detailed instrumentation
* Extracts cycles, slice timings, IRQ latency
* Uploads results to the controller

### **Zephyr Slice Runtime (`zephyr/patches/slice_scheduler.patch`)**

* Hierarchical macro/micro tiling
* Preemption at micro-tile boundaries
* Inline yield checks
* Deterministic pendsv/interrupt timing instrumentation

### **QEMU Cycle Logging (`qemu/patches/cycle_counter_patch.c`)**

* DWT-like cycle counter
* Retired instruction counter
* UART-based structured output

### **Docker Environment (`docker/`)**

* Fully version-pinned toolchain
* Base image, worker image, controller image
* docker-compose orchestration

---

# **4. Setup (Minimal Instructions)**

## **4.1 Build all Docker images**

```bash
cd docker/scripts
./build_all.sh
```

Images generated:

* `slicert-base`
* `slicert-worker`
* `slicert-controller`

---

## **4.2 Launch the farm**

```bash
docker-compose -f docker/compose/docker-compose.yaml up --scale worker=6
```

Workers automatically pull tasks from Redis.

---

## **4.3 Run an experiment**

Example manifest (`experiments/conv2d_5x5.yaml`):

```yaml
experiment:
  name: "conv2d_5x5_latency"
  repeat: 30

  parameters:
    tile_size: [8, 16, 32]
    unroll:    [1, 2, 4]
    slice:     [32, 64]

  runner:
    simulate_irq_load: true

  logging:
    raw: true
    compress: true
```

Submit it:

```bash
python3 farm/controller/controller.py \
    --manifest experiments/conv2d_5x5.yaml
```

---

## **4.4 Result artifacts**

Outputs stored under:

```
results/
  run_0001/
    manifest.json
    tvm_config.json
    zephyr_build.log
    qemu_trace.json
    cycles.txt
  run_0002/
    ...
```

These logs are stable, structured, and reproducible.

---

## **4.5 Train a cost model**

```bash
python3 training/train_cost_model.py \
    --input results/**/qemu_trace.json \
    --output artifacts/cost_model.pkl
```

Workers automatically load the newest model.

---

# **5. Zephyr Slice Runtime Details**

SliceRT introduces a tile-based execution model for predictable, measurable ML kernel scheduling on Cortex-M:

- Hierarchical tiling:
  - Macro-tiles for locality, micro-tiles for deterministic preemption.
- Inline yield checks:
  - Lightweight checks between micro-tiles trigger PendSV when the slice budget expires.
- Tile-boundary preemption:
  - Safe interruption/resume of kernels with preserved registers, accumulators, and DMA state.
- Cycle-level logging:
  - QEMU provides DWT-like cycle counters with logs for tile timing, IRQ latency, and PendSV overhead.

The SliceRT Zephyr runtime provides a **deterministic, tile-aware, preemptive execution model** designed specifically for MCU-scale ML kernels. It mediates between μTVM operators and the Zephyr RTOS scheduler, ensuring predictable slice-level execution, bounded preemption delay, and cycle-level observability inside QEMU.

---

# **6. Limitations**

* QEMU timing excludes wait states, caches, bus contention
* Auto-scheduler optimizes average—not worst-case—latency
* Slice preemption introduces cold-cache effects
* Inline yield overhead can dominate tiny tiles
* No hardware backend currently supported

---

# **7. Future Work**

* Physical hardware backend (J-Link / SWD / pyOCD)
* Extended QEMU timing (Flash wait-states, cache models)
* WCET-aware loss functions for cost models
* EDF and multi-model scheduling
* Kubernetes-based scale-out
* Integration with LakeFS / MLflow
* Automated hardware-in-the-loop CI

---

# **8. Development Workflow**

Enter environment:

```bash
./docker/scripts/run_in_container.sh
```

Build manually:

```bash
west build -b qemu_cortex_m3 zephyr/apps/microbench
```

Run QEMU manually:

```bash
qemu-system-arm -M lm3s6965evb -nographic -kernel build/zephyr/zephyr.elf
```

---

# **9. Maintainers**

Maintained anonymously.
Issues and PRs are welcome.

---

# **10. Summary**

SliceRT-Farm is a unified, reproducible environment for real-time ML research on microcontrollers. It brings together TVM auto-scheduling, slice-based Zephyr execution, QEMU simulation, and scalable distributed automation in one workflow—ideal for:

* real-time ML
* embedded systems optimization
* scheduling and OS research
* reproducible ML benchmarking
* embedded systems education
