# Implementation Plan: Government Scheme Matcher

## Overview

This implementation plan breaks down the Government Scheme Matcher system into discrete, incremental coding tasks. The system is a WhatsApp-based AI chatbot that helps citizens discover and apply for government benefits through natural language conversations in Hindi and English.

The implementation follows a bottom-up approach: starting with core data models and services, then building the conversation orchestration layer, and finally integrating with external APIs (WhatsApp, Claude, Whisper). Each task builds on previous work, with property-based tests and unit tests integrated throughout to validate correctness early.

## Tasks

- [ ] 1. Set up project structure and dependencies
  - Create Python project with virtual environment
  - Install core dependencies: FastAPI, SQLAlchemy, Redis, Motor (MongoDB), Hypothesis, pytest
  - Install external API clients: openai (Whisper), anthropic (Claude), requests (WhatsApp)
  - Set up configuration management for API keys and database connections
  - Create directory structure: `src/`, `tests/unit/`, `tests/property/`, `tests/integration/`, `tests/fixtures/`
  - _Requirements: All (foundational)_

- [ ] 2. Implement data models and database schema
  - [ ] 2.1 Create SQLAlchemy models for PostgreSQL
    - Define User model with all profile fields (phone, name, age, gender, state, district, occupation, income_category, disability_status, caste_category, education_level, family_size, language)
    - Define Scheme model with eligibility_criteria as JSONB field
    - Define Application model with form_data as JSONB field
    - Add indexes on phone, state, district, eligibility_criteria
    - _Requirements: 1.3, 1.4, 4.2, 4.3, 9.2, 9.3_

  - [ ]* 2.2 Write property test for user model validation
    - **Property 2: User Account Creation**
    - **Validates: Requirements 1.4**

  - [ ] 2.3 Create Pydantic models for data validation
    - Define UserProfile, Scheme, Application, Session, FormField, TranscriptionResult classes
    - Add field validators for phone numbers, age ranges, income categories
    - _Requirements: 1.2, 1.3, 4.2, 8.5_

  - [ ]* 2.4 Write property test for phone number validation
    - **Property 1: Phone Number Validation**
    - **Validates: Requirements 1.2**

  - [ ] 2.5 Set up database migrations with Alembic
    - Create initial migration for users, schemes, applications tables
    - Add seed data script for 100+ sample schemes
    - _Requirements: 4.1, 4.2_

- [ ] 3. Implement Authentication Service
  - [ ] 3.1 Create AuthService class with user CRUD operations
    - Implement `authenticate_user(phone)` to lookup user by phone
    - Implement `register_user(phone, profile_data)` to create new user
    - Implement `update_profile(phone, updates)` to modify user data
    - Implement `validate_phone(phone)` for Indian phone format
    - Add encryption for sensitive fields using cryptography library
    - _Requirements: 1.2, 1.4, 1.5, 1.6, 13.1_

  - [ ]* 3.2 Write property tests for authentication
    - **Property 3: Authentication Success for Registered Users**
    - **Property 4: Authentication Failure Triggers Registration**
    - **Validates: Requirements 1.5, 1.6**

  - [ ]* 3.3 Write unit tests for authentication edge cases
    - Test duplicate phone registration
    - Test invalid phone formats
    - Test profile update with partial data
    - _Requirements: 1.2, 1.4, 1.6_

- [ ] 4. Implement Scheme Matching Engine
  - [ ] 4.1 Create SchemeMatchingEngine class
    - Implement `find_eligible_schemes(user_profile)` with eligibility logic
    - Implement `check_eligibility(user_profile, scheme_id)` for single scheme
    - Implement `calculate_relevance_score(user_profile, scheme)` with weighted scoring
    - Add eligibility matching for age, income, location, occupation, gender, disability, caste
    - Add Redis caching for scheme data
    - _Requirements: 5.1, 5.2, 5.3, 5.6_

  - [ ]* 4.2 Write property test for eligibility matching correctness
    - **Property 12: Eligibility Matching Correctness**
    - **Validates: Requirements 5.2**

  - [ ]* 4.3 Write property test for scheme ranking
    - **Property 13: Scheme Ranking by Relevance**
    - **Validates: Requirements 5.3**

  - [ ]* 4.4 Write property test for profile update re-matching
    - **Property 14: Profile Update Triggers Re-matching**
    - **Validates: Requirements 5.6**

  - [ ]* 4.5 Write unit tests for matching edge cases
    - Test user with no matching schemes
    - Test user matching all schemes
    - Test state-specific vs central schemes
    - Test complex multi-condition eligibility
    - _Requirements: 5.1, 5.2, 5.3_

- [ ] 5. Checkpoint - Ensure core services pass tests
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Implement NLP Service with Claude API
  - [ ] 6.1 Create NLPService class
    - Implement `extract_intent(message, language, context)` using Claude API
    - Implement `extract_entities(message, language)` for profile fields
    - Implement `generate_response(intent, data, language)` for bot messages
    - Implement `simplify_text(technical_text, language)` for jargon removal
    - Add system prompts optimized for government scheme domain
    - Add fallback to keyword matching if Claude API fails
    - Add Redis caching for common responses
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6_

  - [ ]* 6.2 Write property test for intent extraction
    - **Property 15: Intent Extraction from Messages**
    - **Validates: Requirements 6.1**

  - [ ]* 6.3 Write property test for scheme inquiry handling
    - **Property 16: Scheme Inquiry Triggers Search**
    - **Validates: Requirements 6.2**

  - [ ]* 6.4 Write property test for unclear intent clarification
    - **Property 17: Unclear Intent Triggers Clarification**
    - **Validates: Requirements 6.3**

  - [ ]* 6.5 Write property test for misspelling tolerance
    - **Property 18: Misspelling Tolerance**
    - **Validates: Requirements 6.6**

  - [ ]* 6.6 Write unit tests for NLP edge cases
    - Test Claude API timeout handling
    - Test fallback to keyword matching
    - Test code-mixing (Hindi-English)
    - Test response caching
    - _Requirements: 6.1, 6.6, 12.1_

- [ ] 7. Implement Voice Service with Whisper API
  - [ ] 7.1 Create VoiceService class
    - Implement `transcribe_audio(audio_url, language)` using Whisper API
    - Implement `detect_language(audio_url)` for language detection
    - Add audio file download from WhatsApp CDN
    - Add audio format conversion (to MP3, 16kHz)
    - Add retry logic for low-confidence transcriptions
    - Store transcriptions in MongoDB for quality monitoring
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ]* 7.2 Write property test for voice transcription processing
    - **Property 6: Voice Transcription Processing**
    - **Validates: Requirements 3.2, 3.3**

  - [ ]* 7.3 Write property test for transcription failure recovery
    - **Property 7: Transcription Failure Recovery**
    - **Validates: Requirements 3.4**

  - [ ]* 7.4 Write unit tests for voice processing edge cases
    - Test empty audio file
    - Test corrupted audio file
    - Test very short audio (<1 second)
    - Test very long audio (>5 minutes)
    - Test Whisper API failure handling
    - _Requirements: 3.1, 3.4, 12.1_

- [ ] 8. Implement Form Filler
  - [ ] 8.1 Create FormFiller class
    - Implement `start_application(user_profile, scheme_id)` to initiate application
    - Implement `auto_fill_fields(user_profile, form_fields)` with field mapping
    - Implement `collect_missing_field(field, language)` for conversational prompts
    - Implement `validate_field(field, value)` with validation rules
    - Implement `generate_summary(application_data, language)` for confirmation
    - Define PROFILE_TO_FORM_MAPPING dictionary for field mapping
    - Store partial application progress in Redis
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 12.5_

  - [ ]* 8.2 Write property test for form field auto-filling
    - **Property 25: Form Field Auto-filling**
    - **Validates: Requirements 8.2, 8.3, 8.4**

  - [ ]* 8.3 Write property test for form field validation
    - **Property 26: Form Field Validation**
    - **Property 27: Validation Failure Explanation**
    - **Validates: Requirements 8.5, 8.6**

  - [ ]* 8.4 Write property test for application summary generation
    - **Property 28: Application Summary Generation**
    - **Validates: Requirements 8.7**

  - [ ]* 8.5 Write unit tests for form filling edge cases
    - Test form with all auto-fillable fields
    - Test form with no auto-fillable fields
    - Test form with optional fields
    - Test partial progress saving and resumption
    - _Requirements: 8.1, 8.2, 8.4, 12.5_

- [ ] 9. Checkpoint - Ensure all services are integrated
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Implement Session and Context Management
  - [ ] 10.1 Create Session class and Redis session store
    - Define Session model with user_phone, language, current_flow, context, message_history
    - Implement session CRUD operations with Redis (24-hour TTL)
    - Implement message history storage (last 20 messages)
    - Implement context update and retrieval methods
    - _Requirements: 15.1, 15.2, 15.3, 15.5, 12.3_

  - [ ]* 10.2 Write property test for conversation context maintenance
    - **Property 42: Conversation Context Maintenance**
    - **Validates: Requirements 15.1**

  - [ ]* 10.3 Write property test for pronoun reference resolution
    - **Property 43: Pronoun Reference Resolution**
    - **Validates: Requirements 15.2**

  - [ ]* 10.4 Write property test for context restoration
    - **Property 44: Context Restoration on Return**
    - **Validates: Requirements 15.3**

  - [ ]* 10.5 Write property test for message history limit
    - **Property 46: Message History Limit**
    - **Validates: Requirements 15.5**

- [ ] 11. Implement Conversation Orchestrator
  - [ ] 11.1 Create ConversationOrchestrator class
    - Implement `process_message(user_phone, message_content, message_type)` as main entry point
    - Implement `get_session(user_phone)` to retrieve or create session
    - Implement `update_context(user_phone, key, value)` for context updates
    - Implement `determine_intent(user_message, context)` using NLP service
    - Implement conversation flow state machines for registration, scheme search, application
    - Add intent routing to appropriate handlers
    - Add topic switching detection
    - _Requirements: 6.1, 6.3, 15.1, 15.4_

  - [ ]* 11.2 Write property test for language preference consistency
    - **Property 5: Language Preference Consistency**
    - **Validates: Requirements 2.2, 2.3, 2.4, 2.5**

  - [ ]* 11.3 Write property test for topic switch recognition
    - **Property 45: Topic Switch Recognition**
    - **Validates: Requirements 15.4**

  - [ ] 11.4 Implement registration conversation flow
    - Create handler for REGISTER intent
    - Implement conversational prompts for name, age, state, district, occupation, income
    - Implement language selection at start of registration
    - Call AuthService to create user after collecting all fields
    - Trigger scheme matching after registration completes
    - _Requirements: 1.1, 1.3, 1.4, 2.1, 2.2, 5.1_

  - [ ] 11.5 Implement scheme search and details flow
    - Create handler for SEARCH_SCHEMES intent
    - Call SchemeMatchingEngine to find eligible schemes
    - Format and present top 10 matches with simple descriptions
    - Create handler for GET_SCHEME_DETAILS intent
    - Retrieve and display complete scheme information
    - Handle document list and application process queries
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 6.2, 7.1, 7.2, 7.3, 7.4_

  - [ ]* 11.6 Write property tests for scheme retrieval
    - **Property 19: Complete Scheme Details Retrieval**
    - **Property 20: Scheme Display Completeness**
    - **Property 21: Document List Completeness**
    - **Property 22: Application Process Guidance**
    - **Validates: Requirements 7.1, 7.2, 7.3, 7.4, 5.4**

  - [ ] 11.7 Implement application submission flow
    - Create handler for START_APPLICATION intent
    - Call FormFiller to initiate application process
    - Collect missing form fields conversationally
    - Validate each field and explain errors
    - Generate and present application summary
    - Handle user confirmation and create application record
    - Generate unique application ID and send confirmation
    - _Requirements: 8.1, 8.4, 8.5, 8.6, 8.7, 9.1, 9.2, 9.3, 9.4_

  - [ ]* 11.8 Write property tests for application workflow
    - **Property 24: Application Initiation**
    - **Property 29: Application Submission Workflow**
    - **Property 30: Application ID Uniqueness**
    - **Validates: Requirements 8.1, 9.1, 9.2, 9.3, 9.4**

  - [ ] 11.9 Implement application status tracking flow
    - Create handler for CHECK_STATUS intent
    - Retrieve all applications for user from database
    - Format and display application status information
    - Handle case when user has no applications
    - _Requirements: 10.1, 10.2, 10.5_

  - [ ]* 11.10 Write property tests for status tracking
    - **Property 31: Application Status Retrieval**
    - **Property 32: Application Status Display Format**
    - **Validates: Requirements 10.1, 10.2**

- [ ] 12. Checkpoint - Ensure conversation flows work correctly
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 13. Implement Message Gateway and WhatsApp integration
  - [ ] 13.1 Create MessageGateway class
    - Implement `handle_webhook(webhook_payload)` to process incoming messages
    - Implement webhook signature validation
    - Extract message type (text, voice, image) and content
    - Implement `send_message(user_phone, message, language)` using WhatsApp Business API
    - Implement `send_voice_message(user_phone, audio_url)` for voice responses
    - Add message splitting for messages >1600 characters
    - Add message queueing in Redis for async processing
    - Add exponential backoff retry logic for failed API calls
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5, 11.6, 7.5_

  - [ ]* 13.2 Write property test for message splitting
    - **Property 23: Message Splitting for Length Limits**
    - **Validates: Requirements 7.5, 11.6**

  - [ ]* 13.3 Write property test for message queueing on failure
    - **Property 34: Message Queueing on API Failure**
    - **Validates: Requirements 11.5**

  - [ ]* 13.4 Write unit tests for WhatsApp integration
    - Test webhook signature validation
    - Test message type extraction
    - Test WhatsApp API rate limit handling
    - Test retry logic with exponential backoff
    - _Requirements: 11.1, 11.2, 11.5_

- [ ] 14. Implement comprehensive error handling
  - [ ] 14.1 Add error handling for external API failures
    - Implement retry logic with exponential backoff for Claude API
    - Implement fallback to keyword matching on Claude failure
    - Implement retry logic for Whisper API
    - Implement user-friendly error messages for all API failures
    - Add error logging with timestamps and details
    - _Requirements: 12.1, 12.2, 12.4_

  - [ ]* 14.2 Write property tests for error handling
    - **Property 35: External API Error Handling**
    - **Property 36: Persistent Unclear Intent Escalation**
    - **Property 37: Session Context Preservation**
    - **Property 38: Application Progress Saving**
    - **Validates: Requirements 12.1, 12.2, 12.3, 12.4, 12.5**

  - [ ] 14.3 Add database failure handling
    - Implement fallback to Redis cache when PostgreSQL unavailable
    - Implement write operation queueing for retry
    - Implement graceful degradation when Redis unavailable
    - Add monitoring alerts for persistent failures
    - _Requirements: 12.4_

  - [ ]* 14.4 Write unit tests for error scenarios
    - Test Claude API timeout
    - Test Whisper API failure
    - Test WhatsApp API rate limit
    - Test PostgreSQL connection failure
    - Test Redis unavailability
    - _Requirements: 12.1, 12.4_

- [ ] 15. Implement data privacy and security features
  - [ ] 15.1 Add encryption for sensitive data
    - Implement AES-256 encryption for name, phone, income, documents fields
    - Add encryption/decryption methods to User model
    - Ensure all external API calls use HTTPS/TLS
    - _Requirements: 13.1, 13.5_

  - [ ]* 15.2 Write property tests for security
    - **Property 39: Sensitive Data Encryption**
    - **Property 41: Encrypted API Connections**
    - **Validates: Requirements 13.1, 13.5**

  - [ ] 15.3 Implement data deletion functionality
    - Add `delete_user_data(phone)` method to AuthService
    - Implement cascade deletion across all tables
    - Add 30-day deletion compliance tracking
    - _Requirements: 13.3_

  - [ ]* 15.4 Write property test for data deletion
    - **Property 40: Data Deletion Compliance**
    - **Validates: Requirements 13.3**

- [ ] 16. Implement application status change notifications
  - [ ] 16.1 Create notification system for status changes
    - Add database trigger or application logic to detect status changes
    - Implement notification sending via WhatsApp when status changes
    - Support all status values: Submitted, Under Review, Approved, Rejected, Documents Required
    - _Requirements: 10.3, 10.4_

  - [ ]* 16.2 Write property test for status notifications
    - **Property 33: Status Change Notifications**
    - **Validates: Requirements 10.3**

- [ ] 17. Implement scheme database management features
  - [ ] 17.1 Add scheme versioning system
    - Create scheme_versions table for version history
    - Implement version creation on scheme updates
    - Store timestamp and changed fields for each version
    - _Requirements: 4.5_

  - [ ]* 17.2 Write property tests for scheme management
    - **Property 8: Scheme Data Completeness**
    - **Property 9: Eligibility Criteria Structure**
    - **Property 10: Scheme Version History**
    - **Validates: Requirements 4.2, 4.3, 4.5**

- [ ] 18. Create FastAPI application and wire all components
  - [ ] 18.1 Create main FastAPI application
    - Set up FastAPI app with CORS, logging, and middleware
    - Create webhook endpoint for WhatsApp messages
    - Create health check endpoint
    - Create admin endpoints for scheme management (optional)
    - Wire MessageGateway, ConversationOrchestrator, and all services together
    - Add dependency injection for services
    - _Requirements: All (integration)_

  - [ ] 18.2 Add application configuration
    - Create config.py with environment variables for API keys, database URLs
    - Add configuration for Redis, PostgreSQL, MongoDB connections
    - Add configuration for external API endpoints (Claude, Whisper, WhatsApp)
    - Add logging configuration
    - _Requirements: All (infrastructure)_

  - [ ] 18.3 Create startup and shutdown handlers
    - Initialize database connections on startup
    - Initialize Redis connection pool
    - Close all connections gracefully on shutdown
    - _Requirements: All (infrastructure)_

- [ ] 19. Checkpoint - Ensure end-to-end integration works
  - Ensure all tests pass, ask the user if questions arise.

- [ ]* 20. Write integration tests for complete workflows
  - [ ]* 20.1 Write integration test for registration flow
    - Test complete registration from first message to account creation
    - Test language selection and preference storage
    - Test automatic scheme matching after registration
    - _Requirements: 1.1, 1.3, 1.4, 2.1, 2.2, 5.1_

  - [ ]* 20.2 Write integration test for application submission flow
    - Test complete flow from scheme selection to application confirmation
    - Test form auto-filling and missing field collection
    - Test validation and error handling
    - Test application ID generation and storage
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 9.1, 9.2, 9.3, 9.4_

  - [ ]* 20.3 Write integration test for WhatsApp webhook processing
    - Test webhook signature validation
    - Test text message processing end-to-end
    - Test voice message processing end-to-end
    - Test response delivery via WhatsApp API
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 3.1, 3.2, 3.3_

  - [ ]* 20.4 Write integration test for session restoration
    - Test session timeout and context preservation
    - Test user return within 24 hours
    - Test context restoration and conversation continuation
    - _Requirements: 12.3, 15.3_

- [ ] 21. Create deployment configuration and documentation
  - [ ] 21.1 Create Docker configuration
    - Create Dockerfile for FastAPI application
    - Create docker-compose.yml for local development (app, PostgreSQL, Redis, MongoDB)
    - Add environment variable templates
    - _Requirements: All (deployment)_

  - [ ] 21.2 Create database seed script
    - Create script to populate 100+ sample schemes
    - Include schemes at central, state, and district levels
    - Include diverse eligibility criteria and categories
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [ ] 21.3 Write deployment documentation
    - Document environment variables and configuration
    - Document database setup and migrations
    - Document WhatsApp Business API setup
    - Document Claude and Whisper API setup
    - Create README with setup instructions
    - _Requirements: All (documentation)_

- [ ] 22. Final checkpoint - Complete system validation
  - Run all unit tests, property tests, and integration tests
  - Verify all 46 correctness properties pass
  - Verify all requirements are covered by tests
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP delivery
- Each task references specific requirements for traceability
- Property tests validate universal correctness properties across many generated inputs
- Unit tests validate specific examples, edge cases, and error conditions
- Integration tests validate complete user workflows end-to-end
- Checkpoints ensure incremental validation and provide opportunities to address issues early
- The implementation uses Python with FastAPI, SQLAlchemy, Redis, MongoDB, Hypothesis for property testing
- External APIs: WhatsApp Business API, Claude (Anthropic), Whisper (OpenAI)
- All 46 correctness properties from the design document are covered by property-based tests
