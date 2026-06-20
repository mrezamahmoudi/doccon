# Python Decorators in This Project — A Cheat Sheet

This document explains the purpose of each decorator you'll encounter in the codebase, with examples and real‑world usage.

---

## 1. `@router.get` / `@router.post` / `@router.put` — FastAPI Route Decorators

**File:** `app/api/routers/incident.py` (and all other routers)

```python
@router.get("/start_incident", status_code=status.HTTP_200_OK)
def start_incident(...):
    ...
What it does:
Registers the function as an HTTP endpoint. When someone calls GET /start_incident, FastAPI executes this function. The three variants correspond to HTTP methods:

Decorator	HTTP Method	Purpose
@router.get(...)	GET	Retrieve / read data
@router.post(...)	POST	Create new resources
@router.put(...)	PUT	Update existing resources
2. @abstractmethod — Forces Subclasses to Implement a Method
File: app/External/channel.py

python
class Channel(ABC):
    @abstractmethod
    def send_message(self, message: str):
        pass
What it does:
Declares a method that must be implemented by any concrete subclass of Channel. If a subclass forgets to override send_message(), Python raises an error when you try to instantiate it. Think of it as a contract: "any channel must know how to send a message."

3. @classmethod — Method that Receives the Class, Not an Instance
File: app/Internal/Model/shedding_type.py

python
@classmethod
def validate_enable_for(cls, v):
    if isinstance(v, str):
        return [v]
    return v
What it does:
Makes the method receive the class itself as the first argument (cls) instead of an instance (self). It can be called without creating an object:
TenantConfig.validate_enable_for("snapp-ir").
In this project, it's used inside @field_validator (see #4 below) — Pydantic requires validators to be classmethods.

4. @field_validator — Pydantic Automatic Data Validation
File: app/Internal/Model/shedding_type.py

python
@field_validator('enable_for', mode='before')
@classmethod
def validate_enable_for(cls, v):
    if isinstance(v, str):
        return [v]
    return v
What it does:
Automatically called by Pydantic when the enable_for field is being set. The mode='before' means it runs before Pydantic's own type checking.
In this example: if someone passes a single string like "snapp-ir", it wraps it into a list ["snapp-ir"] so it matches the expected List[str] type.
Note the stacking: both @field_validator and @classmethod are applied (order matters, but here they work together).

5. @property — Makes a Method Look Like an Attribute
File: app/domain/incident/aggregate.py

python
class Incident:
    def __init__(self):
        self._active = False

    @property
    def is_active(self) -> bool:
        return self._active
What it does:
Lets you access the result as if it were an attribute, without calling it as a function:

python
incident = Incident()
print(incident.is_active)   # ✅ reads like a property — no parentheses
# NOT: incident.is_active()  # this would fail
It's a read‑only computed attribute. The method is_active runs behind the scenes, but to the caller it looks like a simple field access.

6. @retry — Automatically Retries on Failure
File: app/External/keycloak.py

python
@retry(
    stop=stop_after_attempt(10),
    wait=wait_exponential(multiplier=1, min=30, max=60),
    reraise=True,
)
def connect(self):
    ...
What it does:
If connect() throws an exception, it will automatically retry up to 10 times, with an exponentially increasing wait between attempts (starting at 30s, maxing at 60s). If all 10 attempts fail, it re‑raises the last error.
This comes from the tenacity library and is common in SRE/infrastructure code — if Keycloak is temporarily down, the app keeps trying instead of failing immediately.

7. @staticmethod — Method with No Access to Instance or Class
File: app/Utils/adaptive_card_parser.py

python
class AdaptiveCardParser:
    @staticmethod
    def to_markdown(card_data: dict | str) -> str:
        ...
What it does:
Marks a method that doesn't need self (no instance data) or cls (no class data). It's essentially a regular function that lives inside a class for organisational purposes. You can call it as:

AdaptiveCardParser.to_markdown(data) — without creating an instance

parser.to_markdown(data) — on an instance (it ignores self)
