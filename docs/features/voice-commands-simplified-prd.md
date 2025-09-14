# Voice Commands PRD - Simplified Implementation

## Executive Summary

After analyzing multiple implementation approaches for voice-controlled recording (pause/resume/stop functionality), the optimal solution leverages existing STT infrastructure rather than introducing specialized wake-word detection models. This approach dramatically reduces implementation complexity while maintaining full configurability and acceptable resource usage.

## Complexity Comparison

### Approach 1: Continuous Stop-Word Detection (Original)
**Complexity: HIGH**
- Requires constant audio buffering (2-3 second rolling buffer)
- Real-time processing of all audio through detection algorithm
- Complex transcript cleaning to remove stop words
- Word boundary detection to avoid false positives
- Multiple detection modes with different state machines
- Estimated implementation: 2000+ lines of code
- New dependencies: Audio processing libraries, possibly ML models

### Approach 2: Wake-Word Detection with Pause/Resume/Stop
**Complexity: MEDIUM**
- Requires specialized wake-word detection models
- Limited to pre-trained word sets or custom training
- Separate detection pipeline from main STT
- State machine for LISTENING/PAUSED/STOPPED
- Battery-efficient but configuration-constrained
- Estimated implementation: 800-1000 lines of code
- New dependencies: Wake-word detection library (Porcupine, etc.)

### Approach 3: Smart STT Session Management (Recommended)
**Complexity: LOW**
- Reuses existing STT infrastructure entirely
- No new models or specialized detection needed
- Simple session state management
- Full configurability with any words in any language
- Estimated implementation: 100-200 lines of code
- New dependencies: None

## Recommended Implementation

The Smart STT Session Management approach provides the best balance of simplicity, functionality, and maintainability. Here's how it works:

When the user says their configured pause word (e.g., "pause"), the system:
1. Detects the word through normal STT processing
2. Saves the current transcript to session memory
3. Ends the current expensive STT session
4. Starts a new lightweight STT session in "command mode"

During the paused state:
- The system runs STT but only processes the first 1-2 seconds of any utterance
- It checks if the utterance starts with "resume" or "stop"
- If neither command is detected, it discards the audio and continues monitoring
- This provides a 70-80% reduction in processing compared to full continuous STT

When "resume" is detected:
- The system appends a space to the saved transcript
- Returns to full STT mode
- Continues accumulating the transcript

When "stop" is detected:
- The system finalizes the complete accumulated transcript
- Sends it to the LLM
- Ends the session

## Implementation Details

```python
class VoiceCommandSession:
    def __init__(self, config):
        self.pause_word = config.get("pause_word", "pause")
        self.resume_word = config.get("resume_word", "resume")
        self.stop_word = config.get("stop_word", "stop")
        self.transcript_parts = []
        self.state = "LISTENING"

    def process_stt_result(self, text):
        if self.state == "LISTENING":
            if text.rstrip().endswith(self.pause_word):
                # Remove pause word and save transcript
                clean_text = text[:text.rfind(self.pause_word)].rstrip()
                self.transcript_parts.append(clean_text)
                self.state = "PAUSED"
                self.switch_to_command_mode()
                return "PAUSED"

        elif self.state == "PAUSED":
            # Only check first word(s) for commands
            first_words = text.lower().split()[:3]
            if self.resume_word in first_words:
                self.state = "LISTENING"
                self.switch_to_full_stt()
                return "RESUMED"
            elif self.stop_word in first_words:
                return self.finalize_transcript()

    def switch_to_command_mode(self):
        # Configure STT for short bursts
        self.stt_config.max_duration = 2.0
        self.stt_config.vad_sensitivity = "aggressive"

    def finalize_transcript(self):
        return " ".join(self.transcript_parts)
```

## Advantages of This Approach

1. **No New Dependencies**: Uses existing STT infrastructure
2. **Full Configurability**: Any words in any language work immediately
3. **Simple Implementation**: ~100 lines vs 1000+ for other approaches
4. **Maintainable**: No separate detection pipeline to maintain
5. **Predictable Behavior**: Users understand it's just STT with smart processing
6. **Progressive Enhancement**: Can add wake-word optimization later if needed

## Resource Usage During Pause

| Metric | Full STT | Command Mode STT | Savings |
|--------|----------|-----------------|---------|
| Audio Buffer | 30 seconds | 2 seconds | 93% |
| API Calls | Continuous | Every 2s (brief) | ~80% |
| Processing | Full vocabulary | First few words | ~70% |
| Network | Streaming | Burst | ~75% |

## Configuration Schema

```yaml
voice_commands:
  enabled: true
  pause_word: "pause"      # Any word/phrase
  resume_word: "resume"    # Any word/phrase
  stop_word: "stop"        # Any word/phrase
  command_timeout: 2.0     # Seconds to listen for commands
  confirmation_sound: true # Audio feedback on state change
```

## Migration Path

This approach allows for progressive enhancement:

1. **Phase 1**: Ship simple STT-based implementation
2. **Phase 2**: Monitor usage patterns and resource consumption
3. **Phase 3**: If needed, add wake-word optimization for common command choices
4. **Phase 4**: Offer wake-word as an optional performance enhancement

## Risk Assessment

**Risks:**
- Slightly higher resource usage during pause than wake-word approach
- Potential for false positives if command words appear in speech

**Mitigations:**
- Resource usage still 70-80% lower than full recording
- Choose distinctive command words by default
- Add confirmation sounds for state changes
- Allow users to customize words to avoid their common vocabulary

## Conclusion

By cleverly reusing existing STT infrastructure instead of building new detection systems, we can deliver pause/resume/stop functionality in days rather than weeks, with full configurability and minimal complexity. This pragmatic approach prioritizes shipping value quickly while maintaining the option to optimize later based on real usage data.