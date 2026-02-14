# Design Document: Voice-Based AI Assistant for Opportunity Discovery

## Overview

The Voice-Based AI Assistant is a conversational system that helps citizens discover and access opportunities in three key areas: job opportunities, government scheme eligibility, and educational programs. The system uses speech recognition for voice input, a Large Language Model (LLM) for natural language understanding and intelligent matching, and text-to-speech for voice output. The architecture is designed to be accessible, multilingual (Hindi and English), and capable of handling complex eligibility matching and personalized recommendations.

## Architecture

The system follows a pipeline architecture with the following flow:

```
Voice Input → Speech Recognition → LLM Processing → Response Generation → Text-to-Speech → Voice Output
                                          ↓
                                  Opportunity Database
                                          ↓
                                    User Profile Store
```

### High-Level Components

1. **Voice Interface Layer**: Handles speech-to-text and text-to-speech conversion
2. **LLM Processing Layer**: Performs intent classification, entity extraction, opportunity matching, and response generation
3. **Data Layer**: Manages opportunity database and user profiles
4. **Session Management**: Maintains conversation context and state

## Components and Interfaces

### 1. Speech Recognition Service

**Responsibility**: Convert spoken audio to text with language detection

**Interface**:
```typescript
interface SpeechRecognitionService {
  transcribe(audioBuffer: AudioBuffer, languageHint?: Language): Promise<TranscriptionResult>
  detectLanguage(audioBuffer: AudioBuffer): Promise<Language>
}

interface TranscriptionResult {
  text: string
  language: Language
  confidence: number
  duration: number
}

enum Language {
  HINDI = "hi",
  ENGLISH = "en"
}
```

**Implementation Notes**:
- Use a pre-trained multilingual speech recognition model (e.g., Whisper, Google Speech-to-Text)
- Apply noise filtering before transcription
- Set confidence threshold at 0.7 for accepting transcriptions
- Support streaming audio for real-time processing

### 2. Text-to-Speech Service

**Responsibility**: Convert text responses to natural-sounding speech

**Interface**:
```typescript
interface TextToSpeechService {
  synthesize(text: string, language: Language, voice?: VoiceProfile): Promise<AudioBuffer>
  getAvailableVoices(language: Language): VoiceProfile[]
}

interface VoiceProfile {
  id: string
  language: Language
  gender: "male" | "female" | "neutral"
  name: string
}
```

**Implementation Notes**:
- Use neural TTS models for natural-sounding output
- Set speaking rate to 140-160 words per minute
- Add 1-second pauses between list items
- Support SSML for controlling prosody and emphasis

### 3. LLM Service

**Responsibility**: Core intelligence for understanding queries, matching opportunities, and generating responses

**Interface**:
```typescript
interface LLMService {
  classifyIntent(query: string, context: ConversationContext): Promise<IntentClassification>
  extractEntities(query: string): Promise<ExtractedEntities>
  matchOpportunities(intent: IntentClassification, entities: ExtractedEntities, profile: UserProfile): Promise<OpportunityMatch[]>
  generateResponse(matches: OpportunityMatch[], context: ConversationContext): Promise<string>
  evaluateEligibility(scheme: GovernmentScheme, profile: UserProfile): Promise<EligibilityResult>
}

interface IntentClassification {
  category: "job" | "scheme" | "education" | "profile_update" | "clarification" | "help"
  confidence: number
  subCategory?: string
  needsClarification: boolean
}

interface ExtractedEntities {
  location?: string
  skills?: string[]
  education?: string
  experience?: string
  income?: number
  age?: number
  interests?: string[]
}

interface OpportunityMatch {
  opportunity: JobOpportunity | GovernmentScheme | EducationalProgram
  relevanceScore: number
  reasoning: string
}

interface EligibilityResult {
  eligible: boolean
  matchedCriteria: string[]
  unmatchedCriteria: string[]
  explanation: string
}
```

**Implementation Notes**:
- Use a capable LLM (e.g., GPT-4, Claude, or fine-tuned open-source model)
- Implement prompt engineering for consistent intent classification
- Use few-shot examples for entity extraction
- Implement chain-of-thought reasoning for eligibility evaluation
- Cache LLM responses for common queries to reduce latency

### 4. Opportunity Database

**Responsibility**: Store and retrieve opportunities across all three categories

**Data Models**:
```typescript
interface JobOpportunity {
  id: string
  title: string
  company: string
  location: string
  description: string
  requirements: {
    education: string[]
    skills: string[]
    experience: string
  }
  salary: {
    min: number
    max: number
    currency: string
  }
  employmentType: "full-time" | "part-time" | "contract" | "internship"
  applicationUrl: string
  deadline?: Date
  tags: string[]
}

interface GovernmentScheme {
  id: string
  name: string
  nameHindi: string
  description: string
  descriptionHindi: string
  department: string
  benefits: string[]
  eligibilityCriteria: {
    age?: { min: number, max: number }
    income?: { max: number }
    gender?: "male" | "female" | "other" | "any"
    caste?: string[]
    location?: string[]
    education?: string[]
    employment?: "employed" | "unemployed" | "any"
    [key: string]: any
  }
  requiredDocuments: string[]
  applicationProcess: string
  applicationUrl: string
  deadline?: Date
  tags: string[]
}

interface EducationalProgram {
  id: string
  name: string
  institution: string
  type: "course" | "degree" | "certification" | "scholarship" | "training"
  description: string
  duration: string
  fees: {
    amount: number
    currency: string
    scholarshipAvailable: boolean
  }
  eligibility: {
    education: string[]
    age?: { min: number, max: number }
    [key: string]: any
  }
  applicationUrl: string
  deadline?: Date
  tags: string[]
}
```

**Interface**:
```typescript
interface OpportunityDatabase {
  searchJobs(criteria: SearchCriteria): Promise<JobOpportunity[]>
  searchSchemes(criteria: SearchCriteria): Promise<GovernmentScheme[]>
  searchEducation(criteria: SearchCriteria): Promise<EducationalProgram[]>
  getById(id: string, type: OpportunityType): Promise<JobOpportunity | GovernmentScheme | EducationalProgram>
}

interface SearchCriteria {
  keywords?: string[]
  location?: string
  tags?: string[]
  limit?: number
}
```

**Implementation Notes**:
- Use a vector database for semantic search (e.g., Pinecone, Weaviate)
- Index opportunities by embeddings for better matching
- Support full-text search as fallback
- Implement caching for frequently accessed opportunities

### 5. User Profile Manager

**Responsibility**: Manage user profiles and preferences

**Data Model**:
```typescript
interface UserProfile {
  userId: string
  demographics: {
    age: number
    gender: string
    location: string
    caste?: string
  }
  education: {
    level: string
    field?: string
    institution?: string
  }
  employment: {
    status: "employed" | "unemployed" | "student" | "self-employed"
    currentRole?: string
    experience?: string
  }
  financial: {
    income?: number
    dependents?: number
  }
  preferences: {
    language: Language
    interests: string[]
    careerGoals?: string[]
  }
  history: {
    viewedOpportunities: string[]
    savedOpportunities: string[]
    appliedOpportunities: string[]
  }
  createdAt: Date
  updatedAt: Date
}
```

**Interface**:
```typescript
interface UserProfileManager {
  createProfile(userId: string, initialData: Partial<UserProfile>): Promise<UserProfile>
  getProfile(userId: string): Promise<UserProfile>
  updateProfile(userId: string, updates: Partial<UserProfile>): Promise<UserProfile>
  addToHistory(userId: string, opportunityId: string, action: "viewed" | "saved" | "applied"): Promise<void>
}
```

### 6. Conversation Manager

**Responsibility**: Maintain conversation context and state

**Data Model**:
```typescript
interface ConversationContext {
  sessionId: string
  userId: string
  language: Language
  turns: ConversationTurn[]
  currentTopic?: string
  referencedOpportunities: string[]
  lastActivity: Date
}

interface ConversationTurn {
  timestamp: Date
  userInput: string
  intent: IntentClassification
  systemResponse: string
  opportunities?: string[]
}
```

**Interface**:
```typescript
interface ConversationManager {
  createSession(userId: string, language: Language): Promise<ConversationContext>
  getContext(sessionId: string): Promise<ConversationContext>
  addTurn(sessionId: string, turn: ConversationTurn): Promise<void>
  resolveReference(sessionId: string, reference: string): Promise<string | null>
  clearContext(sessionId: string): Promise<void>
}
```

**Implementation Notes**:
- Store last 5 conversation turns for context
- Implement reference resolution for pronouns ("it", "that one", "the first one")
- Auto-clear context after 2 minutes of inactivity
- Use Redis or similar for fast session storage

### 7. Voice Assistant Orchestrator

**Responsibility**: Coordinate all components and manage the conversation flow

**Interface**:
```typescript
interface VoiceAssistantOrchestrator {
  processVoiceInput(audioBuffer: AudioBuffer, sessionId: string): Promise<AudioBuffer>
  handleTextQuery(query: string, sessionId: string): Promise<string>
}
```

**Processing Flow**:
1. Receive audio input
2. Transcribe to text using Speech Recognition Service
3. Get conversation context from Conversation Manager
4. Classify intent and extract entities using LLM Service
5. Based on intent:
   - **Job query**: Search jobs, match with profile, rank results
   - **Scheme query**: Search schemes, evaluate eligibility, rank results
   - **Education query**: Search programs, match with profile, rank results
   - **Profile update**: Update user profile
   - **Clarification**: Ask follow-up questions
6. Generate natural language response using LLM Service
7. Add turn to conversation context
8. Synthesize speech using TTS Service
9. Return audio response

## Data Models

See component-specific data models above for:
- JobOpportunity
- GovernmentScheme
- EducationalProgram
- UserProfile
- ConversationContext

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, I identified several areas where properties can be consolidated:

- **Intent Classification (3.2, 3.3, 3.4)**: These three properties about classifying job, scheme, and education queries can be combined into one comprehensive property about intent classification accuracy.
- **Response Content Completeness (4.4, 4.5, 5.5, 6.4, 10.1)**: Multiple properties check that responses contain required fields. These can be consolidated into properties per opportunity type.
- **Database Search Triggers (4.1, 5.1, 6.1)**: These three properties about triggering searches can be combined into one property about intent-to-search mapping.
- **Language Preference (2.2, 2.3)**: These can be combined into one property about language preference affecting TTS output.
- **SMS Offers (10.2, 10.3)**: These can be combined into one property about offering SMS for contact information.

### Core Properties

**Property 1: Intent Classification Accuracy**
*For any* user query containing domain-specific keywords (jobs/employment/work for jobs, schemes/benefits/subsidies for schemes, education/courses/training for education), the Intent_Classifier should categorize it into the correct category (job, scheme, or education respectively).
**Validates: Requirements 3.2, 3.3, 3.4**

**Property 2: Entity Extraction Completeness**
*For any* user query containing entity information (location, age, education, income, interests), the LLM should extract all present entities into the ExtractedEntities object with correct values.
**Validates: Requirements 3.6**

**Property 3: Profile-Based Job Matching**
*For any* job search request and user profile, the returned job opportunities should be filtered and ranked based on the profile's skills, education, experience, and location attributes.
**Validates: Requirements 4.2, 4.3**

**Property 4: Job Response Completeness**
*For any* job opportunity presented to the user, the response should contain the job title, company, location, key requirements, and salary range.
**Validates: Requirements 4.4, 4.5**

**Property 5: Eligibility Evaluation Correctness**
*For any* government scheme and user profile, the Eligibility_Matcher should correctly evaluate eligibility by comparing all profile attributes (age, income, caste, gender, location, education, employment) against the scheme's eligibility criteria, returning true if all criteria are met and false otherwise.
**Validates: Requirements 5.2, 10.4**

**Property 6: Eligibility Response Appropriateness**
*For any* scheme eligibility check, if the user is eligible, the response should confirm eligibility and explain benefits; if ineligible, the response should explain why and suggest alternatives.
**Validates: Requirements 5.3, 5.4**

**Property 7: Scheme Response Completeness**
*For any* government scheme presented to the user, the response should contain the scheme name, benefits, eligibility criteria, required documents, and application process.
**Validates: Requirements 5.5**

**Property 8: Profile-Based Education Matching**
*For any* education search request and user profile, the returned educational programs should be filtered and ranked based on the profile's education level, interests, career goals, and financial situation.
**Validates: Requirements 6.2, 6.5**

**Property 9: Scholarship Prioritization**
*For any* education search where the user profile indicates low income or financial need, programs with scholarshipAvailable=true should be ranked higher than programs without scholarships.
**Validates: Requirements 6.3**

**Property 10: Education Response Completeness**
*For any* educational program presented to the user, the response should contain the program name, institution, duration, fees, eligibility, and application deadlines.
**Validates: Requirements 6.4**

**Property 11: Urgent Need Prioritization**
*For any* opportunity search where the user profile indicates urgent needs (employment status = "unemployed" or income below poverty threshold), opportunities tagged as immediate assistance should be ranked higher in results.
**Validates: Requirements 7.4**

**Property 12: Recommendation Reasoning**
*For any* opportunity recommendation, the response should include reasoning text explaining why the opportunity is relevant to the user's situation.
**Validates: Requirements 7.5**

**Property 13: Profile Validation**
*For any* profile data input (age, location, education, etc.), if the data format is invalid (e.g., age < 0, age > 150, empty location), the system should reject the input and request correction.
**Validates: Requirements 9.3**

**Property 14: Profile Update Correctness**
*For any* profile update request specifying a field and new value, the system should modify only the specified field in the user's profile, leaving all other fields unchanged.
**Validates: Requirements 9.4**

**Property 15: Opportunity Detail Completeness**
*For any* specific opportunity request, the response should contain the opportunity name, description, eligibility criteria, and application deadline.
**Validates: Requirements 10.1**

**Property 16: SMS Offer for Contact Information**
*For any* opportunity that contains a website URL, phone number, or application URL, the response should include an offer to send this information via SMS.
**Validates: Requirements 10.2, 10.3**

**Property 17: Save Opportunity Persistence**
*For any* save opportunity request, the opportunity ID should be added to the user's savedOpportunities list in their profile, and subsequent profile queries should include this opportunity in the saved list.
**Validates: Requirements 10.5**

**Property 18: Language Preference Consistency**
*For any* user with a language preference set to Hindi or English, all TTS synthesis calls should use the user's preferred language parameter.
**Validates: Requirements 2.2, 2.3**

**Property 19: Language Detection Accuracy**
*For any* audio input in Hindi or English, the Speech_Recognizer should detect the correct language and return it in the TranscriptionResult.
**Validates: Requirements 1.3**

**Property 20: Language Switching**
*For any* conversation where the user switches from one language to another (Hindi to English or vice versa), the system should detect the language change and generate all subsequent responses in the new language.
**Validates: Requirements 11.3**

**Property 21: Opportunity Translation**
*For any* opportunity retrieved from the database, if the user's preferred language is Hindi and the opportunity has Hindi fields (nameHindi, descriptionHindi), the response should use the Hindi versions; otherwise, it should use the English versions.
**Validates: Requirements 11.4**

**Property 22: Conversation Context Retention**
*For any* conversation session, the system should maintain the last 5 conversation turns in the context, and these turns should be accessible for reference resolution.
**Validates: Requirements 12.1**

**Property 23: Pronoun Reference Resolution**
*For any* user query containing pronouns ("it", "that one", "the first one") that refer to previously mentioned opportunities, the system should resolve the pronoun to the correct opportunity ID from the conversation context.
**Validates: Requirements 12.2**

**Property 24: Context-Aware Detail Requests**
*For any* user query asking for "more details" or "tell me more", the system should provide additional information about the most recently mentioned opportunity in the conversation context.
**Validates: Requirements 12.3**

**Property 25: Topic Change Detection**
*For any* conversation where the user's query intent changes to a different category (e.g., from jobs to schemes), the system should detect the topic change and reset the referencedOpportunities list in the context.
**Validates: Requirements 12.4**

**Property 26: Session Timeout**
*For any* conversation session where the lastActivity timestamp is more than 2 minutes ago, the next user interaction should clear the conversation context and start with a greeting.
**Validates: Requirements 12.5**

### Edge Cases and Examples

The following are specific examples and edge cases that should be tested with unit tests rather than property-based tests:

**Example 1: Poor Audio Quality Handling**
When audio transcription confidence is below 0.7, the system should respond with a request to repeat the query.
**Validates: Requirements 1.4, 13.1**

**Example 2: Ambiguous Query Clarification**
When a query like "I need help" doesn't clearly indicate job, scheme, or education intent, the system should ask clarifying questions like "Are you looking for job opportunities, government schemes, or educational programs?"
**Validates: Requirements 3.5, 13.2**

**Example 3: Frustrated User Response**
When a user query contains frustration indicators (e.g., "this isn't working", "I don't understand"), the system should acknowledge their feelings and offer additional help.
**Validates: Requirements 8.4**

**Example 4: New User Onboarding**
When a user with no existing profile first interacts with the system, they should receive profile creation prompts requesting age, location, education level, employment status, and interests.
**Validates: Requirements 9.1, 9.2**

**Example 5: Profile Update Confirmation**
When a user requests to update their profile (e.g., "change my location to Mumbai"), the system should confirm the change verbally before persisting it.
**Validates: Requirements 9.5**

**Example 6: Language Preference Selection**
When a new user first interacts with the system, they should be asked "Would you like to continue in Hindi or English?"
**Validates: Requirements 11.2**

**Example 7: Technical Error Handling**
When a technical error occurs (e.g., LLM service timeout), the system should respond with an apology and suggest trying again or contacting support.
**Validates: Requirements 13.3**

**Example 8: Database Unavailability**
When the Opportunity_Database is unavailable, the system should inform the user of temporary unavailability and suggest trying later.
**Validates: Requirements 13.4**

**Example 9: Escalation to Human Support**
When the system fails to understand a user's intent after 3 consecutive attempts, it should offer to connect the user with human support.
**Validates: Requirements 13.5**

**Example 10: Adaptive Simplification**
When a user requires clarification multiple times (3+ times) on the same topic, the system should simplify its explanations further.
**Validates: Requirements 14.3**

**Example 11: Help Request**
When a user asks "how do I use this system?" or "help", the system should provide voice-based instructions and examples.
**Validates: Requirements 14.5**

## Error Handling

The system must handle various error conditions gracefully to maintain a positive user experience:

### Speech Recognition Errors

1. **Low Confidence Transcription**: When transcription confidence < 0.7, request user to repeat
2. **Language Detection Failure**: Default to user's profile language preference or ask for language selection
3. **Audio Processing Timeout**: After 5 seconds, inform user of processing issue and request retry
4. **Empty Audio Input**: Detect silence and prompt user to speak

### LLM Service Errors

1. **API Timeout**: Implement 10-second timeout with retry logic (max 2 retries)
2. **Rate Limiting**: Queue requests and inform user of brief delay
3. **Invalid Response Format**: Log error, use fallback template response
4. **Context Window Exceeded**: Truncate conversation history to last 3 turns and continue

### Database Errors

1. **Connection Failure**: Return cached results if available, otherwise inform user of temporary unavailability
2. **Query Timeout**: Implement 5-second timeout, return partial results or error message
3. **Empty Results**: Inform user no opportunities found, suggest broadening search criteria
4. **Data Integrity Issues**: Skip malformed records, log for review

### Profile Management Errors

1. **Profile Not Found**: Trigger new user onboarding flow
2. **Invalid Profile Data**: Validate on input, reject and request correction
3. **Update Conflicts**: Use last-write-wins strategy with timestamp checking
4. **Storage Failure**: Retry once, then inform user to try again later

### Text-to-Speech Errors

1. **Synthesis Failure**: Retry once with simplified text, fallback to text display if available
2. **Unsupported Language**: Fall back to English with notification
3. **Audio Playback Issues**: Provide text alternative if possible

### Conversation Context Errors

1. **Session Expired**: Clear context and start fresh conversation
2. **Reference Resolution Failure**: Ask user to clarify which opportunity they mean
3. **Context Corruption**: Reset context and apologize for confusion

### General Error Handling Strategy

- **Graceful Degradation**: System should continue functioning with reduced capabilities rather than failing completely
- **User-Friendly Messages**: All error messages should be in simple language, avoiding technical jargon
- **Logging**: All errors should be logged with context for debugging and monitoring
- **Retry Logic**: Transient errors should trigger automatic retries with exponential backoff
- **Escalation Path**: After 3 failed attempts, offer human support option

## Testing Strategy

The testing strategy employs a dual approach combining unit tests for specific examples and edge cases with property-based tests for universal correctness properties.

### Property-Based Testing

Property-based testing will validate that universal properties hold across a wide range of generated inputs. This approach is particularly valuable for catching edge cases and ensuring correctness across the input space.

**Testing Library**: Use **fast-check** for TypeScript/JavaScript or **Hypothesis** for Python implementations.

**Configuration**:
- Minimum 100 iterations per property test
- Each test must reference its design document property
- Tag format: `Feature: voice-ai-assistant, Property {number}: {property_text}`

**Property Test Coverage**:

Each of the 26 core properties listed above should have a corresponding property-based test:

1. **Intent Classification Tests**: Generate random queries with domain keywords, verify correct categorization
2. **Entity Extraction Tests**: Generate queries with various entity combinations, verify extraction accuracy
3. **Matching and Ranking Tests**: Generate random profiles and opportunities, verify matching logic
4. **Response Completeness Tests**: Generate random opportunities, verify all required fields present in responses
5. **Eligibility Tests**: Generate random profiles and schemes, verify eligibility evaluation correctness
6. **Language Tests**: Generate queries in both languages, verify language detection and preference handling
7. **Context Management Tests**: Generate conversation sequences, verify context retention and reference resolution
8. **Profile Management Tests**: Generate profile updates, verify correct field modifications

**Example Property Test Structure**:
```typescript
// Feature: voice-ai-assistant, Property 1: Intent Classification Accuracy
test('intent classification for job queries', async () => {
  await fc.assert(
    fc.asyncProperty(
      fc.record({
        query: fc.oneof(
          fc.constant('I need a job'),
          fc.constant('looking for employment'),
          fc.constant('find me work'),
          fc.string().map(s => `${s} job opportunities`)
        )
      }),
      async ({ query }) => {
        const result = await llmService.classifyIntent(query, emptyContext);
        expect(result.category).toBe('job');
      }
    ),
    { numRuns: 100 }
  );
});
```

### Unit Testing

Unit tests complement property-based tests by validating specific examples, edge cases, and error conditions that are difficult to express as universal properties.

**Unit Test Coverage**:

1. **Speech Recognition**:
   - Test specific audio samples with known transcriptions
   - Test noise filtering with controlled noise levels
   - Test language detection with bilingual samples
   - Test timeout behavior with long audio

2. **Text-to-Speech**:
   - Test specific text samples for correct audio generation
   - Test language parameter passing
   - Test pause insertion between list items
   - Test error handling for synthesis failures

3. **LLM Service**:
   - Test specific queries with expected intents
   - Test ambiguous queries requiring clarification
   - Test entity extraction with complex queries
   - Test error handling for API failures

4. **Opportunity Matching**:
   - Test specific profile-opportunity pairs with known match scores
   - Test ranking with tied relevance scores
   - Test empty result handling
   - Test database query construction

5. **Eligibility Evaluation**:
   - Test specific profile-scheme pairs with known eligibility
   - Test edge cases (boundary ages, income thresholds)
   - Test missing criteria handling
   - Test complex multi-criteria schemes

6. **Profile Management**:
   - Test profile creation with valid data
   - Test validation for invalid data (negative age, empty location)
   - Test profile updates for each field type
   - Test concurrent update handling

7. **Conversation Management**:
   - Test context retention for exactly 5 turns
   - Test reference resolution for specific pronouns
   - Test topic change detection
   - Test session timeout at 2-minute boundary

8. **Error Handling**:
   - Test each error condition listed in Error Handling section
   - Test retry logic with controlled failures
   - Test fallback behaviors
   - Test error message content

9. **Integration Tests**:
   - Test complete conversation flows (onboarding → query → response)
   - Test multi-turn conversations with context
   - Test language switching mid-conversation
   - Test end-to-end voice input to voice output

### Test Data Management

- **Synthetic Data**: Generate realistic but synthetic user profiles and opportunities for testing
- **Anonymized Data**: If using real data, ensure all PII is removed
- **Test Fixtures**: Maintain a set of standard test profiles and opportunities for consistent testing
- **Multilingual Data**: Ensure test data includes both Hindi and English content

### Testing Best Practices

1. **Isolation**: Each test should be independent and not rely on shared state
2. **Determinism**: Use seeded random generators for reproducible property tests
3. **Fast Feedback**: Unit tests should run in < 5 minutes, property tests in < 15 minutes
4. **Clear Assertions**: Each test should have clear, specific assertions
5. **Error Messages**: Test failures should provide actionable debugging information
6. **Coverage Tracking**: Aim for >80% code coverage, 100% for critical paths
7. **Continuous Integration**: Run all tests on every commit
8. **Performance Testing**: Separate performance tests from correctness tests

### Manual Testing

While automated tests cover correctness, some aspects require manual validation:

1. **Voice Quality**: Human evaluation of TTS naturalness and clarity
2. **User Experience**: Usability testing with real users
3. **Cultural Appropriateness**: Review of Hindi responses by native speakers
4. **Accessibility**: Testing with users of varying literacy levels
5. **End-to-End Flows**: Manual walkthrough of complete user journeys

