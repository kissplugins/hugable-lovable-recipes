# Hugable Lovable Recipes

A monitoring/uptime tracking web application built with React, TypeScript, Vite, and Supabase.

## Architecture

This project follows a **checklist-driven architecture** approach. All architectural decisions, design patterns, and development workflows are documented in:

📋 **[AGENTS.md](./AGENTS.md)** - Checklist-Driven Architecture Guide

The AGENTS.md document provides:
- Pre-build, build, and post-build checklists
- SOLID principles applied to this stack
- State management patterns
- CMS content management guidelines
- Continuous audit and improvement processes

## Tech Stack

- **Frontend**: React + TypeScript + Vite
- **Backend**: Supabase (database + auth + edge functions)
- **State Management**: Zustand
- **UI Components**: Component-based with unified layout system

## Key Features

- User authentication with role-based access (users + admins)
- Monitor management and uptime tracking
- Events feed (max 500 events)
- Admin CMS for content management
- Status page and changelog

## Getting Started

```bash
# Install dependencies
bun install

# Run development server
bun run dev

# Type checking
bun run typecheck

# Linting
bun run lint

# Tests
bun run test

# Database diff
supabase db diff
```

## License

**Code:** This project's code is available under standard open-source terms.

**Documentation (AGENTS.md):**
Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) (Creative Commons Attribution 4.0 International)
© 2026 Hypercart DBA Neochrome, Inc.

When sharing or adapting AGENTS.md, you must:
- Credit "Hypercart DBA Neochrome, Inc." as the original author
- Provide a link to the CC BY 4.0 license
- Indicate if changes were made
- Not remove attribution notices

---

For detailed architectural guidelines and development checklists, see **[AGENTS.md](./AGENTS.md)**.
