# LLM Gateway (vLLM + Ollama) with AWS Serverless + Moto

A single, intelligent API gateway that fronts your local LLM runtimes (vLLM for speed, Ollama for multimodal), using serverless components for production (API Gateway, SQS, DynamoDB) and Moto for free local emulation. Your Mac (or local machine) runs the actual LLM inference.

## What this provides

- API "front door" (Lambda-style handler) that:
  - Fetches conversation history by `conversation_id` from DynamoDB
  - Decides routing: vLLM (text-only) vs Ollama (multimodal)
  - Enqueues jobs to SQS (priority-aware)
- Worker that:
  - Polls priority then default queues
  - Calls local LLMs via OpenAI-compatible `/v1/chat/completions`
  - Updates conversation history in DynamoDB with user+assistant messages
- Local dev:
  - Moto emulation for DynamoDB and SQS
  - Express server that mounts the Lambda handler

## Project structure

- `src/middleware/handler.js` — API handler (for API Gateway/Lambda)
- `src/local/server.js` — Express shim for local dev
- `src/worker/worker.js` — SQS polling worker
- `src/shared/aws.js` — AWS SDK v3 clients with Moto endpoint support
- `src/shared/historyStore.js` — DynamoDB helpers for conversation history
- `src/shared/routing.js` — routing decision logic
- `src/shared/logger.js` — simple file+console logger
- `tests/basic-routing.test.js` — minimal routing test (no framework)

## DynamoDB item shape

Partition key: `conversation_id` (String)

Item example:

```json
{
  "conversation_id": "abc-123",
  "messages": [
    { "role": "user", "content": "Hi" },
    { "role": "assistant", "content": "Hello!" }
  ],
  "updated_at": "2025-11-02T12:34:56.000Z"
}
```

## Local development

Recommended: LocalStack + DynamoDB Local via OpenTofu

1) Provision local infra (LocalStack + DynamoDB Local + queues/table):

```bash
cd ../../infra/opentofu
tofu init
tofu apply -auto-approve
```

2) Export env vars and install deps:

```bash
export AWS_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export SQS_ENDPOINT_URL=http://localhost:4566
export DYNAMODB_ENDPOINT_URL=http://localhost:8000
export DDB_TABLE=Conversations
export SQS_QUEUE_URL=$(tofu output -raw sqs_default_queue_url)
export SQS_PRIORITY_QUEUE_URL=$(tofu output -raw sqs_priority_queue_url)

cd ../../llm-gateway
npm install
```

3) Run the local API and worker in two terminals:

```bash
npm run start
npm run worker
```

- POST http://localhost:8787/chat

```json
{
  "conversation_id": "abc-123",
  "user_message": "Caption this image",
  "images": [{ "image_url": "http://localhost:11434/examples/cat.jpg" }],
  "priority": "high"
}
```

- GET http://localhost:8787/conversation?conversation_id=abc-123

## Production notes

- Deploy `src/middleware/handler.js` as a Lambda behind API Gateway (HTTP API v2)
- Point environment variables to real AWS endpoints (remove local emulator endpoints)
- Use two SQS queues (priority + default) with suitable DLQs
- Permit the worker host to reach local LLMs; when remote, consider tunneling/edge

## Compatibility with Ollama Web UI

This gateway exposes conversation-by-ID and enqueue semantics. You can adapt an Ollama Web UI fork to call this gateway instead of storing chat state on the client. The worker writes both the user and assistant messages to the conversation record to keep long-term context server-side.

## Caveats

- Moto’s API Gateway v2/Lambda emulation is limited. For local HTTP, this repo uses an Express shim that mirrors the Lambda handler behavior.
- vLLM and Ollama must be running locally and reachable. Both paths are coded against the OpenAI-compatible `/v1/chat/completions` interface for simplicity.

## License

MIT
