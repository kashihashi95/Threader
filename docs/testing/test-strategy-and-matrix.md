# Test Strategy & Matrix Document: Redder

## 1. Overview
The Redder testing architecture enforces verification across three distinct boundaries:
1. **Unit Tests (JVM)**: Pure logic executed fast on local JVM (<5s execution).
2. **Integration Tests (JVM + Coroutines)**: Asynchronous concurrency pipelines, backpressure, and cooperative cancellation.
3. **Instrumented / System Tests (Android Emulator / Device)**: Real Android OS lifecycle, MediaSession notification token bindings, and foreground transitions.

---

## 2. Test Execution Matrix

| Test Layer | Target Component | Test Scope & Assertions | Runner / Framework | CI Gate |
| :--- | :--- | :--- | :--- | :--- |
| **Unit** | `RedditUrlResolver` | - Shortlinks (`/s/XYZ`) resolved to canonical permalinks.<br>- Query tracking params (`?utm_source=...`) stripped.<br>- Trailing `.json` extension appended correctly. | JUnit 5 + MockWebServer | Pre-commit / Fast |
| **Unit** | `PolymorphicRedditParser` | - Correctly deserializes `t1` comment objects.<br>- Safely skips or prunes `more` pagination objects.<br>- Handles Reddit's empty reply string `""` vs nested listing array without throwing. | JUnit 5 + Kotlinx.serialization | Pre-commit / Fast |
| **Unit** | `DialogueTreePruner` | - Prunes deep branches exceeding `depthThreshold = 3`.<br>- Caps sibling replies per comment to `maxReplies = 2`.<br>- Discards `[deleted]`, `[removed]`, and AutoModerator stickies.<br>- Retains top-voted comments in descending order. | JUnit 5 | Pre-commit / Fast |
| **Unit** | `DialogueFormatter` | - Strips raw URLs, spoiler blocks `>!...<!`, tables, and markdown.<br>- Cleans usernames (e.g. `cool_coder_42` -> `"cool coder"`).<br>- Injects natural spoken prefixes (`"OP writes:"`, `"Alice replies:"`). | JUnit 5 | Pre-commit / Fast |
| **Unit** | `SpeakerVoiceAllocator` | - Guaranteed alternating voices: Parent and child never share same voice ID.<br>- OP is strictly assigned dedicated anchor voice.<br>- Negative hashcodes (`Int.MIN_VALUE`) do not throw `IndexOutOfBoundsException`. | JUnit 5 | Pre-commit / Fast |
| **Unit** | `WavUtil` | - Correct 44-byte RIFF/WAV header byte offsets.<br>- Valid little-endian byte rate ($44,100\text{ bytes/s}$ for 22.05kHz 16-bit mono).<br>- Matches audio duration calculation to byte count. | JUnit 5 | Pre-commit / Fast |
| **Integration** | `PlaybackOrchestrator` | - **Backpressure**: Producer coroutine suspends when `Channel` has 2 unconsumed items.<br>- **Zero Drift**: Producer stays idle (0% CPU) while consumer is paused.<br>- **Seek Cancellation**: `jumpTo(5)` instantly cancels in-flight inference of chunk 1, drains channel, and begins chunk 5. | `kotlinx.coroutines.test` (`runTest`, `TestScope`) | CI Task Gate |
| **Integration** | `MockTtsEngine` | - Emits mock PCM buffers with deterministic sample rates and durations.<br>- Handles concurrent cancellation without hanging or memory leaks. | `kotlinx.coroutines.test` | CI Task Gate |
| **Instrumented** | `RedderAudioService` | - Service successfully starts as `FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK`.<br>- Foreground notification attaches a valid `MediaSession` token.<br>- Lockscreen transport controls (Play/Pause/Skip) properly route into `ExoPlayer`. | AndroidX `media3-test-utils` + Robolectric / AndroidJUnit4 | CI Full Gate |
| **Instrumented** | `BecomingNoisyReceiver` | - Disconnecting audio routing broadcasts `ACTION_AUDIO_BECOMING_NOISY` and triggers `ExoPlayer.pause()`. | AndroidJUnit4 on Emulator | CI Full Gate |

---

## 3. Unit Test Example: `SpeakerVoiceAllocatorTest`

```kotlin
class SpeakerVoiceAllocatorTest {
    private val voices = listOf("voice_f1", "voice_f2", "voice_m1", "voice_m2")
    private val allocator = SpeakerVoiceAllocator(voices)

    @Test
    fun `parent and direct child comments never share the same voice`() {
        val parentAuthor = "alice_dev"
        val childAuthor = "bob_reviewer"

        val parentVoice = allocator.allocateVoice(parentAuthor, parentAuthor = null, isOp = false)
        val childVoice = allocator.allocateVoice(childAuthor, parentAuthor = parentAuthor, isOp = false)

        assertNotEquals(parentVoice, childVoice, "Direct replies must have contrasting voices")
    }

    @Test
    fun `op always receives designated op voice`() {
        val opVoice = allocator.allocateVoice("thread_starter", parentAuthor = null, isOp = true)
        assertEquals("voice_op_anchor", opVoice)
    }

    @Test
    fun `integer min value hashcode does not crash allocator`() {
        // Mock author whose hashCode returns Int.MIN_VALUE
        val pathologicalAuthor = "poly_pathological_hash_min"
        assertDoesNotThrow {
            allocator.allocateVoice(pathologicalAuthor, parentAuthor = null, isOp = false)
        }
    }
}
```

---

## 4. Concurrency Integration Test: `PlaybackOrchestratorTest`

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class PlaybackOrchestratorTest {
    private val testDispatcher = StandardTestDispatcher()
    private val testScope = TestScope(testDispatcher)

    @Test
    fun `pipeline respects capacity of 2 and suspends producer`() = testScope.runTest {
        val mockEngine = FakeTtsEngine(delayMs = 100)
        val orchestrator = PlaybackOrchestrator(mockEngine, this)

        val chunks = (0..5).map { DialogueChunk(id = "c$it", text = "Comment $it", voice = "v1") }
        orchestrator.startPipeline(chunks, startIndex = 0)

        advanceTimeBy(350) // Enough time for 3 items if unbounded

        // Only 2 items should have been synthesized and buffered
        assertEquals(2, mockEngine.synthesizedCount, "Producer must suspend when channel capacity (2) is reached")

        // Consumer consumes 1 item
        val first = orchestrator.nextChunk()
        assertEquals(0, first.index)

        advanceTimeBy(150) // Producer unblocks and synthesizes chunk 2
        assertEquals(3, mockEngine.synthesizedCount, "Consuming 1 item should unblock synthesis for next item")
    }
}
```
