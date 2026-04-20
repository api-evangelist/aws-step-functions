# AWS Step Functions

AWS Step Functions is a serverless orchestration service that lets you coordinate distributed applications and microservices using visual workflows, integrating with AWS services and supporting error handling and retries.

## APIs

### AWS Step Functions API
Orchestration API for managing state machines and executions via Amazon States Language.
- **Documentation**: https://docs.aws.amazon.com/step-functions/
- **OpenAPI**: [openapi/aws-step-functions-openapi.json](openapi/aws-step-functions-openapi.json) (26 operations)
- **API Reference**: https://docs.aws.amazon.com/step-functions/latest/apireference/Welcome.html

## Artifacts

| Directory | Contents |
|---|---|
| [openapi/](openapi/) | 1 OpenAPI specification (26 operations) |
| [json-schema/](json-schema/) | 173 JSON Schema files |
| [json-structure/](json-structure/) | 173 JSON Structure files |
| [json-ld/](json-ld/) | 1 JSON-LD context file |
| [examples/](examples/) | 173 example files |
| [rules/](rules/) | Spectral ruleset |
| [capabilities/](capabilities/) | Naftiko capability definitions |
| [vocabulary/](vocabulary/) | Domain vocabulary |

## Features

- **Visual Workflow Design** — Design workflows using the Workflow Studio drag-and-drop interface.
- **Amazon States Language** — Define workflows in JSON with built-in error handling and retry logic.
- **AWS Service Integrations** — Natively integrate with over 220 AWS services without custom code.
- **Standard Workflows** — Long-running workflows with exactly-once task execution semantics.
- **Express Workflows** — High-volume, short-duration workflows optimized for cost.
- **Error Handling** — Built-in retry logic and catch blocks for graceful error handling.
- **Parallel Execution** — Run parallel branches simultaneously within executions.
- **Map State** — Process arrays of items in parallel using the Map state.
- **Wait for Callback** — Pause workflows waiting for external events using task tokens.
- **X-Ray Integration** — End-to-end visibility into workflow executions via AWS X-Ray.

## Use Cases

- **Microservice Orchestration** — Coordinate multiple microservices in reliable, fault-tolerant workflows.
- **Data Processing Pipelines** — Build ETL pipelines that process data in parallel.
- **Human Approval Workflows** — Implement approval workflows that wait for human decisions.
- **Event-Driven Automation** — Automate complex multi-step processes triggered by events.
- **Machine Learning Pipelines** — Orchestrate ML model training, evaluation, and deployment.

## Links

- **Website**: https://aws.amazon.com/step-functions/
- **Getting Started**: https://docs.aws.amazon.com/step-functions/latest/dg/getting-started.html
- **Pricing**: https://aws.amazon.com/step-functions/pricing/
- **Blog**: https://aws.amazon.com/blogs/compute/category/application-services/aws-step-functions/
- **Status**: https://health.aws.amazon.com/health/status

## Maintainers

- **Kin Lane** — kin@apievangelist.com
