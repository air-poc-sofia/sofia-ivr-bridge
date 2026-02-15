# Project Guidelines — Agentic IVR (URA)

## Overview

Conversational IVR system built on **Azure Communication Services (ACS)** + **Azure AI Voice Live API**, using **Java 21 / Spring Boot 4.x with Spring MVC + Virtual Threads** (imperative stack). The service acts as an audio bridge between ACS and Voice Live API via two concurrent WebSocket connections per call.

## Architecture

See [URA - Architecture Q&A.md](../URA%20-%20Architecture%20Q%26A.md) for full rationale.

**Core components** (all inside one Spring Boot service):

| Component | Responsibility |
|-----------|---------------|
| **Webhook Controller** (REST) | Receives `IncomingCall` from Event Grid + mid-call callbacks from ACS |
| **WebSocket Server Endpoint** | Accepts audio stream from ACS (`wss://`, PCM 24K mono) via JSR 356 `@ServerEndpoint` |
| **WebSocket Client** | Connects outbound to Voice Live API (`/voice-agent/realtime`) via JDK `java.net.http.WebSocket` |
| **Audio Bridge Service** | Forwards audio bidirectionally between both WebSockets |
| **Tool Call Handler** | Processes `tool_call` events from Voice Live, calls backends via APIM |
| **Call Automation Client** | Answers/hangs up calls via ACS Call Automation SDK |

**Critical design rules:**
- Event Grid → IVR Service: **direct** (no APIM) — calls ring ~30s, latency kills UX
- ACS mid-call webhooks → IVR Service: **direct** — even more latency-sensitive
- APIM is only for **outbound** calls to backend APIs (Oracle BRM, CRM, etc.)
- WebSocket audio path must **never** pass through HTTP gateways or middleware

## Code Style

- **Java 21**, Spring Boot 4.x, **Spring MVC + Virtual Threads** (`spring.threads.virtual.enabled=true`) — imperative non-blocking I/O via Project Loom
- Use standard imperative Java; avoid `Mono`/`Flux` reactive types
- ACS WebSocket server: **JSR 356** `@ServerEndpoint` + `@OnMessage` (Tomcat servlet container, ACS SDK quickstart pattern)
- Voice Live WebSocket client: **`java.net.http.WebSocket`** (JDK built-in) or equivalent
- ACS SDK: `azure-communication-callautomation` — use `CallAutomationClient` (synchronous), `StreamingData.parse()` and `OutStreamingData.getStreamingDataForOutbound()`
- Voice Live connection: raw WebSocket client to `wss://<foundry-resource>/voice-agent/realtime` (OpenAI Realtime API-compatible event schema)
- Package structure: group by feature/layer (e.g., `webhook`, `websocket`, `bridge`, `toolcall`, `config`)

## Build and Test

**Build tool:** Maven (preferred for ACS Java SDK ecosystem).

```bash
# Build
./mvnw clean package

# Run tests
./mvnw test

# Run locally
./mvnw spring-boot:run
```

**Testing approach:** Test-first development. Run full test suite after each task; never proceed with failing tests.

## Security

- **Zero hardcoded secrets** — all secrets in Azure Key Vault, accessed via managed identity
- Authenticate to Voice Live API using `DefaultAzureCredential` (managed identity), not API keys
- Validate **JWT** on every inbound webhook and WebSocket connection from ACS
- Validate Event Grid `SubscriptionValidationEvent` handshake
- All WebSocket connections use `wss://` (never `ws://`)
- ACS webhook callback IP allowlist: `52.112.0.0/14`, `52.122.0.0/15`
- RBAC role for Voice Live: `Cognitive Services User` on the Foundry resource

## Project Conventions

- **BMAD framework** is used for project management — planning artifacts go in `planning/planning-artifacts/`, implementation artifacts in `planning/implementation-artifacts/`, docs in `docs/`
- When implementing stories, read the full story file first, execute tasks in order, mark complete only when tests pass
- Audio format between ACS and service: **PCM 24K mono** — no transcoding needed (Voice Live uses the same format)
- Each active call holds two WebSocket connections — size containers accordingly (512MB–1GB per instance)
- For high concurrency, virtual threads avoid thread-per-connection bottleneck without reactive complexity

## Integration Points

| Integration | Protocol | Auth |
|------------|----------|------|
| ACS Event Grid → Service | HTTPS webhook | Entra ID bearer token |
| ACS Call Automation → Service | HTTPS callback | Signed JWT in headers |
| ACS Audio → Service | WSS | JWT in auth header |
| Service → Voice Live API | WSS | Managed identity / `DefaultAzureCredential` |
| Service → APIM → Backends | HTTPS | Managed identity + OAuth2 bearer |
| Service → Key Vault | HTTPS | Managed identity |

## Key References

- [ACS Call Automation Java SDK](https://learn.microsoft.com/azure/communication-services/how-tos/call-automation/audio-streaming-quickstart)
- [Azure AI Voice Live API](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live)
- [ACS + OpenAI Java Sample](https://github.com/Azure-Samples/communication-services-java-quickstarts/tree/main/CallAutomation_OpenAI_Sample)
- [ACS + Voice Live .NET Sample](https://github.com/Azure-Samples/communication-services-dotnet-quickstarts/tree/main/CallAutomation_AzureAI_VoiceLive)
