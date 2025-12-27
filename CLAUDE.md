# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

Templify is a multi-tenant PDF generation SaaS platform. Users can upload Handlebars templates, manage API keys, and generate PDFs via API or web interface.

## Repository Structure

This is an **orchestrator repository** containing git submodules:

- `templify-backend/` - Serverless Framework API (AWS Lambda, DynamoDB, S3, Cognito)
- `templify-web/` - Next.js 14 frontend dashboard
- `templify-cli/` - Node.js CLI tool (oclif)

Each submodule is an independent repository with its own CI/CD pipeline.

## Development Workflow

### Cloning

```bash
git clone --recurse-submodules https://github.com/templifying/paper-workspace.git
```

### Working with Submodules

```bash
# Update all submodules to latest
git submodule update --remote

# Work in a submodule
cd templify-backend
git checkout dev
# make changes, commit, push
```

## Branch Strategy

| Branch | Environment | Description |
|--------|-------------|-------------|
| main | Production | Stable releases |
| stage | Staging | Client testing |
| dev | Development | Active development |

## CI/CD

- **Backend**: GitHub Actions → Serverless Framework deploy (OIDC auth)
- **Frontend**: GitHub Actions → AWS Amplify
- **CLI**: npm publish

## Key Architecture Decisions

1. **Serverless Framework** over CDK for simpler API-focused deployments
2. **Dual Authentication**: Cognito (web) + API tokens (programmatic)
3. **User Isolation**: Separate S3 prefixes and DynamoDB partition keys per user
4. **Subscription Tiers**: Free, Starter, Professional, Enterprise

## Domain Configuration

- Domain: `templifying.com`
- Route53 Hosted Zone: `Z05722842FNRUUMO9PPEV`
- AWS Account: `rocketeast` profile (197837191835)
