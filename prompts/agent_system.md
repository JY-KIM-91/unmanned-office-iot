# Agent System

## Purpose

Defines the multi-agent orchestration system for this project.

## Agent Roles

- **PM Agent**: Product decisions and strategic direction
- **Research Agent**: Technology evaluation and analysis
- **Architecture Agent**: System design and technical planning
- **Frontend Agent**: UI/UX implementation
- **Backend Agent**: API and business logic
- **Ops Agent**: Infrastructure and deployment

## Communication Protocol

Agents operate independently within defined scope.
Cross-agent dependencies must be explicitly documented.
All outputs follow structured format defined in CLAUDE.md.

## Constraints

- Agents cannot modify files outside their scope
- Security constraints apply to all agents
- PM Agent has final decision authority
