# Professional Networking System

A Java and Spring Boot project for building a professional network where members can connect with colleagues, share career updates, and stay informed about activity in their network.

**Project stage: Planning and architecture design.** The implementation roadmap will guide development, and this README will track progress as milestones are completed.

The project aims to deepen practical understanding of backend engineering and distributed systems through secure APIs, clear service ownership, relational and graph data modeling, event delivery, automated testing, and reproducible operations.

## Planned capabilities

- **Member accounts:** registration, login, and authenticated profile access.
- **Professional connections:** send, accept, and reject requests; list first-degree connections.
- **Posts and media:** publish text updates with an optional image and read posts by author.
- **Engagement:** like and unlike posts with duplicate prevention.
- **In-app notifications:** receive updates when a connection publishes a post or another member likes a post.

## Architecture and technology

The design uses five business services—User, Posts, Connections, Notification, and Uploader—supported by API Gateway and Discovery Server. User Service will own authentication; each business service will control its own data and rules.

| Area | Planned technology |
| --- | --- |
| Backend | Java, Spring Boot, Spring Data, Spring Cloud OpenFeign |
| API access and discovery | Spring Cloud Gateway, Eureka |
| Security | Spring Security, JWT, password hashing |
| Persistence | PostgreSQL for accounts and content; Neo4j for connections |
| Event delivery | Apache Kafka |
| Media | Provider integration through Uploader Service; provider to be selected |
| Development and quality | Docker Compose, automated tests, GitHub Actions |
| Operational visibility | Health checks, structured logs, request IDs, and basic metrics |

Explore the [system architecture and request flows](docs/system-architecture.md) for the service layout and communication patterns.

## Implementation roadmap

Development will progress from accounts and authentication to connections, posts, and media. Kafka integration will add graph projections and notifications, followed by delivery reliability, local packaging, CI, discovery, and observability. Each phase has a demonstrable result and acceptance criteria in the [project plan and implementation roadmap](docs/project-plan-and-roadmap.md).

## Design documentation

- [Project Plan and Implementation Roadmap](docs/project-plan-and-roadmap.md)
- [System Architecture and Request Flows](docs/system-architecture.md)
- [API Specification](docs/api-specification.md)
- [Kafka Topics and Event Contracts](docs/kafka-topics-and-events.md)
- [Database Design and Data Ownership](docs/database-design.md)
