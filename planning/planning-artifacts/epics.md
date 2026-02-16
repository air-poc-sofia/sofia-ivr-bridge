---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - planning/planning-artifacts/prd.md
  - planning/planning-artifacts/architecture.md
---

# agentic-ivr - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for agentic-ivr, decomposing the requirements from the PRD and Architecture into implementable stories. No UX Design document exists — this is a backend-only audio bridge service.

## Requirements Inventory

### Functional Requirements

**50 FRs across 10 capabilities** (FR15 absorbed into FR13+FR31 per PRD elicitation history)

#### Capability 1: Call Ingestion & Lifecycle Management (6 FRs)

- FR1: The system SHALL receive `IncomingCall` events from Azure Event Grid via HTTPS webhook and initiate the call-answer sequence within the configured ring timeout (C-03).
- FR2: The system SHALL answer incoming calls using ACS Call Automation with bidirectional media streaming enabled (PCM 24K mono).
- FR3: The system SHALL handle ACS mid-call callback events (`CallConnected`, `CallDisconnected`, `PlayFailed`, `PlayCompleted`) and update internal call state accordingly.
- FR4: The system SHALL perform graceful teardown of all resources (both WebSocket connections, call state, metrics) when a call ends, regardless of which side initiates disconnection.
- FR5: The system SHALL deduplicate ACS callback events using a cache keyed on `callId` + `eventType` + `correlationId` with a configurable TTL (C-56), ensuring idempotent processing.
- FR6: When the call approaches max duration (C-29 minus C-30 seconds remaining), the system SHALL inject a context message to Voice Live prompting the AI to wrap up the conversation naturally, and enforce a hard disconnect at max duration if the call does not end voluntarily.

#### Capability 2: Real-Time Audio Communication (4 FRs)

- FR7: The system SHALL accept inbound WebSocket connections from ACS at the versioned path `/ws/v1` and parse incoming binary frames into `AudioMetadata` and `AudioData` packets.
- FR8: The system SHALL send outbound audio to ACS via the same WebSocket connection using the ACS SDK's outbound streaming data format.
- FR9: The system SHALL forward audio bidirectionally between ACS and Voice Live in real-time without buffering, inspecting, or transforming the audio content (protocol translation only).
- FR10: The system SHALL enforce a linked lifecycle between the ACS and Voice Live WebSocket connections — closure of either side SHALL trigger graceful teardown of both within the configured timeout (C-31).

#### Capability 3: Conversational AI Integration (5 FRs)

- FR11: The system SHALL establish an outbound WebSocket connection to the Voice Live API endpoint (C-07) with the configured API version (C-08) within the connect timeout (C-10).
- FR12: The system SHALL authenticate to the Voice Live API using managed identity credentials, with no API keys in code or configuration.
- FR13: The system SHALL send a `session.update` event to Voice Live upon connection, configuring the session with the externalized system prompt (C-12), voice model (C-13), tool definitions loaded from the externalized definitions file (C-60), and turn detection settings — including VAD type, threshold, silence duration, noise suppression, and echo cancellation (C-14 through C-18). If Voice Live rejects the session, the system SHALL log the rejection at WARN level including `correlationId`, error code/message, and system prompt size.
- FR14: Upon receiving `input_audio_buffer.speech_started` from Voice Live, the system SHALL immediately (i) stop forwarding Voice Live audio frames to ACS and (ii) cancel any in-progress ACS Play operation, treating both actions as idempotent operations safe to invoke redundantly.
- FR16: The system SHALL support injecting system-level messages to Voice Live mid-session (e.g., max-duration wrap-up notification) when triggered by lifecycle events.

#### Capability 4: Connection Resilience & Recovery (6 FRs)

- FR17: The system SHALL implement heartbeat/keepalive on both WebSocket connections and detect connection staleness within the configured idle timeout (C-05, C-11). Upon detecting staleness, the system SHALL force-close the stale WebSocket connection, triggering the linked lifecycle teardown per FR10.
- FR18: The system SHALL implement a circuit breaker on Voice Live connections that trips to OPEN state after N consecutive connection failures (C-22), preventing new calls from attempting doomed connections.
- FR19: The circuit breaker SHALL transition through CLOSED → OPEN → HALF-OPEN → CLOSED states, with configurable half-open delay (C-23) and success threshold for recovery (C-24).
- FR20: If ACS reconnects within the configured reconnection window (C-28), the system SHALL attempt to resume the existing Voice Live session rather than starting a new one. If resume fails, the system SHALL fall back to FR21 behavior.
- FR21: When the circuit breaker is OPEN or session resumption fails, the system SHALL play the appropriate fallback audio prompt to the caller before disconnecting gracefully.
- FR46: The system SHALL detect when a Voice Live audio response stalls mid-stream — defined as receiving `response.audio.delta` events followed by no further audio events and no `response.done` within C-55 (default 5000ms). On stall detection, the system SHALL log a WARN, play the apology prompt (C-37), and initiate graceful teardown.

#### Capability 5: Security & Authentication (6 FRs)

- FR22: The system SHALL validate JWT tokens in the `Authorization` header during WebSocket upgrade handshake, rejecting connections with HTTP 401 before upgrading if the token is invalid, expired, or missing.
- FR23: The system SHALL authenticate to ACS Call Automation using managed identity credentials, with no connection strings or API keys in code, configuration, or secret stores.
- FR24: The system SHALL authenticate to Voice Live API using managed identity credentials. Infrastructure prerequisite: RBAC role `Cognitive Services User` on the Foundry resource.
- FR25: The system SHALL validate Entra ID bearer tokens on Event Grid webhook requests and complete the `SubscriptionValidationEvent` handshake during subscription registration.
- FR26: All WebSocket connections (both ACS-facing and Voice Live-facing) SHALL use `wss://` protocol exclusively; the system SHALL reject or never initiate `ws://` connections.
- FR27: The system SHALL implement a testable credential health abstraction that reports cached token status without triggering live token refresh, used by the readiness probe.

#### Capability 6: Operational Observability (5 FRs)

- FR28: The system SHALL emit all log events in structured JSON format, with every entry including `correlationId`, `timestamp`, `level`, and `component` fields. When an OTel trace context is active, entries SHALL include `traceId` and `spanId`.
- FR29: The system SHALL emit distributed tracing spans for the 4 defined call lifecycle operations (`ivr.call`, `ivr.call.answer`, `ivr.call.voicelive.connect`, `ivr.call.audio.bridge`).
- FR30: The system SHALL expose three-tier health probes: startup (first managed identity token acquisition), liveness (Spring context alive), readiness (cached credential status + Voice Live DNS resolution).
- FR31: All configurable parameters (per the Configuration Catalog) SHALL be externalized via Spring configuration properties with validation constraints, supporting per-environment overrides via Spring profiles (`test`, `local`, `prod`).
- FR32: The system SHALL expose Micrometer metrics for all 14 defined metric names (prefixed `ivr.*`) covering active calls, latency distributions, circuit breaker state, security rejection counts, and audio forwarding counters.

#### Capability 7: Call Admission & Lifecycle Edge Cases (6 FRs)

- FR33: The system SHALL track instance-local active concurrent calls and reject incoming calls with a spoken busy prompt (C-38) when the instance-local active call count exceeds the per-instance admission threshold (C-26).
- FR34: On SIGTERM, the system SHALL mark readiness probe unhealthy immediately, send a wrap-up notification to in-flight Voice Live sessions, and allow active calls to complete within drain timeout (C-34) before forcefully terminating connections.
- FR35: The system SHALL play pre-recorded audio prompts via ACS Play action for: (a) comfort tone if VL connection exceeds C-33, (b) fallback greeting if no AI audio within C-32, (c) apology prompt on unexpected VL disconnect, (d) busy prompt on admission rejection, (e) service-unavailable prompt when circuit breaker is open.
- FR42: The system SHALL periodically sweep for call sessions exceeding max-duration plus buffer (C-52 interval) and forcefully terminate orphaned sessions where neither WebSocket close nor callback arrived.
- FR43: The system SHALL terminate WebSocket connections that receive no audio frames within the idle timeout (C-54) after successful upgrade.
- FR47: When an ACS Play action fails (`PlayFailed` callback), the system SHALL NOT retry or play a substitute prompt. It SHALL log `PlayFailed` at ERROR level with `correlationId`, prompt identifier, and ACS failure reason, then proceed directly to graceful call termination.

#### Capability 8: Defense-in-Depth Security (5 FRs)

- FR36: The system SHALL generate a cryptographically random token with minimum length (C-50) and embed it in each callback URL registered with ACS, validating the token on every callback request.
- FR37: The system SHALL validate the `eventTime` of each Event Grid event and reject events older than C-45 to prevent replay attacks.
- FR38: The system SHALL enforce per-IP rate limiting on WebSocket handshake attempts, rejecting connections exceeding C-46 per C-47 window with HTTP 429.
- FR39: The system SHALL bind Spring Boot Actuator to a separate management port (C-48), exposing only health/liveness/readiness/startup endpoints, returning 404 for all other actuator paths.
- FR40: The system SHALL return generic JSON error bodies on all HTTP error responses, with no stack traces, class names, or framework-specific details.

#### Capability 9: Operational Safety Nets (3 FRs)

- FR41: The system SHALL periodically probe Voice Live API connectivity (C-25 interval) and preemptively trip the circuit breaker if probes fail.
- FR44: The system SHALL process Event Grid event batches individually, returning HTTP 200 for the batch even if individual events fail processing, logging failures per event.
- FR45: The system SHALL use lenient JSON deserialization for Voice Live events (ignoring unknown fields) and increment a parse error counter metric on deserialization failures.

#### Capability 10: Tool Call Framework (4 FRs)

- FR48: Upon receiving a `tool_call` event from Voice Live, the system SHALL dispatch an HTTPS request to the configured API gateway base URL (C-57) using `java.net.http.HttpClient` on a virtual thread, authenticating with `DefaultAzureCredential` (scope C-57b). The request SHALL include the tool call name, arguments, and `correlationId`.
- FR49: Upon receiving a successful HTTP response from the API gateway, the system SHALL forward the result to Voice Live as a `tool_call_output` event containing the tool call ID and the response payload.
- FR50: If the API gateway does not respond within C-58, or returns an HTTP error (4xx/5xx), the system SHALL return a `tool_call_output` event to Voice Live with an error payload, log the failure at WARN level, and increment `ivr.tool_call.errors`. The system SHALL NOT terminate the call due to a tool call failure.
- FR51: At session initialization, the system SHALL load tool definitions from C-60, validate the JSON structure, and include the tool definitions in the `session.update` event sent to Voice Live (FR13). If the definitions file is missing or contains invalid JSON, the system SHALL log at ERROR level and start the session without tool definitions (degraded mode).

### Non-Functional Requirements

**47 NFRs across 6 categories**

#### Category 1: Performance (10 NFRs)

- NFR-P1: Time-to-First-Audio — elapsed time from `CallConnected` to first audio byte sent to ACS. Target: < 3s p95. JWKS cache MUST be pre-warmed during startup.
- NFR-P2: AI Response Latency — elapsed time from VAD end-of-speech to first `response.audio.delta` from Voice Live. Target: < 1s p95.
- NFR-P3: Barge-In Latency — elapsed time from `speech_started` received to `StopAudio` sent to ACS. Target: < 100ms p95.
- NFR-P4: Audio Forwarding Latency — single audio frame bridge traversal. Target: < 50ms p99.
- NFR-P5: Call Answer Latency — `IncomingCall` received to `answerCall()` response. Target: < 3s p95.
- NFR-P6: Voice Live Connect Latency — WebSocket handshake to `session.created`. Target: ≤ 2s p95, ≤ 3s p99.
- NFR-P7: Per-Call Memory — max heap per active call. Target: ≤ 200 KB per call.
- NFR-P8: Zero-Copy Audio Forwarding — no intermediate buffer copies (protocol translation permitted).
- NFR-P9: Application Startup Time — JVM launch to readiness probe 200. Target: ≤ 30s (cold JVM). Credential acquisition capped at 15s.
- NFR-P10: Tool Call Round-Trip Latency — `tool_call` received to `tool_call_output` sent. Target: ≤ 5s p95.

#### Category 2: Reliability (14 NFRs)

- NFR-R1: Service Availability — ≥ 99.9% uptime.
- NFR-R2: Linked Lifecycle Teardown — peer WebSocket closed + resources freed ≤ 3s. Applies to non-reconnectable closures only (codes 1000, 1001).
- NFR-R3: Circuit Breaker Activation — trips at exactly C-22 consecutive failures.
- NFR-R4: Graceful Shutdown Drain — 100% in-flight calls drained on SIGTERM within C-34.
- NFR-R5: Zero Race Conditions — concurrent state transitions produce no inconsistent state, resource leaks, or duplicate teardown.
- NFR-R6: Callback Deduplication — duplicate ACS callbacks processed exactly once.
- NFR-R7: Zero Orphaned Sessions — no Voice Live sessions remain after call end + timeout.
- NFR-R8: Call Completion Rate — ≥ 99% graceful ends / total answered.
- NFR-R9: Heartbeat Detection Latency — staleness detected ≤ 3s from last heartbeat.
- NFR-R10: Configuration Fail-Fast — app fails to start within 5s if required config is missing/invalid.
- NFR-R11: Observability Signal Completeness — ≥ 99% of all applicable log events + metrics emitted per call lifecycle.
- NFR-R12: Reconnection Window Compliance — VL session held for C-28 (5s) on abnormal ACS close (code 1006), then teardown. Normal closes trigger immediate R2 teardown.
- NFR-R13: Stall Detection Accuracy — < 1% false positives on mid-stream stall detection.
- NFR-R14: Tool Call Timeout Handling — 100% timeout-handling coverage; 0 call terminations from tool call failures.

#### Category 3: Scalability (6 NFRs)

- NFR-S1: Concurrent Call Capacity — 50 calls per instance without latency degradation.
- NFR-S2: Sustained Load Stability — 50 calls × 30 min: zero OOM, zero thread starvation.
- NFR-S3: Instance Memory Limit — ≤ 1 GB total container memory at full capacity.
- NFR-S4: Stateless Horizontal Scaling — zero shared mutable state between instances.
- NFR-S5: Admission Control Isolation — rejection is instance-local, no cross-instance impact.
- NFR-S6: Network Bandwidth — ≤ 10 MB/s per instance at full capacity.

#### Category 4: Security (10 NFRs)

- NFR-SEC1: WebSocket Authentication Coverage — 100% JWT validation on `/ws/v1` upgrades.
- NFR-SEC2: Zero Secrets in Code/Config — all secrets resolved at runtime via managed identity or K8s CSI.
- NFR-SEC3: WSS-Only Enforcement — zero `ws://` connections.
- NFR-SEC4: Zero Critical/High CVEs — no CVSS ≥ 7.0 in production dependency tree.
- NFR-SEC5: Zero Information Leakage — no stack traces, class names, or internal details in external responses.
- NFR-SEC6: Actuator Isolation — management port (C-48), only health endpoints exposed.
- NFR-SEC7: Event Replay Protection — reject events with `eventTime` older than C-45.
- NFR-SEC8: Zero Audio in Log Output — no raw audio bytes in any log level.
- NFR-SEC9: Zero Audio Bytes at Rest — audio exists only in JVM heap during active forwarding.
- NFR-SEC10: Zero Rate-Limiter False Positives for ACS Traffic — ACS IP ranges exempt from per-IP rate limiting.

#### Category 5: Integration Reliability (7 NFRs)

- NFR-I1: Fallback Prompt Coverage — 100% coverage for all 5 failure modes (FR35a–e).
- NFR-I2: No Reactive Types — zero `Mono`, `Flux`, `Publisher` in production code.
- NFR-I3: Lenient Voice Live Deserialization — unknown fields ignored, parse failures increment counter.
- NFR-I4: Config-Only Voice Live Version Changes — version update = config-only deploy.
- NFR-I5: Cloud-Free Test Suite — `./mvnw test` passes with zero Azure env vars, zero network calls. In-process test JWT issuer exercises full validation path.
- NFR-I6: Developer Onboarding Speed — git clone to first call ≤ 30 minutes (pre-provisioned Azure resources).
- NFR-I7: Deterministic Single-Command Build — `./mvnw clean package` produces runnable JAR with zero manual steps.

### Additional Requirements

#### From Architecture: Starter Template (Impacts Epic 1 Story 1)

- **Greenfield Spring Initializr project** (Option C selected) with: `web`, `websocket`, `actuator`, `validation`, `configuration-processor` starters
- Spring Boot 4.0.x baseline (ADR-8)
- `spring.threads.virtual.enabled=true` MUST be present from day one
- `ServerEndpointExporter` bean registration required (Spring Boot does not auto-register JSR 356 endpoints)
- Package layout: `com.sofia.ivr.{config,webhook,websocket,bridge,toolcall,model,exception}`

#### From Architecture: Technical Constraints (17)

- TC-1: Java 21 LTS, Spring Boot 4.x, Spring MVC + virtual threads
- TC-2: JSR 356 `@ServerEndpoint` for ACS WebSocket server
- TC-3: `java.net.http.WebSocket` for Voice Live client
- TC-4: `azure-communication-callautomation` SDK for call control
- TC-5: `DefaultAzureCredential` for all Azure auth
- TC-6: No API keys anywhere — managed identity only
- TC-7: PCM 24K mono — no transcoding
- TC-8: WebSocket audio path direct — no HTTP gateways
- TC-9: Event Grid → Service direct (no APIM)
- TC-10: ACS callbacks → Service direct (no APIM)
- TC-11: APIM only for outbound to backends
- TC-12: Zero audio persistence — biometric data constraint
- TC-13: Single Spring Boot service — no microservice decomposition
- TC-14: AKS deployment, Brazil South region
- TC-15: Virtual thread pinning guard — no `synchronized` blocks
- TC-16: WebSocket send serialization — one sender per connection
- TC-17: Audio path purity — zero business logic in audio forwarding path

#### From Architecture: Cross-Cutting Concerns (7)

- CC-1: Correlation — explicit `CallContext` passing with `correlationId` (ACS call ID)
- CC-2: Call State Machine — `EnumMap` transition table, 4 behavioral states: INITIALIZING → STREAMING → TERMINATING → TERMINATED
- CC-3: Auth Multiplexing — 3 JWT validation flows (Entra ID, ACS webhook, ACS WebSocket) + pre-emptive token refresh at 75% lifetime
- CC-4: Graceful Degradation Hierarchy — ordered fallback behavior per failure type
- CC-5: Configuration Validation — `@ConfigurationProperties` with validation constraints, fail-fast on boot
- CC-6: Resource Cleanup Determinism — `try-finally` on all WebSocket paths
- CC-7: Correlated Failure Isolation — one call's failure cannot cascade to others

#### From Architecture: Key Decisions & ADRs

- ADR-7: Keep Azure SDK Netty default HTTP transport (no swap to JDK HttpClient)
- ADR-8: Spring Boot 4.0.x baseline
- ADR-9: OTel Spring Boot autoconfig over Java Agent
- ADR-10: ConcurrentHashMap over Redis for call state
- ADR-11: Nimbus JOSE+JWT direct for all 3 JWT validation paths (no Spring Security)
- Decision 1: In-memory call state store (`ConcurrentHashMap<String, CallContext>`)
- Decision 3: Event Grid synchronous SubscriptionValidation handshake
- Decision 4: Sealed interfaces + records for Voice Live event schema
- Decision 5: RestClient + Resilience4j circuit breaker for tool call dispatch
- Decision 6: Sealed exception hierarchy with surface-specific translation
- Decision 7: Multi-stage Dockerfile with layered JAR extraction
- Decision 9: Configuration externalization via env vars + Spring relaxed binding
- Decision 10: JSON structured logging with profile switching

#### From Architecture: Mandatory Rules (8)

- M-1: No `Mono`, `Flux`, or reactive types anywhere
- M-2: No hardcoded secrets, URLs, or connection strings
- M-3: No `Thread.sleep()` for delays
- M-4: No `synchronized` blocks (virtual thread pinning)
- M-5: No custom thread pools for I/O work
- M-6: No WebSocket audio through HTTP middleware/APIM
- M-7: All public REST endpoints must validate JWT
- M-8: No `catch (Exception e) {}` — silent swallowing forbidden

#### From Architecture: Dependencies

- `spring-boot-starter-web` — Tomcat, Spring MVC, Jackson
- `spring-boot-starter-websocket` — WebSocket support
- `spring-boot-starter-actuator` — Health, Micrometer metrics
- `spring-boot-starter-validation` — Bean validation
- `spring-boot-configuration-processor` — `@ConfigurationProperties` metadata
- `azure-communication-callautomation` — ACS Call Automation SDK
- `azure-identity` — `DefaultAzureCredential`
- `azure-sdk-bom` — Azure SDK version management
- `opentelemetry-spring-boot-starter` — OTel autoconfig
- `resilience4j-spring-boot3` — Circuit breaker
- `logstash-logback-encoder` — JSON structured logging
- `spring-boot-starter-test` — JUnit 5, Mockito, AssertJ

#### From Architecture: Project Structure

- ~60 files across 7 feature packages + resources
- Complete test directory mirroring source packages
- Integration tests: `WebhookIntegrationTest`, `WebSocketIntegrationTest`, `ToolCallIntegrationTest`
- Test fixtures: JSON events, test JWKS, audio frame binary, WireMock stubs
- Tool definitions: `get-user-data.json`, `send-invoice-registered-email.json`, `send-invoice-provided-email.json`

#### From PRD: Security Requirements (18 SEC-REQs)

- SEC-REQ-1 (P0): JWT validation on all inbound connections
- SEC-REQ-2 (P0): Entra ID auth on Event Grid webhooks
- SEC-REQ-3 (P0): ACS JWT validation on callbacks
- SEC-REQ-4 (P0): Callback URL token validation
- SEC-REQ-5 (P0): Zero audio logging / persistence
- SEC-REQ-9 (P0): Actuator endpoint isolation
- SEC-REQ-10 (P0): Error response hardening
- SEC-REQ-11 (P0): Managed identity everywhere
- SEC-REQ-14 (P0): Event replay protection
- SEC-REQ-15 (P0): WebSocket rate limiting
- SEC-REQ-6 (P1): PII masking in logs
- SEC-REQ-7 (P1): LGPD consent audit trail
- SEC-REQ-8 (P1): LGPD data subject rights
- SEC-REQ-12 (P1): Dependency vulnerability scanning
- SEC-REQ-13 (P1): Container image scanning
- SEC-REQ-16 (P1): TLS 1.2+ enforcement
- SEC-REQ-17 (P1): Network segmentation
- SEC-REQ-18 (P0): Tool call auth via managed identity

#### From PRD: UX Requirements (Caller Experience — 11 UX-REQs)

- UX-REQ-1: Silence tolerance ≤ 3s, then comfort tone
- UX-REQ-2: Consistent AI voice throughout call
- UX-REQ-3: Apologize before disconnect on failures
- UX-REQ-4: Time-to-first-audio < 3s
- UX-REQ-5: Barge-in support (interrupt AI)
- UX-REQ-6: No abrupt mid-sentence cutoffs
- UX-REQ-7: AI response latency < 1s after speech
- UX-REQ-8: Barge-in halts AI audio within 100ms
- UX-REQ-9: Max-duration wrap-up notification
- UX-REQ-10: Graceful circuit-breaker-open message
- UX-REQ-11: Admission rejection spoken message

#### From PRD: Priority Tiers

- **P0 (5-Day MVP Sprint):** 30 items in day-by-day plan — core call lifecycle, audio bridge, Voice Live integration, security, observability, tool calls
- **P1 (Growth):** PII masking, LGPD compliance, call transfer, advanced metrics
- **P2 (Scale):** KEDA autoscaling, multi-region, session resumption

### FR Coverage Map

- FR1: Epic 1 — Receive IncomingCall from Event Grid, initiate call-answer sequence
- FR2: Epic 1 — Answer incoming calls via ACS with bidirectional media streaming
- FR3: Epic 1 — Handle ACS mid-call callback events, update call state
- FR4: Epic 4 — Graceful teardown of all resources on call end
- FR5: Epic 7 — Deduplicate ACS callback events
- FR6: Epic 3 — Max-duration wrap-up notification and hard disconnect
- FR7: Epic 1 — Accept ACS WebSocket connections at /ws/v1, parse audio frames
- FR8: Epic 2 — Send outbound audio to ACS via WebSocket
- FR9: Epic 2 — Forward audio bidirectionally between ACS and Voice Live (zero-copy)
- FR10: Epic 2 — Linked lifecycle between ACS and Voice Live WebSockets
- FR11: Epic 2 — Establish outbound WebSocket to Voice Live API
- FR12: Epic 2 — Authenticate to Voice Live via managed identity
- FR13: Epic 2 — Send session.update with system prompt, voice model, tool defs, VAD config
- FR14: Epic 3 — Barge-in: stop VL audio forwarding + cancel ACS Play
- FR16: Epic 3 — Inject system-level messages to Voice Live mid-session
- FR17: Epic 4 — Heartbeat/keepalive and staleness detection on both WebSockets
- FR18: Epic 4 — Circuit breaker on Voice Live connections
- FR19: Epic 4 — Circuit breaker state transitions (CLOSED→OPEN→HALF-OPEN→CLOSED)
- FR20: Epic 4 — ACS reconnection window with Voice Live session resume
- FR21: Epic 4 — Fallback audio when circuit breaker OPEN or resume fails
- FR22: Epic 6 — JWT validation on WebSocket upgrade handshake
- FR23: Epic 1 — ACS Call Automation managed identity auth
- FR24: Epic 6 — Voice Live managed identity auth + RBAC prerequisite
- FR25: Epic 1 — Event Grid Entra ID token validation + SubscriptionValidation handshake
- FR26: Epic 6 — WSS-only enforcement
- FR27: Epic 6 — Testable credential health abstraction
- FR28: Epic 1 — Structured JSON logging with correlationId, traceId, spanId
- FR29: Epic 1 — Distributed tracing spans for call lifecycle operations
- FR30: Epic 1 — Three-tier health probes (startup, liveness, readiness)
- FR31: Epic 1 — Configuration externalization with Spring properties + validation
- FR32: Epic 1 — Micrometer metrics for all 14 metric names
- FR33: Epic 7 — Concurrent call admission control with busy prompt
- FR34: Epic 7 — Graceful shutdown: SIGTERM drain with wrap-up notification
- FR35: Epic 3 — Pre-recorded audio prompts for all 5 failure modes
- FR36: Epic 6 — Cryptographic callback URL token generation + validation
- FR37: Epic 6 — Event Grid eventTime freshness validation
- FR38: Epic 6 — Per-IP rate limiting on WebSocket handshakes
- FR39: Epic 1 — Actuator isolation on separate management port
- FR40: Epic 1 — Generic JSON error bodies, no information leakage
- FR41: Epic 4 — Periodic Voice Live connectivity probing
- FR42: Epic 7 — Orphan session sweeper
- FR43: Epic 7 — Idle WebSocket timeout after upgrade
- FR44: Epic 7 — Event Grid batch processing (individual handling, batch 200)
- FR45: Epic 7 — Lenient Voice Live JSON deserialization + parse error counter
- FR46: Epic 4 — Mid-stream audio stall detection
- FR47: Epic 3 — PlayFailed handling: log ERROR, proceed to teardown
- FR48: Epic 5 — Tool call dispatch to APIM via HttpClient on virtual thread
- FR49: Epic 5 — Forward tool call result to Voice Live as tool_call_output
- FR50: Epic 5 — Tool call timeout/error handling (error payload, no call termination)
- FR51: Epic 5 — Load tool definitions from external JSON file at session init

## Epic List

### Epic 1: Project Foundation & Call Answering

The service starts, answers incoming calls, accepts audio connections, and provides full operational visibility. A caller dials in → Event Grid delivers `IncomingCall` → the service answers via ACS → ACS opens the audio WebSocket. Includes Spring Boot project setup (starter template, all dependencies, config framework), structured logging, distributed tracing, metrics, health probes, and actuator isolation.

**FRs covered:** FR1, FR2, FR3, FR7, FR23, FR25, FR28, FR29, FR30, FR31, FR32, FR39, FR40

**Notes:**

- Story 1 must be the starter template setup (Architecture requirement)
- FR23 (ACS managed identity) needed to answer calls
- FR25 (Event Grid auth + SubscriptionValidation) needed to receive calls
- FR28/FR29/FR30/FR31/FR32 (logging, tracing, probes, config, metrics) are foundational cross-cutting concerns
- FR39/FR40 (actuator isolation, generic errors) are security baseline

---

### Epic 2: Voice Live Integration & Audio Bridge

The caller speaks and hears AI responses in real-time. The service connects outbound to Voice Live API, authenticates via managed identity, initializes the AI session (system prompt, voice model, VAD settings), and bridges audio bidirectionally between ACS and Voice Live with zero-copy forwarding.

**FRs covered:** FR8, FR9, FR10, FR11, FR12, FR13

**Notes:**

- FR9 is the core audio bridge — protocol translation only, no buffering
- FR10 establishes linked lifecycle: either WebSocket closes → both tear down
- FR12 → managed identity auth to Voice Live (no API keys)
- FR13 → session.update with externalized system prompt, voice model, tool defs, VAD config

---

### Epic 3: Conversational Intelligence & Caller Experience

The caller has a natural, responsive conversation with barge-in support, comfort tones, and graceful time management. When the caller speaks over the AI, barge-in immediately stops AI audio. If connection takes time, the caller hears a comfort tone. At max duration, the AI wraps up naturally. Failed prompt playback is handled without retries.

**FRs covered:** FR6, FR14, FR16, FR35, FR47

**Notes:**

- FR14 → barge-in: stop VL audio forwarding + cancel ACS Play (idempotent)
- FR35 → all 5 prompt types: comfort tone, fallback greeting, apology, busy, service-unavailable
- FR6/FR16 → max-duration wrap-up + hard disconnect enforcement
- FR47 → PlayFailed: log at ERROR, proceed to teardown, no retry

---

### Epic 4: Connection Resilience & Recovery

When infrastructure has issues, callers experience graceful handling instead of dead air or abrupt disconnection. Circuit breaker pattern on Voice Live connections. Heartbeat monitoring detects staleness. ACS reconnection window allows session resume. Mid-stream audio stall detection catches frozen responses. Proactive connectivity probing trips the breaker before callers are affected.

**FRs covered:** FR4, FR17, FR18, FR19, FR20, FR21, FR41, FR46

**Notes:**

- FR18/FR19 → circuit breaker: CLOSED → OPEN → HALF-OPEN → CLOSED
- FR17 → heartbeat/keepalive + staleness detection on both WebSockets
- FR20 → ACS reconnect within 5s window → attempt VL session resume
- FR21 → fallback audio when circuit breaker OPEN or resume fails
- FR41 → periodic VL connectivity probing, preemptive breaker trip
- FR46 → mid-stream stall detection (no audio delta for C-55ms after started)
- FR4 → graceful resource teardown on all disconnect paths

---

### Epic 5: Tool Call Execution

The AI can query customer data and perform actions during the conversation, making the call actionable. Voice Live sends `tool_call` events → the service dispatches HTTPS requests to APIM-fronted backends → forwards results back as `tool_call_output`. Tool definitions are loaded from external JSON files. Timeouts and errors never kill the call.

**FRs covered:** FR48, FR49, FR50, FR51

**Notes:**

- FR48 → dispatch via java.net.http.HttpClient on virtual thread, managed identity auth to APIM
- FR50 → timeout/error → return error payload to VL, increment metric, call continues
- FR51 → load tool defs from external file at session init, degraded mode if invalid

---

### Epic 6: Security Hardening

Defense-in-depth security validates every entry point, prevents replay attacks, limits abuse, and leaks zero information. JWT validation on WebSocket upgrades rejects before protocol escalation. Callback URL tokens defend against forged callbacks. Event timestamp validation blocks replays. Per-IP rate limiting on WebSocket handshakes. Credential health abstraction for readiness probes. WSS-only enforcement.

**FRs covered:** FR22, FR24, FR26, FR27, FR36, FR37, FR38

**Notes:**

- FR22 → JWT validation on /ws/v1 upgrade, reject with 401 before upgrade
- FR24 → VL managed identity auth + RBAC prerequisite (complements FR12 from Epic 2)
- FR36 → cryptographic callback URL token generation + validation
- FR37 → Event Grid eventTime freshness validation
- FR38 → per-IP rate limiting on WS handshakes (ACS IP range exempt per NFR-SEC10)
- FR27 → testable credential health abstraction (used by readiness probe from FR30)

---

### Epic 7: Admission Control & Operational Safety

The service protects itself from overload and handles every edge case — no orphaned sessions, no zombie connections, no dropped events. Concurrent call admission control rejects overflow with spoken message. Graceful shutdown drains in-flight calls on SIGTERM. Orphan session sweeper catches leaked sessions. Idle WebSocket timeout cleans up dead connections. Event batch processing and callback deduplication ensure reliability.

**FRs covered:** FR5, FR33, FR34, FR42, FR43, FR44, FR45

**Notes:**

- FR33 → instance-local admission control, busy prompt (C-38) when at capacity
- FR34 → SIGTERM: mark unhealthy → wrap-up notification → drain within C-34
- FR42 → periodic orphan session sweep
- FR43 → idle timeout on WebSocket after upgrade (no audio within C-54)
- FR5 → callback deduplication via callId+eventType+correlationId cache
- FR44 → Event Grid batch: process individually, 200 for batch, log per-event failures
- FR45 → lenient VL JSON deserialization + parse error counter

---

## Epic 1: Project Foundation & Call Answering

The service starts, answers incoming calls, accepts audio connections, and provides full operational visibility.

### Story 1.1: Spring Boot Project Scaffold & Configuration Framework

As a **developer**,
I want a fully configured Spring Boot 4.x project with all dependencies, package structure, and configuration framework,
So that all subsequent stories have a solid, consistent foundation to build upon.

**Acceptance Criteria:**

**Given** a new Spring Initializr project
**When** generated with starters (`web`, `websocket`, `actuator`, `validation`, `configuration-processor`) and all 12 architecture dependencies
**Then** the project compiles successfully with `./mvnw clean package` producing a runnable JAR (NFR-I7)

**Given** the application configuration
**When** `spring.threads.virtual.enabled=true` is set
**Then** the application uses virtual threads for request processing (TC-1)

**Given** the project structure
**When** examining packages
**Then** all 7 feature packages exist: `config`, `webhook`, `websocket`, `bridge`, `toolcall`, `model`, `exception` (Architecture Section 5)

**Given** the Spring context
**When** `ServerEndpointExporter` bean is registered
**Then** JSR 356 `@ServerEndpoint` annotations are detected and registered

**Given** `@ConfigurationProperties` classes with `sofia.*` prefix
**When** required configuration values are missing or invalid
**Then** the application fails to start within 5s with a clear validation error (FR31, NFR-R10)

**Given** Spring profiles (`test`, `local`, `prod`)
**When** a profile is active
**Then** environment-specific overrides apply via Spring relaxed binding (Decision 9)

**Given** the test profile
**When** running `./mvnw test`
**Then** tests pass with zero Azure environment variables and zero network calls (NFR-I5)

---

### Story 1.2: Observability Foundation (Logging, Tracing, Metrics & Health Probes)

As an **operations engineer**,
I want structured logging, distributed tracing, metrics, and health probes from day one,
So that I can monitor, troubleshoot, and manage the service in production.

**Acceptance Criteria:**

**Given** any log event
**When** emitted
**Then** it is in structured JSON format with `correlationId`, `timestamp`, `level`, and `component` fields (FR28)

**Given** an active OTel trace context
**When** a log event is emitted
**Then** it includes `traceId` and `spanId` fields (FR28, ADR-9)

**Given** defined call lifecycle operations
**When** processing a call
**Then** distributed tracing spans are emitted for `ivr.call`, `ivr.call.answer`, `ivr.call.voicelive.connect`, `ivr.call.audio.bridge` (FR29)

**Given** configured Micrometer metrics
**When** the application starts
**Then** all 14 metric names (prefixed `ivr.*`) are registered and ready to emit (FR32)

**Given** the startup probe
**When** queried after first managed identity token acquisition
**Then** it returns HTTP 200; credential acquisition is capped at 15s (FR30, NFR-P9)

**Given** the liveness probe
**When** queried while Spring context is alive
**Then** it returns HTTP 200 (FR30)

**Given** the readiness probe
**When** queried
**Then** it checks cached credential status and Voice Live DNS resolution (FR30)

**Given** Spring Boot Actuator
**When** bound to management port (C-48)
**Then** only `/health`, `/health/liveness`, `/health/readiness`, `/health/startup` endpoints are exposed; all other actuator paths return 404 (FR39, NFR-SEC6)

---

### Story 1.3: Event Grid Webhook & Call Ingestion

As the **IVR system**,
I want to receive and authenticate Event Grid webhook events,
So that incoming calls are securely received and the call-answer sequence is initiated.

**Acceptance Criteria:**

**Given** an Event Grid `SubscriptionValidationEvent`
**When** received at the webhook endpoint
**Then** the system completes the validation handshake by returning the `validationCode` (FR25, Decision 3)

**Given** an `IncomingCall` event from Event Grid
**When** received with a valid Entra ID bearer token
**Then** the system initiates the call-answer sequence within the configured ring timeout C-03 (FR1)

**Given** an `IncomingCall` event
**When** the `Authorization` header contains an invalid, expired, or missing Entra ID token
**Then** the system rejects the request with HTTP 401 (FR25)

**Given** any HTTP error response from the webhook controller
**When** returned to a client
**Then** the response body is a generic JSON structure with no stack traces, class names, or framework-specific details (FR40, NFR-SEC5)

**Given** a received `IncomingCall` event
**When** processed
**Then** the event is logged in structured JSON with `correlationId` at INFO level

---

### Story 1.4: ACS Call Automation & Call Answering

As a **caller**,
I want the system to answer my call with bidirectional audio streaming,
So that my voice reaches the AI agent and I can hear its responses.

**Acceptance Criteria:**

**Given** a validated `IncomingCall` event
**When** the call-answer sequence starts
**Then** the system answers using `CallAutomationClient` with bidirectional media streaming enabled in PCM 24K mono (FR2, TC-7)

**Given** `CallAutomationClient`
**When** authenticating to ACS
**Then** managed identity credentials are used with no connection strings or API keys (FR23, NFR-SEC2)

**Given** a `CallConnected` callback
**When** received from ACS
**Then** a new `CallContext` is created with the ACS `callId` as `correlationId` and call state transitions to `INITIALIZING` (FR3, CC-1, CC-2)

**Given** a `CallDisconnected` callback
**When** received from ACS
**Then** call state transitions to `TERMINATED` and all associated resources are marked for cleanup (FR3)

**Given** `PlayFailed` or `PlayCompleted` callbacks
**When** received from ACS
**Then** the internal call state is updated accordingly (FR3)

**Given** the call answer operation
**When** measured end-to-end (`IncomingCall` received to `answerCall()` response)
**Then** latency is < 3s at p95 (NFR-P5)

---

### Story 1.5: ACS WebSocket Audio Server Endpoint

As the **IVR system**,
I want to accept and parse ACS audio WebSocket connections,
So that the inbound audio stream is ready for bridging to Voice Live.

**Acceptance Criteria:**

**Given** an ACS audio WebSocket connection request
**When** directed to the versioned path `/ws/v1`
**Then** the system accepts the WSS connection and upgrades the protocol (FR7)

**Given** incoming binary WebSocket frames from ACS
**When** received on the established connection
**Then** they are parsed into `AudioMetadata` and `AudioData` packets using `StreamingData.parse()` (FR7)

**Given** a successful WebSocket upgrade
**When** the connection is established
**Then** it is associated with the correct `CallContext` via the call's `correlationId`

**Given** application startup time
**When** measured from JVM launch to readiness probe HTTP 200
**Then** it completes within ≤ 30s on a cold JVM (NFR-P9)

---

## Epic 2: Voice Live Integration & Audio Bridge

The caller speaks and hears AI responses in real-time.

### Story 2.1: Voice Live WebSocket Client & Session Initialization

As the **IVR system**,
I want to connect to Voice Live API via WebSocket, authenticate with managed identity, and initialize the AI session,
So that the conversational AI is ready to process caller audio.

**Acceptance Criteria:**

**Given** a call in `INITIALIZING` state
**When** the Voice Live connection is initiated
**Then** an outbound WebSocket is established to the configured endpoint (C-07) with API version (C-08) within the connect timeout (C-10) using `java.net.http.WebSocket` (FR11, TC-3)

**Given** the Voice Live WebSocket connection
**When** authenticating
**Then** managed identity credentials (`DefaultAzureCredential`) are used with no API keys in code or configuration (FR12, TC-5, TC-6)

**Given** a successful Voice Live WebSocket connection
**When** `session.created` is received
**Then** the system sends a `session.update` event configuring: externalized system prompt (C-12), voice model (C-13), tool definitions from C-60, and VAD settings — type (C-14), threshold (C-15), silence duration (C-16), noise suppression (C-17), echo cancellation (C-18) (FR13)

**Given** Voice Live rejects the session
**When** `session.update` fails
**Then** the rejection is logged at WARN with `correlationId`, error code/message, and system prompt size (FR13)

**Given** Voice Live connect latency
**When** measured from WebSocket handshake to `session.created`
**Then** it is ≤ 2s at p95, ≤ 3s at p99 (NFR-P6)

**Given** the Voice Live WebSocket
**When** connection uses the `ws://` scheme
**Then** the connection is never initiated — only `wss://` is allowed (NFR-SEC3)

---

### Story 2.2: Bidirectional Audio Bridge

As a **caller**,
I want my voice forwarded to the AI and the AI's voice forwarded back to me in real-time,
So that I can have a natural, low-latency conversation.

**Acceptance Criteria:**

**Given** audio frames arriving from ACS (via Story 1.5)
**When** the Voice Live session is active
**Then** audio is forwarded to Voice Live without buffering, inspecting, or transforming the content — protocol translation only (FR9, NFR-P8, TC-17)

**Given** `response.audio.delta` events from Voice Live
**When** received during an active session
**Then** audio is sent to ACS via the outbound streaming format using `OutStreamingData.getStreamingDataForOutbound()` (FR8)

**Given** a single audio frame traversing the bridge
**When** measured end-to-end
**Then** latency is < 50ms at p99 (NFR-P4)

**Given** time-to-first-audio
**When** measured from `CallConnected` to first audio byte sent to ACS
**Then** it is < 3s at p95; JWKS cache is pre-warmed during startup (NFR-P1)

**Given** per-call memory
**When** measured at steady state with active audio bridging
**Then** heap usage is ≤ 200 KB per call (NFR-P7)

**Given** no reactive types
**When** examining the audio bridge code
**Then** zero `Mono`, `Flux`, or `Publisher` types are used (NFR-I2, M-1)

---

### Story 2.3: Linked WebSocket Lifecycle

As the **IVR system**,
I want closure of either WebSocket to trigger graceful teardown of both,
So that no orphaned connections or sessions remain after a call ends.

**Acceptance Criteria:**

**Given** the ACS WebSocket closes (codes 1000, 1001)
**When** a normal close is detected
**Then** the Voice Live WebSocket is closed and all call resources are freed within ≤ 3s (FR10, NFR-R2)

**Given** the Voice Live WebSocket closes (codes 1000, 1001)
**When** a normal close is detected
**Then** the ACS WebSocket is closed and all call resources are freed within ≤ 3s (FR10, NFR-R2)

**Given** the ACS WebSocket closes abnormally (code 1006)
**When** detected
**Then** the Voice Live session is held for C-28 (5s reconnection window), then teardown proceeds if no reconnection (NFR-R12)

**Given** concurrent state transitions on both WebSockets
**When** both sides attempt teardown simultaneously
**Then** no race conditions, resource leaks, or duplicate teardown occur (NFR-R5)

**Given** resource cleanup
**When** teardown completes
**Then** `CallContext` is removed from the in-memory store, all metrics are updated, and the `ivr.call` span is completed

---

## Epic 3: Conversational Intelligence & Caller Experience

The caller has a natural, responsive conversation with barge-in support, comfort tones, and graceful time management.

### Story 3.1: Barge-In & Speech Interruption Handling

As a **caller**,
I want to interrupt the AI mid-sentence and have it stop immediately,
So that I can redirect the conversation naturally without waiting.

**Acceptance Criteria:**

**Given** Voice Live sends `input_audio_buffer.speech_started`
**When** the AI is currently streaming audio to the caller
**Then** the system immediately stops forwarding Voice Live audio frames to ACS (FR14)

**Given** Voice Live sends `input_audio_buffer.speech_started`
**When** an ACS Play operation is in progress
**Then** the system cancels the Play operation; both stop-forwarding and cancel-play are idempotent — safe to invoke redundantly (FR14)

**Given** barge-in latency
**When** measured from `speech_started` received to `StopAudio` sent to ACS
**Then** it is < 100ms at p95 (NFR-P3, UX-REQ-8)

**Given** AI response latency
**When** measured from VAD end-of-speech to first `response.audio.delta`
**Then** it is < 1s at p95 (NFR-P2, UX-REQ-7)

**Given** the caller barges in
**When** the AI was mid-sentence
**Then** no abrupt mid-sentence cutoffs occur on the caller's side — audio stops cleanly (UX-REQ-6)

---

### Story 3.2: Audio Prompts & Fallback Experiences

As a **caller**,
I want to hear appropriate audio cues during connection delays and failures,
So that I'm never left in silence or confused by abrupt behavior.

**Acceptance Criteria:**

**Given** the Voice Live connection exceeds C-33 (comfort tone threshold)
**When** no AI audio has been sent to the caller yet
**Then** the system plays a comfort tone via ACS Play action (FR35a, UX-REQ-1)

**Given** the Voice Live session established but no AI audio within C-32
**When** the fallback greeting timeout expires
**Then** the system plays the fallback greeting prompt (FR35b, UX-REQ-4)

**Given** an unexpected Voice Live disconnect
**When** the caller is still connected
**Then** the system plays the apology prompt before disconnecting gracefully (FR35c, UX-REQ-3)

**Given** admission rejection (from Epic 7)
**When** the per-instance call limit is exceeded
**Then** the system plays the busy prompt (C-38) to the caller (FR35d, UX-REQ-11)

**Given** the circuit breaker is OPEN (from Epic 4)
**When** a new call arrives
**Then** the system plays the service-unavailable prompt before disconnecting (FR35e, UX-REQ-10)

**Given** all 5 failure mode prompts
**When** evaluated for coverage
**Then** 100% of defined failure modes have a corresponding audio prompt (NFR-I1)

---

### Story 3.3: Max-Duration Management & PlayFailed Handling

As the **IVR system**,
I want to enforce call time limits gracefully and handle prompt playback failures safely,
So that calls end cooperatively and failures don't cascade.

**Acceptance Criteria:**

**Given** an active call approaching max duration
**When** C-29 minus C-30 seconds remain
**Then** the system injects a context message to Voice Live prompting the AI to wrap up naturally (FR6, FR16, UX-REQ-9)

**Given** the call reaches max duration
**When** the call did not end voluntarily after the wrap-up notification
**Then** the system enforces a hard disconnect (FR6)

**Given** a `PlayFailed` callback from ACS
**When** received for any prompt playback
**Then** the system logs at ERROR with `correlationId`, prompt identifier, and ACS failure reason — does NOT retry or play a substitute prompt, proceeds directly to graceful termination (FR47)

**Given** mid-session system messages
**When** injected to Voice Live (e.g., wrap-up notification)
**Then** they do not disrupt or reset the active conversation context (FR16)

---

## Epic 4: Connection Resilience & Recovery

When infrastructure has issues, callers experience graceful handling instead of dead air or abrupt disconnection.

### Story 4.1: Graceful Resource Teardown

As the **IVR system**,
I want all call resources cleaned up deterministically when a call ends,
So that no connections, memory, or sessions leak regardless of how disconnection occurs.

**Acceptance Criteria:**

**Given** a call ending (any trigger — caller hangup, AI disconnect, error, timeout)
**When** teardown begins
**Then** both WebSocket connections are closed, the `CallContext` is removed from the `ConcurrentHashMap`, metrics are updated, and the tracing span is completed (FR4)

**Given** teardown is initiated
**When** a WebSocket close or resource cleanup throws an exception
**Then** cleanup continues via `try-finally` on all paths — no single failure prevents other resources from being freed (FR4, CC-6)

**Given** the call completion rate
**When** measured across all answered calls
**Then** ≥ 99% end gracefully (NFR-R8)

**Given** a completed teardown
**When** no Voice Live sessions remain after call end + timeout
**Then** zero orphaned sessions exist (NFR-R7)

---

### Story 4.2: Heartbeat Monitoring & Staleness Detection

As the **IVR system**,
I want to detect when either WebSocket connection goes stale,
So that dead connections are terminated quickly instead of leaving callers in dead air.

**Acceptance Criteria:**

**Given** either the ACS or Voice Live WebSocket connection
**When** no messages are received within the configured idle timeout (C-05 for ACS, C-11 for Voice Live)
**Then** the connection is detected as stale (FR17)

**Given** a stale WebSocket detected
**When** staleness is confirmed
**Then** the stale connection is force-closed, triggering the linked lifecycle teardown per FR10 (FR17)

**Given** heartbeat detection latency
**When** measured from last heartbeat to staleness detection
**Then** it is ≤ 3s (NFR-R9)

**Given** heartbeat/keepalive mechanisms
**When** active on both connections
**Then** they operate independently — one connection's heartbeat failure does not interfere with the other's detection logic

---

### Story 4.3: Circuit Breaker & Proactive Connectivity Probing

As the **IVR system**,
I want a circuit breaker on Voice Live connections that trips on repeated failures and recovers automatically,
So that new calls aren't directed at a broken dependency.

**Acceptance Criteria:**

**Given** consecutive Voice Live connection failures
**When** the count reaches C-22 (threshold)
**Then** the circuit breaker trips to OPEN state (FR18, NFR-R3)

**Given** the circuit breaker is OPEN
**When** the configurable half-open delay (C-23) elapses
**Then** the breaker transitions to HALF-OPEN, allowing a probe connection attempt (FR19)

**Given** the circuit breaker is HALF-OPEN
**When** C-24 successive connections succeed
**Then** the breaker transitions back to CLOSED (FR19)

**Given** the circuit breaker is OPEN
**When** a new call arrives
**Then** the system plays the service-unavailable prompt (FR35e via Epic 3) and disconnects gracefully — no connection attempt is made to Voice Live (FR21)

**Given** the periodic Voice Live connectivity probe (C-25 interval)
**When** probes fail
**Then** the circuit breaker is preemptively tripped to OPEN before any caller is affected (FR41)

**Given** the circuit breaker state metric `ivr.circuit_breaker.state`
**When** the breaker transitions
**Then** the metric is updated and a structured log event is emitted at WARN level

---

### Story 4.4: ACS Reconnection, Session Resume & Stall Detection

As a **caller**,
I want brief network glitches to not kill my conversation, and frozen AI responses to be caught,
So that I experience resilience instead of dead air.

**Acceptance Criteria:**

**Given** the ACS WebSocket closes abnormally (code 1006)
**When** within the reconnection window C-28 (5s)
**Then** ACS reconnects and the system attempts to resume the existing Voice Live session (FR20, NFR-R12)

**Given** ACS reconnects within the window
**When** the Voice Live session resume fails
**Then** the system falls back to playing the apology prompt and disconnecting gracefully (FR20, FR21)

**Given** the reconnection window expires
**When** no ACS reconnection occurs
**Then** teardown proceeds for both WebSockets (NFR-R12)

**Given** Voice Live is streaming audio (`response.audio.delta` events received)
**When** no further audio events and no `response.done` arrive within C-55 (5000ms)
**Then** a mid-stream stall is detected — the system logs WARN, plays the apology prompt (C-37), and initiates graceful teardown (FR46)

**Given** stall detection accuracy
**When** measured across normal conversations with natural pauses
**Then** false positive rate is < 1% (NFR-R13)

---

## Epic 5: Tool Call Execution

When Voice Live needs external data during a conversation, the bridge dispatches tool calls to the APIM gateway and returns results — enabling the AI to answer billing and account questions in real-time.

**3 Predefined Tools (MVP scope):**

| Tool Name | APIM Endpoint | Purpose |
|-----------|---------------|----------|
| `get-user-data` | `GET {C-57}/api/v1/users/{id}` | Fetch caller's account/billing data |
| `send-invoice-registered-email` | `POST {C-57}/api/v1/invoices/send` | Send invoice copy to the caller's registered email |
| `send-invoice-provided-email` | `POST {C-57}/api/v1/invoices/send` | Send invoice copy to an email the caller dictates |

### Story 5.1: Tool Definition Loading & Session Registration

As the **IVR system**,
I want to load the 3 tool definitions from external files and register them with Voice Live at session start,
So that Voice Live knows which tools are available and can invoke them during conversation.

**Acceptance Criteria:**

**Given** the application starts
**When** `ToolDefinitionRegistry` initializes
**Then** it loads 3 tool definition files from the path specified by C-60 (`classpath:tool-definitions.json` by default): `get-user-data.json`, `send-invoice-registered-email.json`, `send-invoice-provided-email.json` — each following OpenAI function calling schema (FR51)

**Given** a new Voice Live session is established
**When** the `session.update` event is sent (FR13)
**Then** all 3 tool definitions are included in the `tools` array of the session configuration, enabling Voice Live to invoke `get-user-data`, `send-invoice-registered-email`, and `send-invoice-provided-email`

**Given** a tool definitions file is missing or contains invalid JSON
**When** the registry loads at startup
**Then** the system logs at ERROR level with the file name and parse error, and starts the session without tool definitions (degraded mode) — the AI conversation works but cannot invoke tools (FR51)

**Given** the tool registry contents
**When** loaded successfully
**Then** each `ToolDefinition` carries its APIM endpoint path (e.g., `/api/v1/users/{id}`, `/api/v1/invoices/send`), HTTP method, timeout override, and expected response schema — matching Architecture Decision 5

---

### Story 5.2: Tool Call Dispatch, Response & Error Handling

As a **caller**,
I want the AI to fetch my account data and send invoices during our conversation,
So that I can get billing answers and receive invoice copies without being transferred to a human agent.

**Acceptance Criteria:**

**Given** Voice Live sends a `tool_call` event for `get-user-data` with argument `{"phone": "<caller-phone>"}`
**When** the `ToolCallDispatcher` receives the event
**Then** it dispatches `GET {C-57}/api/v1/users/{id}` to the APIM gateway with `Authorization: Bearer <token>` (DefaultAzureCredential, scope C-57b), the call's `correlationId`, and the tool arguments (FR48, SEC-REQ-18)

**Given** Voice Live sends a `tool_call` event for `send-invoice-registered-email` or `send-invoice-provided-email`
**When** the `ToolCallDispatcher` receives the event
**Then** it dispatches `POST {C-57}/api/v1/invoices/send` to the APIM gateway with the same auth model, `correlationId`, and the tool-specific arguments (FR48)

**Given** the APIM gateway returns a successful HTTP response for any tool call
**When** the response is received
**Then** the system wraps it as a `tool_call_output` event (containing `call_id` + response payload) and sends it back to Voice Live within 5s p95 (FR49, AC-23, NFR-P10)

**Given** the APIM gateway does not respond within C-58 (default 5000ms) or returns 4xx/5xx
**When** the timeout or error occurs
**Then** the system returns an error `tool_call_output` to Voice Live (e.g., `{"error": true, "message": "Unable to retrieve data at this time"}`), logs WARN with `correlationId`, tool name, HTTP status (if available), and elapsed time, increments `ivr.tool_call.errors`, and does **NOT** terminate the call — the AI conversation continues gracefully (FR50, AC-24, NFR-R14)

**Given** the `ToolCallDispatcher` uses `RestClient` (Spring Boot 4.x)
**When** dispatching tool calls on virtual threads
**Then** a Resilience4j circuit breaker protects against piling up blocked threads when APIM/backend is down — no retry (fail-fast for voice UX, Architecture Decision 5)

**Given** `C-59` (max concurrent tool calls per call, default 3)
**When** a tool call is dispatched
**Then** concurrent tool calls per call are bounded by C-59; additional tool calls beyond the limit are queued or rejected

**Given** the WireMock gateway stub (test profile)
**When** integration tests run
**Then** WireMock at `localhost:8082` responds to `GET /api/v1/users/{id}` and `POST /api/v1/invoices/send`, including assertions for `Authorization: Bearer` header and `correlationId` propagation; fault injection supports configurable delay > C-58, 4xx/5xx errors, and connection refused

> **Note:** The PRD originally specified `POST /api/v1/users/{id}` for `get-user-data`. Changed to `GET` since this is a read operation. Flagged for Architect review — the bridge is verb-agnostic (reads HTTP method from tool definition config), so this only affects the tool definition file and WireMock stubs.

---

## Epic 6: Security Hardening

All inbound connections are authenticated, credentials are validated, replay attacks are blocked, and rate limiting protects against abuse — zero trust on every boundary.

### Story 6.1: JWT Validation on WebSocket & Callback Endpoints

As the **IVR system**,
I want to validate JWT tokens on every inbound WebSocket upgrade and ACS callback request,
So that only authenticated ACS connections can stream audio or trigger call events.

**Acceptance Criteria:**

**Given** an ACS WebSocket upgrade request to `/ws/v1`
**When** the `Authorization` header contains a valid, non-expired JWT
**Then** the upgrade proceeds and the WebSocket connection is established (FR22)

**Given** an ACS WebSocket upgrade request
**When** the JWT is missing, expired, or invalid
**Then** the connection is rejected with HTTP 401 **before** upgrading — no WebSocket session is created (FR22, SEC-REQ-1)

**Given** an ACS mid-call callback request
**When** received at the callback endpoint
**Then** the signed JWT in the headers is validated against ACS's JWKS (SEC-REQ-3)

**Given** the JWKS endpoint
**When** the service starts
**Then** the key set is pre-fetched and cached to avoid first-request latency (NFR-P1) — cache refresh follows standard JWKS rotation practices

**Given** a callback URL registered with ACS
**When** the call is answered via `answerCall()`
**Then** the URL contains a cryptographically random token (min length C-50), and every callback for that call validates the token — rejecting requests with a missing or mismatched token (FR36, SEC-REQ-4)

---

### Story 6.2: Event Grid Authentication & Replay Protection

As the **IVR system**,
I want to validate Event Grid events and reject stale replayed events,
So that only fresh, legitimate incoming-call events trigger call processing.

**Acceptance Criteria:**

**Given** an Event Grid `SubscriptionValidationEvent`
**When** received at the webhook endpoint
**Then** the system responds with the `validationCode` to complete the handshake (SEC-REQ-2)

**Given** an `IncomingCall` event from Event Grid
**When** the `eventTime` is older than C-45
**Then** the event is rejected with HTTP 400 and logged at WARN with `correlationId` and event age (FR37, SEC-REQ-14)

**Given** an `IncomingCall` event
**When** the `eventTime` is within the C-45 freshness window
**Then** the event is processed normally

**Given** the Event Grid webhook endpoint
**When** receiving requests
**Then** the `Authorization` header is validated as an Entra ID bearer token (SEC-REQ-2)

---

### Story 6.3: Managed Identity & Credential Health

As the **IVR system**,
I want all outbound authentication handled via managed identity with no secrets in config,
So that the attack surface is minimized and credentials rotate automatically.

**Acceptance Criteria:**

**Given** the Voice Live WebSocket connection
**When** the service initiates the outbound connection
**Then** authentication uses `DefaultAzureCredential` (managed identity in prod, `az login` locally) — zero API keys in code or configuration (FR24, SEC-REQ-11)

**Given** the `CredentialHealthIndicator` abstraction
**When** the readiness probe calls it
**Then** it reports the cached token status (valid, expiring soon, expired) without triggering a live token refresh (FR27)

**Given** `DefaultAzureCredential` token acquisition for APIM gateway
**When** a tool call is dispatched
**Then** the acquired bearer token is scoped to C-57b and sent in the `Authorization` header — if acquisition fails, the tool call fails gracefully (FR50 error path) rather than sending an unauthenticated request (SEC-REQ-18)

**Given** the infrastructure
**When** deploying the service
**Then** RBAC role `Cognitive Services User` is assigned on the Foundry resource — this is an infrastructure prerequisite, not application code (FR24)

---

### Story 6.4: WSS Enforcement & WebSocket Rate Limiting

As the **IVR system**,
I want all WebSocket connections to use `wss://` exclusively and handshake attempts rate-limited per IP,
So that plaintext audio never traverses the network and brute-force connection attempts are blocked.

**Acceptance Criteria:**

**Given** the ACS-facing WebSocket server endpoint
**When** any connection attempt uses `ws://` (plaintext)
**Then** the connection is rejected — only `wss://` is accepted (FR26)

**Given** the Voice Live outbound WebSocket client
**When** initiating a connection
**Then** only `wss://` URLs are used — the code never constructs a `ws://` URI (FR26)

**Given** a single IP address
**When** WebSocket handshake attempts exceed C-46 within the C-47 time window
**Then** additional connections are rejected with HTTP 429 (FR38, SEC-REQ-15)

**Given** rate limit rejections
**When** they occur
**Then** the event is logged at WARN with the source IP and rejection count — no PII beyond IP is logged (SEC-REQ-6)

---

## Epic 7: Admission Control & Operational Safety

The system protects itself from overload, cleans up orphaned resources, and handles malformed events gracefully — so that operational edge cases never degrade service for active callers.

### Story 7.1: Concurrent Call Admission & Graceful Shutdown

As the **IVR system**,
I want to reject new calls when the per-instance limit is reached and drain existing calls during shutdown,
So that active callers never experience degradation from overload and deployments are zero-downtime.

**Acceptance Criteria:**

**Given** the number of active calls on this instance equals C-26 (max concurrent calls)
**When** a new `IncomingCall` event arrives
**Then** the system rejects the call with the busy prompt (C-38), logs WARN with `correlationId`, and increments `ivr.calls.rejected` (FR33, FR34, UX-REQ-11)

**Given** `C-26` is not reached
**When** a new `IncomingCall` event arrives
**Then** the call is admitted normally

**Given** SIGTERM is received (container shutdown)
**When** the shutdown signal is detected
**Then** the system stops accepting new calls, waits up to C-34 (graceful shutdown timeout) for active calls to complete, then forcibly tears down remaining sessions (FR34)

**Given** active calls during shutdown
**When** the graceful shutdown timeout (C-34) is reached with calls still active
**Then** remaining calls are force-terminated — both WebSockets closed, `CallContext` removed, metrics updated (FR34)

**Given** the admission counter
**When** a call completes teardown (Epic 4 Story 4.1)
**Then** the active call count is decremented atomically — no counter drift over time

---

### Story 7.2: Callback Deduplication & Event Batch Processing

As the **IVR system**,
I want duplicate ACS callbacks rejected and batch callback arrays processed correctly,
So that duplicate events don't corrupt call state and ACS batch delivery works seamlessly.

**Acceptance Criteria:**

**Given** an ACS callback with an `operationContext` + `eventType` combination already processed for this call
**When** the duplicate arrives
**Then** the callback is logged at DEBUG and discarded — no state mutation, no error (FR44)

**Given** ACS delivers a callback payload as a JSON array (batch mode)
**When** the webhook endpoint receives it
**Then** each event in the array is individually validated and dispatched in order — a failure processing one event does not prevent processing of subsequent events (FR5)

**Given** the deduplication window
**When** sized per-call
**Then** the dedup set is released with the `CallContext` at teardown — no unbounded memory growth (C-56)

---

### Story 7.3: Orphan Session Sweeper & Idle Connection Timeout

As the **IVR system**,
I want a periodic sweeper that detects and cleans up orphaned sessions, and idle connections that missed heartbeat detection,
So that leaked resources don't accumulate and degrade the instance over time.

**Acceptance Criteria:**

**Given** a `CallContext` entry in the active session map
**When** the sweeper runs at interval C-52 and finds a session whose last-activity timestamp exceeds C-54 (orphan threshold)
**Then** the session is force-terminated — both WebSockets closed, `CallContext` removed, `ivr.sessions.orphaned` incremented, logged at WARN with `correlationId` (FR42, FR43)

**Given** the sweeper runs
**When** all sessions have recent activity within C-54
**Then** no action is taken

**Given** the orphan threshold (C-54)
**When** defined
**Then** it is strictly greater than the heartbeat idle timeouts (C-05, C-11) — the sweeper is a safety net, not the primary detection mechanism

**Given** the sweeper
**When** it force-terminates a session
**Then** it uses the same teardown path as Story 4.1 — no separate cleanup logic

---

### Story 7.4: Lenient Event Parsing & Error Resilience

As the **IVR system**,
I want unknown or malformed events from both ACS and Voice Live to be logged and skipped without crashing,
So that protocol additions or transient malformations don't take down active calls.

**Acceptance Criteria:**

**Given** an ACS callback with an unrecognized `eventType`
**When** received at the webhook endpoint
**Then** the event is logged at WARN with the unknown type and `correlationId`, and discarded — no exception, no 5xx response (FR45)

**Given** a Voice Live event with an unrecognized `type` field
**When** received on the WebSocket client
**Then** the event is logged at WARN and ignored — audio bridging and the active conversation continue unaffected (FR45)

**Given** a Voice Live event with valid `type` but malformed payload (e.g., missing required fields)
**When** JSON deserialization partially fails
**Then** the system logs the parse error at ERROR with event type and `correlationId`, skips that single event, and continues processing — the WebSocket connection is NOT closed (FR45, SEC-REQ-10)

**Given** any unhandled exception in event processing
**When** it occurs in the ACS callback handler or Voice Live message handler
**Then** the exception is caught at the handler boundary, logged at ERROR, and does NOT propagate to crash the thread or close the connection (FR45)
