# Architecture & Coding Standards

This document explains the project structure, design patterns, and coding methodology used in the Centralized Email Webhook System. Follow these patterns when extending the system.

---

## Architecture Overview

### Design Philosophy

**Separation of Concerns**: Each component has a single, well-defined responsibility.

```
API Layer → Services Layer → Data Models
                ↓
        Routing/Business Logic
                ↓
        Utility Layer (helpers)
```

### Layered Architecture

**Layer 1: API (api/)**
- **Responsibility**: HTTP endpoints, request/response handling
- **Pattern**: FastAPI routers
- **Dependencies**: Services layer
- **Example**: `webhook.py`, `test_endpoints.py`

**Layer 2: Services (services/)**
- **Responsibility**: Business logic, external integrations
- **Pattern**: Service classes with singleton instances
- **Dependencies**: Models, utils
- **Example**: `GraphService`, `EmailFetcher`, `SubscriptionManager`

**Layer 3: Routing (routing/)**
- **Responsibility**: Email routing logic
- **Pattern**: Static methods, pure functions
- **Dependencies**: Models
- **Example**: `RuleMatcher`, `Dispatcher`

**Layer 4: Models (models/)**
- **Responsibility**: Data structures
- **Pattern**: Dataclasses
- **Dependencies**: None (or minimal)
- **Example**: `EmailMetadata`, `UtilityConfig`

**Layer 5: Utils (utils/)**
- **Responsibility**: Shared utilities
- **Pattern**: Standalone functions or classes
- **Dependencies**: Minimal
- **Example**: `deduplication`, `logging_config`, `retry_handler`

---

## Design Patterns

### 1. Singleton Pattern

Used for services that should have only one instance.

**Example**:
```python
# services/graph_service.py
class GraphService:
    def __init__(self):
        self.token_cache = None
    
    # ... methods

# Global singleton instance
graph_service = GraphService()
```

**Usage**:
```python
from services.graph_service import graph_service

token = graph_service.get_access_token()
```

**Why**: Shared state (token cache), resource efficiency, consistent interface.

### 2. Service Layer Pattern

All external integrations wrapped in service classes.

**Example**:
```python
class GraphService:
    """Encapsulates Microsoft Graph API operations"""
    
    def get_access_token(self) -> str:
        """Handles authentication, caching"""
        pass
    
    def fetch_email(self, mailbox: str, message_id: str) -> dict:
        """Fetches email data"""
        pass
```

**Why**: Testability, maintainability, abstraction of external dependencies.

### 3. Dataclass Pattern

Structured data with type hints.

**Example**:
```python
@dataclass
class EmailMetadata:
    message_id: str
    subject: str
    from_address: str
    # ... more fields
    
    def to_dict(self) -> dict:
        """Serialize for API calls"""
        return asdict(self)
```

**Why**: Type safety, auto-generated methods, clear contracts.

### 4. Static Method Pattern

Stateless utility methods.

**Example**:
```python
class RuleMatcher:
    @staticmethod
    async def find_matching_utilities(
        email: EmailMetadata,
        utilities: List[UtilityConfig]
    ) -> List[UtilityConfig]:
        """Pure function - no instance state"""
        pass
```

**Why**: No instance needed, clear that method doesn't depend on object state.

### 5. Fire-and-Forget Pattern

Webhook endpoint responds immediately, processes in background.

**Example**:
```python
@router.post("/webhook")
async def webhook_notification(request: Request):
    data = await request.json()
    
    # Respond immediately (within 5 seconds)
    asyncio.create_task(process_notifications(data))
    
    return {"status": "accepted"}

async def process_notifications(data):
    # Long-running processing
    pass
```

**Why**: Microsoft requires <5s response, allows async processing.

### 6. Dependency Injection

Pass dependencies explicitly (don't use globals inside functions).

**Example**:
```python
# Good
async def process_email(email: EmailMetadata, utilities: List[UtilityConfig]):
    matched = RuleMatcher.find_matching_utilities(email, utilities)
    await Dispatcher.dispatch_to_utilities(email, matched)

# Bad - hidden dependencies
async def process_email(email: EmailMetadata):
    utilities = global_utilities  # Hidden dependency
    matched = RuleMatcher.find_matching_utilities(email, utilities)
```

**Why**: Testability, clarity, flexibility.

---

## Coding Conventions

### File Naming

- **Modules**: `snake_case.py`
- **Classes**: `PascalCase`
- **Functions**: `snake_case()`
- **Constants**: `UPPER_SNAKE_CASE`

### Directory Structure

```
module_name/
├── __init__.py          # Empty or exports
├── main_service.py      # Primary service
├── helper_service.py    # Supporting service
└── models.py            # Related models (if complex)
```

### Import Organization

```python
# 1. Standard library
import asyncio
import logging
from datetime import datetime

# 2. Third-party
from fastapi import APIRouter
import aiohttp

# 3. Local modules
from services.graph_service import graph_service
from models.email_metadata import EmailMetadata
import config
```

### Type Hints

Always use type hints for:
- Function parameters
- Return types
- Class attributes (in dataclasses)

```python
def fetch_email(
    self, 
    mailbox: str, 
    message_id: str
) -> EmailMetadata:
    pass
```

### Docstrings

Use for:
- All public classes
- All public methods
- Complex functions

```python
def create_subscription(self, mailbox: str, folder: str = "Inbox") -> dict:
    """
    Create a new webhook subscription.
    
    Args:
        mailbox: Email address to monitor
        folder: Folder name (defaults to Inbox)
    
    Returns:
        Subscription data from Microsoft Graph
    """
    pass
```

### Error Handling

```python
try:
    # Specific operation
    result = await risky_operation()
except SpecificException as e:
    # Handle specific error
    logger.error(f"Specific error: {e}")
    raise
except Exception as e:
    # Catch-all for unexpected errors
    logger.error(f"Unexpected error: {e}", exc_info=True)
    raise
```

### Logging

Use structured logging with context:

```python
logger.info(f"Processing email: '{email.subject}' from {email.from_address}")
logger.warning(f"Retry {attempt}/{max_retries} for {utility.name}")
logger.error(f"Failed to process email {email.message_id}: {error}")
```

---

## Configuration Management

### Environment Variables

**Pattern**:
```python
# config.py
import os
from dotenv import load_dotenv

load_dotenv()

# Read all config here
CLIENT_ID = os.getenv('CLIENT_ID')
MAX_RETRIES = int(os.getenv('MAX_RETRIES', 3))

# Services import from config
import config

token = get_token(config.CLIENT_ID)
```

**Why**: Single source of truth, easy to test, explicit dependencies.

### Configuration Files

**Pattern**:
```python
# services/config_service.py
class ConfigService:
    def __init__(self):
        self.config_file = 'config/utility_rules.json'
        self.cache = None
    
    async def get_all_utilities(self) -> List[UtilityConfig]:
        if self.cache is None:
            self.cache = self._load_from_file()
        return self.cache
```

**Why**: Caching, lazy loading, consistent interface.

---

## Async/Await Patterns

### When to Use Async

**Use async for**:
- HTTP requests (aiohttp)
- Database operations
- I/O operations
- Concurrent processing

**Don't use async for**:
- CPU-bound operations
- Simple data transformations
- Synchronous libraries

### Concurrent Processing

**Pattern**:
```python
# Process multiple items concurrently
tasks = [process_item(item) for item in items]
results = await asyncio.gather(*tasks, return_exceptions=True)

# With semaphore for rate limiting
semaphore = asyncio.Semaphore(25)

async def limited_operation():
    async with semaphore:
        # Only 25 concurrent operations
        return await expensive_operation()
```

---

## Testing Patterns

### Unit Tests

```python
# Test pure functions
def test_rule_matcher():
    email = EmailMetadata(...)
    utility = UtilityConfig(...)
    
    matched = RuleMatcher.matches_filters(email, utility)
    
    assert matched == True
```

### Integration Tests

```python
# Test with real services
async def test_email_fetcher():
    notification = {...}
    email = await email_fetcher.fetch_email(notification)
    
    assert email.subject is not None
    assert email.has_attachments
```

### Manual Testing

Use test endpoints:
```python
@router.get("/test/something")
async def test_something():
    # Quick test endpoint
    return {"result": "test data"}
```

---

## Extension Guidelines

### Adding a New Service

1. Create file in `services/`
2. Define service class
3. Create singleton instance
4. Add to imports where needed

```python
# services/new_service.py
class NewService:
    def __init__(self):
        pass
    
    def do_something(self):
        pass

# Global instance
new_service = NewService()
```

### Adding a New Filter Type

1. Update `UtilityConfig` model
2. Add to `RuleMatcher._matches_filters()`
3. Update config examples
4. Test

### Adding a New Utility

1. Create utility API (separate project)
2. Add to `config/utility_rules.json`
3. Test with webhook.site first
4. Update to real endpoint
5. Monitor logs

### Adding a New Feature

1. **Plan**: Document in artifact
2. **Implement**: Follow existing patterns
3. **Test**: Unit + integration
4. **Document**: Update README, add to FUTURE_ENHANCEMENTS if partial
5. **Deploy**: Test on Render
6. **Monitor**: Check logs for issues

---

## Common Pitfalls to Avoid

### 1. Don't Block the Event Loop

```python
# Bad
def fetch_emails():
    response = requests.get(url)  # Blocking call in async context
    return response.json()

# Good
async def fetch_emails():
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()
```

### 2. Don't Swallow Exceptions

```python
# Bad
try:
    critical_operation()
except:
    pass  # Silent failure

# Good
try:
    critical_operation()
except SpecificException as e:
    logger.error(f"Operation failed: {e}")
    # Re-raise or handle appropriately
    raise
```

### 3. Don't Use Mutable Defaults

```python
# Bad
def process(items=[]):
    items.append(new_item)  # Mutates default!

# Good
def process(items=None):
    if items is None:
        items = []
    items.append(new_item)
```

### 4. Don't Hardcode Configuration

```python
# Bad
client_state = "SecretClientState123"

# Good
client_state = config.WEBHOOK_CLIENT_STATE
```

### 5. Don't Create Circular Imports

```python
# Bad - circular dependency
# a.py imports b.py
# b.py imports a.py

# Good - shared module
# a.py imports models.py
# b.py imports models.py
```

---

## Performance Considerations

### 1. Use Caching

```python
class GraphService:
    def __init__(self):
        self.token_cache = None  # Cache expensive operations
```

### 2. Batch Operations

```python
# Process in batches
for i in range(0, len(items), batch_size):
    batch = items[i:i + batch_size]
    await process_batch(batch)
```

### 3. Limit Concurrency

```python
semaphore = asyncio.Semaphore(25)  # Max 25 concurrent
```

### 4. Async Where Possible

```python
# Concurrent API calls (non-blocking)
tasks = [call_api(url) for url in urls]
results = await asyncio.gather(*tasks)
```

---

## Security Best Practices

1. **Never commit secrets**: Use environment variables
2. **Validate all inputs**: Check webhook client state
3. **Use HTTPS only**: All API calls secure
4. **Limit permissions**: Minimum required scopes
5. **Log security events**: Track validation failures

---

## Deployment Checklist

Before deploying new code:

- [ ] All tests pass
- [ ] No hardcoded secrets
- [ ] Environment variables documented
- [ ] Logging added for new features
- [ ] Error handling implemented
- [ ] Type hints added
- [ ] Docstrings written
- [ ] README updated if needed
- [ ] Backward compatible (or migration plan)
- [ ] Tested locally first

---

## Summary

**Core Principles:**
1. Separation of concerns (layers)
2. Single responsibility (each class/function)
3. Explicit dependencies (no hidden globals)
4. Type safety (type hints everywhere)
5. Async for I/O (not for CPU)
6. Comprehensive logging (debug-friendly)
7. Fail-soft design (graceful degradation)
8. Configuration-driven (not hardcoded)

**When in doubt:**
- Look at existing similar code
- Follow the pattern
- Keep it simple
- Document your decision

**Remember:** Consistency is key. Following these patterns makes the codebase:
- Easier to understand
- Easier to maintain
- Easier to test
- Easier to extend
