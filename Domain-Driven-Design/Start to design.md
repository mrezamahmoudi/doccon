Step 1: Start from the Domain — What Does the System Do?
Don't think about databases, APIs, or frameworks yet. Ask:

"If this system had no technology at all — no HTTP, no database, no Teams — what rules would still exist?"

For Kalantar, those rules are:

An incident is either active or inactive
Starting an incident means it becomes active
You can't start an incident that's already active (maybe)
During an active incident, don't update the oncall group
Those are pure business rules. They belong in the domain.

How to find your domain concepts:
Write down every noun and verb in your problem:

Nouns (things)	Verbs (actions)
Incident	start, end
Oncall member	look up
Notification	send
Chat group	update, rename
Postmortem	create
Now ask for each: "Is this a thing that holds state and enforces rules?" or "Is this an action that coordinates multiple things?"


Holds state + enforces rules  →  Domain Aggregate (class)
  Example: Incident (tracks active/inactive)

Coordinates multiple things    →  Application Command (class)
  Example: StartIncident (calls incident.start() + notification.send() + oncall.get())

Talks to external system       →  Infrastructure Adapter (class)
  Example: IncidentNotificationService (wraps Teams/Element/Mattermost)

Pure data, no behavior         →  Value Object or DTO (dataclass)
  Example: OncallMember(name, role)
Step 2: Draw the Dependency Graph on Paper
Before coding, sketch who calls whom:


User clicks "Start Incident"
       │
       ▼
   API Router (HTTP concerns only)
       │
       ▼
   StartIncident.execute()  ← application command
       │
       ├──► Incident.start()           ← domain aggregate
       ├──► notification.send(...)     ← port (abstract)
       └──► oncall.get_members(...)    ← port (abstract)
       
   Infrastructure implements the ports:
       IncidentNotificationService(NotificationPort)
       OncallAdapter(OncallPort)
If your diagram has a line going from the domain outward (e.g., Incident calling TeamsService), that's a violation. The domain should not know about infrastructure.

Step 3: Identify Boundaries — What Should Be a Separate Class?
Use these decision rules:

Rule 1: External system → Always a separate adapter
Every time your code talks to something outside your process, wrap it:

External system	Adapter class
MS Teams API	TeamsNotificationAdapter
PostgreSQL	PostgresIncidentRepo
oncall.snapp.ir	OncallAdapter
Prometheus	PrometheusMetricsAdapter
Kubernetes API	KubernetesAdapter
Why? Because you want to test your business logic without hitting real systems. The adapter is the only place that imports the external library.

Rule 2: Business rule that can be stated as a sentence → Domain entity or value object

"An incident can't be ended if it was never started"     → Incident entity
"A city can only be migrated if it's in the source region" → CityMigration service
"Load shedding applies per tenant"                        → SheddingManager
If you can describe the rule without mentioning any technology, it's domain.

Rule 3: "Do A, then B, then C" → Application command
When an action requires orchestrating multiple steps across different systems, that's a command:


class MigrateCities:
    """Coordinates:
    1. Validate cities are in source region      ← domain rule
    2. Call Kubernetes to migrate                ← infrastructure
    3. Send notification                         ← infrastructure
    4. Record metrics                            ← infrastructure
    """
Rule 4: "Just read some data" → Query
Separate reads from writes:


class GetIncidentMode:
    def execute(self) -> bool:
        return self._incident.is_active

class GetSheddingConfig:
    def execute(self) -> dict:
        return self._shedding.get_config()
Rule 5: If it's just data transformation → Simple function
Not everything needs to be a class:


# This is fine as a function — no dependencies, no orchestration
def format_oncall_display_name(member: dict) -> str:
    name = member.get("name", "")
    roles = [r.strip() for r in member.get("role", "").split(",")]
    return f"{name} ({', '.join(roles)})" if roles else name
Step 4: The Layer Template
For any new feature, you create files in this structure:


app/
├── domain/
│   └── <feature>/
│       ├── aggregate.py      # State + business rules (pure Python)
│       ├── ports.py           # Abstract interfaces for what the domain needs
│       └── service.py         # Domain services (pure logic, no I/O)
│
├── application/
│   └── <feature>/
│       ├── commands.py        # Write operations (orchestration)
│       ├── queries.py         # Read operations
│       └── tasks.py           # Background/periodic tasks
│
├── infrastructure/
│   └── <feature>/
│       ├── notification_adapter.py   # Implements NotificationPort
│       ├── repository_adapter.py     # Implements RepositoryPort
│       └── ...                       # Other adapters
│
└── api/
    └── routers/
        └── <feature>.py      # HTTP endpoints (thin)
Step 5: Concrete Example — Designing a New Feature
Let's say you're adding "Automated Health Check" — a system that periodically checks if services are healthy and alerts when they're not.

5a. Write the rules in plain English

- A health check runs every 5 minutes
- It checks a list of services
- Each service is either healthy or unhealthy
- If a service is unhealthy for 3 consecutive checks → trigger alert
- An alert sends a notification to the SRE channel
- During an active incident, health check alerts are suppressed
5b. Extract domain concepts

Nouns:  ServiceHealth, HealthCheckResult, Alert
Verbs:  check, evaluate, alert, suppress

Domain aggregate:  ServiceHealth (tracks consecutive failures)
Domain port:       HealthCheckPort (abstract — "check if service X is healthy")
Domain port:       NotificationPort (reuse existing!)
Domain rule:       "Don't alert during active incident"
5c. Sketch the layers

# domain/health/aggregate.py
class ServiceHealth:
    def __init__(self, name: str):
        self._name = name
        self._consecutive_failures = 0
    
    def record_success(self):
        self._consecutive_failures = 0
    
    def record_failure(self):
        self._consecutive_failures += 1
    
    @property
    def needs_alert(self) -> bool:
        return self._consecutive_failures >= 3

# domain/health/ports.py
class HealthCheckPort(ABC):
    @abstractmethod
    def check(self, service_url: str) -> bool: ...

# application/health/commands.py
class RunHealthChecks:
    def __init__(self, services: list, health_check: HealthCheckPort, 
                 notification: NotificationPort, incident: Incident):
        self._services = services
        self._health_check = health_check
        self._notification = notification
        self._incident = incident
    
    async def execute(self):
        if self._incident.is_active:
            return  # suppress during incidents
        
        for service in self._services:
            healthy = self._health_check.check(service.url)
            if healthy:
                service.record_success()
            else:
                service.record_failure()
                if service.needs_alert:
                    await self._notification.send_alert(...)

# infrastructure/health/health_check_adapter.py
class HttpHealthCheckAdapter(HealthCheckPort):
    def check(self, service_url: str) -> bool:
        response = requests.get(f"{service_url}/health", timeout=5)
        return response.status_code == 200
5d. Wire it up in the composition root

# In api.py __init__:
self.health_check_adapter = HttpHealthCheckAdapter()
self.run_health_checks_cmd = RunHealthChecks(
    services=[...],
    health_check=self.health_check_adapter,
    notification=self.incident_notification,  # reuse!
    incident=self.incident,                    # reuse!
)
Summary: The Checklist
When designing a new feature, go through this:

Step	Question	Output
1	What are the business rules (no tech mentioned)?	Domain concepts
2	What external systems are involved?	List of ports needed
3	What orchestration is needed?	Command/Query classes
4	Who calls whom?	Dependency diagram
5	Where does it wire up?	Composition root additions
And the golden rule:

The domain should be importable and testable without installing any external library. If you import requests or import sqlalchemy in a domain file, something is in the wrong place.
