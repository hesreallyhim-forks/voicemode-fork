# Voice Commands Implementation Tickets - Small Tasks (15-30 min each)

## Implementation Overview
Breaking down the Smart STT Session Management approach into small, focused tickets.

---

## Ticket #1: Create VoiceCommandSession Class Structure
**Time Estimate:** 15 minutes
**File:** `voice_mode/voice_command_session.py`

**Description:**
Create the basic class structure with initialization and state definitions.

**Acceptance Criteria:**
- [ ] Class created with `__init__` method
- [ ] State constants defined (LISTENING, PAUSED, STOPPED)
- [ ] Configuration parameters initialized (pause_word, resume_word, stop_word)
- [ ] Transcript parts list initialized

**Implementation:**
```python
class VoiceCommandSession:
    LISTENING = "LISTENING"
    PAUSED = "PAUSED"
    STOPPED = "STOPPED"

    def __init__(self, config=None):
        self.config = config or {}
        self.pause_word = self.config.get("pause_word", "pause")
        self.resume_word = self.config.get("resume_word", "resume")
        self.stop_word = self.config.get("stop_word", "stop")
        self.transcript_parts = []
        self.state = self.LISTENING
```

**Tests Required:**
```python
def test_session_initialization():
    session = VoiceCommandSession()
    assert session.state == "LISTENING"
    assert session.transcript_parts == []
    assert session.pause_word == "pause"

def test_custom_config():
    config = {"pause_word": "hold"}
    session = VoiceCommandSession(config)
    assert session.pause_word == "hold"
```

---

## Ticket #2: Add State Transition Methods
**Time Estimate:** 20 minutes
**File:** `voice_mode/voice_command_session.py`

**Description:**
Add methods to handle state transitions with validation.

**Acceptance Criteria:**
- [ ] `set_state()` method validates transitions
- [ ] `can_transition()` helper method added
- [ ] Invalid transitions raise ValueError
- [ ] State change callbacks supported

**Implementation:**
```python
def set_state(self, new_state):
    if not self.can_transition(self.state, new_state):
        raise ValueError(f"Cannot transition from {self.state} to {new_state}")
    old_state = self.state
    self.state = new_state
    return old_state

def can_transition(self, from_state, to_state):
    valid_transitions = {
        self.LISTENING: [self.PAUSED, self.STOPPED],
        self.PAUSED: [self.LISTENING, self.STOPPED],
        self.STOPPED: []
    }
    return to_state in valid_transitions.get(from_state, [])
```

**Tests Required:**
```python
def test_valid_state_transitions():
    session = VoiceCommandSession()
    session.set_state("PAUSED")
    assert session.state == "PAUSED"

def test_invalid_state_transition():
    session = VoiceCommandSession()
    session.state = "STOPPED"
    with pytest.raises(ValueError):
        session.set_state("LISTENING")
```

---

## Ticket #3: Implement Pause Word Detection
**Time Estimate:** 25 minutes
**File:** `voice_mode/voice_command_session.py`

**Description:**
Add logic to detect pause word at the end of transcripts.

**Acceptance Criteria:**
- [ ] `detect_pause_word()` method checks for pause word
- [ ] Case-insensitive matching
- [ ] Handles variations with punctuation
- [ ] Returns cleaned text without pause word

**Implementation:**
```python
def detect_pause_word(self, text):
    if not text:
        return None, text

    # Normalize for comparison
    text_lower = text.lower().rstrip('.,!?')
    words = text_lower.split()

    if words and words[-1] == self.pause_word.lower():
        # Remove pause word from original text
        last_word_index = text.lower().rfind(self.pause_word.lower())
        cleaned_text = text[:last_word_index].rstrip()
        return True, cleaned_text

    return False, text
```

**Tests Required:**
```python
def test_pause_word_detection():
    session = VoiceCommandSession()
    detected, cleaned = session.detect_pause_word("Let me think about this pause")
    assert detected == True
    assert cleaned == "Let me think about this"

def test_pause_word_with_punctuation():
    session = VoiceCommandSession()
    detected, cleaned = session.detect_pause_word("Hold on, pause.")
    assert detected == True
    assert cleaned == "Hold on,"

def test_no_pause_word():
    session = VoiceCommandSession()
    detected, text = session.detect_pause_word("Regular text")
    assert detected == False
    assert text == "Regular text"
```

---

## Ticket #4: Implement Command Word Detection
**Time Estimate:** 20 minutes
**File:** `voice_mode/voice_command_session.py`

**Description:**
Add logic to detect resume/stop commands at the beginning of text.

**Acceptance Criteria:**
- [ ] `detect_command()` method checks for resume or stop
- [ ] Only checks first 3 words
- [ ] Case-insensitive matching
- [ ] Returns command type and remaining text

**Implementation:**
```python
def detect_command(self, text):
    if not text:
        return None, text

    words = text.lower().split()[:3]  # Check first 3 words only

    if self.resume_word.lower() in words:
        # Remove command from text
        idx = text.lower().find(self.resume_word.lower())
        remaining = text[idx + len(self.resume_word):].lstrip()
        return "RESUME", remaining

    if self.stop_word.lower() in words:
        return "STOP", ""

    return None, text
```

**Tests Required:**
```python
def test_resume_command_detection():
    session = VoiceCommandSession()
    command, remaining = session.detect_command("Resume where I left off")
    assert command == "RESUME"
    assert remaining == "where I left off"

def test_stop_command_detection():
    session = VoiceCommandSession()
    command, remaining = session.detect_command("Stop the recording")
    assert command == "STOP"

def test_no_command_detected():
    session = VoiceCommandSession()
    command, text = session.detect_command("Just regular speech")
    assert command is None
```

---

## Ticket #5: Add Main Processing Method
**Time Estimate:** 30 minutes
**File:** `voice_mode/voice_command_session.py`

**Description:**
Implement the main method that processes STT results based on current state.

**Acceptance Criteria:**
- [ ] `process_stt_result()` handles LISTENING state
- [ ] `process_stt_result()` handles PAUSED state
- [ ] Returns action taken (PAUSED, RESUMED, STOPPED, CONTINUE)
- [ ] Updates transcript_parts appropriately

**Implementation:**
```python
def process_stt_result(self, text):
    if self.state == self.LISTENING:
        pause_detected, cleaned_text = self.detect_pause_word(text)
        if pause_detected:
            if cleaned_text:  # Only add if there's content
                self.transcript_parts.append(cleaned_text)
            self.set_state(self.PAUSED)
            return "PAUSED"
        else:
            # Continue accumulating
            return "CONTINUE", text

    elif self.state == self.PAUSED:
        command, remaining = self.detect_command(text)
        if command == "RESUME":
            self.set_state(self.LISTENING)
            if remaining:  # Add any text after resume
                return "RESUMED", remaining
            return "RESUMED", None
        elif command == "STOP":
            self.set_state(self.STOPPED)
            return "STOPPED", self.get_final_transcript()
        else:
            # Ignore non-command speech in paused state
            return "WAITING", None

    return "INVALID_STATE", None
```

**Tests Required:**
```python
def test_process_pause_command():
    session = VoiceCommandSession()
    action = session.process_stt_result("Let me explain this pause")
    assert action == "PAUSED"
    assert session.state == "PAUSED"
    assert "Let me explain this" in session.transcript_parts

def test_process_resume_command():
    session = VoiceCommandSession()
    session.state = "PAUSED"
    action, text = session.process_stt_result("Resume and here's more")
    assert action == "RESUMED"
    assert text == "and here's more"
```

---

## Ticket #6: Add Transcript Management Methods
**Time Estimate:** 15 minutes
**File:** `voice_mode/voice_command_session.py`

**Description:**
Add methods to manage and retrieve transcript parts.

**Acceptance Criteria:**
- [ ] `add_transcript_part()` method adds text
- [ ] `get_final_transcript()` joins all parts
- [ ] `clear_transcript()` resets parts
- [ ] Handles empty parts gracefully

**Implementation:**
```python
def add_transcript_part(self, text):
    if text and text.strip():
        self.transcript_parts.append(text.strip())

def get_final_transcript(self):
    return " ".join(self.transcript_parts)

def clear_transcript(self):
    self.transcript_parts = []

def get_transcript_length(self):
    return sum(len(part) for part in self.transcript_parts)
```

**Tests Required:**
```python
def test_transcript_management():
    session = VoiceCommandSession()
    session.add_transcript_part("First part")
    session.add_transcript_part("Second part")
    assert session.get_final_transcript() == "First part Second part"

def test_empty_transcript_handling():
    session = VoiceCommandSession()
    session.add_transcript_part("")
    session.add_transcript_part("  ")
    assert session.get_final_transcript() == ""
```

---

## Ticket #7: Add Configuration Validation
**Time Estimate:** 20 minutes
**File:** `voice_mode/voice_command_session.py`

**Description:**
Add validation for command word configuration.

**Acceptance Criteria:**
- [ ] Validate command words are different
- [ ] Validate command words are not empty
- [ ] Validate command words don't contain each other
- [ ] Raise clear error messages

**Implementation:**
```python
def validate_config(self):
    # Check all words are provided
    if not all([self.pause_word, self.resume_word, self.stop_word]):
        raise ValueError("All command words must be non-empty")

    # Check words are unique
    words = [self.pause_word.lower(), self.resume_word.lower(), self.stop_word.lower()]
    if len(words) != len(set(words)):
        raise ValueError("Command words must be different")

    # Check words don't contain each other
    for w1 in words:
        for w2 in words:
            if w1 != w2 and w1 in w2:
                raise ValueError(f"Command word '{w1}' is contained in '{w2}'")
```

**Tests Required:**
```python
def test_valid_configuration():
    config = {"pause_word": "pause", "resume_word": "continue", "stop_word": "end"}
    session = VoiceCommandSession(config)
    session.validate_config()  # Should not raise

def test_duplicate_command_words():
    config = {"pause_word": "stop", "stop_word": "stop"}
    session = VoiceCommandSession(config)
    with pytest.raises(ValueError, match="must be different"):
        session.validate_config()
```

---

## Ticket #8: Create Integration with Config Module
**Time Estimate:** 25 minutes
**File:** `voice_mode/config.py`

**Description:**
Add voice command configuration to existing VoiceModeConfig.

**Acceptance Criteria:**
- [ ] Add voice command fields to config dataclass
- [ ] Support environment variables
- [ ] Add defaults for all fields
- [ ] Backward compatible (disabled by default)

**Implementation:**
```python
# In voice_mode/config.py
@dataclass
class VoiceModeConfig:
    # ... existing fields ...

    # Voice command configuration
    voice_commands_enabled: bool = field(
        default=False,
        metadata={"env": "VOICEMODE_COMMANDS_ENABLED"}
    )
    voice_command_pause: str = field(
        default="pause",
        metadata={"env": "VOICEMODE_COMMAND_PAUSE"}
    )
    voice_command_resume: str = field(
        default="resume",
        metadata={"env": "VOICEMODE_COMMAND_RESUME"}
    )
    voice_command_stop: str = field(
        default="stop",
        metadata={"env": "VOICEMODE_COMMAND_STOP"}
    )
    voice_command_timeout: float = field(
        default=2.0,
        metadata={"env": "VOICEMODE_COMMAND_TIMEOUT"}
    )
```

**Tests Required:**
```python
def test_voice_command_config_defaults():
    config = VoiceModeConfig()
    assert config.voice_commands_enabled == False
    assert config.voice_command_pause == "pause"

def test_voice_command_env_vars(monkeypatch):
    monkeypatch.setenv("VOICEMODE_COMMANDS_ENABLED", "true")
    monkeypatch.setenv("VOICEMODE_COMMAND_PAUSE", "hold")
    config = VoiceModeConfig()
    assert config.voice_commands_enabled == True
    assert config.voice_command_pause == "hold"
```

---

## Ticket #9: Add STT Mode Configuration
**Time Estimate:** 20 minutes
**File:** `voice_mode/voice_command_session.py`

**Description:**
Add methods to configure STT for command mode vs full mode.

**Acceptance Criteria:**
- [ ] `get_command_mode_config()` returns optimized settings
- [ ] `get_full_mode_config()` returns normal settings
- [ ] Settings include duration, VAD, and streaming options
- [ ] Configuration is returned as dict

**Implementation:**
```python
def get_command_mode_config(self):
    """STT config optimized for command detection"""
    return {
        "max_duration": self.config.get("command_timeout", 2.0),
        "vad_sensitivity": "aggressive",
        "streaming": False,  # Use burst mode
        "language": self.config.get("language", "en"),
        "temperature": 0.0,  # Deterministic for commands
    }

def get_full_mode_config(self):
    """STT config for normal transcription"""
    return {
        "max_duration": self.config.get("max_duration", 30.0),
        "vad_sensitivity": "normal",
        "streaming": True,
        "language": self.config.get("language", "en"),
        "temperature": self.config.get("temperature", 0.2),
    }
```

**Tests Required:**
```python
def test_command_mode_config():
    session = VoiceCommandSession()
    config = session.get_command_mode_config()
    assert config["max_duration"] == 2.0
    assert config["streaming"] == False

def test_full_mode_config():
    session = VoiceCommandSession()
    config = session.get_full_mode_config()
    assert config["streaming"] == True
```

---

## Ticket #10: Add Audio Feedback Hooks
**Time Estimate:** 15 minutes
**File:** `voice_mode/voice_command_session.py`

**Description:**
Add methods to determine when audio feedback should play.

**Acceptance Criteria:**
- [ ] `should_play_feedback()` method checks config
- [ ] `get_feedback_sound()` returns appropriate sound file
- [ ] Different sounds for pause/resume/stop
- [ ] Can be disabled via config

**Implementation:**
```python
def should_play_feedback(self, action):
    return self.config.get("confirmation_sound", True)

def get_feedback_sound(self, action):
    sounds = {
        "PAUSED": "sounds/pause_chime.wav",
        "RESUMED": "sounds/resume_chime.wav",
        "STOPPED": "sounds/stop_chime.wav",
    }
    return sounds.get(action, None)

def get_feedback_config(self, action):
    return {
        "sound_file": self.get_feedback_sound(action),
        "volume": self.config.get("feedback_volume", 0.5),
        "enabled": self.should_play_feedback(action),
    }
```

**Tests Required:**
```python
def test_feedback_configuration():
    session = VoiceCommandSession({"confirmation_sound": True})
    assert session.should_play_feedback("PAUSED") == True
    assert "pause" in session.get_feedback_sound("PAUSED")

def test_feedback_disabled():
    session = VoiceCommandSession({"confirmation_sound": False})
    assert session.should_play_feedback("PAUSED") == False
```

---

## Ticket #11: Create Session Factory
**Time Estimate:** 20 minutes
**File:** `voice_mode/voice_command_session.py`

**Description:**
Add factory function to create sessions from various config sources.

**Acceptance Criteria:**
- [ ] `create_session()` factory function
- [ ] Accepts config dict or VoiceModeConfig object
- [ ] Validates configuration
- [ ] Returns configured session instance

**Implementation:**
```python
def create_voice_command_session(config=None):
    """Factory function to create properly configured session"""
    if config is None:
        from voice_mode.config import get_config
        config = get_config()

    if hasattr(config, '__dict__'):  # VoiceModeConfig object
        session_config = {
            "pause_word": config.voice_command_pause,
            "resume_word": config.voice_command_resume,
            "stop_word": config.voice_command_stop,
            "command_timeout": config.voice_command_timeout,
            "confirmation_sound": config.voice_command_feedback,
        }
    else:
        session_config = config

    session = VoiceCommandSession(session_config)
    session.validate_config()
    return session
```

**Tests Required:**
```python
def test_factory_with_dict():
    config = {"pause_word": "halt"}
    session = create_voice_command_session(config)
    assert session.pause_word == "halt"

def test_factory_with_voice_mode_config():
    from voice_mode.config import VoiceModeConfig
    config = VoiceModeConfig()
    session = create_voice_command_session(config)
    assert session.pause_word == "pause"
```

---

## Ticket #12: Add Session Status Method
**Time Estimate:** 15 minutes
**File:** `voice_mode/voice_command_session.py`

**Description:**
Add method to get current session status for debugging/UI.

**Acceptance Criteria:**
- [ ] `get_status()` returns dict with all session info
- [ ] Includes state, transcript length, word count
- [ ] Includes configuration summary
- [ ] JSON serializable

**Implementation:**
```python
def get_status(self):
    transcript = self.get_final_transcript()
    return {
        "state": self.state,
        "transcript_parts": len(self.transcript_parts),
        "transcript_length": len(transcript),
        "word_count": len(transcript.split()) if transcript else 0,
        "config": {
            "pause_word": self.pause_word,
            "resume_word": self.resume_word,
            "stop_word": self.stop_word,
        },
        "can_pause": self.state == self.LISTENING,
        "can_resume": self.state == self.PAUSED,
        "can_stop": self.state in [self.LISTENING, self.PAUSED],
    }
```

**Tests Required:**
```python
def test_session_status():
    session = VoiceCommandSession()
    session.add_transcript_part("Test transcript")
    status = session.get_status()
    assert status["state"] == "LISTENING"
    assert status["word_count"] == 2
    assert status["can_pause"] == True
```

---

## Ticket #13: Integration Hook for Converse Tool
**Time Estimate:** 30 minutes
**File:** `voice_mode/tools/converse.py`

**Description:**
Add integration point in converse tool to use VoiceCommandSession.

**Acceptance Criteria:**
- [ ] Add `use_commands` parameter to converse
- [ ] Create session when enabled
- [ ] Route STT results through session
- [ ] Handle state changes appropriately

**Implementation Outline:**
```python
# In voice_mode/tools/converse.py
async def converse(
    message: str,
    wait_for_response: bool = True,
    use_commands: bool = None,  # New parameter
    **kwargs
):
    config = get_config()

    # Determine if commands should be used
    if use_commands is None:
        use_commands = config.voice_commands_enabled

    if use_commands and wait_for_response:
        from voice_mode.voice_command_session import create_voice_command_session
        session = create_voice_command_session(config)

        # Modified recording loop using session
        while session.state != session.STOPPED:
            stt_config = (session.get_command_mode_config()
                         if session.state == session.PAUSED
                         else session.get_full_mode_config())

            text = await stt_transcribe(stt_config)
            action, content = session.process_stt_result(text)

            if action == "STOPPED":
                return session.get_final_transcript()
            # ... handle other actions
```

---

## Ticket #14: Add Unit Test Suite
**Time Estimate:** 30 minutes
**File:** `tests/test_voice_command_session.py`

**Description:**
Create comprehensive test file with all unit tests.

**Acceptance Criteria:**
- [ ] All methods have at least one test
- [ ] Edge cases covered
- [ ] State transitions fully tested
- [ ] 90%+ code coverage

**Implementation:**
```python
# tests/test_voice_command_session.py
import pytest
from voice_mode.voice_command_session import VoiceCommandSession

class TestVoiceCommandSession:
    def test_initialization(self):
        # Test default initialization
        pass

    def test_state_transitions(self):
        # Test all valid and invalid transitions
        pass

    def test_pause_detection(self):
        # Test pause word detection scenarios
        pass

    def test_command_detection(self):
        # Test resume/stop detection
        pass

    # ... all other test methods
```

---

## Ticket #15: Add Integration Test
**Time Estimate:** 25 minutes
**File:** `tests/test_voice_command_integration.py`

**Description:**
Create integration test simulating full conversation flow.

**Acceptance Criteria:**
- [ ] Test full pause/resume/stop flow
- [ ] Test multiple pause/resume cycles
- [ ] Test error scenarios
- [ ] Test with actual config object

**Implementation:**
```python
def test_full_conversation_flow():
    session = VoiceCommandSession()

    # Simulate conversation
    assert session.process_stt_result("First part pause") == "PAUSED"
    assert session.state == "PAUSED"

    assert session.process_stt_result("resume and continue") == ("RESUMED", "and continue")
    assert session.state == "LISTENING"

    session.add_transcript_part("and continue")

    assert session.process_stt_result("final part pause") == "PAUSED"
    assert session.process_stt_result("stop") == ("STOPPED", _)

    final = session.get_final_transcript()
    assert "First part" in final
    assert "and continue" in final
    assert "final part" in final
```

---

## Sprint Organization

### Sprint 1 (Day 1 - 2.5 hours)
- Tickets 1-5: Core session implementation

### Sprint 2 (Day 2 - 2.5 hours)
- Tickets 6-10: Supporting features

### Sprint 3 (Day 3 - 2.5 hours)
- Tickets 11-15: Integration and testing

## Total Estimate
- 15 tickets × ~20 minutes average = 5 hours
- Can be completed by one developer in 1-2 days
- Significantly simpler than original proposals (weeks → days)