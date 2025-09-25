# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a CMS research repository comparing Strapi and WordPress. It contains:

- **strapi/**: Complete Strapi monorepo source code
- **wordpess/**: Complete WordPress source code  
- **Analysis documents**: Markdown files with research findings
- **QA-Documentation.md**: Questions and answers about the CMS implementations

## Architecture

### Strapi (strapi/)
- **Monorepo structure**: Uses Nx workspace with packages in `packages/` and `examples/`
- **Language**: TypeScript/JavaScript with Node.js backend
- **Database**: Supports PostgreSQL, MySQL, MariaDB, SQLite
- **Architecture**: Headless CMS with REST/GraphQL APIs

### WordPress (wordpess/)
- **Monolithic structure**: Traditional PHP application
- **Language**: PHP with MySQL database
- **Architecture**: Traditional CMS with themes and plugins system

## Development Commands

### Strapi Development
```bash
cd strapi/
yarn install
yarn build          # Build all packages
yarn build:watch    # Watch mode for development
yarn test:unit      # Run unit tests
yarn test:e2e       # Run end-to-end tests
yarn lint           # Run linting
yarn format         # Format code
```

### Key Strapi Scripts
- `yarn setup` - Clean install and build
- `yarn watch` - Watch all packages for changes
- `yarn test:api` - Run API tests
- `yarn test:front` - Run frontend tests

## Working with Source Code

When analyzing or answering questions about the CMS implementations:

1. **Strapi source**: Look in `strapi/packages/` for core functionality
2. **WordPress source**: Examine `wordpess/wp-includes/` and `wordpess/wp-admin/`
3. **Add Q&A entries**: Append questions and answers to `QA-Documentation.md`

## Key Directories

### Strapi
- `strapi/packages/core/` - Core Strapi functionality
- `strapi/packages/cli/` - Command line interface
- `strapi/packages/admin/` - Admin panel
- `strapi/examples/` - Example applications

### WordPress
- `wordpess/wp-includes/` - Core WordPress functions and classes
- `wordpess/wp-admin/` - Administration interface
- `wordpess/wp-content/` - Themes, plugins, uploads

## Research Focus

This repository is specifically for comparing:
- Architecture patterns between headless (Strapi) and traditional (WordPress) CMS
- Database abstraction layers
- Tag and category systems
- API design approaches
- Content management workflows