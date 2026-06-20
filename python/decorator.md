1. @router.get / @router.post / @router.put — FastAPI route decorators
File: app/api/routers/incident.py (and all other routers)


@router.get("/start_incident", status_code=status.HTTP_200_OK)
def start_incident(...):
    ...
What it does: Registers the function as an HTTP endpoint. When someone calls GET /start_incident, FastAPI will execute this function. The three variants map to HTTP methods:

Decorator	HTTP Method	Purpose
@router.get(...)	GET	Retrieve/read data
@router.post(...)	POST	Create new resources
@router.put(...)	PUT	Update existing resources
2. @abstractmethod — Forces subclasses to implement a method
File: app/External/channel.py


class Channel(ABC):
    @abstractmethod
    def send_message(self, message: str):
        pass
What it does: Declares a method that must be implemented by any class that inherits from Channel. If a subclass forgets to implement send_message(), Python raises an error when you try to create an instance. Think of it as a contract: "any channel must know how to send a message."

3. @classmethod — Method that receives the class, not an instance
File: app/Internal/Model/shedding_type.py


@classmethod
def validate_enable_for(cls, v):
    if isinstance(v, str):
        return [v]
    return v
What it does: Makes the method receive the class itself as the first argument (cls) instead of an instance (self). You can call it without creating an object: TenantConfig.validate_enable_for("snapp-ir"). In this project, it's used inside @field_validator (see #4 below) — Pydantic requires validators to be classmethods.

4. @field_validator — Pydantic automatic data validation
File: app/Internal/Model/shedding_type.py


@field_validator('enable_for', mode='before')
@classmethod
def validate_enable_for(cls, v):
    if isinstance(v, str):
        return [v]
    return v
What it does: Automatically called by Pydantic when the enable_for field is being set. The mode='before' means it runs before Pydantic's own type checking. In this case: if someone passes a single string like "snapp-ir", it wraps it into a list ["snapp-ir"] so it matches the expected List[str] type. Note it's stacked on top of @classmethod — both decorators are applied.

5. @property — Makes a method look like an attribute
File: app/domain/incident/aggregate.py


class Incident:
    def __init__(self):
        self._active = False

    @property
    def is_active(self) -> bool:
        return self._active
What it does: Lets you access the result like an attribute instead of calling it as a function:


incident = Incident()
print(incident.is_active)   # ✅ reads like a property — no parentheses
# NOT: incident.is_active()  # this would fail
It's a read-only computed attribute. The method is_active runs behind the scenes, but to the caller it looks like a simple field access.

6. @retry — Automatically retries on failure
File: app/External/keycloak.py


@retry(
    stop=stop_after_attempt(10),
    wait=wait_exponential(multiplier=1, min=30, max=60),
    reraise=True,
)
def connect(self):
    ...
What it does: If connect() throws an exception, it will automatically retry up to 10 times, with an exponentially increasing wait between attempts (starting at 30s, maxing at 60s). If all 10 attempts fail, it re-raises the last error. This is from the tenacity library and is very common in SRE/infrastructure code — if Keycloak is temporarily down, the app keeps trying instead of failing immediately.

7. @staticmethod — Method with no access to instance or class
File: app/Utils/adaptive_card_parser.py


class AdaptiveCardParser:
    @staticmethod
    def to_markdown(card_data: dict | str) -> str:
        ...
What it does: Marks a method that doesn't need self (no instance data) or cls (no class data). It's just a regular function that lives inside a class for organizational purposes. You can call it as:

AdaptiveCardParser.to_markdown(data) — without creating an instance
parser.to_markdown(data) — on an instance (it ignores self)
