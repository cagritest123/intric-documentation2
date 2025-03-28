# Domain-Driven Design in Intric

This document outlines how Domain-Driven Design (DDD) principles are applied within the Intric platform, particularly for new feature development.

## Table of Contents
- [Introduction to DDD](#introduction-to-ddd)
- [Core DDD Concepts](#core-ddd-concepts)
- [Implementation in Intric](#implementation-in-intric)
- [Feature Structure](#feature-structure)
- [Development Guidelines](#development-guidelines)
- [Example Implementation](#example-implementation)
- [Common Patterns](#common-patterns)
- [Testing Approach](#testing-approach)

## Introduction to DDD

Domain-Driven Design is an approach to software development that focuses on creating a rich domain model that reflects the business processes and requirements. It prioritizes:

- **Understanding the core business domain**
- **Collaboration** between technical and domain experts
- Creating a **ubiquitous language** shared by all team members
- Focusing on the core domain and domain logic
- Maintaining a **model-driven design**

In Intric, we apply DDD principles to ensure our codebase directly reflects the business domain of AI-powered knowledge management systems.

## Core DDD Concepts

### Ubiquitous Language

A shared language between developers and domain experts that is used consistently in:
- **Code**: Class and method names
- **Documentation**: Technical and business documentation
- **Conversations**: Team discussions and requirements gathering

### Bounded Contexts

Explicit boundaries within which a particular domain model applies:

- Each bounded context has its own ubiquitous language
- Models across contexts may differ even for the same concept
- Contexts are integrated through defined relationships

> *In Intric, these often correspond to the top-level directories under `backend/src/intric/` like `assistants`, `spaces`, `users`*

### Entities and Value Objects

- **Entities**: Objects defined by their identity (e.g., Assistants, Spaces)
  - Have unique identifiers
  - Mutable over time
  - Lifecycle tracked
  
- **Value Objects**: Immutable objects defined by their attributes (e.g., Embedding, Prompt)
  - No identity of their own
  - Immutable
  - Replaceable rather than modifiable

### Aggregates

Clusters of domain objects treated as a single unit:

- Each aggregate has a **root entity**
- External references are only to the aggregate root
- Changes within the aggregate maintain consistency rules

### Repositories

Objects that provide collection-like access to aggregates:

- Abstract the underlying persistence mechanism (e.g., SQLAlchemy interactions)
- Provide methods to find and save aggregates

### Domain Services

Stateless operations that don't naturally belong to entities or value objects:

- Operate on multiple aggregates
- Implement complex domain processes

## Implementation in Intric

In Intric, we implement DDD with a focus on maintainability and clarity:

### Domain Layer

The core domain layer contains:

- **Domain entities and value objects** (e.g., `<domain>.py`)
- **Domain services** (interfaces or base classes if needed)
- **Repository interfaces** (often implicitly defined by `*_repo.py` usage)
- **Domain events** (if applicable)

### Application Layer

Coordinates domain objects to perform application tasks:

- **Application services** orchestrate domain objects (e.g., `*_service.py`)
- **DTOs** for transferring data (API models in `api/*_models.py`)
- **Event handlers** for domain events (if applicable)

### Infrastructure Layer

Implements technical concerns:

- **Repository implementations** (`*_repo.py` using SQLAlchemy)
- **External system integrations** (e.g., LLM clients)
- **Persistence mechanisms** (SQLAlchemy, Alembic)
- **Messaging systems** (ARQ via Redis)

### Interface Layer

Handles interaction with external systems:

- **API endpoints** (FastAPI routers in `api/*_router.py`)
- **User interface components** (Frontend SvelteKit app)
- **External service adapters** (e.g., under `ai_models/`)

## Feature Structure

For new features in Intric, we follow this standard structure (found within `backend/src/intric/`):

```
feature_x/
├── api/
│   ├── feature_x_models.py      # API schema definitions (Pydantic)
│   ├── feature_x_assembler.py   # Translates domain objects to API schema
│   └── feature_x_router.py      # API endpoints and routes (FastAPI)
├── feature_x.py                 # Main domain entity/object class
├── feature_x_repo.py            # Repository for persistence (SQLAlchemy)
├── feature_x_service.py         # Application service layer (Use Cases)
└── feature_x_factory.py         # Factory for creating domain objects (Optional but common)
```

### Component Responsibilities

#### Domain Entity (`feature_x.py`)

The main domain object containing state and business logic methods. Example Snippet:

```python
# Example based on intric/spaces/space.py
class Space:
    def __init__(self, id: UUID, name: str, tenant_id: UUID, /* ... */):
        # ... attributes ...
        self._members: list[SpaceMember] = []

    def add_member(self, member: SpaceMember) -> None:
        """Add a member to the space with validation."""
        if self._find_member_index(member.id) is not None:
            raise DomainError("User is already a member")
        self._members.append(member)
        # ... potentially raise domain events ...

    # ... other methods like change_member_role, remove_member, can_access ...
```

#### Repository (`feature_x_repo.py`)

Abstracts data persistence operations using SQLAlchemy. Example Snippet:

```python
# Example based on intric/spaces/space_repo.py
class SpaceRepository:
    def __init__(self, session: AsyncSession):
        self._session = session

    async def get(self, id: UUID) -> Space | None:
        """Retrieve a space by its ID."""
        stmt = select(SpacesTable).where(SpacesTable.id == id)
        # ... execute query, handle relationships, map to domain object ...
        db_space = (await self._session.execute(stmt)).scalar_one_or_none()
        return _map_to_domain(db_space) if db_space else None

    async def add(self, space: Space) -> Space:
        """Save a new space to the database."""
        db_space = _map_to_table(space)
        self._session.add(db_space)
        await self._session.flush() # Flush to get IDs if needed
        return space # Or return mapped object if needed

    # ... other methods like update, delete, list_by_member ...
```

#### Service (`feature_x_service.py`)

Coordinates domain objects, uses repositories, and implements application use cases. Example Snippet:

```python
# Example based on intric/spaces/space_service.py
class SpaceService:
    def __init__(
        self,
        user: UserInDB, # Injected current user
        factory: SpaceFactory,
        repo: SpaceRepository,
        user_repo: UsersRepository,
        # ... other dependencies ...
    ):
        # ... initialize dependencies ...

    async def create_space(self, name: str) -> Space:
        """Create a new space for the current user."""
        space = self.factory.create_space(name=name, user_id=self.user.id)
        space.tenant_id = self.user.tenant_id

        # ... add default models, etc. ...

        admin_member = SpaceMember(id=self.user.id, /* ... */ role=SpaceRoleValue.ADMIN)
        space.add_member(admin_member)

        return await self.repo.add(space)

    async def add_member(self, space_id: UUID, member_id: UUID, role: SpaceRoleValue) -> SpaceMember:
        """Add a member to a space, checking permissions."""
        space = await self.repo.get(space_id)
        # ... check permissions (e.g., using ActorManager/SpaceActor) ...
        # ... find user via user_repo ...
        # ... create SpaceMember object ...
        space.add_member(new_member)
        await self.repo.update(space)
        return new_member

    # ... other use cases like update_space, delete_space ...
```

#### Factory (`feature_x_factory.py`)

Creates properly initialized domain objects, often encapsulating default values or complex setup. Example Snippet:

```python
# Example based on intric/spaces/space_factory.py
class SpaceFactory:
    def create_space(self, name: str, user_id: UUID) -> Space:
        """Create a new space instance with defaults."""
        # ... generate UUID, set timestamps, default description ...
        return Space(id=new_uuid, name=name, user_id=user_id, /* ... */)
```

#### API Layer (`api/feature_x_*.py`)

Handles HTTP concerns (FastAPI routing), data validation (Pydantic models), and transformation between domain objects and API responses (Assemblers). Uses dependency injection (`@inject`, `Depends`, `Provide`) to get services. Example Snippet:

```python
# feature_x_models.py - Pydantic API models
class SpaceCreateRequest(BaseModel):
    name: str

class SpaceResponse(BaseModel):
    id: UUID
    name: str
    # ... other fields ...

# feature_x_assembler.py - Maps domain to API models
class SpaceAssembler:
    @staticmethod
    def to_response(space: Space) -> SpaceResponse:
        # ... map space attributes to SpaceResponse ...
        return SpaceResponse(id=space.id, name=space.name, /* ... */)

# feature_x_router.py - FastAPI endpoints
@router.post("/spaces", response_model=SpaceResponse, status_code=201)
@inject
async def create_space(
    space_data: SpaceCreateRequest,
    space_service: SpaceService = Depends(Provide[Container.space_service]),
    # current_user is injected into space_service via container/dependencies
):
    """API endpoint to create a new space."""
    created_space = await space_service.create_space(space_data.name)
    return SpaceAssembler.to_response(created_space)
```

## Development Guidelines

When implementing new features using DDD in Intric:

1.  **Start with the Domain**: 
    - Identify concepts, relationships, and the ubiquitous language first
    - Model the core entities and value objects

2.  **Follow Separation of Concerns**: 
    - Keep domain logic within domain entities
    - Use services for application logic and coordinating multiple domain objects
    - Keep persistence and infrastructure details out of the domain layer

3.  **Use Value Objects**: 
    - For attributes defining a concept (e.g., configuration settings, coordinates)
    - Make them immutable

4.  **Protect Aggregate Consistency**: 
    - Define clear boundaries (e.g., a `Space` aggregate might include its `Members`)
    - Ensure operations on the aggregate root maintain internal rules

5.  **Use Dependency Injection**:
    - Leverage FastAPI's built-in DI for request-scoped dependencies
    - Use the `dependency-injector` library for application-scoped components

## Example Implementation (Knowledge Base)

Applying DDD to a Knowledge Base feature:

```python
# Example: intric/knowledge/knowledge_base.py (Domain Entity)
class KnowledgeBase:
    # ... attributes: id, name, space_id ...
    _sources: list[KnowledgeSource] # Child objects

    def add_source(self, source: KnowledgeSource):
        # ... validation logic ...
        self._sources.append(source)

# Example: intric/knowledge/knowledge_base_repo.py (Repository)
class KnowledgeBaseRepository:
    async def get(self, kb_id: UUID) -> KnowledgeBase | None: ...
    async def save(self, kb: KnowledgeBase) -> KnowledgeBase: ...

# Example: intric/knowledge/knowledge_base_service.py (Service)
class KnowledgeBaseService:
    # ... depends on repo, maybe file_service ...
    async def add_file_source(self, kb_id: UUID, file_id: UUID):
        kb = await self.repo.get(kb_id)
        # ... permission checks ...
        file_info = await self.file_service.get_file(file_id)
        source = self.factory.create_file_source(file_info)
        kb.add_source(source)
        await self.repo.save(kb)
        # ... potentially trigger background job via task_service ...
```

## Common Patterns

### Repository Pattern
Abstracts data access using SQLAlchemy. Repositories handle mapping between domain entities and database table models (defined in `database/tables/`).

### Factory Pattern
Used to encapsulate the creation logic for domain entities, especially when setup is complex or involves defaults.

### Assembler Pattern
Used in the API layer (`api/*_assembler.py`) to convert domain entities into API response models (Pydantic) and sometimes vice-versa, decoupling the API representation from the internal domain model.

## Testing Approach

DDD facilitates focused testing:

1.  **Domain Tests**: 
    - Unit test domain entities and value objects in isolation
    - Focus on business logic and validation rules
    - Use `pytest` for testing

2.  **Application Tests**: 
    - Test application services (`*_service.py`) 
    - Mock dependencies (like repositories, other services)
    - Use `pytest` fixtures and mocking libraries
    - Verify services correctly orchestrate domain objects

3.  **Integration Tests**: 
    - Test interactions between components
    - May involve a real (test) database or external services

4.  **API/E2E Tests**: 
    - Test API endpoints using `pytest` with `httpx` or a test client
    - Consider Playwright for full end-to-end browser tests

By following these DDD principles, Intric aims for a maintainable, expressive codebase that aligns closely with the business domain.