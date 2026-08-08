> **Note on this fork:** FirstAId was built by a four-person team at a 2026 hackathon.
> I designed the system architecture and API contract the team built against: the
> two-stage parser/formatter split, the request/response surface, and the Bedrock tool
> schemas, and built the retrieval layer and its similarity gate. The frontend, most of
> the pipeline implementation, and the knowledge base provisioning are teammates' work.

# FirstAid AI

**Track:** Human-Centered Design  
**Demo:** https://d32xsl7uhmmecy.cloudfront.net/ (deactivated)  
**Video:** https://www.youtube.com/watch?v=1BiMNqMJajo&t=2s

## Overview

FirstAid AI delivers structured, grounded first-aid triage guidance in under 5 seconds for underserved populations facing emergency healthcare navigation barriers. Users describe emergencies in plain language; the system returns color-coded severity assessments, imperative action steps, care tier recommendations, and nearby facility maps.

## Target Impact

**Affected Communities:**
- Uninsured individuals who can't afford wrong care-tier decisions
- Rural populations facing long transport times  
- Non-native English speakers needing clear, imperative instructions
- Low-income households requiring free, no-account emergency guidance
- First-time caregivers lacking experience to calibrate severity

**Problem Solved:** The 30-second window between "something just happened" and "what do I do" — replacing SEO-farmed results, hedged AI prose, and hold music with instant, grounded medical triage.

## Technical Innovation

**Serverless RAG Pipeline:**
- Two-stage LLM: Parser (retrieve/clarify) → Formatter (structured triage)
- Bedrock Knowledge Base with curated medical corpus (NHS, CDC, Red Cross)
- Similarity threshold gating prevents hallucinated medical advice
- Sub-5-second end-to-end latency target

**Safety-First Design:**
- One clarification round-trip maximum (prevents conversation loops)
- Out-of-scope detection for non-medical queries
- Reasoning field logged server-side, stripped from client responses
- Over-escalation bias in severity assessment

**Equity Architecture:**
- No accounts, authentication, or data retention
- No payment tiers or premium features
- WCAG AAA accessibility compliance
- Mobile-first responsive design (375px target)


## Kiro Development Methodology

Built entirely using Kiro's full feature spectrum:

- **Spec-driven development:** Complete architecture before coding
- **Steering docs:** Persistent context preventing regressions
- **Agent hooks:** Automated build validation and diff tracking
- **MCP servers:** Accurate AWS/React API integration
- **Vibe coding:** Generated complete modules from specifications

**Result:** Zero integration bugs, no stale builds, no hallucinated API signatures — eliminating the three most expensive hackathon failure modes.

## Repository Structure

```
├── frontend/          # React + Vite + TypeScript SPA
├── backend/           # Lambda function (Node.js 20.x)
├── cdk/              # AWS CDK infrastructure
├── .kiro/steering/   # Persistent AI context files
└── KIRO_POWERS.md    # Detailed Kiro usage write-up
```


## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Browser                             │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │   React SPA     │    │  Geolocation    │                     │
│  │  (TypeScript)   │    │   Prefetch      │                     │
│  └─────────────────┘    └─────────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
                                │ HTTPS
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CloudFront Distribution                      │
│  ┌─────────────────┐              ┌─────────────────┐           │
│  │   S3 Bucket     │              │  /api/triage    │           │
│  │ (Static Files)  │              │   → API GW      │           │
│  └─────────────────┘              └─────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                                                │
                                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Lambda Function                              │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │  Stage 1:       │    │  Stage 2:       │                     │
│  │  Parser         │    │  Formatter      │                     │
│  │  (Clarify?)     │    │  (Triage Card)  │                     │
│  └─────────────────┘    └─────────────────┘                     │
│                                │                                │
│  ┌─────────────────┐           │    ┌─────────────────┐         │
│  │  Google Places  │           │    │  SSM Parameter  │         │
│  │  API (Nearby)   │           │    │  Store (Keys)   │         │
│  └─────────────────┘           │    └─────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Amazon Bedrock                               │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │  Claude Sonnet  │    │  Knowledge Base │                     │
│  │  (Parser +      │    │  (Medical       │                     │
│  │   Formatter)    │    │   Corpus)       │                     │
│  └─────────────────┘    └─────────────────┘                     │
│                                │                                │
│  ┌─────────────────┐           │                                │
│  │  Titan Text     │           │                                │
│  │  Embeddings v2  │           │                                │
│  └─────────────────┘           │                                │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      S3 Vector Store                            │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │  Vector Store   │    │  Medical        │                     │
│  │  (Embeddings)   │    │  Protocols      │                     │
│  └─────────────────┘    └─────────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

**Data Flow:**
1. User submits query → CloudFront → API Gateway → Lambda
2. Lambda Stage 1: Parser (retrieve vs. clarify decision)
3. If retrieve: Query Bedrock KB → S3 Vector similarity search
4. Lambda Stage 2: Formatter (structured triage response)
5. Optional: Google Places API for nearby facilities
6. Response → CloudFront → User (reasoning stripped)

**Cost Estimate:**
- **Development:** ~$50/month (Lambda + API Gateway + CloudFront)
- **Production (1K users/day):** ~$200/month
- **S3 Vector Store:** ~$1/month (storage only, no idle charges)

## Performance Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| End-to-end latency | <5s | 2.8s |
| Parser response | <2s | 1.1s |
| KB retrieval | <1s | 0.4s |
| Formatter response | <2s | 1.2s |
| Places API | <1s | 0.3s |

## License

MIT License - See [LICENSE](LICENSE) for details.

