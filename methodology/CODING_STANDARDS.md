# CopperForge — Coding Standards

These standards apply to all code written by CopperForge agents. QAM enforces these at Code Gate.

## Language and Framework

- **PHP 8.2+** for all backend work unless explicitly noted otherwise
- **Laravel** for application framework
- **Livewire** for reactive components where appropriate
- **Tailwind CSS** for styling

## Package Selection

**Laravel native packages are preferred over community packages.** If a first-party Laravel package exists for a given need, use it. Deviations require:

1. Explicit justification in the task comments
2. CTO approval before implementation
3. Documentation in the repo README explaining why

Examples:

- Use `Laravel\Sanctum` over third-party auth packages
- Use `Laravel\Cashier` over custom Stripe integration
- Use `Laravel\Horizon` for queue monitoring
- Use `Laravel\Scout` for search unless requirements exceed its scope

## Code Style

- **PHP Pint** for code formatting — run before every commit
- **PSR-12** as the underlying style standard
- **Type declarations** required on all method signatures (parameters and return types)
- **Strict types** (`declare(strict_types=1);`) at the top of every PHP file
- **Named arguments** for methods with more than 3 parameters

## Testing

- **Pest PHP** for new tests unless the project uses PHPUnit
- **Feature tests** for user-facing behavior and integration
- **Unit tests** for isolated logic
- **Red-Green-Refactor** — tests must fail before implementation begins
- **One assertion concept per test** — multiple assertions are fine, but each test should validate one behavior
- **Test names describe behavior** — `it_sends_ack_within_ten_minutes_of_classification` not `testAck`

## Version Control

- **Bitbucket** for hosted repositories (default; GitHub for skills repo and other public-facing work)
- **Feature branches** from main, merged via pull request
- **Commit messages** follow conventional commits: `feat:`, `fix:`, `refactor:`, `test:`, `docs:`, `chore:`
- **One logical change per commit** — no "misc updates" commits
- **Squash merge** to main unless history matters

## Deployment

- **Laravel Forge** on `cpLaravel2` for Laravel apps
- **WordPress** projects deployed per project convention (varies by client)
- **Migrations always reversible** — every `up()` has a corresponding `down()`
- **No destructive migrations** in production without board approval

## Security

- **Never commit secrets.** Use `.env` files, never hardcode.
- **Never log secrets or PII.** Sanitize logs before writing.
- **CSRF protection** on all state-changing routes
- **Rate limiting** on all public endpoints
- **Input validation** via Form Requests — never trust user input
- **Output escaping** in Blade is automatic; in Livewire, confirm component escaping
- **Prepared statements** for all database queries — no raw SQL concatenation

## Architecture Patterns

- **Service classes** for business logic — not fat controllers, not fat models
- **Form Requests** for validation
- **Events and Listeners** for cross-cutting concerns
- **Jobs** for async work, queued through Redis
- **Repositories** only when a real abstraction benefit exists — don't cargo-cult
- **Action classes** (single-method invokables) for discrete operations

## Performance

- **Eager loading** relationships to avoid N+1 queries — QAM checks for `->with()` calls
- **Database indexes** on all foreign keys and queried columns
- **Caching** for expensive operations — document TTL and invalidation strategy
- **Queue** any operation that might take longer than 500ms

## Documentation

- **DocBlocks** on public methods explaining non-obvious behavior
- **README.md** in every project with setup instructions
- **Architecture decisions** documented in `docs/decisions/` as ADRs
- **Current-state documentation** in `docs/current-state/`, updated by Dev per subtask, reconciled by QA at parent issue completion
- **API documentation** via OpenAPI/Swagger when an API is exposed

## Client Code (WordPress, frontend)

- **WordPress coding standards** for plugin and theme code
- **No modifications to core** — always extend via hooks
- **Escape output** with `esc_html`, `esc_attr`, `esc_url` as appropriate
- **Sanitize input** with `sanitize_text_field`, `wp_kses_post`, etc.
- **Nonces** on all admin forms

## When Standards Conflict

If a task requires violating one of these standards, the violation must be:

1. Flagged in the task comments before implementation
2. Justified with a specific reason
3. Approved by CTO
4. Documented in the code with a comment explaining why

No silent deviations.
