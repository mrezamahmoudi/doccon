# Domain-Driven Design: Where Does My Function Go?

In Domain-Driven Design (DDD), deciding where a specific function belongs is one of the most common challenges for a junior developer. The short answer is: where a function goes depends entirely on its responsibility and whether it contains business rules, coordination logic, or technical details.

Here is a practical "cheat sheet" based on DDD best practices to help you decide exactly where to put your code, followed by an explanation of how the Ports and Adapters (Hexagonal) architecture fits into the picture.

---

## The "Where Does My Function Go?" Cheat Sheet

When you have a new piece of logic to write, ask yourself what it does and match it to one of these building blocks:

### Scenario A: It changes or protects the state of a specific, identifiable business concept.
- **Where it goes:** An Entity.
- **Best Practice:** If an object is distinguished by its unique identity (like a `Customer` or a `Cargo`), put the logic that modifies its state directly inside that class. Do not just use dumb data holders with public "getters and setters". Design intention-revealing methods (e.g., `customer.makePreferred()`) rather than simply updating attributes.

### Scenario B: It computes, measures, or describes something without needing a unique identity.
- **Where it goes:** A Value Object.
- **Best Practice:** DDD highly favors Value Objects over Entities because they are easier to test and maintain. Put logic here if it can be written as a "Side-Effect-Free Function"—a function that calculates and returns a result without modifying the original object's state. For example, combining two paint colors to produce a new color is logic that belongs in a `Color` Value Object.

### Scenario C: It is a significant business process or calculation that involves multiple domain objects.
- **Where it goes:** A Domain Service.
- **Best Practice:** Eric Evans coined the rule: "Sometimes, it just isn't a thing". If a function represents a business rule, a transformation, or a calculation requiring input from multiple Aggregates/Entities, and it feels awkward to force it into one specific Entity, make it a Domain Service. Keep Domain Services stateless.

### Scenario D: It directs a use case, fetches data, or manages a database transaction, but contains NO business rules.
- **Where it goes:** An Application Service.
- **Best Practice:** Application Services are the direct clients of your domain model. They should be kept "thin". Their only job is to receive a request from the user interface, use a Repository to fetch the required domain objects, delegate the actual work to those domain objects (or Domain Services), and commit the database transaction. Never put business logic in an Application Service.

### Scenario E: It contains complex logic just for creating a new object.
- **Where it goes:** A Factory.
- **Best Practice:** Shift the responsibility for complex assembly into a separate Factory object or method to keep the Entity's constructor clean and to encapsulate the creation rules.

### Scenario F: It saves, searches for, or retrieves domain objects from a database.
- **Where it goes:** A Repository.
- **Best Practice:** A Repository gives the illusion of an in-memory collection of all objects of a certain type. Hide your SQL queries or database framework code behind a Repository interface so your business logic doesn't get tangled with database technology.

---

## How "Ports and Adapters" (Hexagonal Architecture) Solves Your Problem

You mentioned not knowing where to use Ports and Adapters. This pattern (also known as Hexagonal Architecture) is the perfect solution for organizing the building blocks listed above so that your core business logic is completely isolated from technical infrastructure like databases, UI, and external APIs.

In this architecture, your system is divided into two primary areas: **the Inside** and **the Outside**.

### The Inside (The Core)
- This is where your **Domain Model** (Entities, Value Objects, Domain Services) and your **Application Services** live.
- The "Inside" is designed purely based on your business use cases and functional requirements, completely ignorant of whether the application is a web app, a mobile app, or a background worker.

### The Ports (The Interfaces)
- The Inside defines "Ports" (interfaces) for anything it needs to communicate with:
    - **Input Ports:** Interfaces defining the use cases your application can perform (usually implemented by your Application Services).
    - **Output Ports:** Interfaces defining the things your application needs from the outside world (like a `CustomerRepository` interface defining how to save/fetch a customer).

### The Adapters (The Outside)
- This is where all the technical code lives. Adapters sit on the outside and translate messages between the outside world and your Inside Ports.
    - **Input Adapters:** A REST API controller, a web UI, or a RabbitMQ message listener. They take external requests and invoke your Application Services.
    - **Output Adapters:** A Hibernate/SQL database class that implements your `CustomerRepository` interface, or a class that sends emails.
