# Design Document: Government Scheme Matcher

## Overview

The Jargon-Free Government Scheme Matcher is a WhatsApp-based conversational AI system that helps citizens discover and apply for government benefits. The system architecture follows a microservices pattern with clear separation between the messaging layer (WhatsApp), conversation management, NLP processing, scheme matching logic, and data persistence.

The MVP focuses on delivering core functionality: bilingual conversational interface (Hindi/English), voice message support, intelligent scheme matching, basic form filling assistance, and application tracking. The design prioritizes simplicity, scalability, and user experience for citizens with varying literacy levels.

## Architecture

The system follows a layered architecture with these primary components:

```
┌─────────────────────────────────────────────────────────────┐
│                        WhatsApp Users                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   WhatsApp Business API                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Message Gateway                           │
│  (Webhook Handler, Message Queue, Rate Limiting)            │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                 Conversation Orchestrator                    │
│  (Session Management, Context Tracking, Flow Control)       │
└─────┬──────────┬──────────┬──────────┬─────────────────────┘
      │          │          │          │
      ▼          ▼          ▼          ▼
┌──────────┐ ┌────────┐ ┌─────────┐ ┌──────────────┐
│   NLP    │ │ Voice  │ │ Scheme  │ │    Form      │
│ Service  │ │Service │ │Matching │ │   Filler     │
│ (Claude) │ │(Whisper│ │ Engine  │ │              │
└──────────┘ └────────┘ └─────────┘ └──────────────┘
      │          │          │          │
      └──────────┴──────────┴──────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                              │
│  ┌──────────────┐  ┌──────────┐  ┌─────────────────┐      │
│  │  PostgreSQL  │  │  Redis   │  │    MongoDB      │      │
│  │  (Users,     │  │ (Session │  │  (Conversation  │      │
│  │  Schemes,    │  │  Cache)  │  │   Logs)         │      │
│  │Applications) │  └──────────┘  └─────────────────┘      │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

### Architecture Principles

1. **Stateless Services**: All business logic services are stateless; session state is managed in Redis
2. **Async Processing**: Voice transcription and scheme matching run asynchronously to avoid blocking
3. **Idempotent Operations**: All message processing is idempotent to handle WhatsApp delivery retries
4. **Graceful Degradation**: System continues operating with reduced functionality if external APIs fail
5. **Language Agnostic Core**: Business logic is language-independent; translation happens at the presentation layer

## Components and Interfaces

### 1. Message Gateway

**Responsibility**: Receives webhooks from WhatsApp Business API, validates messages, and routes them to the Conversation Orchestrator.

**Interface**:
```python
class MessageGateway:
    def handle_webhook(webhook_payload: dict) -> Response:
        """
        Processes incoming WhatsApp webhook
        Returns: HTTP 200 response to acknowledge receipt
        """
        
    def send_message(user_phone: str, message: str, language: str) -> bool:
        """
        Sends a message to user via WhatsApp API
        Returns: True if sent successfully, False otherwise
        """
        
    def send_voice_message(user_phone: str, audio_url: str) -> bool:
        """
        Sends a voice message to user
        Returns: True if sent successfully, False otherwise
        """
```

**Key Behaviors**:
- Validates webhook signatures to ensure messages are from WhatsApp
- Extracts message type (text, voice, image) and content
- Queues messages in Redis for async processing
- Implements exponential backoff for failed WhatsApp API calls
- Splits long messages (>1600 chars) into multiple WhatsApp messages

### 2. Conversation Orchestrator

**Responsibility**: Manages conversation flow, maintains session context, and coordinates between services.

**Interface**:
```python
class ConversationOrchestrator:
    def process_message(user_phone: str, message_content: str, message_type: str) -> None:
        """
        Main entry point for processing user messages
        Coordinates between NLP, matching, and response generation
        """
        
    def get_session(user_phone: str) -> Session:
        """
        Retrieves or creates user session
        Returns: Session object with conversation context
        """
        
    def update_context(user_phone: str, key: str, value: any) -> None:
        """
        Updates conversation context for a user
        """
        
    def determine_intent(user_message: str, context: dict) -> Intent:
        """
        Uses NLP service to determine user intent
        Returns: Intent enum (REGISTER, SEARCH_SCHEMES, ASK_QUESTION, etc.)
        """
```

**Session Structure**:
```python
class Session:
    user_phone: str
    language: str  # 'hi' or 'en'
    current_flow: str  # 'registration', 'scheme_search', 'application', etc.
    context: dict  # Conversation-specific data
    message_history: list[Message]  # Last 20 messages
    created_at: datetime
    last_active: datetime
```

**Key Behaviors**:
- Maintains session state in Redis with 24-hour TTL
- Implements conversation flows as state machines
- Resolves pronouns and references using message history
- Routes to appropriate handler based on intent
- Handles context switching when user changes topics

### 3. NLP Service

**Responsibility**: Processes natural language input using Claude API to extract intent and entities.

**Interface**:
```python
class NLPService:
    def extract_intent(message: str, language: str, context: dict) -> Intent:
        """
        Determines user intent from message
        Returns: Intent object with confidence score
        """
        
    def extract_entities(message: str, language: str) -> dict:
        """
        Extracts structured data from message (age, location, occupation, etc.)
        Returns: Dictionary of entity_type -> value
        """
        
    def generate_response(intent: Intent, data: dict, language: str) -> str:
        """
        Generates natural language response
        Returns: Response text in specified language
        """
        
    def simplify_text(technical_text: str, language: str) -> str:
        """
        Converts technical/jargon text to simple language
        Returns: Simplified text
        """
```

**Intent Types**:
- REGISTER: User wants to create account
- SEARCH_SCHEMES: User wants to find schemes
- GET_SCHEME_DETAILS: User wants details about specific scheme
- START_APPLICATION: User wants to apply for scheme
- CHECK_STATUS: User wants application status
- ASK_QUESTION: General question about schemes/process
- CHANGE_LANGUAGE: User wants to switch language
- UNCLEAR: Intent cannot be determined

**Key Behaviors**:
- Uses Claude API with system prompts optimized for government scheme domain
- Maintains conversation context in API calls for better understanding
- Handles code-mixing (Hindi-English mixed messages)
- Falls back to keyword matching if Claude API fails
- Caches common responses in Redis to reduce API calls

### 4. Voice Service

**Responsibility**: Converts voice messages to text using OpenAI Whisper API.

**Interface**:
```python
class VoiceService:
    def transcribe_audio(audio_url: str, language: str) -> TranscriptionResult:
        """
        Downloads audio from WhatsApp and transcribes it
        Returns: TranscriptionResult with text and confidence score
        """
        
    def detect_language(audio_url: str) -> str:
        """
        Detects language of audio message
        Returns: Language code ('hi', 'en', etc.)
        """
```

**TranscriptionResult Structure**:
```python
class TranscriptionResult:
    text: str
    language: str
    confidence: float  # 0.0 to 1.0
    duration_seconds: float
    success: bool
    error_message: str | None
```

**Key Behaviors**:
- Downloads audio files from WhatsApp CDN
- Converts audio to format compatible with Whisper (MP3, 16kHz)
- Uses Whisper's language detection if user's preferred language is uncertain
- Retries transcription once if confidence < 0.7
- Stores transcriptions in MongoDB for quality monitoring

### 5. Scheme Matching Engine

**Responsibility**: Matches users to eligible government schemes based on their profile.

**Interface**:
```python
class SchemeMatchingEngine:
    def find_eligible_schemes(user_profile: UserProfile) -> list[SchemeMatch]:
        """
        Finds all schemes user is eligible for
        Returns: List of SchemeMatch objects sorted by relevance
        """
        
    def check_eligibility(user_profile: UserProfile, scheme_id: str) -> EligibilityResult:
        """
        Checks if user is eligible for specific scheme
        Returns: EligibilityResult with pass/fail and reasons
        """
        
    def calculate_relevance_score(user_profile: UserProfile, scheme: Scheme) -> float:
        """
        Calculates relevance score for ranking
        Returns: Score from 0.0 to 1.0
        """
```

**UserProfile Structure**:
```python
class UserProfile:
    phone: str
    name: str
    age: int
    gender: str  # 'male', 'female', 'other'
    state: str
    district: str
    occupation: str
    income_category: str  # 'BPL', 'APL', 'below_5L', '5L_to_10L', 'above_10L'
    disability_status: bool
    caste_category: str  # 'general', 'obc', 'sc', 'st'
    education_level: str
    family_size: int
    language: str
```

**Scheme Structure**:
```python
class Scheme:
    id: str
    name: str
    description: str
    eligibility_criteria: dict  # Structured conditions
    required_documents: list[str]
    benefits: str
    application_process: str
    deadline: datetime | None
    level: str  # 'central', 'state', 'district'
    state: str | None
    district: str | None
    category: str  # 'agriculture', 'education', 'health', 'business', etc.
```

**Eligibility Matching Logic**:
- Age: Check if user age falls within scheme's age range
- Income: Check if user's income category qualifies
- Location: Match state/district for state/district-level schemes
- Occupation: Match occupation categories (farmer, business owner, student, etc.)
- Gender: Check gender-specific schemes
- Disability: Check disability-specific schemes
- Caste: Check caste-specific schemes

**Relevance Scoring**:
```
relevance_score = (benefit_amount_weight * 0.4) + 
                  (application_simplicity_weight * 0.3) +
                  (deadline_urgency_weight * 0.2) +
                  (category_match_weight * 0.1)
```

**Key Behaviors**:
- Queries PostgreSQL with optimized indexes on eligibility fields
- Caches scheme data in Redis for fast repeated lookups
- Returns top 10 matches to avoid overwhelming users
- Explains why user is eligible in simple terms
- Re-evaluates eligibility when profile is updated

### 6. Form Filler

**Responsibility**: Assists users in completing application forms by auto-filling from profile and collecting missing data.

**Interface**:
```python
class FormFiller:
    def start_application(user_profile: UserProfile, scheme_id: str) -> ApplicationSession:
        """
        Initiates application process for a scheme
        Returns: ApplicationSession with form fields and pre-filled data
        """
        
    def auto_fill_fields(user_profile: UserProfile, form_fields: list[FormField]) -> dict:
        """
        Auto-fills form fields from user profile
        Returns: Dictionary of field_name -> value
        """
        
    def collect_missing_field(field: FormField, language: str) -> str:
        """
        Generates conversational prompt to collect missing field
        Returns: Prompt text in specified language
        """
        
    def validate_field(field: FormField, value: str) -> ValidationResult:
        """
        Validates field value against scheme requirements
        Returns: ValidationResult with pass/fail and error message
        """
        
    def generate_summary(application_data: dict, language: str) -> str:
        """
        Creates human-readable summary of application
        Returns: Summary text for user confirmation
        """
```

**FormField Structure**:
```python
class FormField:
    name: str
    label: str
    field_type: str  # 'text', 'number', 'date', 'select', 'file'
    required: bool
    validation_rules: dict
    help_text: str
    options: list[str] | None  # For select fields
```

**Field Mapping**:
```python
PROFILE_TO_FORM_MAPPING = {
    'name': ['applicant_name', 'full_name', 'name'],
    'age': ['age', 'applicant_age'],
    'phone': ['mobile', 'phone_number', 'contact'],
    'state': ['state', 'applicant_state'],
    'district': ['district', 'applicant_district'],
    'occupation': ['occupation', 'profession'],
    'income_category': ['income', 'annual_income'],
    # ... more mappings
}
```

**Key Behaviors**:
- Maps profile fields to form fields using fuzzy matching
- Asks for missing fields one at a time conversationally
- Validates each field immediately and explains errors
- Stores partial progress in Redis to allow resumption
- Generates final summary in bullet-point format for easy review

### 7. Authentication Service

**Responsibility**: Manages user authentication and registration.

**Interface**:
```python
class AuthService:
    def authenticate_user(phone: str) -> User | None:
        """
        Authenticates user by phone number
        Returns: User object if exists, None otherwise
        """
        
    def register_user(phone: str, profile_data: dict) -> User:
        """
        Creates new user account
        Returns: Created User object
        """
        
    def update_profile(phone: str, updates: dict) -> User:
        """
        Updates user profile
        Returns: Updated User object
        """
        
    def validate_phone(phone: str) -> bool:
        """
        Validates Indian phone number format
        Returns: True if valid, False otherwise
        """
```

**Key Behaviors**:
- Uses phone number as primary identifier (no passwords)
- Validates phone numbers match Indian format (10 digits, starts with 6-9)
- Stores user data in PostgreSQL with encrypted sensitive fields
- Creates audit log for profile changes
- Handles duplicate registration attempts gracefully

## Data Models

### PostgreSQL Schema

**users table**:
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone VARCHAR(10) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    age INTEGER NOT NULL,
    gender VARCHAR(10),
    state VARCHAR(100) NOT NULL,
    district VARCHAR(100) NOT NULL,
    occupation VARCHAR(100),
    income_category VARCHAR(50),
    disability_status BOOLEAN DEFAULT FALSE,
    caste_category VARCHAR(20),
    education_level VARCHAR(50),
    family_size INTEGER,
    language VARCHAR(5) DEFAULT 'en',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_state_district ON users(state, district);
```

**schemes table**:
```sql
CREATE TABLE schemes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(500) NOT NULL,
    description TEXT,
    eligibility_criteria JSONB NOT NULL,
    required_documents JSONB,
    benefits TEXT,
    application_process TEXT,
    deadline TIMESTAMP,
    level VARCHAR(20) NOT NULL,
    state VARCHAR(100),
    district VARCHAR(100),
    category VARCHAR(50),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_schemes_level ON schemes(level);
CREATE INDEX idx_schemes_state ON schemes(state);
CREATE INDEX idx_schemes_category ON schemes(category);
CREATE INDEX idx_schemes_eligibility ON schemes USING GIN(eligibility_criteria);
```

**applications table**:
```sql
CREATE TABLE applications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id VARCHAR(50) UNIQUE NOT NULL,
    user_id UUID REFERENCES users(id),
    scheme_id UUID REFERENCES schemes(id),
    form_data JSONB NOT NULL,
    status VARCHAR(50) DEFAULT 'submitted',
    submitted_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    notes TEXT
);

CREATE INDEX idx_applications_user ON applications(user_id);
CREATE INDEX idx_applications_status ON applications(status);
CREATE INDEX idx_applications_application_id ON applications(application_id);
```

### Redis Data Structures

**Session Cache**:
```
Key: session:{phone}
Value: JSON serialized Session object
TTL: 24 hours
```

**Scheme Cache**:
```
Key: scheme:{scheme_id}
Value: JSON serialized Scheme object
TTL: 1 hour
```

**Response Cache**:
```
Key: response:{intent}:{language}:{hash(context)}
Value: Generated response text
TTL: 15 minutes
```

**Message Queue**:
```
Key: message_queue
Type: List (LPUSH/RPOP)
Value: JSON serialized message payloads
```

### MongoDB Collections

**conversation_logs**:
```javascript
{
    _id: ObjectId,
    user_phone: String,
    timestamp: ISODate,
    message_type: String,  // 'user' or 'bot'
    content: String,
    intent: String,
    language: String,
    metadata: Object
}
```

**transcriptions**:
```javascript
{
    _id: ObjectId,
    user_phone: String,
    audio_url: String,
    transcribed_text: String,
    language: String,
    confidence: Number,
    duration_seconds: Number,
    timestamp: ISODate
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property 1: Phone Number Validation

*For any* phone number input, the validation function should return true if and only if it is exactly 10 digits and starts with 6, 7, 8, or 9.

**Validates: Requirements 1.2**

### Property 2: User Account Creation

*For any* complete set of required profile fields (name, age, state, district, occupation, income category), creating a user account should result in a stored user record with all provided fields.

**Validates: Requirements 1.4**

### Property 3: Authentication Success for Registered Users

*For any* registered user phone number, authentication should succeed and return the user's profile.

**Validates: Requirements 1.5**

### Property 4: Authentication Failure Triggers Registration

*For any* unregistered phone number, authentication should fail and initiate the registration flow.

**Validates: Requirements 1.6**

### Property 5: Language Preference Consistency

*For any* user with a stored language preference, all system messages should be generated in that language, and all NLP processing should use that language. When the user changes their language preference, all subsequent interactions should use the new language.

**Validates: Requirements 2.2, 2.3, 2.4, 2.5**

### Property 6: Voice Transcription Processing

*For any* successfully transcribed voice message, the resulting text should be processed identically to a text message with the same content.

**Validates: Requirements 3.2, 3.3**

### Property 7: Transcription Failure Recovery

*For any* voice transcription failure, the system should request the user to resend the message or use text input.

**Validates: Requirements 3.4**

### Property 8: Scheme Data Completeness

*For any* scheme stored in the database, it must include all required fields: name, description, eligibility criteria, required documents, application process, and benefits.

**Validates: Requirements 4.2**

### Property 9: Eligibility Criteria Structure

*For any* scheme's eligibility criteria, they must be structured as a queryable JSON object with typed fields (age_min, age_max, income_categories, states, occupations, etc.).

**Validates: Requirements 4.3**

### Property 10: Scheme Version History

*For any* update to a scheme's information, a new version entry should be created in the version history with timestamp and changed fields.

**Validates: Requirements 4.5**

### Property 11: Scheme Matching Triggers on Registration

*For any* user who completes registration, the matching engine should automatically analyze their profile against all active schemes.

**Validates: Requirements 5.1**

### Property 12: Eligibility Matching Correctness

*For any* user profile and scheme, the matching engine should return the scheme as a match if and only if the user meets all mandatory eligibility criteria (age range, income category, location, occupation, gender, disability status, caste category where applicable).

**Validates: Requirements 5.2**

### Property 13: Scheme Ranking by Relevance

*For any* set of matching schemes for a user, the results should be ordered by descending relevance score, with higher-benefit and simpler-application schemes ranked higher.

**Validates: Requirements 5.3**

### Property 14: Profile Update Triggers Re-matching

*For any* update to a user's profile fields that affect eligibility (age, income, location, occupation), the matching engine should re-evaluate scheme eligibility.

**Validates: Requirements 5.6**

### Property 15: Intent Extraction from Messages

*For any* user text message, the NLP service should extract an intent classification (REGISTER, SEARCH_SCHEMES, GET_SCHEME_DETAILS, START_APPLICATION, CHECK_STATUS, ASK_QUESTION, CHANGE_LANGUAGE, or UNCLEAR).

**Validates: Requirements 6.1**

### Property 16: Scheme Inquiry Triggers Search

*For any* message classified with SEARCH_SCHEMES intent, the system should query the scheme database and return relevant results.

**Validates: Requirements 6.2**

### Property 17: Unclear Intent Triggers Clarification

*For any* message classified with UNCLEAR intent, the chatbot should respond with clarifying questions to better understand the user's needs.

**Validates: Requirements 6.3**

### Property 18: Misspelling Tolerance

*For any* message containing common misspellings or grammatical variations of known intents/entities, the NLP service should still extract the correct intent and entities.

**Validates: Requirements 6.6**

### Property 19: Complete Scheme Details Retrieval

*For any* scheme selection by a user, the system should retrieve and return all scheme fields from the database.

**Validates: Requirements 7.1**

### Property 20: Scheme Display Completeness

*For any* scheme being displayed to a user, the output should include scheme name, description, eligibility criteria, required documents, benefits, and application deadline (if applicable).

**Validates: Requirements 7.2, 5.4**

### Property 21: Document List Completeness

*For any* request for required documents for a scheme, the system should return all documents listed in the scheme's required_documents field.

**Validates: Requirements 7.3**

### Property 22: Application Process Guidance

*For any* request for application process information, the system should return the step-by-step guidance from the scheme's application_process field.

**Validates: Requirements 7.4**

### Property 23: Message Splitting for Length Limits

*For any* message exceeding 1600 characters (WhatsApp limit), the system should split it into multiple messages, each under the limit, preserving message order and readability.

**Validates: Requirements 7.5, 11.6**

### Property 24: Application Initiation

*For any* user decision to apply for a scheme, the system should create an application session and initiate the form filling flow.

**Validates: Requirements 8.1**

### Property 25: Form Field Auto-filling

*For any* application form field that maps to a user profile field, the form filler should pre-populate it with the profile value. For any field that cannot be mapped, the chatbot should prompt the user for input.

**Validates: Requirements 8.2, 8.3, 8.4**

### Property 26: Form Field Validation

*For any* form field input, the system should validate it against the field's validation rules and return success or a descriptive error message.

**Validates: Requirements 8.5**

### Property 27: Validation Failure Explanation

*For any* field validation failure, the system should provide an error explanation in simple language and request corrected information.

**Validates: Requirements 8.6**

### Property 28: Application Summary Generation

*For any* completed application form, the system should generate a human-readable summary containing all field values for user confirmation.

**Validates: Requirements 8.7**

### Property 29: Application Submission Workflow

*For any* user confirmation of an application summary, the system should: (1) generate a completed application, (2) assign a unique application ID, (3) store it with status "Submitted" and timestamp, and (4) send confirmation with the application ID to the user.

**Validates: Requirements 9.1, 9.2, 9.3, 9.4**

### Property 30: Application ID Uniqueness

*For any* two applications in the system, they must have different application IDs.

**Validates: Requirements 9.2**

### Property 31: Application Status Retrieval

*For any* user status inquiry, the system should retrieve and return all applications associated with that user's profile.

**Validates: Requirements 10.1**

### Property 32: Application Status Display Format

*For any* application status being displayed, the output should include scheme name, application ID, submission date, and current status.

**Validates: Requirements 10.2**

### Property 33: Status Change Notifications

*For any* change to an application's status field, the system should send a WhatsApp notification to the associated user.

**Validates: Requirements 10.3**

### Property 34: Message Queueing on API Failure

*For any* WhatsApp API failure when sending a message, the system should add the message to a retry queue for later delivery.

**Validates: Requirements 11.5**

### Property 35: External API Error Handling

*For any* external API failure (Claude, Whisper, WhatsApp), the system should log the error with timestamp and details, and send a user-friendly error message to the user.

**Validates: Requirements 12.1, 12.4**

### Property 36: Persistent Unclear Intent Escalation

*For any* user message that remains unclear after 3 clarification attempts, the system should offer to connect the user with human support.

**Validates: Requirements 12.2**

### Property 37: Session Context Preservation

*For any* user session that times out, the conversation context should be preserved in storage for 24 hours to allow resumption.

**Validates: Requirements 12.3**

### Property 38: Application Progress Saving

*For any* critical error during application submission, the system should save all collected form data to allow the user to resume from the same point.

**Validates: Requirements 12.5**

### Property 39: Sensitive Data Encryption

*For any* user data being stored in the database, sensitive fields (name, phone, income, documents) should be encrypted using AES-256 encryption.

**Validates: Requirements 13.1**

### Property 40: Data Deletion Compliance

*For any* user data deletion request, all personal information should be removed from all databases within 30 days.

**Validates: Requirements 13.3**

### Property 41: Encrypted API Connections

*For any* external API call (Claude, Whisper, WhatsApp), the connection should use HTTPS/TLS encryption.

**Validates: Requirements 13.5**

### Property 42: Conversation Context Maintenance

*For any* sequence of messages from a user within a session, the system should maintain context including the last 20 messages and current conversation state.

**Validates: Requirements 15.1**

### Property 43: Pronoun Reference Resolution

*For any* user message containing pronouns ("it", "that", "this") referring to a previously mentioned scheme, the system should resolve the reference using conversation history.

**Validates: Requirements 15.2**

### Property 44: Context Restoration on Return

*For any* user who returns to the conversation within 24 hours of their last message, the system should restore their previous conversation context.

**Validates: Requirements 15.3**

### Property 45: Topic Switch Recognition

*For any* user message that introduces a new topic different from the current conversation flow, the system should recognize the context change and adapt the conversation accordingly.

**Validates: Requirements 15.4**

### Property 46: Message History Limit

*For any* user, the system should store exactly the last 20 messages in their conversation history, removing older messages as new ones arrive.

**Validates: Requirements 15.5**

## Error Handling

The system implements comprehensive error handling across all components:

### External API Failures

**Claude API Failures**:
- Retry with exponential backoff (1s, 2s, 4s)
- Fall back to keyword-based intent matching if all retries fail
- Log failure details for monitoring
- Notify user: "I'm having trouble understanding right now. Could you try rephrasing?"

**Whisper API Failures**:
- Retry once after 2 seconds
- If retry fails, request user to resend or use text
- Log failure with audio metadata
- Notify user: "I couldn't hear that clearly. Could you send it again or type your message?"

**WhatsApp API Failures**:
- Queue messages in Redis with retry metadata
- Retry with exponential backoff (5s, 15s, 45s, 135s)
- Alert monitoring system if failures persist > 5 minutes
- Messages remain queued for up to 24 hours

### Database Failures

**PostgreSQL Unavailable**:
- Return cached data from Redis if available
- Queue write operations for retry
- Notify user: "We're experiencing technical difficulties. Please try again in a few minutes."
- Alert operations team immediately

**Redis Unavailable**:
- Continue operating without cache (slower but functional)
- Session state falls back to PostgreSQL
- Log all cache misses for monitoring

### Validation Errors

**Invalid User Input**:
- Explain specific validation failure in simple language
- Provide example of valid input
- Allow up to 3 retry attempts before offering human support
- Example: "Age should be a number between 1 and 120. You entered 'twenty'. Please enter your age as a number like 25."

**Incomplete Profile Data**:
- Identify missing required fields
- Ask for missing fields one at a time
- Save partial progress to allow resumption
- Example: "I need a bit more information. What is your occupation?"

### Session Management Errors

**Session Timeout**:
- Preserve context in Redis for 24 hours
- On user return, check for saved context
- Restore context and continue from last point
- Notify user: "Welcome back! Let's continue where we left off."

**Concurrent Session Conflicts**:
- Use phone number as session key (one session per user)
- Latest message overwrites session state
- Log concurrent access for monitoring

### Application Submission Errors

**Partial Form Data**:
- Save all collected fields to Redis
- Generate resume token
- Allow user to resume with "continue application"
- Notify user: "Your progress has been saved. You can continue anytime by saying 'continue application'."

**Duplicate Submission**:
- Check for existing application with same user + scheme + recent timestamp
- Prevent duplicate if submission within last 24 hours
- Notify user: "You already applied for this scheme today. Your application ID is XYZ123."

## Testing Strategy

The testing strategy employs a dual approach combining property-based testing for universal correctness guarantees and unit testing for specific examples and edge cases.

### Property-Based Testing

Property-based testing validates that universal properties hold across many generated inputs. We will use **Hypothesis** (Python) for property-based testing.

**Configuration**:
- Minimum 100 iterations per property test
- Each test tagged with: `# Feature: government-scheme-matcher, Property N: [property text]`
- Custom generators for domain objects (UserProfile, Scheme, FormField, etc.)
- Shrinking enabled to find minimal failing examples

**Key Property Tests**:

1. **Phone Validation (Property 1)**: Generate random strings and valid/invalid phone numbers, verify validation logic
2. **Eligibility Matching (Property 12)**: Generate random user profiles and schemes, verify matching logic correctness
3. **Language Consistency (Property 5)**: Generate message sequences, verify all responses use user's language
4. **Form Auto-fill (Property 25)**: Generate profiles and form schemas, verify correct field mapping
5. **Message Splitting (Property 23)**: Generate messages of varying lengths, verify splits maintain order and stay under limit
6. **Application ID Uniqueness (Property 30)**: Generate multiple applications, verify all IDs are unique
7. **Context Preservation (Property 42)**: Generate message sequences, verify last 20 messages are stored
8. **Encryption (Property 39)**: Generate user data, verify sensitive fields are encrypted in storage

**Custom Generators**:
```python
@st.composite
def user_profile(draw):
    return UserProfile(
        phone=draw(st.from_regex(r'[6-9]\d{9}')),
        name=draw(st.text(min_size=1, max_size=100)),
        age=draw(st.integers(min_value=1, max_value=120)),
        state=draw(st.sampled_from(INDIAN_STATES)),
        district=draw(st.text(min_size=1, max_size=100)),
        occupation=draw(st.sampled_from(OCCUPATIONS)),
        income_category=draw(st.sampled_from(INCOME_CATEGORIES)),
        language=draw(st.sampled_from(['hi', 'en']))
    )

@st.composite
def scheme(draw):
    return Scheme(
        name=draw(st.text(min_size=1, max_size=500)),
        eligibility_criteria=draw(eligibility_criteria()),
        level=draw(st.sampled_from(['central', 'state', 'district'])),
        # ... other fields
    )
```

### Unit Testing

Unit tests validate specific examples, edge cases, and integration points.

**Focus Areas**:

1. **Registration Flow Examples**:
   - Test complete registration with valid data
   - Test registration with missing fields
   - Test duplicate phone number registration

2. **Scheme Matching Edge Cases**:
   - User with no matching schemes
   - User matching all schemes
   - Schemes with complex eligibility (multiple conditions)
   - State-specific vs central schemes

3. **Voice Processing Edge Cases**:
   - Empty audio file
   - Corrupted audio file
   - Very short audio (<1 second)
   - Very long audio (>5 minutes)

4. **Form Filling Edge Cases**:
   - Form with all auto-fillable fields
   - Form with no auto-fillable fields
   - Form with optional fields
   - Form with conditional fields

5. **Error Handling Examples**:
   - Claude API timeout
   - WhatsApp API rate limit
   - Database connection failure
   - Invalid user input formats

6. **Integration Tests**:
   - End-to-end registration flow
   - End-to-end application submission
   - WhatsApp webhook processing
   - Session restoration after timeout

**Test Organization**:
```
tests/
├── unit/
│   ├── test_auth_service.py
│   ├── test_matching_engine.py
│   ├── test_form_filler.py
│   ├── test_nlp_service.py
│   └── test_voice_service.py
├── property/
│   ├── test_properties_auth.py
│   ├── test_properties_matching.py
│   ├── test_properties_forms.py
│   └── test_properties_conversation.py
├── integration/
│   ├── test_registration_flow.py
│   ├── test_application_flow.py
│   └── test_whatsapp_integration.py
└── fixtures/
    ├── sample_schemes.json
    ├── sample_profiles.json
    └── sample_audio.mp3
```

### Testing Best Practices

1. **Mock External APIs**: Use mocks for Claude, Whisper, and WhatsApp APIs in unit tests
2. **Test Database Isolation**: Each test uses a separate test database or transaction rollback
3. **Deterministic Tests**: Seed random generators for reproducible property tests
4. **Performance Benchmarks**: Separate performance tests for response time requirements
5. **Continuous Integration**: Run all tests on every commit, property tests on nightly builds

### Coverage Goals

- Unit test coverage: >80% of code
- Property test coverage: All 46 correctness properties
- Integration test coverage: All major user workflows
- Edge case coverage: All identified edge cases from requirements
