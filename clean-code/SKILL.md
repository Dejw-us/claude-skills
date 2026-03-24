---
name: clean-code
description: Write clean, scalable, production-ready code following senior engineering best practices. Use this skill when the user asks to write, review, or refactor any code. Applies to all languages and frameworks. Enforces architectural discipline, naming clarity, separation of concerns, and long-term maintainability over clever shortcuts.
---

This skill enforces the coding standards of a senior engineer with 10+ years of production experience. Every output must be code that a team could maintain, extend, and reason about without the original author present.

## Core Philosophy

Code is written once but read hundreds of times. Optimize for the reader, not the writer. Every decision must answer: "Will this be obvious to someone reading this in 6 months?"

**Non-negotiable principles:**
- **Explicitness over cleverness** — if it requires a comment to understand, rewrite it
- **Single responsibility** — one function, one job. One module, one concern
- **Fail loudly** — errors must be handled, never silently swallowed
- **No premature abstraction** — abstract when you see the third repetition, not the first
- **Composition over inheritance** — prefer small composable units over deep hierarchies

## Naming

Names are the most important documentation. Never abbreviate unless the abbreviation is universally known (e.g. `id`, `url`, `api`).

**Rules:**
- Functions: verb + noun describing what they DO (`calculateMonthlyRevenue`, not `calc` or `process`)
- Booleans: prefix with `is`, `has`, `can`, `should` (`isActive`, `hasPermission`)
- Collections: always plural (`users`, `orderItems`)
- Constants: SCREAMING_SNAKE_CASE for true constants, descriptive names (`MAXIMUM_RETRY_ATTEMPTS`, not `MAX`)
- Avoid noise words: `data`, `info`, `manager`, `helper`, `utils` — these say nothing
- If you need `And` in a function name, it does two things — split it

**Bad:**
```
fn proc(d: &[u8]) -> Res
let flag = true
let mgr = UserManager::new()
```

**Good:**
```
fn parse_authentication_token(raw_bytes: &[u8]) -> Result<AuthToken>
let is_email_verified = true
let user_repository = UserRepository::new()
```

## Function Design

- Maximum 50 lines per function — if longer, extract
- Maximum 3 parameters — if more, use a struct/object
- One level of abstraction per function — don't mix high-level orchestration with low-level details
- Guard clauses over nested conditionals — return early, keep the happy path unindented

**Bad:**
```rust
fn handle(user: &User, order: &Order, db: &Pool, cache: &Cache, notify: bool) {
    if user.is_active {
        if order.total > 0.0 {
            if let Ok(result) = db.save(order) {
                cache.invalidate(user.id);
                if notify {
                    send_email(user.email);
                }
            }
        }
    }
}
```

**Good:**
```rust
fn process_order(user: &User, order: &Order, context: &OrderContext) -> Result<()> {
    validate_user_is_active(user)?;
    validate_order_has_positive_total(order)?;
    
    context.order_repository.save(order)?;
    context.cache.invalidate_user(user.id);
    
    if context.notifications_enabled {
        context.notification_service.send_order_confirmation(user, order)?;
    }
    
    Ok(())
}
```

## Error Handling

- Never ignore errors — every `Result`, `Option`, or nullable must be handled explicitly
- Error messages must be actionable — say what failed AND what the caller can do about it
- Propagate errors up, handle them at the boundary where you have enough context
- Don't log AND return an error — pick one (usually return, let the caller log)
- Use typed errors, not strings — makes error handling exhaustive and refactor-safe

**Bad:**
```rust
let user = db.find_user(id).unwrap();
let _ = cache.set(key, value);
```

**Good:**
```rust
let user = db.find_user(id)
    .map_err(|_| Error::not_found(format!("User with id {} does not exist", id)))?;

cache.set(key, value)
    .map_err(|error| Error::internal(format!("Failed to cache user {}: {}", id, error)))?;
```

## Architecture & Structure

- **Separate layers** — data access, business logic, and presentation must never mix
- **Repository pattern** for data access — business logic never writes SQL
- **Service layer** for business logic — orchestrates repositories, never touches HTTP/DB directly
- **Dependency injection** — pass dependencies in, never instantiate them inside
- **No god objects** — if a struct has more than 5 responsibilities, split it
- **Domain objects** over primitive obsession — `UserId(Uuid)` not raw `Uuid`, `Email(String)` not raw `String`

## Code Smells to Refuse

Immediately refactor or flag these:

- **Magic numbers/strings** — every literal that isn't 0, 1, or an empty string needs a named constant
- **Boolean parameters** — `fn send(user, true)` is unreadable. Use enums: `fn send(user, NotificationType::Email)`
- **Commented-out code** — delete it, git history exists
- **TODO without a ticket** — either fix it now or create a tracking issue
- **Nested ternaries** — maximum one level
- **Functions that return different types based on a flag** — split into two functions
- **Mutable global state** — never

## Testing Philosophy

- Test behavior, not implementation — tests should survive refactors
- One assertion concept per test — not one `assert`, but one logical thing being verified
- Test names are sentences: `should_return_error_when_user_is_not_found`
- Arrange-Act-Assert structure, always
- No logic in tests — no loops, no conditionals
- Test edge cases explicitly: empty inputs, zero values, boundary conditions

## Documentation

- Code should be self-documenting — if you need a comment to explain WHAT, rename things
- Comments explain WHY, not WHAT
- Document public APIs with examples
- Never leave stale comments — wrong documentation is worse than no documentation

**Bad:**
```rust
// increment i
i += 1;

// check if user exists
if user.is_some() {
```

**Good:**
```rust
// RevenueCat requires idempotency — duplicate events must be silently ignored
if self.webhook_repository.is_already_processed(&event.id).await? {
    return Ok(());
}
```

## Review Checklist

Before outputting any code, verify:

- [ ] Every function has a single, clear responsibility
- [ ] All errors are handled explicitly
- [ ] No magic literals — all constants are named
- [ ] No abbreviations in names
- [ ] Dependencies are injected, not instantiated
- [ ] No dead code, no commented-out blocks
- [ ] Edge cases are handled (empty collections, null/None, zero values)
- [ ] The code reads like prose — top to bottom, no mental gymnastics required
