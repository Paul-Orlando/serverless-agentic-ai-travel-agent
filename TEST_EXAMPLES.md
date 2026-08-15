# Test Examples

Comprehensive test cases for the Serverless Agentic AI Travel Agent API.

## Table of Contents

1. [Setup](#setup)
2. [Test Cases](#test-cases)
3. [Success Scenarios](#success-scenarios)
4. [Error Scenarios](#error-scenarios)
5. [Load Testing](#load-testing)
6. [Integration Tests](#integration-tests)

---

## Setup

### Prerequisites

```bash
# Set environment variables
export API_URL="https://pci9npwhe4.execute-api.us-west-2.amazonaws.com/prod/chat"
export ID_TOKEN="<your-cognito-token>"
export USER_POOL_ID="us-west-2_zFcPHDP2W"
export CLIENT_ID="4nav7pme5d8bp4k42f24mt5900"
```

### Get Fresh Token

```bash
# Run this to get a new token
TOKEN_RESPONSE=$(aws cognito-idp admin-initiate-auth \
  --user-pool-id $USER_POOL_ID \
  --client-id $CLIENT_ID \
  --auth-flow ADMIN_NO_SRP_AUTH \
  --auth-parameters USERNAME=testuser,PASSWORD="MyPassword123!")

export ID_TOKEN=$(echo $TOKEN_RESPONSE | jq -r '.AuthenticationResult.IdToken')
echo "Token expires in: $(echo $TOKEN_RESPONSE | jq -r '.AuthenticationResult.ExpiresIn') seconds"
```

---

## Test Cases

### Test 1: Simple Flight Search

**Request:**
```bash
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "What are the flight options to Seattle?"
  }' | jq
```

**Expected Response:**
```json
{
  "response": "Here are the flight options to Seattle: 1. Alaska Airlines 2. Delta Airlines - Please let me know if you need further assistance with your travel plans."
}
```

**Success Criteria:**
- ✅ Status code: 200
- ✅ Response contains "Alaska Airlines" or "Delta Airlines"
- ✅ Response is in natural language
- ✅ No internal reasoning tags visible

---

### Test 2: Attraction Reservation

**Request:**
```bash
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Reserve your ticket for the Space Needle on December 20th at 9 AM"
  }' | jq
```

**Expected Response:**
```json
{
  "response": "Your ticket for the Space Needle on December 20th at 9 AM has been reserved. The reservation code is RES20231220900AM."
}
```

**Success Criteria:**
- ✅ Status code: 200
- ✅ Response confirms reservation
- ✅ Reservation code is provided (format: RES\d{8}\d+)
- ✅ Date and time are acknowledged

---

### Test 3: Multi-Turn Conversation

**Request 1:** Reserve attraction
```bash
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Book the Chihuly Garden for December 25th at 2 PM"
  }' | jq

# Save reservation code from response
```

**Request 2:** Reference previous booking
```bash
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "What time is my Chihuly Garden reservation?"
  }' | jq
```

**Expected Response:**
```json
{
  "response": "Your Chihuly Garden reservation is scheduled for December 25th at 2 PM. Your reservation code is RES20231225200PM."
}
```

**Success Criteria:**
- ✅ Agent remembers previous booking
- ✅ Context is maintained across requests
- ✅ Correct details are recalled
- ✅ Uses same conversation session

---

### Test 4: Inquiry About Available Times

**Request:**
```bash
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "What times are available for the Space Needle on December 20th?"
  }' | jq
```

**Expected Response:**
```json
{
  "response": "Here are the available times for the Space Needle on December 20th: 9:00 AM, 12:00 PM, 3:00 PM, 6:00 PM. Please let me know which time works best for you."
}
```

**Success Criteria:**
- ✅ Lists multiple time slots
- ✅ Prompts for user selection
- ✅ Times are realistic

---

### Test 5: Cancel Reservation

**Request:**
```bash
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Cancel my reservation code RES20231220900AM"
  }' | jq
```

**Expected Response:**
```json
{
  "response": "Your reservation with code RES20231220900AM has been successfully cancelled."
}
```

**Success Criteria:**
- ✅ Confirms cancellation
- ✅ References correct reservation code
- ✅ Uses past tense (has been cancelled)

---

### Test 6: Weather Forecast Request

**Request:**
```bash
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "What is the weather forecast for Seattle next week?"
  }' | jq
```

**Expected Response:**
```json
{
  "response": "Here is the 5-day forecast for Seattle: [Weather details including temperature, precipitation, and alerts]. This should help you plan your activities accordingly."
}
```

**Success Criteria:**
- ✅ Calls National Weather Service API
- ✅ Returns realistic weather data
- ✅ Includes temperature and conditions
- ✅ Limited to 5-day summary

---

### Test 7: Natural Language Intent Recognition

**Request:**
```bash
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "I want to go to the Needle on New Years Eve around dinner time"
  }' | jq
```

**Expected Response:**
```json
{
  "response": "The Space Needle has availability on December 31st at 6:00 PM (dinner time). Would you like me to reserve this for you?"
}
```

**Success Criteria:**
- ✅ Interprets "the Needle" as "Space Needle"
- ✅ Converts "New Years Eve" to December 31st
- ✅ Maps "dinner time" to appropriate hour (6 PM)
- ✅ Asks for confirmation

---

### Test 8: Multiple Tools in One Request

**Request:**
```bash
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Show me flights to Seattle, the weather for next week, and book Pike Place Market tour for tomorrow at 10 AM"
  }' | jq
```

**Expected Response:**
```json
{
  "response": "I found flights to Seattle: Alaska Airlines and Delta Airlines. The weather next week looks mild with scattered showers. I've also booked Pike Place Market for tomorrow at 10 AM. Your reservation code is RES[date][time]."
}
```

**Success Criteria:**
- ✅ Handles multiple requests in one prompt
- ✅ Uses multiple tools correctly
- ✅ Provides coherent response
- ✅ All information is accurate

---

## Success Scenarios

### Scenario 1: New User First Booking

```bash
# Step 1: Explore options
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "What attractions are available?"}' | jq

# Step 2: Get details
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Tell me about the Space Needle"}' | jq

# Step 3: Check availability
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "What times are available tomorrow?"}' | jq

# Step 4: Make reservation
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Book the 2 PM slot for me"}' | jq

# Expected: Smooth progression through information gathering to booking
```

### Scenario 2: Returning User Modifying Booking

```bash
# Step 1: Check existing reservations
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "What bookings do I have?"}' | jq

# Step 2: Cancel old booking
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Cancel my existing booking"}' | jq

# Step 3: Make new booking
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Book a different time slot instead"}' | jq

# Expected: Smooth transition with conversation context
```

---

## Error Scenarios

### Error 1: Missing Authorization Header

**Request:**
```bash
curl -X POST $API_URL \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Test"}' | jq
```

**Expected Response:**
```
HTTP/1.1 403 Forbidden
{
  "message": "Unauthorized"
}
```

**Success Criteria:**
- ✅ Returns 403 Forbidden
- ✅ Rejects unauthenticated request

---

### Error 2: Invalid Token

**Request:**
```bash
curl -X POST $API_URL \
  -H "Authorization: Bearer invalid_token_12345" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Test"}' | jq
```

**Expected Response:**
```
HTTP/1.1 401 Unauthorized
{
  "message": "Unauthorized"
}
```

**Success Criteria:**
- ✅ Returns 401 Unauthorized
- ✅ Rejects malformed token

---

### Error 3: Missing Prompt in Body

**Request:**
```bash
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' | jq
```

**Expected Response:**
```json
{
  "error": "KeyError: 'prompt'"
}
```

**Success Criteria:**
- ✅ Returns 500 error
- ✅ Indicates missing required field

---

### Error 4: Empty Prompt

**Request:**
```bash
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt": ""}' | jq
```

**Expected Response:**
```json
{
  "response": "Please provide a travel-related question or request to get started."
}
```

**Success Criteria:**
- ✅ Gracefully handles empty prompt
- ✅ Asks user to provide valid input

---

### Error 5: Request Timeout

**Request:**
```bash
# Send request with very large prompt (>10KB)
curl -X POST $API_URL \
  -H "Authorization: $ID_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "'$(printf 'A%.0s' {1..50000})'"}' | jq
```

**Expected Response:**
```
HTTP/1.1 413 Payload Too Large
```

**Success Criteria:**
- ✅ Rejects oversized requests
- ✅ Returns 413 error

---

## Load Testing

### Test Parameters

```bash
# Configuration
CONCURRENT_REQUESTS=10
TOTAL_REQUESTS=100
TEST_PROMPT="What are the flight options to Seattle?"
```

### Apache Bench Test

```bash
# Generate Authorization header with token
AUTH_HEADER="Authorization: $ID_TOKEN"

# Run load test
ab -n $TOTAL_REQUESTS \
   -c $CONCURRENT_REQUESTS \
   -H "$AUTH_HEADER" \
   -H "Content-Type: application/json" \
   -p /tmp/request.json \
   $API_URL/

# Where /tmp/request.json contains:
# {"prompt": "What are the flight options to Seattle?"}
```

### Expected Results

```
Requests per second: 50+ (warm Lambda)
Time per request: 20-100ms (average)
Success rate: 99%+ (API Gateway + Lambda working)
Error rate: <1% (timeout or temporary issues)
```

---

### Artillery Load Test

```bash
# Install Artillery
npm install -g artillery

# Create test file: load-test.yml
cat > /tmp/load-test.yml << 'EOF'
config:
  target: "https://pci9npwhe4.execute-api.us-west-2.amazonaws.com"
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Ramp up"
    - duration: 120
      arrivalRate: 50
      name: "Sustained load"
    - duration: 60
      arrivalRate: 10
      name: "Ramp down"

scenarios:
  - name: "Travel Agent Flow"
    flow:
      - post:
          url: "/prod/chat"
          headers:
            Authorization: "Bearer {{ $processEnvironment.ID_TOKEN }}"
            Content-Type: "application/json"
          json:
            prompt: "What are the flight options to Seattle?"
          expect:
            - statusCode: 200
          capture:
            json: "$.response"
            as: "agentResponse"
EOF

# Run test
ID_TOKEN=$ID_TOKEN artillery run /tmp/load-test.yml
```

---

## Integration Tests

### Python Test Suite

```python
import requests
import json
import os

class TravelAgentTests:
    def __init__(self):
        self.api_url = os.environ['API_URL']
        self.id_token = os.environ['ID_TOKEN']
        self.headers = {
            'Authorization': self.id_token,
            'Content-Type': 'application/json'
        }
    
    def test_flight_search(self):
        """Test flight search functionality"""
        payload = {"prompt": "What are flights to Seattle?"}
        response = requests.post(self.api_url, headers=self.headers, json=payload)
        assert response.status_code == 200
        assert 'Alaska Airlines' in response.json()['response'] or 'Delta' in response.json()['response']
        print("✓ Flight search test passed")
    
    def test_attraction_reservation(self):
        """Test attraction reservation"""
        payload = {"prompt": "Reserve Space Needle for Dec 20th at 9 AM"}
        response = requests.post(self.api_url, headers=self.headers, json=payload)
        assert response.status_code == 200
        assert 'reserved' in response.json()['response'].lower()
        assert 'RES' in response.json()['response']
        print("✓ Attraction reservation test passed")
    
    def test_authentication_required(self):
        """Test that authentication is required"""
        payload = {"prompt": "Test"}
        response = requests.post(self.api_url, json=payload)
        assert response.status_code == 403
        print("✓ Authentication requirement test passed")
    
    def test_conversation_context(self):
        """Test multi-turn conversation"""
        # First request
        payload1 = {"prompt": "Book Chihuly for Dec 25"}
        resp1 = requests.post(self.api_url, headers=self.headers, json=payload1)
        assert resp1.status_code == 200
        
        # Second request (should remember context)
        payload2 = {"prompt": "What time is it?"}
        resp2 = requests.post(self.api_url, headers=self.headers, json=payload2)
        assert resp2.status_code == 200
        assert 'Dec 25' in resp2.json()['response'] or 'Chihuly' in resp2.json()['response']
        print("✓ Conversation context test passed")
    
    def run_all_tests(self):
        """Run all tests"""
        self.test_flight_search()
        self.test_attraction_reservation()
        self.test_authentication_required()
        self.test_conversation_context()
        print("\n✅ All tests passed!")

# Run tests
if __name__ == "__main__":
    tests = TravelAgentTests()
    tests.run_all_tests()
```

**Run the tests:**
```bash
export API_URL="https://pci9npwhe4.execute-api.us-west-2.amazonaws.com/prod/chat"
export ID_TOKEN="<your-token>"
python3 test_agent.py
```

---

## Performance Metrics

### Typical Performance

| Metric | Value | Notes |
|--------|-------|-------|
| Cold start | 2.7s | First Lambda invocation |
| Warm start | 300-500ms | Subsequent invocations |
| API latency | 600-800ms | Including Bedrock inference |
| Tool execution | 50-200ms | Per tool call |
| Session retrieval | 50ms | From S3 |
| Concurrent users | 100+ | Limited by Lambda concurrency |

### Load Test Results (Example)

```
Requests: 1000
Successful: 995 (99.5%)
Failed: 5 (0.5%)
Average response time: 650ms
Min response time: 350ms
Max response time: 1200ms
Throughput: 50 requests/sec (sustained)
```

---

## Monitoring & Debugging

### CloudWatch Logs

```bash
# View Lambda logs
aws logs tail /aws/lambda/strands-travel-agent --follow

# Search for errors
aws logs filter-log-events \
  --log-group-name /aws/lambda/strands-travel-agent \
  --filter-pattern "ERROR"

# Get logs for specific request
aws logs filter-log-events \
  --log-group-name /aws/lambda/strands-travel-agent \
  --filter-pattern "RequestId: <request-id>"
```

### X-Ray Tracing (Optional)

```bash
# Enable X-Ray tracing
aws lambda update-function-configuration \
  --function-name strands-travel-agent \
  --tracing-config Mode=Active

# View traces in AWS Console or via CLI
aws xray get-trace-summaries \
  --start-time $(date -u -d '10 minutes ago' +%s) \
  --end-time $(date -u +%s)
```

---

## Summary

This test suite covers:
- ✅ Basic functionality
- ✅ Multi-turn conversations
- ✅ Authentication
- ✅ Error handling
- ✅ Load testing
- ✅ Integration scenarios

Use these tests regularly to ensure the agent works correctly as you make changes or add new features.
