# ADR-003: Kotlin Channel Backpressure vs. Shared Ring Buffer for Synthesis Streaming

## Status
Accepted

## Date
2026-09-20

## Context
Redder streams synthesized dialogue from an on-device neural TTS engine (Producer) into an ExoPlayer playback pipeline (Consumer). Reddit threads can contain dozens or hundreds of dialogue chunks.

Running neural speech inference for an entire 100-comment thread upfront is unacceptable:
1. **Latency (TTFA)**: The user would wait 30–60 seconds before hearing the first word.
2. **Memory Exhaustion**: Storing 100 raw uncompressed audio chunks in memory consumes $>100\text{ MB}$ of RAM.
3. **Wasted Battery/Compute**: If the user listens to 3 comments and stops or skips to another post, all compute expended synthesizing comments 4 through 100 is wasted, causing severe battery drain and CPU heating.

A bounded streaming pipeline is required to produce audio *just-in-time*. Two concurrency architectures were evaluated:
1. **Custom Synchronized Ring Buffer** (e.g., circular array guarded by `ReentrantLock` and `Condition` variables).
2. **Kotlin Coroutines `Channel`** with bounded capacity and cooperative cancellation.

## Decision
We select **Kotlin Coroutines `Channel<SynthesizedChunk>(capacity = 2, onBufferOverflow = BufferOverflow.SUSPEND)`** to manage the producer-consumer synthesis pipeline.

## Alternatives Considered

### Custom Shared Ring Buffer (`ArrayBlockingQueue` or synchronized Circular Array)
- **Pros**: Familiar legacy pattern; low abstraction level.
- **Cons**:
  1. *Thread Blocking*: When the buffer is full, the producer thread must block using `Condition.await()` or `Thread.sleep()`. On mobile, blocked threads hold stack memory and complicate thread pool management.
  2. *Brittle Cancellation*: When the user taps "Skip to Comment #15" or scrubs backward, active threads must be interrupted via `Thread.interrupt()`. ONNX runtime native C++ code often does not respond cleanly to Java thread interrupts, risking deadlock or corrupted memory states.
  3. *State Synchronization*: Managing ring pointers (read head, write head, wrap-around index arithmetic) introduces subtle off-by-one and race-condition vulnerabilities during concurrent seeks.

### RxJava / Reactive Streams (`Flowable` with backpressure)
- **Pros**: Established backpressure operators (`onBackpressureBuffer(2)`).
- **Cons**: Substantial extra dependency overhead; less idiomatic in modern Kotlin-first Android codebases than Coroutines; complex error propagation across service boundaries.
- **Rejected**: Kotlin Coroutines are already standard across Jetpack libraries and Media3.

### Infinite Background Coroutine Queue with Unbounded Buffer
- **Pros**: Simple to write.
- **Cons**: No backpressure; synthesizes ahead continuously until memory exhausts or CPU overheats.
- **Rejected**: Violates memory and thermal constraints.

## Consequences

### Positive
- **Automatic Non-Blocking Backpressure**: With `capacity = 2` and `BufferOverflow.SUSPEND`, the producer coroutine automatically suspends execution at `audioChannel.send()` once 2 upcoming dialogue chunks are ready. The thread is released back to the dispatcher thread pool.
- **Instant Seek / Cooperative Cancellation**: When the user seeks or skips:
  ```kotlin
  fun jumpTo(newIndex: Int) {
      synthesisJob?.cancel() // Instantly cancels in-flight ONNX inference
      while (audioChannel.tryReceive().isSuccess) { /* Drain stale buffer */ }
      startPipeline(dialogueList, startIndex = newIndex)
  }
  ```
  Calling `Job.cancel()` terminates the active coroutine cooperatively at the next `ensureActive()` suspension point without hard thread interrupts.
- **Deterministic RAM Ceiling**: Only $N$ (currently playing) plus 2 future chunks are ever in memory at any instant. For 22.05 kHz 16-bit mono audio, 2 chunks of average duration (10s each) represent $<1\text{ MB}$ of buffered PCM data.
- **Zero Thermal Drift**: When playback is paused, the consumer stops calling `receive()`. The channel remains full, keeping the producer suspended indefinitely with **0% CPU utilization**.

### Negative / Trade-offs
- **Buffer Underrun Risk on Extreme Rapid Scrubbing**: If a user frantically taps "Next" 5 times in 1 second, the pipeline must cancel and re-synthesize from scratch. This is mitigated by a 200ms debounce on the UI skip button before triggering full synthesis restart.
- Coroutines require disciplined structured concurrency scoping (`CoroutineScope` tied to `RedderAudioService` lifecycle).
