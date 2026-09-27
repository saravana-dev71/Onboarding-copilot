# 🚀 Ramp Onboarding Copilot

### AI-Powered Developer Onboarding for Unfamiliar Repositories

Built for the **IBM Bob 2.0 Hackathon**

---

## 📌 Overview

**Ramp Onboarding Copilot** is a Bob-native AI developer onboarding system designed to help developers understand and contribute to an unfamiliar codebase faster.

When a new developer joins an existing project, they usually need to:

- Understand the project architecture
- Explore different services and entry points
- Read and validate existing documentation
- Set up the development environment
- Run tests, linting, and builds
- Identify suitable starter tasks

This information is often scattered across the repository.

**Ramp Onboarding Copilot brings these activities together into one automated onboarding workflow using IBM Bob.**

---

## 🎯 Problem

Getting started with an unfamiliar repository can be time-consuming.

A new contributor has to manually answer questions such as:

> What does this project contain?

> How are the services connected?

> Can I trust the existing documentation?

> Does the project actually build and pass its tests?

> What should I work on first?

Ramp Onboarding Copilot automates these discovery and verification steps.

---

## 💡 Our Solution

The system uses **four specialized IBM Bob custom modes** coordinated through an `onboarding-kit` skill.

```text
                    Repository
                        │
                        ▼
              ┌───────────────────┐
              │ /onboarding-kit    │
              └─────────┬─────────┘
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
 Architecture       Documentation    Environment
    Mapper             Auditor          Verifier
        │               │                │
        └───────────────┼────────────────┘
                        ▼
                  Task Curator
                        │
                        ▼
                  ONBOARDING.md
