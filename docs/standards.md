# Coding Standards & Preferences

> Load this file when writing or reviewing Python or CDK code.
> Edit freely — this file captures *your* preferences, not defaults.

---

## Python Version

Target: **Python 3.12**. Use modern syntax exclusively. Do not use compatibility shims for older versions.

---

## File Headers

Every `.py` file starts with a module docstring. Keep it concise — 3 sections max:

```python
"""
Brief one-line summary of what this module does.

Responsibilities:
- What this file owns (not what it calls)
- ...

Usage:
    # Only include if the module has a non-obvious entry point
    handler(event, context)
"""
```

Rules:
- The one-liner must fit on a single line
- Responsibilities describe *this file's* job, not its dependencies
- Skip the Usage section for CDK constructs and stacks (not directly invoked)
- No `Author:`, `Date:`, or `Version:` fields — that's what git is for

---

## Inline Comments

Write comments to explain **why**, not **what**. The code shows what; comments explain intent.

```python
# Good — explains a non-obvious decision
# maxReceiveCount=3: enough retries for transient failures, fast enough to catch poison messages
dead_letter_queue=sqs.DeadLetterQueue(max_receive_count=3, queue=dlq)

# Bad — restates the code
# Create a dead letter queue
dead_letter_queue=sqs.DeadLetterQueue(max_receive_count=3, queue=dlq)
```

Rules:
- One comment per logical block, not per line
- If you need a comment to explain a variable name, rename the variable instead
- Mark intentional edge-case handling with `# NOTE:` and tech debt with `# TODO:`
- Never leave commented-out code — delete it (git remembers)

---

## Type Hints (Python 3.12 style)

Use built-in generics — never import from `typing` unless strictly necessary.

```python
# Correct
def process(events: list[dict]) -> dict | None: ...
lookup: dict[str, int] = {}
items: tuple[str, ...] = ()

# Wrong — legacy style
from typing import List, Dict, Optional, Tuple
def process(events: List[Dict]) -> Optional[Dict]: ...
```

Use `TypeAlias` for complex reused types:

```python
type FlightId = str          # Python 3.12 soft keyword syntax
type EventDetail = dict[str, str | int | float]
```

Use `match/case` for dispatching on event types instead of `if/elif` chains:

```python
match event["detail-type"]:
    case "FlightDelayed":
        handle_delay(detail)
    case "FlightCancelled":
        handle_cancellation(detail)
    case _:
        logger.warning("Unrecognized event type", event_type=event["detail-type"])
```

---

## Pydantic Models (v2)

```python
from pydantic import BaseModel, field_validator, model_validator
from datetime import datetime
from uuid import UUID

class FlightInfo(BaseModel):
    model_config = {"frozen": True}   # immutable after creation

    airline: str
    flight_number: str
    departure_date: str                # ISO date string: "YYYY-MM-DD"
    origin: str                        # IATA airport code
    destination: str
```

Rules:
- Use `model_config = {"frozen": True}` on value objects (they shouldn't change)
- Validate at the boundary — parse once at the Lambda handler entry point
- Prefer `model.model_dump()` over `dict(model)` (v2 style)
- Use `@field_validator` for single-field rules, `@model_validator(mode="after")` for cross-field rules

---

## Lambda Handler Structure

Every handler follows this exact structure — no exceptions:

```python
"""
One-liner summary.

Responsibilities:
- ...
"""
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger(service="ingestion")   # module-level, not inside handler
tracer = Tracer(service="ingestion")

@tracer.capture_lambda_handler
@logger.inject_lambda_context(correlation_id_path="headers.x-correlation-id")
def handler(event: dict, context: LambdaContext) -> dict:
    # 1. Parse & validate input
    # 2. Business logic
    # 3. Return structured response
    ...
```

Rules:
- `Logger` and `Tracer` are module-level singletons — never instantiate inside the handler
- Always use `@logger.inject_lambda_context` to auto-populate `correlation_id`
- Return a `dict` with `statusCode` and `body` (API Gateway format) or plain `dict` (SQS consumers)
- No bare `except Exception` — catch specific exceptions and log them with `logger.exception()`

---

## Structured Logging

```python
# Good — key=value pairs, searchable in CloudWatch Insights
logger.info("Event published", event_id=str(event.event_id), detail_type=detail_type)
logger.warning("Duplicate event skipped", event_id=str(event_id))
logger.error("DynamoDB write failed", event_id=str(event_id), error=str(e))

# Bad — unstructured string
logger.info(f"Published event {event_id} of type {detail_type}")
```

Standard keys to always include when available:
- `event_id` — UUID of the event being processed
- `correlation_id` — injected automatically by Powertools
- `flight_number` — when processing a flight-related event
- `detail_type` — the EventBridge `detail-type` string

---

## CDK Construct Structure

```python
"""
One-liner summary of what this construct creates.

Responsibilities:
- Resource A
- Resource B
"""
from constructs import Construct
from aws_cdk import aws_dynamodb as dynamodb

class DynamoDBTables(Construct):

    def __init__(self, scope: Construct, construct_id: str) -> None:
        super().__init__(scope, construct_id)

        self.flight_state_table = dynamodb.Table(   # expose as public attribute
            self, "FlightStateTable",
            ...
        )
```

Rules:
- Expose created resources as `self.<name>` public attributes — never return them
- `construct_id` must match the class name in PascalCase (e.g., `"DynamoDBTables"`)
- One construct per file, one concern per construct
- No business logic inside constructs — only resource definitions and IAM grants

---

## CDK Stack Structure

```python
"""
One-liner summary.

Responsibilities:
- Which constructs this stack composes
- Which outputs it exposes
"""
from aws_cdk import Stack, CfnOutput
from constructs import Construct

class DataStack(Stack):

    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        tables = DynamoDBTables(self, "DynamoDBTables")

        # Expose for cross-stack references
        self.flight_state_table = tables.flight_state_table

        CfnOutput(self, "FlightStateTableName", value=self.flight_state_table.table_name)
```

---

## Naming Conventions

| Thing | Convention | Example |
|---|---|---|
| Python files | `snake_case.py` | `ingestion_lambda.py` |
| Classes | `PascalCase` | `FlightDelayed`, `DataStack` |
| Functions / methods | `snake_case` | `handle_delay()` |
| Constants | `SCREAMING_SNAKE` | `MAX_RETRY_COUNT = 3` |
| Environment variables | `SCREAMING_SNAKE` | `EVENT_BUS_NAME` |
| DynamoDB table names | `kebab-case` | `flight-state`, `disruption-log` |
| DynamoDB attribute names | `snake_case` | `flight_number`, `occurred_at` |
| SQS queue names | `kebab-case` | `notify-queue`, `audit-dlq` |
| CDK construct IDs | `PascalCase` | `"FlightStateTable"` |
| EventBridge source | `dot.notation` | `flight.ops` |

---

## Import Order

Follow PEP 8 with this project-specific grouping:

```python
# 1. Standard library
import json
import os
from datetime import datetime
from uuid import UUID, uuid4

# 2. Third-party (alphabetical within group)
import boto3
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.utilities.typing import LambdaContext
from pydantic import BaseModel, ValidationError

# 3. Local / project
from services.shared.models import EventEnvelope, FlightInfo
from services.shared.events import FlightDelayed, EVENT_TYPE_MAP
```

No blank lines within a group. One blank line between groups.

---

## What to Avoid

- `Optional[X]` → use `X | None`
- `Union[X, Y]` → use `X | Y`
- `List[X]`, `Dict[X, Y]` → use `list[X]`, `dict[X, Y]`
- `Any` type hint → name the actual type or use a `TypeAlias`
- `print()` statements → always use `logger`
- Hardcoded ARNs or table names → always from environment variables
- `time.sleep()` in Lambda handlers → signals a design problem
- Mutable default arguments → `def f(items: list = [])` is a bug
- Bare `except:` or `except Exception:` without logging
