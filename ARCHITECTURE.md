# System Architecture

## Table of Contents

1. [High-Level Overview](#high-level-overview)
2. [Data Flow](#data-flow)
3. [Component Details](#component-details)
4. [Tool Routing](#tool-routing)
5. [Session Management](#session-management)
6. [Security Architecture](#security-architecture)
7. [Scalability Considerations](#scalability-considerations)
8. [Design Decisions](#design-decisions)

---

## High-Level Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                            │
│          (Web App, Mobile App, CLI, Third-party Service)       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           │ HTTPS Request + Cognito Token
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    API GATEWAY LAYER                            │
│  • REST API endpoint: /chat POST                               │
│  • Cognito authorizer validates token                          │
│  • Request validation & rate limiting                          │
│  • Response formatting (JSON)                                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           │ Lambda Proxy Integration
                           │ (Request context passed as-is)
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    LAMBDA COMPUTE LAYER                         │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  REQUEST HANDLER                                         │   │
│  │  • Extract session_id from Cognito username              │   │
│  │  • Parse prompt from request body                        │   │
│  │  • Initialize S3SessionManager                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│  ┌────────────────────────▼─────────────────────────────────┐   │
│  │  STRANDS AGENT (Nova Lite)                               │   │
│  │                                                           │   │
│  │  Planning:                                               │   │
│  │  • Parse user prompt                                     │   │
│  │  • Decide which tool(s) to use                           │   │
│  │  • Generate reasoning for decisions                      │   │
│  │                                                           │   │
│  │  Acting:                                                 │   │
│  │  • Execute selected tools                                │   │
│  │  • Gather results                                        │   │
│  │  • Refine responses based on results                     │   │
│  │                                                           │   │
│  │  Memory:                                                 │   │
│  │  • Retrieve conversation history from S3                 │   │
│  │  • Maintain context across multiple turns                │   │
│  │  • Save conversation state after each interaction        │   │
│  └────────────────────────┬────────────────────────────────┘   │
│                           │                                     │
│           ┌───────────────┼───────────────────┐                │
│           │               │                   │                │
│  ┌────────▼──┐  ┌────────▼──┐  ┌────────────▼┐  ┌──────────┐  │
│  │   Tool    │  │  External │  │   Session  │  │ Response │  │
│  │ Routing   │  │  Service  │  │  Manager   │  │ Formatter│  │
│  │           │  │  Calls    │  │            │  │          │  │
│  └────────────┘  └───────────┘  └────────────┘  └──────────┘  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           │ Sanitized JSON Response
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    PERSISTENCE LAYER                            │
│                                                                 │
│  ┌──────────────────┐  ┌───────────────┐  ┌──────────────────┐ │
│  │  S3 Buckets      │  │  Bedrock      │  │  Knowledge Base  │ │
│  │  (Sessions)      │  │  (Inference)  │  │  (RAG/Search)    │ │
│  │                  │  │               │  │                  │ │
│  │ User1/session... │  │ Nova Lite     │  │ Tour Operators   │ │
│  │ User2/session... │  │ • Reasoning   │  │ • Attractions    │ │
│  │ User3/session... │  │ • Planning    │  │ • Reviews        │ │
│  └──────────────────┘  │ • Acting      │  │ • Details        │ │
│                        └───────────────┘  └──────────────────┘ │
│                                                                 │
│  ┌──────────────────┐  ┌───────────────┐                       │
│  │  Weather API     │  │  MCP Gateway  │                       │
│  │  (External)      │  │  (Attractions)│                       │
│  │                  │  │               │                       │
│  │ National Weather │  │ OAuth2 Secured│                       │
│  │ Service          │  │ Tool Server   │                       │
│  └──────────────────┘  └───────────────┘                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## Data Flow

### Request → Response Lifecycle

```
1. CLIENT SENDS REQUEST
   └─ POST /chat
   └─ Header: Authorization: Bearer <cognito_token>
   └─ Body: {"prompt": "Reserve Space Needle for Dec 20th at 9 AM"}

2. API GATEWAY RECEIVES REQUEST
   └─ Validates Cognito token with authorizer
   └─ Passes full event (with requestContext) to Lambda
   └─ Sets 15-minute timeout

3. LAMBDA HANDLER STARTS
   └─ Extract: session_id = event["requestContext"]["authorizer"]["claims"]["cognito:username"]
   └─ Extract: prompt = json.loads(event["body"])["prompt"]
   └─ Create: S3SessionManager for user
   └─ Retrieve: Conversation history from S3 (if exists)

4. AGENT PLANNING PHASE
   └─ System Prompt + User Prompt + Tools Description + History
   └─ Send to Bedrock Nova Lite
   └─ Nova decides: which tool(s) to use and why
   └─ Response includes: reasoning, tool selection, parameters

5. AGENT ACTING PHASE
   └─ If tool_name == "flight_search":
   │  └─ Execute local tool → return flight options
   │
   └─ If tool_name == "list_attractions":
   │  └─ Execute local tool → return available attractions
   │
   └─ If tool_name == "reserve_ticket":
   │  └─ Execute local tool → return reservation code
   │
   └─ If tool_name == "http_request":
   │  └─ Call external API (e.g., weather service)
   │  └─ Return formatted response
   │
   └─ If tool_name == "retrieve":
   │  └─ Query Knowledge Base (RAG)
   │  └─ Get relevant documents
   │  └─ Return search results

6. AGENT REASONING PHASE (Continued)
   └─ Evaluate tool results
   └─ If successful: craft final response
   └─ If needs more info: loop back to planning phase
   └─ Multiple turns until confidence reached

7. RESPONSE FORMATTING
   └─ Clean response (remove <thinking> tags)
   └─ Sanitize (no internal reasoning exposed)
   └─ Format as JSON

8. SESSION PERSISTENCE
   └─ Save conversation history to S3
   └─ Key: s3://bucket/agent-sessions/{user_id}/{session_id}.json
   └─ Contents: full conversation history for next request

9. LAMBDA RETURNS
   └─ Format: {statusCode: 200, body: JSON}
   └─ Body: {"response": "Your ticket has been reserved..."}

10. API GATEWAY RETURNS RESPONSE
    └─ HTTPS 200 OK
    └─ Body: {"response": "..."}

11. CLIENT RECEIVES RESPONSE
    └─ Display: "Your ticket has been reserved..."
    └─ Ready for next prompt in conversation
```

---

## Component Details

### 1. API Gateway

**Responsibility:** HTTP interface and request routing

```
Configuration:
├─ REST API: strands-travel-agent
├─ Endpoint: Regional (us-west-2)
├─ Resource: /
│  └─ Resource: /chat
│     └─ Method: POST
│        ├─ Integration: Lambda Function
│        ├─ Lambda Proxy: Enabled
│        ├─ Response Mode: Buffered
│        └─ Authorization: CognitoAuthorizer
├─ Authorizer: CognitoAuthorizer
│  ├─ Type: Cognito User Pool
│  ├─ User Pool: StrandsAgentUserPool
│  ├─ Token Source: Authorization header
│  └─ Identity Source: method.request.header.Authorization
└─ Deployment: prod stage
```

**Flow:**
1. Client sends HTTPS request with Cognito token
2. API Gateway extracts token from Authorization header
3. Cognito Authorizer validates token signature
4. If valid: passes full event to Lambda (includes claims)
5. If invalid: returns 401 Unauthorized

### 2. Cognito Authentication

**Responsibility:** User identity and authentication

```
User Pool: StrandsAgentUserPool
├─ OAuth2 Client Credentials Flow (for MCP)
├─ Authorization Code Flow (for API users)
└─ Token Validation on every request

Token Structure (JWT):
{
  "sub": "username",
  "cognito:username": "testuser",
  "email": "test@example.com",
  "exp": 1234567890,
  "iat": 1234567800,
  "auth_time": 1234567800
}

Lambda receives in: event["requestContext"]["authorizer"]["claims"]["cognito:username"]
```

**Session Isolation:**
- User's `cognito:username` becomes their `session_id`
- All conversation history stored under user's namespace in S3
- One user cannot access another user's conversations

### 3. Lambda Function

**Responsibility:** Agent orchestration and tool execution

```python
Handler Signature:
def handler(event: Dict[str, Any], _context) -> Dict[str, Any]:
    # event structure from API Gateway Lambda Proxy:
    {
        "body": '{"prompt": "..."}',  # JSON string (must parse)
        "requestContext": {
            "authorizer": {
                "claims": {
                    "cognito:username": "testuser"
                }
            }
        },
        "headers": {
            "Authorization": "Bearer <token>"
        }
    }

Memory: 512 MB (configurable)
Timeout: 900 seconds (max, can reduce to 60-120s)
Runtime: Python 3.12
Layers: Strands SDK, MCP client, dependencies
```

**Execution Flow:**
```
1. Parse request body (event["body"])
2. Extract session_id (from Cognito claims)
3. Initialize S3SessionManager
4. Load conversation history (if exists)
5. Create Strands Agent with tools
6. Pass prompt to agent
7. Agent iterates (plan → act → reason)
8. Save conversation state to S3
9. Return sanitized response
```

### 4. Strands Agent

**Responsibility:** Reasoning, planning, and tool orchestration

```
Agent Configuration:
├─ Model: us.amazon.nova-lite-v1:0 (Bedrock)
├─ System Prompt: Travel assistant instructions
├─ Tools: [flight_search, list_attractions, reserve_ticket, ...]
├─ Session Manager: S3SessionManager
└─ Max Iterations: Implicit (controlled by SDK)

Agent Loop (Agentic Reasoning):
Loop:
  1. PLANNING:
     └─ Send (system prompt + user message + tool specs + history)
     └─ LLM decides which tool(s) to call
     └─ LLM decides with what parameters

  2. ACTING:
     └─ Execute selected tool
     └─ Capture tool output

  3. REASONING:
     └─ Evaluate: "Is response complete?"
     └─ If NO: loop back to step 1 with tool output
     └─ If YES: exit loop and return response
```

### 5. Tool Routing

**Responsibility:** Directing agent to appropriate tool based on intent

```
Tools Available:

┌─ LOCAL TOOLS (Strands @tool decorator)
│  ├─ flight_search(city: str) → dict
│  │  └─ Returns airline options for destination
│  │
│  ├─ list_attractions(date: str) → dict
│  │  └─ Returns available attractions + times + prices
│  │
│  ├─ reserve_ticket(attraction, date, time) → dict
│  │  └─ Creates reservation, returns confirmation code
│  │
│  └─ cancel_ticket(reservation_code) → dict
│     └─ Cancels reservation
│
├─ EXTERNAL TOOLS (Strands built-ins)
│  ├─ http_request(url, method, headers, body) → str
│  │  └─ Calls external APIs (e.g., weather service)
│  │
│  └─ current_time() → str
│     └─ Returns current date/time
│
└─ RAG TOOL (Bedrock Knowledge Base)
   └─ retrieve(query: str) → str
      └─ Searches private documents (tour operators)
      └─ Returns relevant excerpts

Tool Selection Logic (LLM-driven):
User: "Reserve Space Needle for Dec 20th"
  ↓
Agent Planning: "User wants attractions → need reserve_ticket tool"
  ↓
Parameter inference: attraction="Space Needle", date="Dec 20th", time=?
  ↓
If time missing: call list_attractions first to get available times
  ↓
Execute: reserve_ticket("Space Needle", "Dec 20", "9:00 AM")
  ↓
Return: {"status": "success", "reservation_code": "RES20231220900AM"}
```

### 6. Session Management

**Responsibility:** Persistent conversation state across requests

```
S3 SessionManager:
├─ Bucket: <SESSIONS_BUCKET>
├─ Key Structure: agent-sessions/{user_id}/session.json
└─ Contents: Full conversation history

Session Data Structure:
{
    "user_id": "testuser",
    "session_id": "testuser",
    "created": "2024-01-15T20:30:00Z",
    "last_updated": "2024-01-15T20:35:00Z",
    "conversation_history": [
        {
            "role": "user",
            "content": "What flights to Seattle?"
        },
        {
            "role": "assistant",
            "content": "Here are the flight options..."
        },
        {
            "role": "user",
            "content": "Reserve Space Needle for Dec 20th"
        },
        {
            "role": "assistant",
            "content": "Your reservation code is RES20231220900AM"
        }
    ]
}

Lifecycle:
1. First request: Create new session file in S3
2. Second request: Load session from S3 (context-aware)
3. Third+ request: Append to conversation, save back
4. Benefits:
   ├─ Multi-turn conversations work
   ├─ Agent has conversation context
   ├─ User session is persistent (logout → login = same conversation)
   └─ Stateless Lambda (no in-memory state)
```

---

## Security Architecture

### 1. Authentication Flow

```
Client Request:
  ├─ Step 1: Authenticate with Cognito (out-of-band)
  │  └─ POST /oauth2/token
  │  └─ Headers: {client_id, client_secret}
  │  └─ Response: access_token + id_token
  │
  └─ Step 2: Call agent API with token
     ├─ POST /chat
     ├─ Header: Authorization: Bearer {id_token}
     └─ Body: {"prompt": "..."}

API Gateway:
  └─ Intercepts request
  └─ Extracts token from Authorization header
  └─ Validates token signature with Cognito public key
  └─ Verifies token expiration
  └─ Extracts claims (username, email, etc.)
  └─ Passes to Lambda in requestContext.authorizer.claims

Lambda:
  └─ Receives token claims
  └─ Uses cognito:username as session_id
  └─ Creates per-user S3 path
  └─ No need to re-validate token (API Gateway did it)
```

### 2. Data Isolation

```
User1 (alice@example.com):
  └─ Session stored at: s3://bucket/agent-sessions/alice/session.json
  └─ Can only access own sessions
  └─ Cannot access Bob's sessions

User2 (bob@example.com):
  └─ Session stored at: s3://bucket/agent-sessions/bob/session.json
  └─ Can only access own sessions
  └─ Cannot access Alice's sessions

Mechanism:
  ├─ session_id derived from Cognito username (immutable)
  ├─ S3 key = agent-sessions/{username}/session.json
  ├─ No cross-user access possible at application level
  └─ S3 bucket policy can enforce additional restrictions
```

### 3. Secret Management

```
Environment Variables (set via Lambda Configuration):
├─ CLIENT_ID: OAuth2 client ID (for MCP)
├─ CLIENT_SECRET: OAuth2 client secret (sensitive)
├─ DOMAIN_URL: Cognito domain
├─ GATEWAY_URL: MCP gateway endpoint
└─ SESSIONS_BUCKET: S3 bucket name

Never in Code:
├─ No hardcoded credentials
├─ No credentials in environment
├─ All secrets passed via AWS Secrets Manager or Lambda env vars
└─ Log files sanitized (no token logging)
```

### 4. Transport Security

```
In Transit:
├─ Client → API Gateway: HTTPS (TLS 1.2+)
├─ API Gateway → Lambda: Internal AWS network (encrypted)
├─ Lambda → Bedrock: HTTPS (TLS 1.2+)
├─ Lambda → S3: HTTPS (TLS 1.2+)
└─ Lambda → Weather API: HTTPS (TLS 1.2+)

At Rest:
├─ S3 conversations: Server-side encryption (KMS or S3-managed)
├─ Bedrock requests: AWS encrypted
└─ Database: Not applicable (stateless compute)
```

---

## Scalability Considerations

### 1. Horizontal Scaling

```
Lambda (Automatic):
├─ Cold starts: First invocation takes 2.7s
├─ Warm invocations: 300-500ms
├─ Concurrent executions: 1000+ default (can increase)
├─ No server management
└─ Pay only for execution time (100ms granularity)

API Gateway:
├─ Default throttling: 10,000 req/s
├─ Automatic scaling: Handles traffic spikes
├─ No provisioning needed
└─ Can increase quota if needed

S3:
├─ Automatic scaling: Handles millions of requests
├─ Partitioned by session ID (good distribution)
└─ No manual scaling needed
```

### 2. Performance Optimization

```
Latency Breakdown (typical request):
├─ API Gateway processing: ~50ms
├─ Lambda cold start (first only): ~2700ms
├─ Lambda warm start: ~150ms
├─ Bedrock inference: ~400-600ms
├─ S3 operations: ~50ms
└─ Total (warm): ~650-800ms

Optimizations Applied:
├─ Lambda layers: Pre-compiled Strands SDK (no install time)
├─ Connection pooling: Reuse Bedrock/S3 connections
├─ S3 caching: Local session cache in Lambda
├─ Buffered response mode: Complete response before streaming
└─ Async where possible: Non-blocking I/O
```

### 3. Cost Optimization

```
Cost Drivers:
├─ Lambda: $0.0000002 per ms (0.2 ms per request = $0.0000004/request)
├─ API Gateway: $3.50 per million requests ($0.0000035/request)
├─ Bedrock: $0.00030 per 1K input tokens (~$0.005/request)
├─ S3: $0.023 per GB/month (~$0.001/request for small objects)
└─ Total: ~$0.015-0.02 per request

Techniques:
├─ Use Lambda only when needed (no idle costs)
├─ Compress conversation history (S3 storage)
├─ Cache frequently used data
├─ Monitor logs for inefficient tool calls
└─ Set API Gateway throttling limits
```

---

## Design Decisions

### 1. Why Serverless?

**Pros:**
- ✅ No server management (AWS handles scaling)
- ✅ Pay only for usage (no idle costs)
- ✅ Automatic horizontal scaling
- ✅ Built-in high availability
- ✅ Integrates well with AWS services

**Cons:**
- ❌ Cold start latency (2.7s first call)
- ❌ Max execution time: 900s
- ❌ Limited to AWS Lambda runtime

**Decision:** Serverless is right for agent workloads (bursty traffic, variable demand)

### 2. Why S3 for Sessions?

**Alternatives Considered:**
- DynamoDB: More complex, higher cost for small objects
- ElastiCache: More expensive, requires provisioning
- RDS: Overkill for simple JSON storage

**Why S3:**
- ✅ Simple key-value storage (session_id → JSON)
- ✅ Automatic scaling
- ✅ Built-in encryption
- ✅ Versioning (audit trail of conversations)
- ✅ Cost-effective (~$0.001/request)

### 3. Why Strands SDK?

**Alternatives Considered:**
- LangChain: More complex, heavier dependencies
- LlamaIndex: Focused on RAG, less on agentic patterns
- Manual tool calling: Too much boilerplate

**Why Strands:**
- ✅ Built for production agents
- ✅ Clean tool decorator syntax
- ✅ Handles iteration/looping automatically
- ✅ Great session management
- ✅ MCP support built-in

### 4. Why API Gateway + Cognito?

**Alternatives Considered:**
- ALB + API authentication: More management
- Lambda Authorizer with JWT: More complex
- API Key only: Insufficient for user tracking

**Why API Gateway + Cognito:**
- ✅ OAuth2 standard (interoperable)
- ✅ User identity built-in
- ✅ Token validation automatic
- ✅ MFA support if needed
- ✅ Managed by AWS (no ops burden)

### 5. Why Bedrock Nova Lite?

**Alternatives Considered:**
- Claude 3.5 Sonnet: More capable but 3x cost
- GPT-4: External API, higher latency, privacy concerns
- Llama 2: Lower quality, requires fine-tuning

**Why Nova Lite:**
- ✅ Fast inference (400-600ms)
- ✅ Low cost ($0.00030 per 1K tokens)
- ✅ Good quality for travel domain
- ✅ Running in AWS (lower latency)
- ✅ Managed service (no servers to run)

---

## Future Enhancements

Potential improvements:

1. **Streaming Responses** — Use Lambda streaming mode for real-time responses
2. **Function Calling** — Use Bedrock tool_use format instead of agentic loop
3. **Multi-Agent** — Add supervisor agent for complex workflows
4. **Vector DB** — Use OpenSearch for semantic search instead of basic retrieval
5. **Observability** — Add X-Ray tracing for performance analysis
6. **Rate Limiting** — Add user-based rate limits (not just global)
7. **Feedback Loop** — Collect user feedback to improve agent behavior
8. **Fine-tuning** — Fine-tune Nova on travel domain for better accuracy

---

## References

- [AWS Lambda Architecture Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html)
- [Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/)
- [Strands SDK Documentation](https://strandsagents.com/latest/documentation/docs/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [OAuth 2.0 Specification](https://tools.ietf.org/html/rfc6749)
