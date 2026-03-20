# Project Config

- **Project**: FinanceServer — a trading platform with UI dashboard, backend services, and K8s-based infrastructure
- **Server SSH**: `ssh ai@192.168.8.8` (K8s cluster host)
- **Kubectl**: `kubectl XXX` (direnv auto-sets `KUBECONFIG` to the microk8s-trader context when you `cd` into this project — no `--context` flag needed. See `.envrc` in project root.)
- **Domain (production)**: `kellytrade.org` (UI: `https://www.kellytrade.org`, API: `api.kellytrade.org`, wildcard: `*.kellytrade.org`)
- **Local dev URL**: `http://localhost:3000`
- **K8s namespace**: `trading-svc`
- **Cloudflare config**: `/etc/cloudflared/config.yml` on server
- **Token file**: `claude_tokens.md` (contains `CLAUDE_SERVICE_TOKEN` and `CLAUDE_TOKEN_EXPIRES`)
- **UI directory**: `ui/`


# Core Philosophy

Write code that is **simple, maintainable, and production-ready**. Prioritize clarity over cleverness, and always leave the codebase cleaner than you found it.

## Key Principles

1. **Simplicity First**: Choose the simplest solution that meets requirements (KISS principle)
2. **Consistency**: Maintain tech stack consistency unless there's strong justification for change
3. **Maintainability**: Prioritize code clarity and self-documentation over clever solutions
4. **Scalability**: Design for growth without premature optimization
5. **Best Practices**: Follow established patterns, idioms, and community conventions
6. **Clean Architecture**: Apply SOLID principles and maintain separation of concerns
7. **Quality First**: Continuous refactoring and cleanup are not optional

## Code Quality Standards

### SOLID Principles (Non-negotiable)

- **Single Responsibility**: Each class/function has exactly one reason to change
- **Open/Closed**: Open for extension, closed for modification
- **Liskov Substitution**: Subtypes must be substitutable for their base types
- **Interface Segregation**: Many specific interfaces beat one general-purpose interface
- **Dependency Inversion**: Depend on abstractions, not concrete implementations

### Clean Code Practices

**Code Organization:**

- Keep functions small (< 20 lines ideally, < 100 lines maximum)
- One level of abstraction per function
- Use meaningful, pronounceable names (avoid abbreviations unless widely known)
- Self-documenting code is the goal; comments explain "why", not "what"

**Code Quality:**

- **DRY Principle**: Don't Repeat Yourself - eliminate duplication through abstraction
- **YAGNI**: You Aren't Gonna Need It - don't add features speculatively
- Write code that's easy to delete, not easy to extend
- Prefer composition over inheritance

**Error Handling:**

- Fail fast and explicitly
- Use typed errors/exceptions with clear messages
- Never silently ignore errors
- Validate inputs at system boundaries

---

# Technology Stack

### Frontend & UI

**JavaScript/TypeScript:**

- Package Manager: `npm`
- Runtime: Node.js
- Framework: React
- UI Components: shadcn-ui (https://ui.shadcn.com/), Radix UI primitives
- Charts: lightweight-charts (https://github.com/tradingview/lightweight-charts)
- Styling: Tailwind CSS
- Type Safety: TypeScript with strict mode enabled

### Backend

**Python:**

- **CRITICAL**: Use `uv` exclusively - NEVER use `pip` directly
- Virtual Environment: `uv venv` followed by `source .venv/bin/activate`
- Dependency Management: `uv sync` (not `pip install`)
- CLI Reference: `uv --help`
- **AVOID**: `hatchling.build` in pyproject.toml
- API Framework: FastAPI with Uvicorn
- Type Hints: Use type annotations consistently (Python 3.10+ syntax)

**Node.js:**

- Use only when justified, prefered is python
- Package Manager: `npm`
- API Framework: Express.js
- Bundler: `esbuild` (target both CommonJS and ESM if possible, otherwise ESM only)

**Go:**

- Use only when justified (performance-critical services, system tools), prefered is python

### Scripting & Automation

- **Default**: Python for all scripts
- **Avoid**: Bash/Shell scripts (unless trivial one-liners), PowerShell
- Keep scripts simple, documented, and testable

### Operating System

- **Primary**: Linux (Ubuntu LTS preferred)
- **Containers**: Alpine or distroless for production images
- **Alternative**: Windows only when specifically required

### AI & Machine Learning

**Frameworks:**

- ML/DL: PyTorch (primary), avoid TensorFlow
- Model Hub: Hugging Face for pretrained models
- Embeddings: sentence-transformers

**Agent Development:**

- **Preferred**: Claude Agent SDK (https://docs.claude.com/en/docs/agent-sdk/overview)
- **Alternative**: LangChain >= 1.0.0 + LangGraph >= 1.0.0 (https://docs.langchain.com/)
- Keep agent chains simple and debuggable

### Infrastructure & DevOps

**Containerization:**

- Docker for local development and production
- Multi-stage builds for minimal image sizes
- .dockerignore to exclude unnecessary files

**Orchestration:**

- Kubernetes via Google Kubernetes Engine (GKE)
- Helm for K8s resource templating
- Keep manifests DRY and well-documented

**Infrastructure as Code:**

- Terraform for cloud resources
- Version control all IaC configurations
- Use modules for reusability

**Networking:**

- Traefik as ingress/reverse proxy

---

# Code Review Checklist

Before considering code complete, verify:

- [ ] All unused code removed (functions, variables, imports, comments)
- [ ] Comments updated to reflect current implementation
- [ ] No code duplication (DRY principle applied)
- [ ] Functions are small and focused (single responsibility)
- [ ] Error handling is explicit and comprehensive
- [ ] Type safety maintained (TypeScript/Python type hints)
- [ ] Tests written/updated and passing
- [ ] No hardcoded values (use configuration/environment variables)
- [ ] Security best practices followed (no secrets in code, input validation, etc.)
- [ ] Performance considered (no obvious bottlenecks)

---

# Communication Guidelines

**Context:**

- Assume 20+ years of software engineering experience
- Skip basic explanations unless specifically requested
- Be direct and technical in communication
- Ask for clarification when requirements are ambiguous

**Code Explanations:**

- Focus on "why" decisions were made, not "what" the code does
- Highlight tradeoffs and alternatives considered
- Point out areas that may need future attention

## Anti-Patterns to Avoid

- No commented-out code "just in case"
- No TODO comments
- No copy-paste instead of abstracting
- No premature optimization
- No over-engineering simple solutions
- No ignoring compiler/linter warnings
- No writing code without understanding its purpose
- No implementing features "we might need someday"

## Remember

> "Code is read far more often than it is written." - Guido van Rossum

> "Any fool can write code that a computer can understand. Good programmers write code that humans can understand." - Martin Fowler

Write code you'd be proud to maintain in 2 years.