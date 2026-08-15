# System Architecture

## Data Flow
Client (HTTP + Token)
  → API Gateway (REST + Cognito)
  → Lambda (Strands Agent)
  → Bedrock Nova Lite
  → Tools: flight_search, http_request, attractions, weather, RAG

## Key Design Decisions
- Why serverless? Scalability, cost, no ops burden
- Why Strands SDK? Production-ready, tool routing, memory mgmt
- Why MCP? Dynamic tool discovery, extensibility
- Why session state in S3? Durability, simple, no DB ops

## Security
- OAuth2 via Cognito
- Stateless API Gateway
- User-isolated sessions
- No credentials in code
