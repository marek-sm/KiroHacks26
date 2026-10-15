## My role

> FirstAId was built by a four-person team at KiroHacks 2026. I designed the system
> architecture and the API contract the team built against, including the two-stage
> parser/formatter split and the Bedrock tool schemas. I also built the retrieval layer
> with its similarity gate, and the backend and frontend deploy scripts
> (`scripts/deploy-backend.sh`, `scripts/deploy-frontend.sh`). Teammates built the
> frontend and most of the pipeline implementation. The knowledge base provisioning is
> also theirs. My commits are in this repo's history (`git log --author="Marek"`), and
> the design documents are `design.md` and `.kiro/steering/`.

# FirstAid AI

**Status:** hackathon build, May 2026. The hosted demo is deactivated.  
**Demo:** https://d32xsl7uhmmecy.cloudfront.net/ (deactivated)  
**Video:** https://www.youtube.com/watch?v=1BiMNqMJajo&t=2s

## Overview

FirstAid AI turns a plain-language description of an emergency into structured first-aid triage guidance grounded in a curated medical corpus. It returns a color-coded severity level, imperative action steps, a care-tier recommendation, and a map of nearby facilities.

## Design

**Serverless RAG pipeline:**
- Two-stage LLM: Parser (retrieve or clarify) → Formatter (structured triage)
- Bedrock Knowledge Base over first-aid protocols from the NHS, MedlinePlus, Mayo Clinic, British Red Cross, St John Ambulance, American Red Cross, and Healthdirect Australia
- A similarity threshold gate: when no retrieved source scores above the threshold, the handler skips the formatter and returns an out-of-scope result with no triage steps
- Design target: under 5 seconds end to end (not benchmarked)

**Safety decisions:**
- At most one clarifying question; a second clarify request is treated as out of scope
- Out-of-scope detection for non-medical queries
- The model's reasoning field is logged server-side and never returned to the client
- The formatter prompt biases severity upward when it is uncertain (emergency over urgent care)

**Access decisions:**
- No sign-in or accounts
- Mobile-first layout (375px design target)

## Repository Structure

```
├── frontend/          # React + Vite + TypeScript SPA
├── backend/           # Lambda function (Node.js 20.x)
├── cdk/               # AWS CDK infrastructure (deploy steps in cdk/DEPLOY.md)
├── scripts/           # Setup and deploy scripts
├── .kiro/steering/    # Architecture and API contract docs
└── design.md          # System design and specification
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

## Running it

Deployment needs an AWS account with Bedrock access, the AWS CLI, Node.js 20, and a Google Places API key. `./scripts/setup-all.sh` installs dependencies and builds the backend, then checks for the Places key. The stack deploy and knowledge base ingestion steps are in [`cdk/DEPLOY.md`](cdk/DEPLOY.md). The medical corpus documents are not included in this repo.

## Design targets

These latency targets come from the design. The repo contains no benchmark for them.

| Step | Target |
|--------|--------|
| End-to-end latency | <5s |
| Parser response | <2s |
| KB retrieval | <1s |
| Formatter response | <2s |
| Places API | <1s |

## License

MIT License - See [LICENSE](LICENSE) for details.
