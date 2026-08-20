# Therapy App

A serverless backend that connects clients and therapists — managing their relationship, therapy sessions, and messaging — built entirely on AWS with infrastructure defined as Java code.

## Overview

Therapy App models the core workflow of an online therapy platform: clients and therapists each have accounts, a client can be *mapped* to a therapist to start a therapeutic relationship, therapists log session notes (shared and private), and both sides can exchange messages. The entire stack — API, compute, and data — is provisioned through the AWS CDK, so the infrastructure and the business logic live in the same repository and are versioned together.

## Architecture

```
Client / Therapist
        │
        ▼
  Amazon API Gateway  (Client-Therapist API)
        │
        ▼
   AWS Lambda (Java)  ── one handler per operation, grouped by entity
        │
        ▼
   Amazon DynamoDB     ── one table per entity, pay-per-request
```

The project is split into two Maven modules:

| Module | Purpose |
|---|---|
| `infrastructure` | AWS CDK (Java) app that provisions DynamoDB tables, Lambda functions, IAM permissions, and the REST API Gateway. |
| `api-handlers` | The Lambda function code itself — request handlers, DynamoDB repositories, and data models, packaged as a single shaded JAR that the CDK stack deploys. |

## Tech stack

- **Language / build:** Java, Maven (multi-module)
- **Infrastructure as code:** AWS CDK (Java)
- **Compute:** AWS Lambda
- **API layer:** Amazon API Gateway (REST)
- **Data store:** Amazon DynamoDB (DynamoDB Enhanced Client)
- **Other:** Jackson (JSON), Log4j2 (logging)

## Project structure

```
therapy-app/
├── api-handlers/                    # Lambda function code
│   └── src/main/java/com/app/
│       ├── handler/                 # One package per entity: clients, therapists,
│       │                            # mappings, sessions, messages
│       ├── model/                   # DynamoDB-mapped POJOs (Client, Therapist, ...)
│       ├── repository/              # DynamoDB Enhanced Client wrappers per entity
│       └── util/                    # Shared ApiResponse + DynamoDB client helpers
│
├── infrastructure/                  # AWS CDK app
│   └── src/main/java/com/myorg/
│       ├── db/                      # DynamoDBStack — all table definitions
│       ├── lambda/                  # LambdaStack + per-entity Lambda stacks + factory
│       └── apigw/                   # ApiStack — REST API Gateway routes
│
├── dynamodb_crudl_api.yaml          # OpenAPI spec: full API design & roadmap
└── Database_Schema.txt              # Data model notes and query patterns
```

## Data model

| Table | Partition key | Sort key | Notable GSIs | Purpose |
|---|---|---|---|---|
| `Clients` | `clientId` | — | — | Client accounts |
| `Therapists` | `therapistId` | — | — | Therapist accounts and specialization |
| `Mappings` | `clientId` | `therapistId` | — | Client ↔ therapist relationships |
| `Sessions` | `sessionId` | `sessionDate` | `SessionDateIndex` (therapistId + sessionDate) | Therapy sessions, shared and private notes |
| `Messages` | `messageId` | `timestamp` | `ConversationIndex`, `SenderIndex` | Messages exchanged between clients and therapists |
| `Journals` | `clientId` | `timestamp` | — | Emotion journal entries *(table provisioned; not yet wired to an API)* |
| `Appointments` | `appointmentId` | `timestamp` | — | Appointment requests *(table provisioned; not yet wired to an API)* |

## API reference

Routes currently wired up in `ApiStack`:

**Clients**
| Method | Path | Description |
|---|---|---|
| POST | `/clients` | Create a client |
| DELETE | `/clients/{clientId}` | Delete a client |

**Therapists**
| Method | Path | Description |
|---|---|---|
| POST | `/therapists` | Create a therapist |
| DELETE | `/therapists/{therapistId}` | Delete a therapist |

**Mappings**
| Method | Path | Description |
|---|---|---|
| POST | `/mappings/client/{clientId}/therapist/{therapistId}` | Map a client to a therapist |
| DELETE | `/mappings/client/{clientId}/therapist/{therapistId}` | Remove the mapping |

**Sessions**
| Method | Path | Description |
|---|---|---|
| POST | `/sessions` | Create a session |
| PUT | `/sessions` | Update session notes |
| GET | `/sessions` | List all sessions |
| GET | `/sessions/by-therapist/{therapistId}/{sessionDate}` | List a therapist's sessions on a given date |
| DELETE | `/sessions/by-id/{sessionId}/{sessionDate}` | Delete a session |

**Messages**
| Method | Path | Description |
|---|---|---|
| POST | `/messages` | Send a message |
| GET | `/conversations/{senderId}/{receiverId}` | Get the message history between two users |
| GET | `/messages/{senderId}/{timestamp}` | Get a specific message by sender and timestamp |

The broader API surface — login, journal access, appointments, and full CRUDL for every resource — is designed in [`dynamodb_crudl_api.yaml`](dynamodb_crudl_api.yaml) and represents the roadmap beyond what's listed above.

## Getting started

### Prerequisites

- Java 17+ (Lambda functions run on the Java 21 runtime)
- Maven
- AWS CLI, configured with credentials for the target account
- AWS CDK Toolkit: `npm install -g aws-cdk`

### Build

Build the Lambda code first — the CDK stack loads the shaded JAR directly from `api-handlers/target`:

```bash
cd api-handlers
mvn package
```

### Deploy

```bash
cd infrastructure
cdk synth      # emit the synthesized CloudFormation template
cdk diff       # compare against what's currently deployed
cdk deploy     # deploy the stack
```

The CDK app doesn't pin an account/region by default — it deploys using your AWS CLI's configured account/region (or `CDK_DEFAULT_ACCOUNT` / `CDK_DEFAULT_REGION` if set). Note that the DynamoDB client in `api-handlers` currently targets the `ap-south-1` region directly, so keep that in mind when deploying elsewhere.

## Roadmap / known limitations

- **Journals and Appointments** — tables are provisioned but not yet exposed through Lambda handlers or API routes.
- **Authentication** — login endpoints and JWT-based auth are designed in the OpenAPI spec but not yet implemented; passwords are currently stored unhashed.
- **Testing & CI** — no automated test suite or CI pipeline yet.
- **Configuration** — table names and the AWS region are currently hardcoded in source rather than environment-driven.