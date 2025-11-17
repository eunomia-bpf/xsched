# XSched Transparent Scheduling Experiment Results

This document describes the experimental reproduction of results from the XSched OSDI 2025 paper, demonstrating transparent scheduling with priority-based preemption on CUDA GPUs.

## Overview

XSched enables transparent, priority-based GPU scheduling without requiring application code modifications. This experiment demonstrates how XSched can prioritize high-priority tasks while maintaining fairness for low-priority tasks running concurrently on the same GPU.

## Hardware & Software Setup

- **Platform**: Linux with CUDA-enabled GPU
- **XSched Version**: Built from source with CUDA support
- **Test Application**: Simple vector addition benchmark (100 kernel launches per task)
- **Scheduling Policy**: HPF (Highest Priority First)

## Experimental Setup

### 1. Build XSched

```bash
# Clone and build XSched
cd xsched
git submodule update --init --recursive
make cuda
```

This will install XSched to `xsched/output` by default.

### 2. Build the Example

```bash
cd examples/Linux/1_transparent_sched
make cuda
```

## Experiments

### Experiment 1: Baseline (Single App, No XSched)

Run a single application without XSched to establish baseline performance.

```bash
./app
```

**Expected Output:**
```
Task 0 completed in 25 ms
Task 1 completed in 25 ms
Task 2 completed in 25 ms
...
```

**Result**: ~25ms per task (baseline performance)

### Experiment 2: Concurrent Execution (Two Apps, No XSched)

Run two applications concurrently without XSched to observe GPU interference.

**Terminal 1:**
```bash
./app
```

**Terminal 2:**
```bash
./app
```

**Expected Output (both terminals):**
```
Task 0 completed in 470-510 ms
Task 1 completed in 470-510 ms
Task 2 completed in 470-510 ms
...
```

**Result**: Both apps experience ~470-510ms per task (**20x slowdown**) due to GPU interference and lack of scheduling control.

### Experiment 3: XSched Priority-Based Scheduling

Run two applications with XSched using different priorities.

#### Step 1: Start XSched Server

```bash
# Start the XSched server with HPF (Highest Priority First) policy
# 50000 is the server port for xcli communication
<install_path>/bin/xserver HPF 50000
```

You should see:
```
[INFO] scheduler created with policy HPF
[INFO] HTTP server listening on port 50000
```

#### Step 2: Run High Priority Application

**Terminal 1:**
```bash
# Configure XSched for high priority
export XSCHED_SCHEDULER=GLB
export XSCHED_AUTO_XQUEUE=ON
export XSCHED_AUTO_XQUEUE_PRIORITY=1
export XSCHED_AUTO_XQUEUE_LEVEL=1
export XSCHED_AUTO_XQUEUE_THRESHOLD=16
export XSCHED_AUTO_XQUEUE_BATCH_SIZE=8
export LD_LIBRARY_PATH=<install_path>/lib:$LD_LIBRARY_PATH

./app
```

#### Step 3: Run Low Priority Application

**Terminal 2:**
```bash
# Configure XSched for low priority
export XSCHED_SCHEDULER=GLB
export XSCHED_AUTO_XQUEUE=ON
export XSCHED_AUTO_XQUEUE_PRIORITY=0
export XSCHED_AUTO_XQUEUE_LEVEL=1
export XSCHED_AUTO_XQUEUE_THRESHOLD=4
export XSCHED_AUTO_XQUEUE_BATCH_SIZE=2
export LD_LIBRARY_PATH=<install_path>/lib:$LD_LIBRARY_PATH

./app
```

#### Step 4: Monitor with XCLI (Optional)

**Terminal 3:**
```bash
# Monitor XQueue status in real-time
<install_path>/bin/xcli top -f 10
```

## Results

### Performance Comparison

| Scenario | App 1 Time | App 2 Time | Slowdown |
|----------|-----------|-----------|----------|
| Single App (Baseline) | ~25ms | N/A | 1x |
| Two Apps Without XSched | ~480ms | ~490ms | 20x |
| **XSched High Priority (Priority=1)** | **~25ms** | **N/A** | **1x** |
| **XSched Low Priority (Priority=0)** | **N/A** | **25-52ms** | **1-2x** |

### Detailed Results

#### High Priority App (Priority=1)
```
[INFO] using global scheduler
Task 0 completed in 25 ms
Task 1 completed in 25 ms
Task 2 completed in 25 ms
Task 3 completed in 25 ms
...
```

#### Low Priority App (Priority=0)
```
[INFO] using global scheduler
Task 0 completed in 51 ms
Task 1 completed in 50 ms
Task 2 completed in 51 ms
Task 3 completed in 52 ms
Task 4 completed in 36 ms
Task 5 completed in 30 ms
Task 6 completed in 25 ms
...
```

## Key Findings

✅ **XSched successfully prioritizes high-priority tasks**
- High priority app maintains baseline performance (~25ms) even with concurrent low priority workload

✅ **Low priority tasks receive fair GPU time**
- Low priority app achieves 25-52ms per task when high priority is idle
- No starvation occurs

✅ **Dramatic performance improvement**
- **19x speedup** for high priority app compared to unscheduled concurrent execution
- High priority: 480ms → 25ms
- Low priority: 490ms → 25-52ms (still better than without XSched)

✅ **Transparent operation**
- No application code changes required
- Works through environment variables and LD_LIBRARY_PATH interception

## Understanding the Configuration

### Environment Variables

- `XSCHED_SCHEDULER=GLB`: Use global scheduler (xserver)
- `XSCHED_AUTO_XQUEUE=ON`: Automatically create XQueues for CUDA streams
- `XSCHED_AUTO_XQUEUE_PRIORITY`: Task priority (higher = more important)
- `XSCHED_AUTO_XQUEUE_LEVEL`: Preemption level (1 = basic preemption)
- `XSCHED_AUTO_XQUEUE_THRESHOLD`: In-flight command limit (affects preemption granularity)
- `XSCHED_AUTO_XQUEUE_BATCH_SIZE`: Commands launched per batch

### Performance Tuning

**High Priority Configuration:**
- Higher threshold (16) and batch size (8) for better throughput
- Priority=1 ensures preemption of lower priority tasks

**Low Priority Configuration:**
- Lower threshold (4) and batch size (2) for faster preemption response
- Priority=0 yields to higher priority tasks

## Reproducing Paper Results

This experiment reproduces the key findings from the XSched OSDI 2025 paper:

1. **Transparent Scheduling**: Applications run unmodified with XSched
2. **Priority-Based Preemption**: High priority tasks maintain baseline performance
3. **Fairness**: Low priority tasks receive GPU time without starvation
4. **Efficiency**: Minimal overhead compared to baseline performance

## Troubleshooting

### Apps don't use XSched
- Verify `LD_LIBRARY_PATH` includes `<install_path>/lib`
- Check that xserver is running
- Look for `[INFO] using global scheduler` in app output

### Performance not as expected
- Ensure GPU is not being used by other processes
- Check that CUDA drivers are properly installed
- Verify GPU compute capability is supported (see main README)

### Server connection issues
- Ensure port 50000 is not in use
- Check firewall settings
- Verify xserver log for errors

## References

- XSched Paper: [OSDI 2025](https://www.usenix.org/conference/osdi25/presentation/shen-weihang)
- Main Repository: [https://github.com/XpuOS/xsched](https://github.com/XpuOS/xsched)
- Documentation: See `xsched/README.md` and `xsched/examples/README.md`

## Citation

```bibtex
@inproceedings{Shen2025xsched,
  title = {{XSched}: Preemptive Scheduling for Diverse {XPU}s},
  author = {Weihang Shen and Mingcong Han and Jialong Liu and Rong Chen and Haibo Chen},
  booktitle = {19th USENIX Symposium on Operating Systems Design and Implementation (OSDI 25)},
  year = {2025},
  pages = {671--692},
  url = {https://www.usenix.org/conference/osdi25/presentation/shen-weihang},
  publisher = {USENIX Association}
}
```
