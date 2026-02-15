# Requirements Document

## Introduction

The Jargon-Free Government Scheme Matcher is a WhatsApp-based AI chatbot that helps citizens discover and apply for government benefits through natural language conversations. The system simplifies the complex process of finding eligible schemes, filling applications, and tracking submissions by using voice recognition, intelligent matching, and automated form filling.

This document covers Phase 1 (MVP) requirements focusing on core functionality: WhatsApp integration, user authentication, scheme matching, basic form filling, and bilingual support (Hindi and English).

## Glossary

- **System**: The Jargon-Free Government Scheme Matcher application
- **User**: A citizen seeking government benefits (farmers, small business owners, women, students, senior citizens, differently-abled individuals)
- **Scheme**: A government benefit program with eligibility criteria and application requirements
- **Chatbot**: The conversational AI interface that interacts with users via WhatsApp
- **Matching_Engine**: The component that matches users to eligible schemes based on their profile
- **Profile**: User information including demographics, occupation, income, and other eligibility factors
- **Application**: A submission for a specific government scheme
- **WhatsApp_API**: The WhatsApp Business API used for message delivery
- **NLP_Service**: The Claude API service for natural language understanding
- **Voice_Service**: The Whisper API service for speech-to-text conversion
- **Auth_Service**: The authentication service handling user verification
- **Scheme_Database**: The database containing government scheme information
- **Form_Filler**: The component that auto-populates application forms

## Requirements

### Requirement 1: User Registration and Authentication

**User Story:** As a new user, I want to register through a conversational interface, so that I can access personalized scheme recommendations.

#### Acceptance Criteria

1. WHEN a user sends their first message to the WhatsApp number, THE System SHALL initiate a registration conversation
2. WHEN the user provides their phone number, THE Auth_Service SHALL validate it is a 10-digit Indian mobile number
3. WHEN registration is initiated, THE System SHALL collect name, age, state, district, occupation, and income category through conversational prompts
4. WHEN all required profile fields are collected, THE System SHALL create a user account and store the Profile
5. WHEN a registered user sends a message, THE Auth_Service SHALL authenticate them using their WhatsApp phone number
6. IF authentication fails, THEN THE System SHALL restart the registration process

### Requirement 2: Multi-Language Support

**User Story:** As a user who speaks Hindi or English, I want to interact in my preferred language, so that I can understand and use the system comfortably.

#### Acceptance Criteria

1. WHEN registration begins, THE System SHALL ask the user to select Hindi or English as their preferred language
2. WHEN a language is selected, THE System SHALL store the language preference in the user's Profile
3. WHEN sending messages to the user, THE System SHALL use the language stored in their Profile
4. WHEN a user sends a text message, THE NLP_Service SHALL process it in the user's preferred language
5. WHEN a user requests to change language, THE System SHALL update their Profile and switch all subsequent interactions

### Requirement 3: Voice Message Processing

**User Story:** As a user with limited literacy, I want to send voice messages instead of text, so that I can interact with the system naturally.

#### Acceptance Criteria

1. WHEN a user sends a voice message via WhatsApp, THE System SHALL receive the audio file
2. WHEN an audio file is received, THE Voice_Service SHALL convert it to text using the Whisper API
3. WHEN transcription completes, THE System SHALL process the text as if it were a text message
4. IF transcription fails, THEN THE System SHALL ask the user to resend the message or use text
5. WHEN transcription is in Hindi, THE Voice_Service SHALL accurately transcribe Hindi speech with 85% or higher accuracy

### Requirement 4: Scheme Database Management

**User Story:** As a system administrator, I want to maintain an up-to-date database of government schemes, so that users receive accurate information.

#### Acceptance Criteria

1. THE Scheme_Database SHALL store at least 100 government schemes for the MVP phase
2. WHEN a scheme is stored, THE System SHALL include scheme name, description, eligibility criteria, required documents, application process, and benefits
3. WHEN eligibility criteria are defined, THE System SHALL structure them as queryable conditions (age range, income limit, occupation, state, gender, disability status)
4. THE System SHALL support schemes at central, state, and district levels
5. WHEN scheme information is updated, THE Scheme_Database SHALL maintain version history

### Requirement 5: Intelligent Scheme Matching

**User Story:** As a user, I want to discover schemes I'm eligible for, so that I can access benefits I didn't know existed.

#### Acceptance Criteria

1. WHEN a user completes registration, THE Matching_Engine SHALL analyze their Profile against all schemes in the Scheme_Database
2. WHEN matching is performed, THE Matching_Engine SHALL return schemes where the user meets all mandatory eligibility criteria
3. WHEN multiple schemes match, THE Matching_Engine SHALL rank them by relevance score based on benefit amount and application complexity
4. WHEN presenting matches, THE System SHALL show scheme name, brief description, and estimated benefit amount in simple language
5. THE Matching_Engine SHALL achieve 90% or higher accuracy in eligibility matching
6. WHEN a user's Profile is updated, THE Matching_Engine SHALL re-evaluate scheme eligibility

### Requirement 6: Natural Language Conversation

**User Story:** As a user, I want to ask questions in natural language, so that I can get information without learning technical terms.

#### Acceptance Criteria

1. WHEN a user sends a text message, THE NLP_Service SHALL interpret the intent using Claude API
2. WHEN the intent is a scheme inquiry, THE System SHALL search the Scheme_Database and return relevant results
3. WHEN the intent is unclear, THE Chatbot SHALL ask clarifying questions to understand the user's needs
4. WHEN responding to queries, THE Chatbot SHALL use simple, jargon-free language appropriate for the user's education level
5. WHEN a user asks about eligibility, THE System SHALL explain criteria in conversational terms without technical jargon
6. THE NLP_Service SHALL handle common misspellings and grammatical variations in both Hindi and English

### Requirement 7: Scheme Information Retrieval

**User Story:** As a user, I want to get detailed information about a specific scheme, so that I can decide whether to apply.

#### Acceptance Criteria

1. WHEN a user selects a scheme from matched results, THE System SHALL retrieve complete scheme details from the Scheme_Database
2. WHEN displaying scheme details, THE System SHALL present eligibility criteria, required documents, benefits, and application deadline
3. WHEN a user asks about required documents, THE System SHALL list all documents in simple language with examples
4. WHEN a user asks about the application process, THE System SHALL provide step-by-step guidance
5. WHEN scheme information is presented, THE System SHALL break long messages into multiple WhatsApp messages for readability

### Requirement 8: Basic Form Filling Assistance

**User Story:** As a user, I want help filling application forms, so that I can complete applications without confusion.

#### Acceptance Criteria

1. WHEN a user decides to apply for a scheme, THE System SHALL initiate the application process
2. WHEN the application process starts, THE Form_Filler SHALL identify which profile fields can auto-populate form fields
3. WHEN a form field can be auto-filled, THE Form_Filler SHALL pre-populate it with data from the user's Profile
4. WHEN a form field cannot be auto-filled, THE Chatbot SHALL ask the user for the required information conversationally
5. WHEN collecting form data, THE System SHALL validate each field according to the scheme's requirements
6. WHEN validation fails, THE System SHALL explain the error in simple language and request corrected information
7. WHEN all form fields are completed, THE System SHALL present a summary for user confirmation before submission

### Requirement 9: Application Submission

**User Story:** As a user, I want to submit my application through the chatbot, so that I don't need to visit government offices.

#### Acceptance Criteria

1. WHEN a user confirms the application summary, THE System SHALL generate a completed application form
2. WHEN the application is generated, THE System SHALL assign a unique application ID
3. WHEN the application is submitted, THE System SHALL store it with status "Submitted" and timestamp
4. WHEN submission completes, THE System SHALL send the user their application ID and confirmation message via WhatsApp
5. THE System SHALL complete the application submission process within 15 minutes on average

### Requirement 10: Application Status Tracking

**User Story:** As a user, I want to check my application status, so that I know the progress of my submissions.

#### Acceptance Criteria

1. WHEN a user asks about application status, THE System SHALL retrieve all applications associated with their Profile
2. WHEN displaying application status, THE System SHALL show scheme name, application ID, submission date, and current status
3. WHEN an application status changes, THE System SHALL send a WhatsApp notification to the user
4. THE System SHALL support status values: Submitted, Under Review, Approved, Rejected, Documents Required
5. WHEN a user has no applications, THE System SHALL inform them and suggest exploring available schemes

### Requirement 11: WhatsApp Integration

**User Story:** As a user, I want to interact through WhatsApp, so that I can use a familiar platform without installing new apps.

#### Acceptance Criteria

1. THE System SHALL integrate with WhatsApp Business API for message delivery
2. WHEN a user sends a message to the registered WhatsApp number, THE WhatsApp_API SHALL deliver it to the System within 5 seconds
3. WHEN the System sends a response, THE WhatsApp_API SHALL deliver it to the user's WhatsApp within 5 seconds
4. THE System SHALL support text messages, voice messages, and image messages through WhatsApp
5. WHEN the WhatsApp_API is unavailable, THE System SHALL queue messages and retry delivery
6. THE System SHALL handle WhatsApp message size limits by splitting long responses into multiple messages

### Requirement 12: Error Handling and Recovery

**User Story:** As a user, I want the system to handle errors gracefully, so that I can continue my task even when problems occur.

#### Acceptance Criteria

1. WHEN an external API fails (Claude, Whisper, WhatsApp), THE System SHALL log the error and notify the user with a friendly message
2. WHEN the NLP_Service cannot understand a message after clarification attempts, THE System SHALL offer to connect the user with human support
3. WHEN a user's session times out, THE System SHALL preserve their conversation context for 24 hours
4. WHEN the Scheme_Database is unavailable, THE System SHALL inform the user and suggest trying again later
5. IF a critical error occurs during application submission, THEN THE System SHALL save the user's progress and allow them to resume later

### Requirement 13: Data Privacy and Security

**User Story:** As a user, I want my personal information protected, so that my data remains confidential.

#### Acceptance Criteria

1. WHEN storing user data, THE System SHALL encrypt sensitive fields (name, phone number, income, documents)
2. THE System SHALL not share user data with third parties without explicit consent
3. WHEN a user requests data deletion, THE System SHALL remove all personal information within 30 days
4. THE System SHALL comply with Indian data protection regulations
5. WHEN transmitting data to external APIs, THE System SHALL use encrypted connections (HTTPS/TLS)

### Requirement 14: Performance and Scalability

**User Story:** As a user, I want fast responses, so that I can complete tasks efficiently.

#### Acceptance Criteria

1. WHEN a user sends a text message, THE System SHALL respond within 3 seconds for 95% of requests
2. WHEN processing a voice message, THE System SHALL respond within 10 seconds for 95% of requests
3. WHEN performing scheme matching, THE Matching_Engine SHALL return results within 2 seconds
4. THE System SHALL support at least 1000 concurrent users without performance degradation
5. WHEN system load is high, THE System SHALL maintain response times within acceptable limits by queuing requests

### Requirement 15: Conversation Context Management

**User Story:** As a user, I want the chatbot to remember our conversation, so that I don't have to repeat information.

#### Acceptance Criteria

1. WHEN a user sends multiple messages, THE System SHALL maintain conversation context for the session
2. WHEN a user refers to "it" or "that scheme", THE System SHALL resolve the reference using conversation history
3. WHEN a user returns after a break, THE System SHALL restore conversation context if less than 24 hours have passed
4. WHEN switching topics, THE System SHALL recognize the context change and adapt accordingly
5. THE System SHALL store the last 20 messages of conversation history per user
