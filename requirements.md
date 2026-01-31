# Ariveon - Requirements Specification

## Project Overview

**Product Name**: Ariveon  
**Type**: AI-Driven Adaptive Learning Platform  
**Target Users**: Students, professionals, hobbyists seeking personalized learning experiences

## Problem Statement

Traditional learning platforms fail to adapt to individual learning styles, pace, and knowledge gaps. Learners face:
- Static, one-size-fits-all content that doesn't match their level
- No real-time adaptation based on performance
- Limited feedback on mistakes and misconceptions
- Inefficient learning paths that waste time on known material

## Solution

An intelligent learning platform that:
- Generates personalized learning roadmaps from natural language goals
- Adapts teaching methodology in real-time based on user responses
- Automatically detects knowledge gaps and creates remediation workflows
- Provides AI-powered tutoring with contextual feedback

## User Stories

### Epic 1: User Onboarding & Authentication

**US-1.1**: As a new user, I want to register with my email and learning preferences so that the system can personalize my experience.

**Acceptance Criteria**:
- User provides email, password, name, education level, learning style, preferred mode, and urgency
- Password is securely hashed (bcrypt, 10 rounds)
- JWT token is generated and returned upon successful registration
- User preferences are stored for workflow generation

**US-1.2**: As a returning user, I want to log in securely so that I can access my learning workflows.

**Acceptance Criteria**:
- User authenticates with email and password
- JWT token with 24-hour expiry is returned
- Token is required for all protected endpoints

### Epic 2: Workflow Generation

**US-2.1**: As a learner, I want to describe my learning goal in natural language so that the system creates a personalized learning path.

**Acceptance Criteria**:
- User provides intent like "I want to learn React.js for web development"
- Tool Orchestrator (Node 0) analyzes intent using AWS Bedrock Claude Sonnet 3.5
- System determines topic, learning mode, difficulty, duration, and required nodes
- Workflow is created with "pending" status
- Rationale for decisions is provided

**US-2.2**: As a learner, I want the system to generate a detailed roadmap so that I know what topics I'll cover.

**Acceptance Criteria**:
- Roadmap Generator (Node 1) creates week-by-week breakdown using AWS Bedrock Llama 3.3 70B Instruct
- Each week includes subtopics, concepts, objectives, outcomes, and difficulty
- Subtopics are stored in database with sequence order
- Progress records are initialized for all subtopics
- Workflow status updates to "in_progress"

### Epic 3: AI Tutoring

**US-3.1**: As a learner, I want to interact with an AI tutor that adapts to my learning style so that I can understand concepts effectively.

**Acceptance Criteria**:
- Tutor system prompt is generated dynamically (Node 2.0) based on subtopic, user preferences, and knowledge level
- Teaching mode adapts between Socratic questioning and direct explanation
- Conversation history is persisted in database
- Tutor responses are contextual and reference previous messages

**US-3.2**: As a learner, I want to mark a subtopic as completed so that I can progress to the next topic.

**Acceptance Criteria**:
- User can complete subtopic after sufficient interaction
- System checks if all subtopics in current phase are completed
- If all completed, workflow transitions to practice phase
- Progress status updates to "completed" for the subtopic

### Epic 4: Practice & Assessment

**US-4.1**: As a learner, I want to practice what I've learned with AI-generated questions so that I can test my understanding.

**Acceptance Criteria**:
- Practice Generator (Node 3) creates questions based on covered material
- Supports MCQ, Short Answer, and Code question types
- Questions match difficulty level and learning objectives
- Practice attempt is saved with timestamp

**US-4.2**: As a learner, I want my answers to be evaluated with detailed feedback so that I know what I got right and wrong.

**Acceptance Criteria**:
- Evaluator (Node 4) scores each answer using AWS Bedrock Llama 3.3 70B Instruct
- MCQ: Binary scoring (100% or 0%)
- Short Answer: Semantic similarity scoring (0-100%)
- Code: Test case pass rate + quality assessment
- Overall score, percentage, strengths, weaknesses, and suggestions provided
- Confidence score updated in progress table

### Epic 5: Mistake Analysis & Remediation

**US-5.1**: As a learner who scored below 70%, I want the system to analyze my mistakes so that I understand what went wrong.

**Acceptance Criteria**:
- Mistake Analysis (Node 5) identifies patterns, root causes, and conceptual gaps
- Analysis includes severity assessment and prerequisite topics
- Remediation strategy is determined based on score and mistake types
- Mistake analysis is stored in database

**US-5.2**: As a struggling learner, I want automatic remediation sessions so that I can improve before retrying.

**Acceptance Criteria**:
- Remediation strategy selection:
  - Score < 40%: "reteach" (complete re-teaching)
  - Conceptual gaps: "concept_review" (targeted review)
  - Multiple mistake types: "tutor_session" (interactive clarification)
  - Minor mistakes: "practice_more" (additional practice)
- Remediation tutor session created automatically
- Readiness criteria set (min 2 messages, 60% confidence)
- Progress status updates to "struggling"

**US-5.3**: As a learner in remediation, I want to know when I'm ready to retry practice so that I don't waste attempts.

**Acceptance Criteria**:
- System checks message count and confidence level
- Provides clear feedback on readiness status
- Shows requirements and recommendations
- Allows retry only when criteria are met

### Epic 6: Progress Tracking

**US-6.1**: As a learner, I want to see my progress across all workflows so that I can track my learning journey.

**Acceptance Criteria**:
- Dashboard shows all workflows with completion percentages
- Each workflow displays total subtopics, completed count, and average confidence
- Visual progress indicators (progress bars, badges)
- Workflows are sortable by status and date

**US-6.2**: As a learner, I want detailed progress for each subtopic so that I can see my performance history.

**Acceptance Criteria**:
- Subtopic view shows status, confidence score, time spent
- Lists all tutor sessions with message counts
- Shows practice attempts with scores and dates
- Displays latest evaluation with feedback
- Shows remediation history if applicable

## Functional Requirements

### FR-1: Authentication System
- JWT-based authentication with 24-hour token expiry
- Secure password hashing using bcrypt (10 rounds)
- Protected routes requiring valid JWT token
- User profile management with learning preferences

### FR-2: Workflow Management
- Create workflows from natural language input
- AI-powered workflow structure generation
- Sequential node execution (roadmap → tutor → practice → evaluation → mistake analysis)
- Workflow status tracking (pending, in_progress, completed, failed)
- Automatic progression between workflow phases

### FR-3: AI Integration
- Integration with AWS Bedrock Claude Sonnet 3.5 for fast intent analysis
- Integration with AWS Bedrock Llama 3.3 70B Instruct for educational content
- N8n webhook orchestration for AI tasks
- 60-second timeout for LLM calls
- Fallback to templates on AI failure

### FR-4: Learning Content Generation
- Dynamic roadmap generation with week-by-week breakdown
- Adaptive tutor system prompt generation
- Context-aware practice question generation
- Multi-format question support (MCQ, Short Answer, Code)

### FR-5: Evaluation & Feedback
- Automated answer evaluation with detailed feedback
- Question-specific scoring based on type
- Overall performance metrics (score, percentage, confidence)
- Identification of strengths and weaknesses
- Actionable improvement suggestions

### FR-6: Remediation System
- Automatic mistake pattern analysis
- Root cause identification
- Remediation strategy selection based on performance
- Targeted re-teaching sessions
- Readiness assessment before retry

### FR-7: Progress Tracking
- Real-time progress updates
- Confidence score calculation based on evaluations
- Time tracking per subtopic
- Status transitions (not_started → in_progress → completed/struggling)
- Workflow-level and subtopic-level statistics

## Non-Functional Requirements

### NFR-1: Performance
- API response time < 200ms for non-AI endpoints
- AI endpoint response time < 5s (excluding LLM processing)
- Database query optimization with indexes
- Support for 100+ concurrent users

### NFR-2: Scalability
- Modular architecture for easy feature addition
- Database schema supports millions of records
- Stateless API design for horizontal scaling
- Efficient pagination for large datasets

### NFR-3: Security
- HTTPS for all communications
- JWT tokens with secure secret
- Password hashing with industry-standard algorithm
- SQL injection prevention via parameterized queries
- User data isolation (ownership verification)
- Cascade deletes for data consistency

### NFR-4: Reliability
- Graceful error handling with user-friendly messages
- Automatic retry for transient failures
- Database transactions for data integrity
- Comprehensive logging for debugging

### NFR-5: Usability
- Intuitive UI with clear navigation
- Real-time feedback during interactions
- Progress visualization with charts and indicators
- Responsive design for mobile and desktop
- Accessible components (WCAG 2.1 AA)

### NFR-6: Maintainability
- TypeScript for type safety
- Modular code organization
- Comprehensive API documentation
- Database schema with clear relationships
- Consistent naming conventions

## Technical Constraints

### TC-1: Technology Stack
- Backend: Node.js 18+, Express.js, TypeScript
- Database: Amazon DynamoDB
- AI: AWS Bedrock Claude Sonnet 3.5, AWS Bedrock Claude Haiku 3.5, AWS Bedrock Llama 3.3 70B Instruct
- Workflow Engine: N8n (self-hosted)
- Frontend: Next.js 14+, React 18+, Tailwind CSS

### TC-2: API Integrations
- AWS Bedrock API for educational content generation
- N8n webhooks for AI task orchestration

### TC-3: Data Storage
- Amazon DynamoDB for development and deployment
- 10 normalized tables with foreign key constraints
- JSON columns for flexible data structures
- Automatic timestamp triggers

## Success Metrics

### SM-1: User Engagement
- Average session duration > 20 minutes
- Workflow completion rate > 60%
- Daily active users growth rate > 10% monthly

### SM-2: Learning Effectiveness
- Average confidence score improvement > 20 points per workflow
- Remediation success rate > 75% (retry score ≥ 70%)
- User satisfaction rating > 4.0/5.0

### SM-3: System Performance
- API uptime > 99.5%
- Average AI response time < 5 seconds
- Error rate < 1% of total requests

### SM-4: Content Quality
- Practice question relevance rating > 4.0/5.0
- Tutor response helpfulness rating > 4.0/5.0
- Roadmap accuracy rating > 4.0/5.0

## Out of Scope (Future Enhancements)

- Multi-modal learning (video, interactive diagrams)
- Collaborative learning features (study groups, peer review)
- Mobile native applications
- Content marketplace for community-contributed courses
- Advanced analytics and predictive modeling
- Real-time collaboration features
- Gamification elements (badges, leaderboards)
- Integration with external learning platforms

## Dependencies

### External Services
- AWS Bedrock API (requires AWS credentials and access to models)
- N8n instance (self-hosted or cloud)

### Development Tools
- Node.js 18+ runtime
- npm/pnpm package manager
- Amazon DynamoDB database engine
- Git for version control

## Assumptions

- Users have stable internet connection for AI interactions
- N8n instance is properly configured and accessible
- API keys for LLM services are valid and have sufficient quota
- Users are comfortable with English language interface
- Browser supports modern JavaScript features (ES2020+)

## Risks & Mitigations

### Risk 1: LLM API Failures
**Impact**: High - Core functionality depends on AI  
**Mitigation**: Implement fallback to template-based generation, retry logic, and comprehensive error handling

### Risk 2: Token Expiry During Long Sessions
**Impact**: Low - User inconvenience  
**Mitigation**: Implement token refresh mechanism and auto-save functionality

### Risk 3: Inconsistent AI Responses
**Impact**: Medium - Affects learning quality  
**Mitigation**: Use structured prompts, response validation, and fallback to human review for critical content

## Acceptance Criteria Summary

The project is considered complete when:
1. All user stories are implemented and tested
2. Authentication system is secure and functional
3. Workflow generation creates valid learning paths
4. AI tutor provides contextual, adaptive responses
5. Practice system generates relevant questions and evaluates accurately
6. Remediation system automatically triggers and guides learners
7. Progress tracking displays accurate, real-time data
8. All API endpoints are documented and tested
9. Frontend provides intuitive, responsive user experience
10. System meets all non-functional requirements

