# Design Document: AI Call Interceptor

## Overview

The AI Call Interceptor is a cloud-based system that intelligently manages incoming calls during focus periods. When Meeting Mode is active, the system intercepts calls using Twilio's Voice API, conducts a short natural conversation with the caller to understand urgency, evaluates urgency using Claude LLM, and sends a high-priority SMS notification to the user for truly urgent situations.

The system is designed for hackathon constraints with a focus on demonstrable core functionality, reliable integrations, and fail-safe behavior.

---

## Architecture

The system follows a webhook-driven, event-based architecture optimized for serverless deployment.

```mermaid
graph TB
    A[Incoming Call] --> B[Twilio Voice API]
    B --> C[Webhook Handler]
    C --> D{Meeting Mode Active?}
    D -->|No| E[Allow Call to Ring Normally]
    D -->|Yes| F[AI Conversation Flow]
    F --> G[TwiML Say + Gather (Speech)]
    G --> H[Claude Urgency Scorer]
    H --> I{Score > Threshold (0.8)?}
    I -->|No| J[Polite Call End]
    I -->|Yes| K[AWS SNS SMS Alert]


**Key Architectural Decisions:**
- **Stateless Design**: Each call interaction is self-contained for reliability
- **Webhook-Based**: Leverages Twilio's webhook model for real-time call handling
- **Cloud-Native**: Designed for AWS Lambda or similar serverless platforms
- **Fail-Safe**: Graceful degradation allows calls through if system fails

## Components and Interfaces

### 1. Webhook Handler (Entry Point)
**Responsibility**: Receives Twilio webhook requests and orchestrates call flow

**Interface**:
```python
def handle_incoming_call(request: TwilioWebhookRequest) -> TwiMLResponse:
    """
    Processes incoming call webhook from Twilio
    Returns TwiML instructions for call handling
    """
```

**Key Functions**:
- Validate Twilio webhook signatures for security
- Check Meeting Mode status from persistent storage
- Route to appropriate call handling logic
- Generate initial TwiML response

### 2. Meeting Mode Manager
**Responsibility**: Manages user's Meeting Mode state and preferences

**Interface**:
```python
class MeetingModeManager:
    def is_meeting_mode_active(user_id: str) -> bool
    def get_emergency_threshold(user_id: str) -> float
    def activate_meeting_mode(user_id: str) -> None
    def deactivate_meeting_mode(user_id: str) -> None
```

**Storage**: Simple key-value store (DynamoDB or Redis) for hackathon simplicity

### 3. AI Conversation Handler
**Responsibility**: Manages the conversational flow with callers using TwiML

**Interface**:
```python
def generate_greeting_twiml() -> TwiMLResponse
def process_caller_response(speech_result: str, confidence: float) -> TwiMLResponse"""
```

**TwiML Flow**:
1. **Greeting**: "Hi! [User] is currently in a meeting. I'm their AI assistant. Could you please tell me the purpose of your call and how urgent it is?"
2. **Gather**: Use `<Gather input="speech" timeout="30" speechTimeout="5">` for voice input
3. **Fallback**: If speech unclear, ask caller to repeat or use keypad

### 4. Urgency Scoring Engine
**Responsibility**: Analyzes caller's message using Claude API to determine urgency

**Interface**:
```python
class UrgencyScorer:
    def score_urgency(message: str) -> UrgencyScore:
        """
        Returns urgency score (0.0-1.0) and reasoning
        """
```

**Claude Integration**:
- Model: `claude-3-haiku-20240307` (fast, cost-effective for hackathon)
- Prompt engineering for consistent 0.0-1.0 scoring
- Emergency keyword detection with context awareness
- Confidence scoring for reliability

**Scoring Criteria**:
- **0.9-1.0**: Medical emergencies, accidents, immediate safety concerns
- **0.7-0.8**: Time-sensitive business matters, family emergencies
- **0.4-0.6**: Important but not urgent matters
- **0.0-0.3**: Routine calls, sales, non-urgent inquiries

### 5. Notification Service
**Responsibility**: Sends SMS notifications via AWS SNS for urgent calls

**Interface**:
```python
class NotificationService:
    def send_urgent_notification(
        user_phone: str,
        caller_info: CallerInfo,
        urgency_summary: str
    ) -> bool"""
```

**Message Format**:
```
🚨 URGENT CALL ALERT
From: +91XXXXXXXXXX
Urgency Score: 0.92
Message: Caller reports hospital emergency
Time: 2026-01-25 14:30 IST
```

### 6. Call State Manager
**Responsibility**: Tracks call progression and handles multi-step conversations

**Interface**:
```python
class CallStateManager:
    def get_call_state(call_sid: str) -> CallState
    def update_call_state(call_sid: str, state: CallState) -> None
```

**States**: `GREETING`, `GATHERING_INPUT`, `PROCESSING`, `NOTIFYING`, `COMPLETED`

## Data Models

### CallSession
```python
@dataclass
class CallSession:
    call_sid: str
    caller_number: str
    user_id: str
    transcript: Optional[str]
    urgency_score: Optional[float]
    notification_sent: bool
    created_at: datetime
```

### UrgencyScore
```python
@dataclass
class UrgencyScore:
    score: float  # 0.0 to 1.0
    confidence: float  # 0.0 to 1.0
    reasoning: str
    keywords_detected: List[str]
    emergency_indicators: List[str]
```

### UserPreferences
```python
@dataclass
@dataclass
class UserPreferences:
    user_id: str
    phone_number: str
    meeting_mode_active: bool
    emergency_threshold: float
```

### TwilioWebhookRequest
```python
@dataclass
class TwilioWebhookRequest:
    CallSid: str
    From: str
    To: str
    CallStatus: str
    SpeechResult: Optional[str]
    Confidence: Optional[float]

```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Meeting Mode State Management
*For any* user and any sequence of mode activation/deactivation operations, the system should correctly maintain and persist the Meeting Mode state, with calls being intercepted when active and passing through when inactive.
**Validates: Requirements 1.1, 1.2, 1.4, 8.1, 8.2**

### Property 2: Call Interception Flow Consistency
*For any* incoming call when Meeting Mode is active, the system should consistently execute the complete interception flow: answer immediately, provide greeting with explanation, and request caller purpose and urgency.
**Validates: Requirements 1.3, 2.1, 2.2**

### Property 3: Transcription and Retry Logic
*For any* caller speech input, the system should provide transcription with confidence scores, and when confidence is below threshold, should request clarification from the caller.
**Validates: Requirements 2.3, 2.4, 5.2, 5.3**

### Property 4: Urgency Scoring Consistency
*For any* valid transcribed message, the Urgency Scorer should return a score between 0.0 and 1.0, with emergency keywords consistently producing higher scores than non-emergency content.
**Validates: Requirements 3.1, 3.2, 3.3**

### Property 5: Threshold-Based Decision Making
*For any* urgency score and configured threshold, the system should consistently send notifications when score exceeds threshold and politely dismiss calls when score is below threshold.
**Validates: Requirements 4.1, 4.3, 8.3**

### Property 6: Notification Content Completeness
*For any* urgent call requiring notification, the SMS message should contain all required information: caller details, urgency score, message summary, and timestamp.
**Validates: Requirements 4.2, 4.4, 8.4**

### Property 7: System Reliability and Graceful Degradation
*For any* system component failure, the AI Call Interceptor should gracefully degrade by allowing calls to pass through normally rather than blocking them entirely.
**Validates: Requirements 7.3**

### Property 8: AWS SNS Integration Reliability
*For any* valid notification request with proper credentials, the Notification Service should successfully authenticate with AWS SNS and deliver the message.
**Validates: Requirements 7.2**

### Property 9: Comprehensive Logging
*For any* call interaction or system operation, appropriate logs should be generated with sufficient detail for debugging and system improvement.
**Validates: Requirements 7.5**

### Property 10: User Preference Persistence
*For any* user configuration changes (threshold adjustments, preferences), the settings should persist across system restarts and be correctly applied to subsequent calls.
**Validates: Requirements 8.5**

## Error Handling

### Call Processing Errors
- **Twilio Webhook Failures**: Return TwiML that allows call to ring through normally
- **Speech Recognition Failures**: Provide fallback to DTMF input or human operator
- **LLM API Failures**: Default to high urgency score (0.9) to err on side of caution
- **Database Unavailability**: Use in-memory fallback with conservative defaults

### Integration Failures
- **AWS SNS Failures**: Log error and attempt retry with exponential backoff
- **Claude API Rate Limits**: Implement request queuing with timeout fallbacks
- **Network Timeouts**: Graceful degradation with predefined responses

### User Experience Errors
- **Unclear Speech**: Maximum 2 retry attempts before escalating to human
- **Long Silence**: 30-second timeout with polite prompt for response
- **System Overload**: Queue management with "please hold" messaging

## Testing Strategy

### Dual Testing Approach
The system requires both unit testing and property-based testing for comprehensive coverage:

**Unit Tests** focus on:
- Specific TwiML generation examples
- AWS SNS message formatting
- Error condition handling
- Integration points between components
- Edge cases like empty transcriptions or malformed webhooks

**Property-Based Tests** focus on:
- Universal properties across all call scenarios
- Urgency scoring consistency across diverse inputs
- State management across various operation sequences
- Comprehensive input coverage through randomization

### Property-Based Testing Configuration
- **Framework**: Use `hypothesis` for Python implementation
- **Iterations**: Minimum 100 iterations per property test
- **Test Tagging**: Each property test tagged with format: **Feature: ai-call-interceptor, Property {number}: {property_text}**
- **Data Generation**: Custom generators for phone numbers, speech transcripts, urgency scenarios

### Integration Testing
- **Twilio Webhook Testing**: Use Twilio's webhook testing tools and ngrok for local development
- **AWS SNS Testing**: Use AWS SNS sandbox mode for safe testing
- **End-to-End Testing**: Automated call scenarios using Twilio's test credentials
- **Performance Testing**: Load testing with concurrent call scenarios

### Hackathon Testing Priorities
1. **Core Flow Testing**: Ensure basic interception → conversation → scoring → notification works
2. **Demo Scenario Testing**: Test specific scenarios planned for hackathon demonstration
3. **Error Recovery Testing**: Ensure system fails gracefully for demo reliability
4. **Integration Testing**: Verify all external API integrations work correctly

The testing strategy balances comprehensive coverage with hackathon time constraints, prioritizing functionality that will be demonstrated and core system reliability.