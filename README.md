# Templify - PDF Generation SaaS

Multi-tenant PDF generation platform using Handlebars templates.

## Repository Structure

This is an orchestrator repository containing submodules:

```
paper-workspace/
├── templify-backend/    # Serverless API (Lambda, DynamoDB, S3, Cognito)
├── templify-web/        # Next.js frontend dashboard
└── templify-cli/        # Command-line interface
```

## Getting Started

Clone with submodules:

```bash
git clone --recurse-submodules https://github.com/templifying/paper-workspace.git
```

Or if already cloned:

```bash
git submodule update --init --recursive
```

## Environments

| Environment | API URL | Frontend URL | Branch |
|-------------|---------|--------------|--------|
| Development | dev.apis.templifying.com | dev.app.templifying.com | dev |
| Staging | stage.apis.templifying.com | stage.app.templifying.com | stage |
| Production | prod.apis.templifying.com | app.templifying.com | main |

## Documentation

- [Architecture](docs/architecture/ARCHITECTURE.md)
- [API Reference](docs/api/API_REFERENCE.md)
