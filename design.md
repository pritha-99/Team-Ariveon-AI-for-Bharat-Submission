# Ariveon - System Design Document

## Architecture Overview

Ariveon uses a three-tier architecture with AI orchestration layer:

```
Frontend (Next.js) ↔ Backend API (Express) ↔ Database (DynamoDB)
                            ↕
                    AI Layer (N8n + AWS Bedrock)
```

## System Components

### 1. Frontend Layer (Next.js 14+)

**Purpose**: User interface and client-side logic

**Key Components**:
- Authentication pages (login, register)
- Dashboard with workflow overview
- Tutor chat interface
- Practice question interface
- Progress tracking views
- Remediation flow UI

**Technology Choices**:
- Next.js App Router for file-based routing and SSR
- shadcn/ui for accessible, customizable components
- Tailwind CSS for utility-first styling
- Axios for API communication with JWT interceptors

### 2. Backend API Layer (Express.js)

**Purpose**: Business logic, authentication, workflow orchestration

**Core Modules**:

#### Authentication System
- JWT token generation and verification
- bcrypt password hashing (10 rounds)
- Auth middleware for protected routes
- User profile management

#### Tool Orchestrator (Node 0)
- Analyzes natural language learning goals
- Uses AWS Bedrock Claude 3.5 Sonnet
- Generates workflow structure with node selection
- Provides rationale for decisions
- Lazy initialization with fallback to templates

#### Workflow Manager
- CRUD operations for workflows
- Status management (pending → in_progress → completed)
- Roadmap storage and retrieval
- Ownership verification

#### Workflow Executor
- Sequential node execution engine
- Calls N8n webhooks for AI processing
- Parses and stores results
- Manages transitions between phases

#### Tutor Session Manager
- Creates sessions with AI-generated system prompts
- Manages conversation history
- Handles subtopic completion
- Triggers workflow transitions

#### Practice System
- Generates questions via N8n Node 3
- Stores practice attempts
- Submits for evaluation via N8n Node 4
- Triggers remediation if score < 70%

#### Remediation Manager
- Analyzes mistakes via N8n Node 5
- Determines remediation strategy
- Creates remediation sessions
- Checks readiness for retry

#### Progress Tracker
- Updates progress status
- Calculates confidence scores
- Tracks time spent
- Provides statistics

### 3. AI Processing Layer (N8n)

**Purpose**: Orchestrate LLM calls and data transformation

**Node Architecture**:

#### Node 0: Tool Orchestrator (Backend)
- **LLM**: AWS Bedrock Claude 3.5 Sonnet
- **Input**: User intent, preferences
- **Output**: Workflow structure with node selection
- **Execution**: Direct API call from backend

#### Node 1: Roadmap Generator
- **LLM**: AWS Bedrock Llama 3.3 70B Instruct
- **Webhook**: `/webhook/gen-found`
- **Input**: Topic, level, duration, learning style
- **Output**: Week-by-week roadmap with subtopics
- **Processing**: Structured JSON generation

#### Node 2.0: Tutor Prompt Generator
- **LLM**: AWS Bedrock Claude 3.5 Haiku
- **Webhook**: `/webhook/tutor-prompt-generator`
- **Input**: Subtopic details, user preferences, knowledge level
- **Output**: Adaptive system prompt (500-1500 chars)
- **Processing**: Dynamic prompt engineering

#### Node 3: Practice Generator
- **LLM**: AWS Bedrock Llama 3.3 70B Instruct
- **Webhook**: `/webhook/practice-generator`
- **Input**: Topic, covered units, difficulty, question count
- **Output**: Array of questions (MCQ, Short Answer, Code)
- **Processing**: Question generation with answers

#### Node 4: Evaluator
- **LLM**: AWS Bedrock Llama 3.3 70B Instruct
- **Webhook**: `/webhook/evaluator`
- **Input**: Questions and user answers
- **Output**: Scores, feedback, strengths, weaknesses
- **Processing**: Answer evaluation with detailed analysis

#### Node 5: Mistake Analyzer
- **LLM**: AWS Bedrock Llama 3.3 70B Instruct
- **Webhook**: `/webhook/mistake-analysis`
- **Input**: Evaluation results, incorrect answers
- **Output**: Patterns, root causes, remediation plan
- **Processing**: Deep mistake analysis

### 4. Data Layer (Amazon DynamoDB)

**Purpose**: Scalable NoSQL storage with high performance

**Table Design**: Single-table design with composite keys

**Key Entities**:
- `users`: Account and preferences (PK: USER#userId)
- `workflows`: Learning paths (PK: USER#userId, SK: WORKFLOW#workflowId)
- `workflow_nodes`: Execution tracking (PK: WORKFLOW#workflowId, SK: NODE#nodeId)
- `subtopics`: Learning units (PK: WORKFLOW#workflowId, SK: SUBTOPIC#subtopicId)
- `progress`: User progress per subtopic (PK: USER#userId, SK: PROGRESS#workflowId#subtopicId)
- `tutor_sessions`: Conversation history (PK: SUBTOPIC#subtopicId, SK: SESSION#sessionId)
- `practice_attempts`: Question attempts (PK: SUBTOPIC#subtopicId, SK: ATTEMPT#attemptId)
- `evaluations`: Scoring and feedback (PK: ATTEMPT#attemptId, SK: EVALUATION#evaluationId)
- `mistake_analyses`: Error analysis (PK: SUBTOPIC#subtopicId, SK: ANALYSIS#analysisId)
- `node_transitions`: Workflow progression (PK: WORKFLOW#workflowId, SK: TRANSITION#timestamp)

**Optimization**:
- Global Secondary Indexes (GSI) for query patterns
- Composite sort keys for hierarchical data
- DynamoDB Streams for audit logging
- On-demand billing for cost efficiency

## Data Flow Diagrams

### Workflow Creation Flow

```
User Input → Tool Orchestrator (Node 0) → Workflow Created
                    ↓
            Analyze Intent
                    ↓
        Generate Node Structure
                    ↓
            Save to Database
```

### Roadmap Generation Flow

```
Start Workflow → Workflow Executor → N8n Node 1 (Roadmap)
                        ↓
                Parse Roadmap JSON
                        ↓
                Create Subtopics
                        ↓
            Initialize Progress Records
                        ↓
                Status: in_progress
```

### Tutor Session Flow

```
Create Session → N8n Node 2.0 (Prompt Gen) → System Prompt
                        ↓
                Store in Database
                        ↓
User Message → Append to History → LLM Response
                        ↓
                Update Session
                        ↓
        Complete Subtopic? → Check Transition
```

### Practice & Evaluation Flow

```
Generate Practice → N8n Node 3 → Questions Created
                        ↓
                User Answers
                        ↓
Submit → N8n Node 4 (Evaluator) → Scores & Feedback
                        ↓
        Score < 70%? → N8n Node 5 (Mistake Analysis)
                        ↓
                Remediation Session Created
```

### Remediation Flow

```
Mistake Analysis → Determine Strategy → Create Session
                        ↓
                Tutor Interaction
                        ↓
        Check Readiness (2+ msgs, 60% confidence)
                        ↓
                Retry Practice
                        ↓
                Re-evaluate
```

## API Design

### RESTful Endpoints

**Authentication**:
- `POST /api/auth/register` - Create account
- `POST /api/auth/login` - Get JWT token
- `GET /api/auth/me` - Get current user

**Workflows**:
- `POST /api/workflows/generate` - Create workflow
- `POST /api/workflows/:id/start` - Generate roadmap
- `GET /api/workflows` - List workflows
- `GET /api/workflows/:id` - Get workflow details
- `DELETE /api/workflows/:id` - Delete workflow

**Tutor**:
- `POST /api/tutor/session` - Create session
- `PUT /api/tutor/session/:id` - Send message
- `POST /api/tutor/session/:id/complete-subtopic` - Mark complete

**Practice**:
- `POST /api/practice/generate` - Generate questions
- `POST /api/practice/submit` - Submit answers

**Progress**:
- `GET /api/progress/workflow/:workflow_id` - Workflow progress
- `GET /api/progress/subtopic/:subtopic_id` - Subtopic details
- `PUT /api/progress/subtopic/:subtopic_id` - Update progress

**Remediation**:
- `POST /api/remediation/check-readiness/:subtopic_id` - Check retry readiness
- `POST /api/remediation/retry-practice/:subtopic_id` - Authorize retry
- `GET /api/remediation/history/:subtopic_id` - Get history

### Request/Response Patterns

**Standard Success Response**:
```json
{
  "data": { ... },
  "message": "Success message"
}
```

**Standard Error Response**:
```json
{
  "error": "Error message",
  "details": { ... }
}
```

**Authentication Header**:
```
Authorization: Bearer <jwt_token>
```

## Database Schema Design

### DynamoDB Single-Table Design

**Primary Table**: `ariveon-main`

**Access Patterns**:
1. Get user by email (GSI: email-index)
2. Get user workflows (Query: PK=USER#userId, SK begins_with WORKFLOW#)
3. Get workflow details (Query: PK=WORKFLOW#workflowId)
4. Get workflow subtopics (Query: PK=WORKFLOW#workflowId, SK begins_with SUBTOPIC#)
5. Get user progress (Query: PK=USER#userId, SK begins_with PROGRESS#)
6. Get tutor sessions (Query: PK=SUBTOPIC#subtopicId, SK begins_with SESSION#)
7. Get practice attempts (Query: PK=SUBTOPIC#subtopicId, SK begins_with ATTEMPT#)

**Entity Relationships**:

```
USER#userId
  ├─ WORKFLOW#workflowId (SK)
  └─ PROGRESS#workflowId#subtopicId (SK)

WORKFLOW#workflowId
  ├─ NODE#nodeId (SK)
  ├─ SUBTOPIC#subtopicId (SK)
  └─ TRANSITION#timestamp (SK)

SUBTOPIC#subtopicId
  ├─ SESSION#sessionId (SK)
  ├─ ATTEMPT#attemptId (SK)
  └─ ANALYSIS#analysisId (SK)

ATTEMPT#attemptId
  └─ EVALUATION#evaluationId (SK)
```

**Global Secondary Indexes**:
1. **email-index**: GSI on email attribute for login
2. **status-index**: GSI on status attribute for filtering workflows
3. **user-workflow-index**: GSI for reverse lookups (workflow → user)

### Key Design Decisions

**Why Single-Table Design?**
- Reduces number of queries (related data in one query)
- Better performance with composite keys
- Cost-effective (fewer read operations)
- Follows DynamoDB best practices

**Why Composite Sort Keys?**
- Hierarchical data organization
- Efficient range queries
- Natural data grouping
- Supports multiple access patterns

**Why JSON Attributes?**
- Flexible storage for dynamic data (roadmaps, conversation history)
- Reduces attribute complexity
- Easy to extend without schema changes
- Native DynamoDB support

**Why Cascade Deletes?**
- Automatic cleanup of related data with DynamoDB Streams
- Maintains data consistency
- Prevents orphaned records
- Lambda triggers for complex cleanup logic

## Security Design

### Authentication Flow

```
1. User submits credentials
2. Backend verifies password (bcrypt.compare)
3. JWT token generated with userId payload
4. Token signed with SECRET from .env
5. Token returned to client (24h expiry)
6. Client stores token (localStorage)
7. Client includes token in Authorization header
8. Middleware verifies token on each request
9. userId extracted and injected into request
```

### Security Measures

**Password Security**:
- bcrypt hashing with 10 rounds
- No plain text storage
- Secure comparison function

**Token Security**:
- HS256 algorithm
- Secret stored in environment variable
- 24-hour expiration
- Verified on every protected route

**Data Access Control**:
- Ownership verification in all queries
- User can only access their own data
- Partition key includes user context
- Fine-grained IAM policies

**NoSQL Injection Prevention**:
- Use AWS SDK parameter binding
- Validate input with Zod schemas
- Sanitize attribute names and values
- Use expression attribute names/values

## AI Integration Design

### AWS Bedrock Integration

**Service Setup**:
- AWS Bedrock Runtime API for model invocations
- IAM role with bedrock:InvokeModel permissions
- Model access enabled in AWS Console
- Regional endpoint configuration (us-east-1)

**Available Models**:
- Claude 3.5 Sonnet (anthropic.claude-3-5-sonnet-20241022-v2:0)
- Claude 3.5 Haiku (anthropic.claude-3-5-haiku-20241022-v1:0)
- Llama 3.3 70B Instruct (meta.llama3-3-70b-instruct-v1:0)

**SDK Integration**:
```typescript
import { BedrockRuntimeClient, InvokeModelCommand } from '@aws-sdk/client-bedrock-runtime';

const client = new BedrockRuntimeClient({ region: 'us-east-1' });

const response = await client.send(new InvokeModelCommand({
  modelId: 'anthropic.claude-3-5-sonnet-20241022-v2:0',
  body: JSON.stringify({
    anthropic_version: 'bedrock-2023-05-31',
    messages: [{ role: 'user', content: prompt }],
    max_tokens: 4096,
    temperature: 0.7
  })
}));
```

### Model Selection Strategy

**AWS Bedrock Claude 3.5 Haiku** (Node 2.0):
- Fast response (<2s)
- Cost-effective
- Good for prompt generation
- Used for frequent, lightweight tasks

**AWS Bedrock Claude 3.5 Sonnet** (Node 0):
- Balanced performance and cost
- Excellent for intent analysis
- Strong reasoning capabilities
- Used for workflow orchestration

**AWS Bedrock Llama 3.3 70B Instruct** (Nodes 1, 3, 4, 5):
- Superior reasoning
- Better structured output
- More detailed analysis
- Used for educational content generation

### Prompt Engineering Patterns

**Structured Output Prompts**:
```
Generate a JSON object with the following structure:
{
  "field1": "description",
  "field2": ["array", "of", "items"]
}

Requirements:
- Specific constraint 1
- Specific constraint 2
```

**Adaptive Tutor Prompts**:
```
You are teaching {{subtopic}} to a {{level}} learner.
Teaching mode: {{mode}}
Previous knowledge: {{context}}
Adapt your responses to their learning style.
```

**Evaluation Prompts**:
```
Evaluate the following answer:
Question: {{question}}
Expected: {{expected_answer}}
User Answer: {{user_answer}}

Provide:
1. Score (0-100)
2. Feedback
3. What was correct
4. What needs improvement
```

### Error Handling Strategy

**LLM Call Failures**:
1. Log error with context
2. Retry once after 2s delay
3. If still fails, use fallback template
4. Return user-friendly error message

**Timeout Handling**:
- 60s timeout for all N8n webhook calls
- Prevents hanging requests
- Returns timeout error to user

**Response Validation**:
- Validate JSON structure
- Check required fields
- Sanitize output before storage

## Performance Optimization

### Database Optimization

**DynamoDB Access Patterns**:
- Single-table design with composite keys
- GSI for email-based login queries
- GSI for workflow status filtering
- Batch operations for related items
- Consistent reads for critical data
- Eventually consistent reads for analytics

**Query Optimization**:
- Use Query instead of Scan operations
- Implement pagination with LastEvaluatedKey
- Batch GetItem for multiple records
- Projection expressions to fetch only needed attributes
- Parallel scans for analytics workloads

### API Optimization

**Response Time Targets**:
- Auth endpoints: <100ms
- CRUD operations: <200ms
- AI endpoints: <5s (excluding LLM processing)

**Caching Strategy**:
- DynamoDB Accelerator (DAX) for read-heavy workloads
- Cache user profiles in DAX
- Cache workflow roadmaps
- TTL for session data
- CloudFront for static assets

### Frontend Optimization

**Next.js Features**:
- Server-side rendering for initial load
- Automatic code splitting per route
- Image optimization
- Static generation where possible

**React Optimization**:
- Lazy loading for heavy components
- Memoization for expensive calculations
- Debouncing for search inputs
- Virtualization for long lists

## Scalability Considerations

### Current Architecture Limits

**DynamoDB Advantages**:
- Fully managed, serverless
- Automatic scaling
- High availability and durability
- No connection pooling needed

**Scaling Strategy**:
- On-demand capacity mode for variable workloads
- Provisioned capacity with auto-scaling for predictable loads
- DynamoDB Accelerator (DAX) for caching
- Global tables for multi-region deployment

### Horizontal Scaling Strategy

**Stateless API Design**:
- No server-side session storage
- JWT tokens for authentication
- All state in database

**Load Balancing**:
- Multiple API server instances
- Round-robin distribution
- Health check endpoints

### Future Architecture

```
CloudFront / API Gateway
      ↓
[Lambda/ECS] [Lambda/ECS] [Lambda/ECS]
      ↓              ↓              ↓
    DynamoDB (with DAX Cache)
      ↓
    AWS Bedrock
      ↓
    N8n (ECS/EC2)
```

## Error Handling Design

### Error Categories

**Client Errors (4xx)**:
- 400 Bad Request - Invalid input
- 401 Unauthorized - Missing/invalid token
- 403 Forbidden - Insufficient permissions
- 404 Not Found - Resource doesn't exist
- 409 Conflict - Duplicate resource

**Server Errors (5xx)**:
- 500 Internal Server Error - Unexpected error
- 502 Bad Gateway - N8n webhook failure
- 503 Service Unavailable - Database connection lost
- 504 Gateway Timeout - LLM timeout

### Error Response Format

```json
{
  "error": "Human-readable error message",
  "code": "ERROR_CODE",
  "details": {
    "field": "Additional context"
  },
  "timestamp": "2026-01-15T10:30:00Z"
}
```

### Logging Strategy

**Winston Logger Levels**:
- `error`: System failures, exceptions
- `warn`: Degraded performance, fallbacks used
- `info`: Important events (user actions, workflow transitions)
- `debug`: Detailed execution flow
- `verbose`: Full request/response data

**Log Format**:
```json
{
  "timestamp": "2026-01-15T10:30:00Z",
  "level": "info",
  "message": "Workflow created",
  "userId": "user-uuid",
  "workflowId": "workflow-uuid",
  "metadata": { ... }
}
```

## Testing Strategy

### Unit Testing

**Backend**:
- Test individual functions in isolation
- Mock DynamoDB calls with aws-sdk-mock
- Mock AWS Bedrock responses
- Mock N8n webhook responses
- Test error handling paths

**Frontend**:
- Test component rendering
- Test user interactions
- Test state management
- Mock API calls

### Integration Testing

**API Tests**:
- Test complete request/response cycles
- Use DynamoDB Local for testing
- Verify database state changes
- Test authentication flow

**N8n Integration**:
- Test webhook endpoints
- Verify request/response formats
- Test timeout handling
- Test error scenarios

### End-to-End Testing

**User Flows**:
- Complete workflow creation to completion
- Tutor session interaction
- Practice submission and evaluation
- Remediation flow

## Deployment Architecture

### Development Environment

```
localhost:3000 - Next.js frontend
localhost:3001 - Express backend
localhost:5678 - N8n instance
AWS DynamoDB - Cloud database (dev table)
AWS Bedrock - LLM API endpoints
```

### Production Environment

```
CloudFront (CDN) → Next.js (AWS Amplify)
      ↓
API Gateway → Express (AWS Lambda/ECS)
      ↓
DynamoDB (with DAX)
      ↓
AWS Bedrock (Claude & Llama models)
      ↓
N8n (Self-hosted on ECS)
```

### Environment Variables

**Backend (.env)**:
```
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
AWS_REGION=us-east-1
DYNAMODB_TABLE_NAME=ariveon-dev
SESSION_SECRET=xxx
N8N_WEBHOOK_URL=http://localhost:5678/webhook
PORT=3001
```

**Frontend (.env.local)**:
```
NEXT_PUBLIC_API_URL=http://localhost:3001/api
```

## Monitoring & Observability

### Metrics to Track

**System Metrics**:
- API response times
- Error rates
- DynamoDB read/write capacity usage
- AWS Bedrock invocation latency
- N8n webhook success rate

**Business Metrics**:
- User registrations
- Workflows created
- Completion rates
- Average confidence scores
- Remediation trigger rate

### Logging Requirements

**Request Logging**:
- HTTP method, path, status code
- Response time
- User ID (if authenticated)
- Error details (if failed)

**AI Interaction Logging**:
- Node type and execution time
- Input/output sizes
- Success/failure status
- Fallback usage

## Design Decisions & Rationale

### Why Express.js over NestJS?
- Simpler, more flexible
- Less boilerplate
- Easier to understand for team
- Sufficient for project scope

### Why DynamoDB over RDS/PostgreSQL?
- Fully managed, serverless
- Automatic scaling without capacity planning
- High availability built-in
- Pay-per-request pricing model
- Native AWS integration
- No database administration overhead
- Ideal for AWS competition requirements

### Why N8n over Direct LLM Calls?
- Visual workflow editor
- Easier to modify prompts
- Built-in error handling
- Workflow versioning
- Non-technical team can adjust

### Why Next.js over Create React App?
- Better performance (SSR)
- Built-in routing
- API routes capability
- Image optimization
- Production-ready defaults

### Why JWT over Session Cookies?
- Stateless authentication
- Easier to scale horizontally
- Works across domains
- Mobile app friendly
- Standard industry practice

## Future Enhancements

### Phase 2 Features
- Real-time collaboration
- Video content integration
- Mobile native apps
- Advanced analytics dashboard

### Technical Improvements
- DynamoDB DAX caching layer
- Microservices architecture with Lambda
- GraphQL API with AppSync
- WebSocket for real-time updates (API Gateway WebSocket)
- Step Functions for complex workflows

### AI Enhancements
- Multi-modal learning (images, video)
- Voice interaction
- Personalized learning paths
- Predictive performance modeling
- Automated content curation

