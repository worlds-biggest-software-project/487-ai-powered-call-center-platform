# Standards & API Reference

> Project: AI-Powered Call Center Platform · Generated: 2026-05-07

---

## Industry Standards & Specifications

### ISO Standards

**ISO 18295-1:2017 — Customer Contact Centres: Requirements for Customer Contact Centres**
- URL: https://www.iso.org/standard/64739.html
- Specifies service requirements for customer contact centres of all sizes and sectors, covering inbound and outbound interactions across all channels. Defines KPIs that a compliant CCC must monitor (AHT, FCR, CSAT, abandon rate), reporting requirements, and the contractual relationship between CCCs and client organisations. Replaces the earlier European standard EN 15838. Any commercial call center platform should be designed to support the KPI reporting and SLA tracking requirements defined in this standard.

**ISO 18295-2:2017 — Customer Contact Centres: Requirements for Clients Using Customer Contact Centres**
- URL: https://www.iso.org/standard/64740.html
- Companion document to ISO 18295-1. Specifies requirements on the client (buyer) side of a contact centre engagement, including governance, performance management, and obligation to share customer data appropriately. Relevant when designing the multi-tenant or white-label feature set of a call center platform.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/82875.html
- Framework for information security management systems (ISMS). Widely required by enterprise buyers as a vendor qualification criterion. A call center platform storing call recordings, transcripts, and customer PII must demonstrate controls aligned with ISO 27001 to compete in enterprise and government segments.

---

### W3C & IETF Standards

**RFC 3261 — Session Initiation Protocol (SIP)**
- URL: https://datatracker.ietf.org/doc/html/rfc3261
- The foundational IETF standard for establishing, modifying, and terminating voice/video sessions over IP. All PSTN-connected voice AI systems use SIP for call setup and teardown, either directly or via a SIP trunk provider. The platform must be compatible with SIP trunking for PSTN connectivity and SIP-based media servers (Asterisk, FreeSWITCH).

**RFC 3550 — RTP: A Transport Protocol for Real-Time Applications**
- URL: https://datatracker.ietf.org/doc/html/rfc3550
- Defines the Real-time Transport Protocol (RTP), which carries voice (and video) media over IP networks. RTP is used by every PSTN/SIP-based voice call. The AI voice pipeline must consume RTP audio streams from the media server and return RTP audio from the TTS engine. RTCP (RFC 3551) provides quality monitoring (jitter, packet loss) for diagnosing latency issues.

**RFC 8825 — Overview: Real-Time Protocols for Browser-Based Applications (WebRTC)**
- URL: https://datatracker.ietf.org/doc/html/rfc8825
- The WebRTC standards suite enables browser-native real-time voice/video without plugins, using DTLS-SRTP for encryption and ICE/STUN/TURN (RFC 8445, RFC 5389, RFC 5766) for NAT traversal. Used by every modern CCaaS agent desktop. The human agent desktop component of the platform must implement WebRTC for in-browser calls.

**RFC 6716 — Definition of the Opus Audio Codec**
- URL: https://datatracker.ietf.org/doc/html/rfc6716
- Opus is the mandatory audio codec for WebRTC (RFC 7874), with adaptive bitrate from 6 to 510 kbps. It provides superior quality at low bitrates compared to G.711. The platform's WebRTC agent desktop and any streaming audio pipelines should use Opus; PSTN interop may require transcoding to G.711 (PCMU/PCMA, ITU-T G.711).

**RFC 6787 — Media Resource Control Protocol Version 2 (MRCPv2)**
- URL: https://datatracker.ietf.org/doc/html/rfc6787
- MRCPv2 is the IETF standard for controlling speech resources (ASR, TTS) over IP. Supported by legacy IVR infrastructure (Cisco CVP, Genesys PureConnect, Avaya). Relevant for platforms integrating with existing enterprise telephony stacks that still use MRCP-based ASR/TTS servers rather than cloud STT/TTS APIs.

**RFC 8226 — Secure Telephone Identity Credentials: Certificates (STIR)**
- URL: https://www.rfc-editor.org/rfc/rfc8226.html
- Part of the STIR/SHAKEN framework for authenticating caller ID in VoIP networks. RFC 8226 defines how telephone number authority is established via certificates; RFC 8588 (SHAKEN) defines the PaSSporT token extension for call signing. Outbound AI calling campaigns require STIR/SHAKEN compliance to avoid carrier-level call blocking and FCC regulatory exposure.

**RFC 8588 — PaSSporT Extension for SHAKEN**
- URL: https://www.rfc-editor.org/rfc/rfc8588.html
- Defines the SHAKEN Personal Assertion Token (PaSSporT) extension used by carriers to digitally sign outbound calls with the originating carrier's certificate. Any platform supporting outbound calling from a US PSTN number must be routed through a STIR/SHAKEN-compliant SIP trunk (Twilio, Bandwidth, Vonage all provide this) and ensure proper caller ID registration.

**W3C VoiceXML 2.1**
- URL: https://www.w3.org/TR/voicexml21/
- Voice Extensible Markup Language (VoiceXML) is the W3C standard for defining voice application dialogs. While AI-native platforms are replacing VoiceXML-based IVRs, legacy enterprise environments still run VoiceXML-based call flows. A platform that needs to support migration from legacy IVR to AI agents should be able to ingest or import VoiceXML flow definitions.

**W3C SSML — Speech Synthesis Markup Language 1.1**
- URL: https://www.w3.org/TR/speech-synthesis11/
- SSML allows fine-grained control of TTS output: prosody, pauses, phonemes, emphasis, and voice selection. Major TTS providers (ElevenLabs, Google TTS, Amazon Polly, Azure TTS) support SSML. The platform's TTS integration layer should support SSML passthrough for cases where the LLM or flow designer needs precise control over voice delivery.

---

### Data Model & API Specifications

**OpenAPI 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de facto standard for REST API specification. All major CCaaS vendors (Twilio, Vonage, Retell AI, Genesys, Talkdesk) publish OpenAPI-compatible API documentation. The platform's public API should be documented using OpenAPI 3.1 to enable SDK auto-generation, Postman integration, and partner onboarding.

**AsyncAPI 2.x / 3.x**
- URL: https://www.asyncapi.com/docs/reference/specification/v3.0.0
- Standard for documenting event-driven (webhook/WebSocket) APIs. Complements OpenAPI for the real-time event streams that are core to a call center platform (call started, transcript ready, sentiment alert, escalation triggered). Documenting webhook payloads with AsyncAPI improves developer experience and reduces integration errors.

**JSON:API**
- URL: https://jsonapi.org/
- A widely adopted specification for structuring REST API request and response bodies. Many CCaaS integrations (Zendesk, HubSpot, Salesforce) follow JSON:API conventions. Aligning to this standard reduces custom mapping code for integration partners.

**Protocol Buffers (protobuf) + gRPC**
- URL: https://grpc.io/ / https://protobuf.dev/
- High-performance binary serialisation and RPC framework used for latency-sensitive internal microservice communication. For a voice AI platform where the STT → LLM → TTS pipeline has strict latency budgets, using gRPC between internal services (rather than REST/JSON) reduces serialisation overhead and enables bidirectional streaming.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) + PKCE (RFC 7636)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749 / https://datatracker.ietf.org/doc/html/rfc7636
- The industry-standard authorisation framework for API access delegation. The platform must implement OAuth 2.0 for third-party CRM and integration access (Salesforce, HubSpot, etc.), and use PKCE for public clients (browser-based agent desktops). Token introspection (RFC 7662) and token revocation (RFC 7009) are required for enterprise security posture.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0, providing authentication (not just authorisation). Required for enterprise SSO integration (Okta, Azure AD, Google Workspace). Contact center platforms are typically procured by enterprise IT departments that require SAML 2.0 or OIDC SSO as a prerequisite for deployment.

**PCI DSS 4.0**
- URL: https://www.pcisecuritystandards.org/document_library/
- Payment Card Industry Data Security Standard version 4.0 (effective March 2025). Prohibits recording, storing, or transmitting CVV data, full PAN over recordings, or PIN entries. Mandates MFA for all cardholder data environment access, encryption in transit and at rest, and network segmentation. The platform must implement pause-and-resume recording triggered automatically when DTMF payment digits are detected, ensuring card data never enters the recording or AI pipeline.

**HIPAA Security Rule (45 CFR Part 164)**
- URL: https://www.hhs.gov/hipaa/for-professionals/security/index.html
- US federal regulation protecting electronic protected health information (ePHI). Requires encryption (TLS 1.2+, AES-256 at rest), role-based access controls, 6-year audit trail retention, breach notification within 60 days (with industry best practice of 24–72 hours), and Business Associate Agreements (BAA) with all vendors processing ePHI. Healthcare contact center deployments require HIPAA-eligible infrastructure with BAA support.

**GDPR (EU Regulation 2016/679)**
- URL: https://gdpr.eu/
- EU data protection regulation. Requires lawful basis for processing (consent or legitimate interest), data minimisation, the right to erasure, and data portability. Call recordings and AI-generated transcripts are personal data under GDPR; the platform must support per-caller consent capture at call start, configurable retention periods, and automated deletion workflows to honour erasure requests.

**TCPA — Telephone Consumer Protection Act (47 U.S.C. § 227) + 2025 Amendments**
- URL: https://www.fcc.gov/consumers/guides/stopping-unwanted-calls-texts
- US federal law restricting automated/AI-initiated outbound calls. The 2025 TCPA amendments require one-to-one written consent per seller (pre-checked boxes invalid), honour consent revocations within 10 business days, restrict call timing windows, and mandate real-time DNC (Do Not Call) list checking before each call attempt. The platform's outbound campaign scheduler must enforce all TCPA requirements automatically, log consent records immutably, and provide an audit trail per call attempt.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/www-project-api-security/
- Industry-standard checklist for API security vulnerabilities (broken object-level authorisation, authentication weaknesses, excessive data exposure, etc.). A public API for a voice platform handling sensitive customer data must be validated against the OWASP API Security Top 10 before production launch.

**NIST SP 800-63B — Digital Identity Guidelines: Authentication**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- NIST guidelines for authenticator assurance levels (AAL1, AAL2, AAL3). Enterprise contact center platform deployments, especially in government or financial services, may reference NIST 800-63B as an authentication requirement baseline. MFA (AAL2) is the minimum expected for admin console access.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open standard for connecting AI models to external tools, data sources, and systems via a structured protocol. MCP is directly relevant to the AI-powered call center platform in two ways: (1) the voice AI agent can use MCP tool calls to interact with CRM systems, ticketing platforms, and knowledge bases during calls without custom integration code for each target; (2) the platform itself could expose an MCP server interface, allowing other AI agents (e.g., Claude-based workflows) to query call transcripts, summaries, or customer interaction history as structured context. MCP adoption is growing rapidly as a vendor-neutral integration layer for agentic AI applications.

---

## Similar Products — Developer Documentation & APIs

### Retell AI

- **Description:** AI voice agent platform for building phone-call-handling agents; supports inbound, outbound, and batch campaigns with bring-your-own LLM.
- **API Documentation:** https://docs.retellai.com/general/introduction
- **SDKs/Libraries:** Python SDK: https://github.com/RetellAI/retell-python-sdk · TypeScript SDK: https://github.com/RetellAI/retell-typescript-sdk · npm: https://www.npmjs.com/package/retell-sdk · PyPI: https://pypi.org/project/retell-sdk/
- **Developer Guide:** https://docs.retellai.com/get-started/sdk
- **Standards:** REST/JSON, WebSocket for real-time audio streams, Webhook events, OpenAPI-compatible docs
- **Authentication:** API key (Bearer token)

---

### Bland AI

- **Description:** Voice AI platform for automating inbound and outbound calls using Pathway-based conversation flows and custom voice cloning.
- **API Documentation:** https://www.bland.ai/docs (referenced from vendor site)
- **SDKs/Libraries:** REST API only; no official published SDK repository found
- **Developer Guide:** Available via vendor portal at https://www.bland.ai/
- **Standards:** REST/JSON; webhook call events
- **Authentication:** API key

---

### Twilio Voice API

- **Description:** Programmable telephony API for initiating, receiving, and controlling voice calls over PSTN and WebRTC; widely used as the telephony layer for AI call center stacks.
- **API Documentation:** https://www.twilio.com/docs/voice
- **SDKs/Libraries:** Node.js, Python, Java, C#, Ruby, PHP, Go — https://www.twilio.com/docs/libraries
- **Developer Guide:** https://www.twilio.com/docs/voice/quickstart
- **Standards:** REST/JSON, TwiML (XML-based call control), WebSocket Media Streams for real-time audio to AI pipelines, OpenAPI 3.0
- **Authentication:** HTTP Basic Auth (Account SID + Auth Token); API Keys for production

---

### Vonage Voice API

- **Description:** Carrier-grade programmable voice API supporting PSTN, SIP, WebRTC; includes a Contact Center API for embedding CC capabilities into web apps via WebRTC.
- **API Documentation:** https://developer.vonage.com/en/voice/voice-api/overview
- **SDKs/Libraries:** Node.js, Python, Java, PHP, Ruby, .NET — https://developer.vonage.com/en/documentation
- **Developer Guide:** https://developer.vonage.com/en/voice/voice-api/overview
- **Standards:** REST/JSON, NCCO (Nexmo Call Control Objects — JSON call instructions), WebSocket for streaming audio
- **Authentication:** JWT (JSON Web Token) generated from API Key + Secret

---

### Deepgram API

- **Description:** Real-time and batch speech-to-text API with sub-300 ms latency, speaker diarisation, PII redaction, and contact centre-specific models; also provides TTS (Aura) and voice agent APIs.
- **API Documentation:** https://developers.deepgram.com/reference/deepgram-api-overview
- **SDKs/Libraries:** Python, Node.js, Go, .NET, Rust — https://github.com/deepgram
- **Developer Guide:** https://developers.deepgram.com/home
- **Standards:** REST/JSON for batch; WebSocket for real-time streaming; OpenAPI 3.0 spec available
- **Authentication:** API key (Bearer token)

---

### ElevenLabs API

- **Description:** Text-to-speech and voice cloning API providing ultra-low-latency streaming synthesis (75 ms Flash model), multilingual support, and conversational AI agent capabilities.
- **API Documentation:** https://elevenlabs.io/docs/overview/intro
- **SDKs/Libraries:** Python, TypeScript — https://elevenlabs.io/docs/developer-guides/python-sdk
- **Developer Guide:** https://elevenlabs.io/docs/overview/capabilities/text-to-speech
- **Standards:** REST/JSON, WebSocket streaming; OpenAPI 3.0 spec available
- **Authentication:** API key (xi-api-key header)

---

### Amazon Connect API (with Amazon Q in Connect)

- **Description:** AWS-native CCaaS with full API surface for contact flows, queue management, real-time transcription (Contact Lens), generative AI agent assist (Amazon Q in Connect), and post-contact summarisation.
- **API Documentation:** https://docs.aws.amazon.com/connect/latest/APIReference/
- **SDKs/Libraries:** AWS SDK for all major languages (Python/boto3, Node.js, Java, .NET, Go) — https://aws.amazon.com/tools/
- **Developer Guide:** https://docs.aws.amazon.com/connect/latest/adminguide/
- **Standards:** AWS REST APIs (SigV4 authentication), Kinesis data streams for real-time event processing, EventBridge for contact events, S3 for recordings
- **Authentication:** AWS IAM (SigV4 request signing); supports IAM roles for service-to-service auth

---

### OpenAI Realtime API

- **Description:** Low-latency speech-to-speech API that eliminates separate STT + LLM + TTS pipeline stages, enabling native voice conversations with GPT-4o; officially integrated with Twilio Media Streams for contact center use.
- **API Documentation:** https://developers.openai.com/api/docs/guides/realtime
- **SDKs/Libraries:** Official Python and Node.js Realtime Agents SDK — https://github.com/openai/openai-realtime-agents
- **Developer Guide:** https://www.twilio.com/en-us/blog/voice-ai-assistant-openai-realtime-api-node
- **Standards:** WebSocket-based bidirectional audio streaming; JSON message framing
- **Authentication:** Bearer token (OpenAI API key)

---

### Genesys Cloud API

- **Description:** CCaaS REST API covering conversations, routing, analytics, quality management, workforce management, and AI features (agent copilot, virtual agent, predictive routing).
- **API Documentation:** https://developer.genesys.cloud/api/rest/
- **SDKs/Libraries:** JavaScript, Python, Java, Go, .NET — https://developer.genesys.cloud/sdk
- **Developer Guide:** https://developer.genesys.cloud/
- **Standards:** REST/JSON, OAuth 2.0 (PKCE for browser clients), WebSocket notifications API for real-time events, OpenAPI 3.0 spec
- **Authentication:** OAuth 2.0 (client credentials for server-side; PKCE for browser-side)

---

### Salesforce Agentforce / Service Cloud Voice API

- **Description:** Salesforce-native APIs for contact center operations, voice channel management (Service Cloud Voice), Einstein AI for agent assist, and Agentforce autonomous agent configuration.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/
- **SDKs/Libraries:** Salesforce DX (CLI), Apex (server-side), Lightning Web Components (client-side) — https://developer.salesforce.com/
- **Developer Guide:** https://developer.salesforce.com/docs/service/voice/guide
- **Standards:** REST/JSON, SOAP (legacy), Streaming API (CometD), Platform Events, OpenAPI 3.0 (via API Connect)
- **Authentication:** OAuth 2.0 (Connected App), JWT Bearer Flow for server-to-server

---

## Notes

**Emerging Standards**
- **WebRTC WHIP/WHEP** (RFC drafts): Ingress (WHIP) and egress (WHEP) protocols for WebRTC-based media streaming are emerging as simpler alternatives to full SIP for connecting AI voice pipelines to media servers. Early adoption is visible in real-time streaming platforms.
- **OCSF (Open Cybersecurity Schema Framework)**: AWS, Splunk, and IBM-backed schema for normalising security event logs. Relevant for contact center security telemetry if targeting regulated industries.
- **AI Act (EU, effective 2025–2026)**: Automated AI phone agents may qualify as "limited risk" AI systems under the EU AI Act, requiring transparency disclosures to callers that they are interacting with an AI. This is not an IETF or ISO standard but a regulatory compliance requirement for EU deployments.

**Standards Gaps**
- There is no established standard for voice AI agent behavioural guardrails or hallucination detection — this remains a vendor-specific implementation concern.
- Consent capture and consent record formats for TCPA/GDPR are not standardised; each platform implements proprietary schemas. This is an opportunity for an open platform to define a portable consent record format.
