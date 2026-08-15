# Module Breakdown

Detailed explanation of each workshop module and what was implemented.

## Table of Contents

1. [Module 1: Building the Agent](#module-1-building-the-agent)
2. [Module 2: Integration with External APIs](#module-2-integration-with-external-apis)
3. [Module 3: Adding Agent Memory](#module-3-adding-agent-memory)
4. [Module 4: Using Bedrock Knowledge Bases for RAG](#module-4-using-bedrock-knowledge-bases-for-rag)
5. [Module 5: Using MCP](#module-5-using-mcp)
6. [Module 6: Exposing the Agent via API Gateway](#module-6-exposing-the-agent-via-api-gateway)

---

## Module 1: Building the Agent

### Objective
Create a basic AI agent using Strands SDK that can respond to user queries about flights.

### What Was Built

**Agent Framework:**
- Strands SDK for multi-tool orchestration
- Amazon Bedrock Nova Lite as the LLM backbone
- Simple tool: `flight_search(city: str) → dict`

**Code Structure:**
```python
from strands import Agent, tool

@tool
def flight_search(city: str) -> dict:
    """Get available flight options to a city."""
    flights = {
        "Seattle": ["Alaska Airlines", "Delta Airlines"],
        "New York": ["United Airlines", "JetBlue"],
        "Atlanta": ["Delta Airlines", "Spirit Airlines"]
    }
    return flights.get(city, [])

# Create agent
agent = Agent(
    model="us.amazon.nova-lite-v1:0",
    system_prompt="You are a helpful travel assistant.",
    tools=[flight_search]
)

# Use agent
response = agent("What flights go to Seattle?")
# Output: "Here are flights to Seattle: Alaska Airlines and Delta Airlines"
```

### Key Concepts Introduced

1. **Tool Definition** — `@tool` decorator for Strands
2. **Agent Planning** — LLM decides which tools to use
3. **Tool Execution** — Agent calls selected tools
4. **Response Generation** — LLM creates natural response from tool output

### Performance
- Execution time: ~1.8 seconds (including Bedrock inference)
- Memory usage: 107MB
- Cold start: 2.7 seconds

### Takeaway
Agents are just LLMs + tools + reasoning loop. The agent decides *when* and *how* to use tools based on user intent.

---

## Module 2: Integration with External APIs

### Objective
Integrate the agent with a real external API (National Weather Service) to fetch live data.

### What Was Built

**External API Integration:**
- `http_request` tool from Strands toolkit
- National Weather Service API for weather forecasts
- Multi-step API calls (get grid info, then forecast)

**Code Structure:**
```python
from strands_tools import http_request

# Agent uses http_request tool to call external APIs
# System prompt instructs how to use weather API:
# 1. POST to NWS grid API to get forecast URL
# 2. GET forecast from returned URL
# 3. Extract 5-day summary

# Example agent flow:
# User: "What's the weather for Seattle?"
# Agent: "I'll fetch the weather for you"
# Agent calls: http_request("https://api.weather.gov/points/47.6061,-122.3328")
# Agent parses: forecast URL from response
# Agent calls: http_request(forecast_url)
# Agent returns: "Here's the 5-day forecast for Seattle: ..."
```

### Key Concepts Introduced

1. **External API Calls** — Using `http_request` tool
2. **Multi-Step Reasoning** — Get data, parse it, use it
3. **Error Handling** — What if API is down?
4. **Instruction Following** — Prompt tells agent how to use API

### New Tools
- `http_request` — Make HTTP calls (GET, POST, etc.)
- `current_time` — Get current date/time

### Real-World Scenario
```
User: "Should I pack an umbrella for Seattle next week?"
↓
Agent thinks: "User asking about weather"
↓
Agent uses: http_request to get 5-day forecast
↓
Agent analyzes: "Scattered showers predicted"
↓
Agent responds: "Yes, pack an umbrella. Forecast shows 60% chance of rain."
```

### Takeaway
Agents can reason across multiple API calls. The LLM decides the sequence of calls needed to answer questions.

---

## Module 3: Adding Agent Memory

### Objective
Enable the agent to remember conversation history across multiple requests (stateful agent).

### What Was Built

**Session Management:**
- S3SessionManager for persistent storage
- Automatic conversation history retrieval
- User-isolated sessions (per user namespace)

**Code Structure:**
```python
from strands.session.s3_session_manager import S3SessionManager

# Create session manager
session_manager = S3SessionManager(
    session_id=user_id,  # e.g., "testuser"
    bucket="strands-agent-sessions",
    prefix="agent-sessions"
)

# Agent automatically uses session manager
agent = Agent(
    model="us.amazon.nova-lite-v1:0",
    system_prompt="You are a travel assistant.",
    tools=[flight_search, http_request],
    session_manager=session_manager  # ← Enable memory
)

# Conversation persists across requests
# Request 1: "What flights to Seattle?"  → Agent searches, saves to session
# Request 2: "What about pricing?"       → Agent recalls previous context
# Request 3: "Book one for me"           → Agent knows which flight you meant
```

### Session Storage Format

**S3 Key Structure:**
```
s3://strands-agent-sessions/agent-sessions/{user_id}/session.json
```

**Session Content:**
```json
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
      "content": "Here are the options: Alaska Airlines, Delta Airlines..."
    }
  ]
}
```

### Key Concepts Introduced

1. **Stateful Computing** — Lambda is stateless, S3 provides state
2. **Session Isolation** — Each user has their own conversation
3. **Automatic Context** — Agent retrieves history automatically
4. **Persistence** — Survives Lambda invocation cycles

### Multi-Turn Example
```
Turn 1:
  User: "Reserve Space Needle for Dec 20th"
  Agent: "What time would you prefer?"

Turn 2:
  User: "9 AM"
  Agent: "Confirmed: Space Needle, Dec 20th, 9 AM. Code: RES..."
  (Agent remembers context from Turn 1)

Turn 3:
  User: "What was my confirmation code?"
  Agent: "RES20231220900AM"
  (Agent recalls from session history)
```

### Takeaway
Serverless agents can maintain conversational context using external storage (S3). Each request is isolated but has full access to conversation history.

---

## Module 4: Using Bedrock Knowledge Bases for RAG

### Objective
Integrate Retrieval-Augmented Generation (RAG) so the agent can access private company data.

### What Was Built

**Knowledge Base Setup:**
- Created Bedrock Knowledge Base: `StrandsTravelAgentKB`
- Ingested 25 Seattle tour operators
- 9 categories of information
- Semantic search via Knowledge Base

**Code Structure:**
```python
# Knowledge Base created in AWS Bedrock console
# Contains: Tour operators, attractions, prices, reviews, etc.

# Agent can use `retrieve` tool
@tool
def retrieve(query: str) -> str:
    """Search knowledge base for tour information."""
    # Internally calls Bedrock Knowledge Base
    # Returns relevant documents
    pass

# Agent uses it like:
# User: "Tell me about Pike Place Market tours"
# Agent thinks: "Need tour operator info"
# Agent uses: retrieve("Pike Place Market tours")
# Agent gets: [Document 1, Document 2, ...]
# Agent responds: "Based on our tour operators, Pike Place..."
```

### RAG Flow

```
User Question
    ↓
Agent decides: "Need private data about tours"
    ↓
Agent calls: retrieve("query")
    ↓
Knowledge Base searches documents
    ↓
Returns top-k relevant passages
    ↓
Agent combines: LLM reasoning + knowledge base results
    ↓
Response: Grounded, factual answer
```

### Data Ingested

```
strandsTravelAgentKB
├─ Seattle Tour Operators (25)
│  ├─ Pike Place Market Guide
│  ├─ Space Needle Experience
│  ├─ Chihuly Garden & Glass
│  └─ ... (22 more)
├─ Pricing Information
├─ Reviews & Ratings
├─ Availability
└─ Contact Information
```

### Example Usage

```
User: "Which tour has the best reviews?"

Agent thinks: "User asking for opinion about tours"
Agent calls: retrieve("tour reviews ratings")
Agent gets: [Chihuly (4.9 stars), Pike Place (4.8 stars), ...]
Agent responds: "Based on our tour database, Chihuly Garden 
                and Glass has the best reviews with 4.9 stars."
```

### Key Concepts Introduced

1. **Retrieval-Augmented Generation** — LLM + external knowledge
2. **Semantic Search** — Find relevant documents, not keyword matching
3. **Grounding** — Prevent hallucinations with real data
4. **Knowledge Bases** — Store and search company data

### Benefits
- ✅ Up-to-date information (without retraining)
- ✅ Factual accuracy (based on company data)
- ✅ Reduced hallucinations
- ✅ Private data access

### Takeaway
RAG combines LLM reasoning with external knowledge. The agent uses semantic search to find relevant facts, then uses LLM to synthesize answers.

---

## Module 5: Using MCP

### Objective
Integrate Model Context Protocol (MCP) to dynamically load tools from external servers.

### What Was Built

**MCP Architecture:**
- AgentCore Gateway: Converts Lambda to MCP server
- OAuth2 Authentication: Cognito secures the gateway
- Attractions Lambda: Provides list_attractions, reserve_ticket, cancel_ticket tools

**MCP Server Setup:**
```
Attractions Lambda
    ↓ (wrapped by)
AgentCore Gateway
    ↓ (secured with)
Cognito OAuth2
    ↓ (used by)
Strands Agent (MCP Client)
```

**Code Structure:**
```python
from strands.tools.mcp.mcp_client import MCPClient
from mcp.client.streamable_http import streamablehttp_client

# 1. Fetch OAuth2 token
def fetch_access_token(client_id, client_secret, token_url):
    response = requests.post(
        token_url,
        data=f"grant_type=client_credentials&client_id={client_id}&client_secret={client_secret}",
        headers={'Content-Type': 'application/x-www-form-urlencoded'}
    )
    return response.json()['access_token']

# 2. Create authenticated transport
def create_streamable_http_transport(mcp_url: str, access_token: str):
    return streamablehttp_client(mcp_url, headers={"Authorization": f"Bearer {access_token}"})

# 3. Create MCP client
access_token = fetch_access_token(CLIENT_ID, CLIENT_SECRET, TOKEN_URL)
mcp_client = MCPClient(lambda: create_streamable_http_transport(GATEWAY_URL, access_token))

# 4. Use with agent
with mcp_client:
    tools_mcp = mcp_client.list_tools_sync()  # Dynamically load tools
    tools_mcp += [flight_search, http_request, current_time]
    
    agent = Agent(
        model="us.amazon.nova-lite-v1:0",
        tools=tools_mcp,  # Include MCP tools
        session_manager=session_manager
    )
```

### MCP Tools Provided

```
list_attractions(date: str)
├─ Returns: Available attractions for date
├─ Example: "Space Needle: 9AM, 12PM, 3PM, 6PM"

reserve_ticket(attraction: str, date: str, time: str)
├─ Returns: Reservation code + confirmation
├─ Example: "RES20231220900AM"

cancel_ticket(reservation_code: str)
└─ Returns: Cancellation confirmation
```

### Key Concepts Introduced

1. **Dynamic Tool Loading** — Discover tools at runtime
2. **OAuth2 Authentication** — Secure communication
3. **Server/Client Pattern** — Attractions as server, Agent as client
4. **Extensibility** — Add new tools without redeploying agent

### Architecture

```
Agent (MCP Client)
  ↓ HTTPS + OAuth2 Token
AgentCore Gateway (MCP Server)
  ↓ Proxy
Attractions Lambda (Tool Provider)
  ├─ list_attractions
  ├─ reserve_ticket
  └─ cancel_ticket
```

### Example Conversation

```
User: "Reserve Space Needle for Dec 20th at 9 AM"

Agent thinks: "Need to reserve attraction"
Agent uses: MCP → list_attractions("Dec 20th")
  Returns: {Space Needle: [9AM, 12PM, 3PM, 6PM]}
Agent uses: MCP → reserve_ticket("Space Needle", "Dec 20", "9 AM")
  Returns: {code: "RES20231220900AM", status: "success"}
Agent responds: "Reserved! Code: RES20231220900AM"
```

### Takeaway
MCP enables extensible agent systems. Tools can be added/updated on servers without changing the agent code. Perfect for multi-team environments.

---

## Module 6: Exposing the Agent via API Gateway

### Objective
Create a secure HTTP API so clients can call the agent from anywhere.

### What Was Built

**API Gateway Setup:**
- REST API: `strands-travel-agent`
- Endpoint: `/chat` (POST)
- Authentication: Cognito OAuth2
- Integration: Lambda proxy

**HTTP Request/Response:**

Request:
```bash
curl -X POST https://<api-id>.execute-api.us-west-2.amazonaws.com/prod/chat \
  -H "Authorization: Bearer <cognito-token>" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Reserve Space Needle for Dec 20th at 9 AM"}'
```

Response:
```json
{
  "response": "Your ticket for the Space Needle on December 20th at 9 AM has been reserved. The reservation code is RES20231220900AM."
}
```

**Architecture:**

```
Client (Web/Mobile/CLI)
  ↓ HTTPS + Token
API Gateway (REST)
  ├─ Authorization: Cognito
  ├─ Rate limiting
  ├─ Request validation
  └─ Response formatting
    ↓ Lambda Proxy
Lambda Handler
  ├─ Parse request
  ├─ Extract session_id (from Cognito)
  ├─ Initialize agent
  ├─ Process prompt
  ├─ Save session
  └─ Return response
```

**Code Structure:**

```python
def handler(event: Dict[str, Any], _context) -> Dict[str, Any]:
    try:
        # 1. Extract authenticated user
        session_id = event["requestContext"]["authorizer"]["claims"]["cognito:username"]
        
        # 2. Parse request body
        body = json.loads(event['body'])
        prompt = body['prompt']
        
        # 3. Create session manager
        session_manager = S3SessionManager(
            session_id=session_id,
            bucket=SESSIONS_BUCKET,
            prefix="agent-sessions"
        )
        
        # 4. Create agent
        agent = Agent(
            model="us.amazon.nova-lite-v1:0",
            system_prompt=TRAVEL_AGENT_PROMPT,
            tools=[flight_search, http_request, list_attractions, reserve_ticket, cancel_ticket],
            session_manager=session_manager
        )
        
        # 5. Process prompt
        response = agent(prompt)
        
        # 6. Clean response (remove thinking tags)
        clean_response = re.sub(THINKING_PATTERN, '', str(response)).strip()
        
        # 7. Return API response
        return {
            'statusCode': 200,
            'body': json.dumps({'response': clean_response})
        }
    
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

### API Features

1. **Authentication** — Cognito OAuth2
   - Token validated on every request
   - User identity in request context

2. **Session Isolation** — Per-user sessions
   - User 1 cannot access User 2's conversations
   - Session ID = Cognito username

3. **Scalability** — Fully serverless
   - API Gateway: 10K req/sec
   - Lambda: Auto-scaling (1000+ concurrent)
   - S3: Unlimited capacity

4. **Cost-Effective** — Pay-per-use
   - Lambda: $0.0000002 per ms
   - API Gateway: $3.50 per million requests
   - S3: $0.023 per GB/month

### Request Flow

```
1. Client sends HTTPS request + Cognito token
2. API Gateway:
   ├─ Validates token with Cognito
   ├─ Extracts user claims
   └─ Passes to Lambda with context
3. Lambda:
   ├─ Receives request + user context
   ├─ Creates user-specific session
   ├─ Initializes agent with all tools
   ├─ Processes prompt
   ├─ Saves conversation to S3
   └─ Returns response
4. Client receives JSON response
5. Conversation history saved for next request
```

### Example Multi-Turn

```
Turn 1:
  Client: POST /chat {"prompt": "Flights to Seattle?"}
  Response: {"response": "Alaska Airlines, Delta Airlines"}
  Session saved in S3

Turn 2 (same user, new request):
  Client: POST /chat {"prompt": "What did I ask about?"}
  Lambda: Loads session from S3 (retrieves previous conversation)
  Response: {"response": "You asked about flights to Seattle"}
  (Agent has context from Turn 1)
```

### Key Concepts Introduced

1. **API Gateway** — HTTP interface to serverless
2. **Lambda Proxy Integration** — Full event details passed to function
3. **Request Authentication** — Cognito token validation
4. **Session Isolation** — Per-user namespacing
5. **Response Formatting** — JSON for clients

### Takeaway
Serverless APIs can be production-grade with proper authentication and session management. No servers to manage, unlimited scalability.

---

## Module Summary Table

| Module | Concept | Key Technology | Achievement |
|--------|---------|---|---|
| 1 | Agent Basics | Strands SDK + Bedrock | Agent decides when to use tools |
| 2 | External APIs | http_request tool | Agent fetches live data |
| 3 | Memory | S3SessionManager | Agent remembers conversations |
| 4 | RAG | Bedrock Knowledge Bases | Agent grounds answers in company data |
| 5 | Extensibility | MCP + OAuth2 | Agent dynamically loads tools |
| 6 | HTTP API | API Gateway + Cognito | Agent accessible from anywhere |

---

## Skills Acquired

✅ Designing AI agents that reason and act  
✅ Multi-tool orchestration and routing  
✅ Integration with external APIs  
✅ Persistent conversation memory  
✅ Retrieval-Augmented Generation (RAG)  
✅ Dynamic tool loading (MCP)  
✅ Secure HTTP API design  
✅ Serverless architecture patterns  
✅ Production-grade authentication  
✅ Scalable AI systems  

---

## Next Steps

1. **Extend Tools** — Add more capabilities via MCP
2. **Fine-tune Agent** — Improve prompts for your domain
3. **Add Monitoring** — CloudWatch logs, X-Ray tracing
4. **Collect Feedback** — Iterate on user interactions
5. **Scale Out** — Deploy to multiple regions

---

## References

- [Strands SDK Docs](https://strandsagents.com/latest/documentation/docs/)
- [Amazon Bedrock Guide](https://docs.aws.amazon.com/bedrock/)
- [MCP Specification](https://modelcontextprotocol.io/)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [API Gateway Security](https://docs.aws.amazon.com/apigateway/latest/developerguide/security.html)
