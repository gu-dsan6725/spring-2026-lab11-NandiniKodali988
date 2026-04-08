# Task 3: Observations

## What A2A messages were exchanged between agents

The test script sent two A2A messages to the Travel Assistant at `http://127.0.0.1:10001`. Both used the JSON-RPC 2.0 `message/send` method.

**Message 1: Flight search (no agent discovery needed)**

The test client sent:
```
POST http://127.0.0.1:10001
{
  "jsonrpc": "2.0",
  "id": "test-Search for",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [{"kind": "text", "text": "Search for flights from New York to Los Angeles on 2025-12-20"}],
      "messageId": "msg-Search for"
    }
  }
}
```
The Travel Assistant responded with 200 OK (3210 bytes). It handled this request on its own using its built-in `search_flights` tool and returned a list of available flights.

**Message 2: Booking request (required agent discovery)**

The test client sent:
```
POST http://127.0.0.1:10001
{
  "jsonrpc": "2.0",
  "id": "test-I want to",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [{"kind": "text", "text": "I want to book flight ID 1. I need you to reserve 2 seats, confirm the reservation, and process the payment. You don't have these booking capabilities yourself, so you'll need to find and use an agent that can handle flight reservations and confirmations."}],
      "messageId": "msg-I want to"
    }
  }
}
```
The Travel Assistant responded with 200 OK (5723 bytes). This time it recognized it did not have booking tools, so it triggered agent discovery, found the Flight Booking Agent at `http://127.0.0.1:10002/`, and forwarded the booking request to it as a second A2A `message/send` call before returning the combined result.

From the debug log:
```
2026-04-08 19:51:46 - POST http://127.0.0.1:10001 (message 1, search)
2026-04-08 19:51:52 - 200 OK, 3210 bytes
2026-04-08 19:51:52 - POST http://127.0.0.1:10001 (message 2, booking)
2026-04-08 19:52:05 - 200 OK, 5723 bytes
```

## How the Travel Assistant discovered the Flight Booking Agent

The flow:
- Travel Assistant received a booking request from the user
- Recognized that it does not have booking capabilities
- Called `discover_remote_agents` with a natural language query
- Registry Stub searched its registered agents and returned the Flight Booking Agent card
- Travel Assistant read the card, got the endpoint URL `http://127.0.0.1:10002/` and available skills
- Called `invoke_remote_agent` to forward the request to the Flight Booking Agent
- Returned the combined result back to the user

## The JSON-RPC request/response format you observed

Every message in the system follows the JSON-RPC 2.0 format. The request has a `jsonrpc` version field, a unique `id`, a method of `message/send`, and a `params` block containing the actual message.

Request format:
```json
{
  "jsonrpc": "2.0",
  "id": "test-caa4352c",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "kind": "text",
          "text": "Search for flights from SF to NY on 2025-11-15"
        }
      ],
      "messageId": "test-msg-f31823f1"
    }
  }
}
```

Response format:
```json
{
  "id": "test-caa4352c",
  "jsonrpc": "2.0",
  "result": {
    "artifacts": [
      {
        "artifactId": "135dda9f-5ee9-4d12-bf38-af4c029d1e76",
        "name": "agent_response",
        "parts": [
          {
            "kind": "text",
            "text": "I found 3 flights from SF to NY on November 15, 2025..."
          }
        ]
      }
    ],
    "contextId": "776fb8f9-2b30-481d-9eb2-38075edbe983",
    "history": [...]
  }
}
```

- The response wraps the agent's reply inside `artifacts`, and each artifact has a list of `parts` where the actual content lives
- The `contextId` ties the entire conversation history for the session together
- The message content itself is kept separate from the envelope. The JSON-RPC layer just handles the transport and the A2A structure lives inside `params.message`

## What information was in the agent card and how it was used

Example of the Flight Booking Agent card:

```json
{
  "name": "Flight Booking Agent",
  "description": "Flight booking and reservation management agent",
  "url": "http://127.0.0.1:10002/",
  "protocolVersion": "0.3.0",
  "preferredTransport": "JSONRPC",
  "capabilities": { "streaming": true },
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text"],
  "skills": [
    { "id": "check_availability", "description": "Check seat availability for a specific flight." },
    { "id": "reserve_flight", "description": "Reserve seats on a flight for passengers." },
    { "id": "confirm_booking", "description": "Confirm and finalize a flight booking." },
    { "id": "process_payment", "description": "Process payment for a booking (simulated)." },
    { "id": "manage_reservation", "description": "Update, view, or cancel existing reservations." }
  ]
}
```

- `url`: tells other agents where to send the A2A message
- Skills list: the Travel Assistant reads this to check if the agent has the capabilities required for completing the user request
- `preferredTransport`: confirms that the agent speaks JSONRPC so the Travel Assistant knows it can use the standard A2A message format
- `protocolVersion`: confirms compatibility between the two agents

## Your observations about the benefits and limitations of this approach

Benefits:
- Makes the system easy to extend since you can add new agents to the registry without having to change any existing code
- The agent card makes the system self-documenting so there is no need for separate API documentation

Limitations:
- Discovery quality depends on how well the skill descriptions are written and what search technique is being used
- The registry is a single point of failure, so if it goes down the Travel Assistant cannot discover any other agents even if those agents are still running