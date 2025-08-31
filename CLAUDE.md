# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NUcore is a comprehensive core facility management system written in Ruby on Rails. It's designed to manage scientific equipment reservations, billing, accounts, and user access across research facilities. The project began as a Rails 2.x application and has evolved to Rails 7.x while maintaining backwards compatibility with legacy code patterns.

## Development Commands

### Setup and Development
```bash
# Development setup (Docker recommended)
./docker-setup.sh                    # Initial setup with database and secrets configuration
docker-compose up                    # Start all services (Rails server, assets, database)

# Local development
bin/dev                              # Start Rails server with asset compilation (uses Procfile.dev)
bin/rails server                     # Rails server only
bundle exec rails c                  # Rails console

# Database operations
rake db:create db:schema:load db:seed  # Initial database setup (run separately to avoid issues)
rake demo:seed                       # Populate with demo data
rake db:migrate                      # Run migrations
```

### Testing
```bash
# Run all tests
bundle exec rspec                    # All RSpec tests
rake spec                           # Same as above
rake spec:models                    # Model tests only
rake spec:controllers               # Controller tests only

# Parallel testing (faster)
rake parallel:create                # Create test databases
rake parallel:load_schema           # Load schema for parallel tests  
rake parallel:prepare               # Prepare test databases after migrations
rake parallel:spec                  # Run tests in parallel

# JavaScript tests
bundle exec rake teaspoon           # JavaScript test suite

# Engine tests (located in vendor/engines/*/spec)
bundle exec rspec vendor/engines/*/spec/**/*_spec.rb

# Test single file
bundle exec rspec spec/path/to/file_spec.rb
```

### Code Quality
```bash
bundle exec rubocop                 # Ruby linting and style check
bundle exec rubocop --auto-correct  # Auto-fix style issues
```

### Asset Management
```bash
yarn build                         # Build JavaScript assets (uses esbuild)
yarn build --watch                 # Watch and rebuild assets
bundle exec rails assets:precompile # Precompile all assets for production
```

### Background Jobs
```bash
./script/delayed_job start         # Start background job processor
./script/delayed_job run           # Run jobs once
./script/delayed_job stop          # Stop background job processor
```

## Architecture Overview

### Core Domain Models
- **Facility**: Represents a research facility/core with instruments and services
- **Product**: Base class for instruments, items, services, and bundles that can be ordered
- **Order/OrderDetail**: Purchase transactions and individual line items with pricing
- **Account**: Financial accounts (credit card, PO, chart strings) for billing
- **User**: System users with role-based permissions across facilities
- **Reservation**: Time-based bookings for instruments with schedule rules
- **Statement**: Billing statements grouping transactions for accounts

### Key Architectural Patterns

#### Rails Engines for Modularity
Optional features are implemented as Rails engines in `vendor/engines/`:
- `c2po`: Credit card and purchase order payment processing
- `sanger_sequencing`: DNA sequencing workflow management
- `split_accounts`: Split billing across multiple accounts
- `secure_rooms`: Card reader access control integration
- `bulk_email`: Mass email functionality
- `ldap_authentication`: LDAP user authentication
- `saml_authentication`: SAML SSO authentication

#### Service Objects Pattern
Business logic is extracted into service classes in `app/services/`:
- `OrderPurchaser`: Handles order processing and validation
- `ReservationCreator`: Creates instrument reservations with conflict checking
- `StatementCreator`: Generates billing statements
- `JournalRowBuilder`: Creates accounting journal entries
- `AccountBuilder`: Creates and configures user accounts

#### Authorization via CanCanCan
Role-based permissions are managed through:
- `app/lib/ability.rb`: Main authorization rules
- Engine-specific ability extensions in `vendor/engines/*/app/models/*/ability_extension.rb`
- User roles: Admin, Facility Director, Staff, PI, User

#### Price Policy System
Dynamic pricing based on user roles and facility rules:
- `PricePolicy` base class with instrument/item/service variants
- `PriceGroup`: User categories (internal, external, etc.) with different rates
- `DurationRate`: Time-based pricing for instruments
- Support for subsidies, cancellation fees, and usage caps

### File Organization

#### Models (`app/models/`)
- Domain models following Active Record pattern
- Concerns in `app/models/concerns/` for shared functionality
- External service integrations in `app/models/external_services/`
- Reporting models in `app/models/reports/`

#### Controllers (`app/controllers/`)
- Resource-based controllers following RESTful conventions  
- Facility-scoped controllers with `facility_` prefix
- Shared controller concerns in `app/controllers/concerns/`
- API controllers in `app/controllers/api/` 

#### Views (`app/views/`)
- Haml templates with Bootstrap 3.4.1 styling
- Internationalization via `text()` helper from text-helpers gem
- View hooks system for engine extensibility

### Database Considerations
- Supports both MySQL (primary) and Oracle databases
- Extensive migration history dating back to Rails 2.x
- Soft deletes using paranoia gem for some models
- Full-text search capabilities for MySQL and Oracle

### External Integrations
- **Relay Control**: Power relay integration for instrument control (Dataprobe, SynAccess)
- **Research Safety**: Certification requirement checking
- **LDAP/SAML**: External authentication systems
- **File Storage**: Paperclip (legacy) and Active Storage with S3/Azure support
- **Form.io**: Dynamic form generation for custom order forms

## Development Guidelines

### Code Style
- Follow Ruby style guide enforced by RuboCop
- Prefer double quotes over single quotes
- Use Ruby 1.9+ hash syntax (`key: value`)
- Prefer `before_action` over `before_filter`
- Use I18n/locales for all user-facing text via `text()` helper

### Testing Practices  
- Write tests for all bug fixes before implementing the fix
- Use FactoryBot for test data creation
- Model tests in `spec/models/`, controller tests in `spec/controllers/`
- System tests with Capybara for integration testing
- JavaScript tests using Teaspoon with Jasmine

### Architecture Decisions
- **Service Objects**: Extract complex business logic from controllers/models
- **Presenters**: Use for view-specific logic rather than helpers
- **Engines**: Isolate optional/institution-specific features
- **Hooks**: Use ViewHook system for engine extensibility rather than view overrides
- Avoid DCI pattern (used in some legacy code like `PriceDisplayment`)

### Working with Legacy Code
- Refactor incrementally - leave code better than you found it
- Don't fix everything at once - focus on the area you're working in  
- Consider effort vs. value when refactoring
- Maintain backwards compatibility when possible

### Pull Request Process
- All changes must go through pull request review
- Never commit directly to master branch
- Include ticket numbers in PR titles: `[#12345] Fix critical issue`
- Use "Squash and Merge" to maintain clean commit history
- Write descriptive commit messages following Chris Beam's guidelines

### Engine Development
When creating institution-specific features:
- Develop in separate engine in `vendor/engines/`
- Add gem dependency to engine's gemspec, not main Gemfile
- Use hooks and extension points rather than overriding core files
- See existing engines like `projects` for extension patterns

### Debugging and Development Tools
- **Mailcatcher**: Email testing at http://localhost:1080 (Docker setup)
- **Rails Console**: `docker-compose exec app bundle exec rails c`
- **Delayed Job**: Monitor background job processing
- **Bullet**: Detect N+1 queries and suggest eager loading
- **Parallel Tests**: Faster test execution during development

## Common Gotchas
- Run database setup commands separately (`db:create`, `db:schema:load`, `db:seed`) to avoid "splits table exists" error
- Ruby version must match `.ruby-version` (currently 3.4.4)
- Some legacy code patterns exist - modernize incrementally when working in those areas
- Oracle support requires specific gem configuration (see documentation)
- Engine migrations live in engine directories, not main `db/migrate`