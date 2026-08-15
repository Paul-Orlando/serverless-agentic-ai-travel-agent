# Serverless Agentic AI Travel Agent

A production-grade AI agent built with Strands SDK, Amazon Bedrock, 
and AWS Lambda. Demonstrates enterprise patterns for tool routing, 
memory management, RAG, MCP integration, and secure API exposure.

## Features
- ✅ Multi-tool orchestration (flight search, weather, attractions)
- ✅ Persistent conversation memory (S3 sessions)
- ✅ Retrieval-Augmented Generation (Knowledge Bases)
- ✅ Model Context Protocol (MCP) integration
- ✅ Secure HTTP API (Cognito + API Gateway)
- ✅ Serverless architecture (Lambda, DynamoDB-compatible)

## Tech Stack
- Strands SDK (agent framework)
- Amazon Bedrock (Nova Lite foundation model)
- AWS Lambda (compute)
- API Gateway (HTTP endpoint)
- Cognito (authentication)
- S3 (session persistence)
- Knowledge Bases (RAG)

## Architecture
[Include ASCII diagram or architecture.png]

## Quick Start
[Include CLI commands to deploy]

## Modules
1. Agent building with Strands SDK
2. External API integration (Weather)
3. Conversation memory management
4. RAG with Knowledge Bases
5. MCP server integration
6. Secure API exposure

## Results
- Successfully reserves attractions
- Fetches real weather data
- Maintains multi-turn conversations
- 658ms latency end-to-end
