---
name: graphql-patterns
description: GraphQL API design patterns including schema design, resolvers, DataLoader batching, caching, subscriptions, and security for Apollo Server, Strawberry, and Ariadne.
---

# GraphQL Development Patterns

Patterns for building performant, secure, and maintainable GraphQL APIs.

## When to Activate

- Designing GraphQL schemas (types, queries, mutations, subscriptions)
- Implementing resolvers with DataLoader for N+1 prevention
- Adding authentication/authorization to GraphQL
- Optimizing query complexity and depth limiting
- Building real-time features with subscriptions
- Migrating from REST to GraphQL or federation

## Core Principles

### 1. Schema-First Design

Design the schema from the client's perspective, not the database.

```graphql
# Good: Client-focused schema
type LicensePlate {
  id: ID!
  plateNumber: String!
  region: String!
  vehicle: Vehicle
  detections: DetectionConnection!
  lastSeen: DateTime
}

type Query {
  licensePlate(id: ID!): LicensePlate
  searchPlates(
    query: String!
    filter: PlateFilter
    first: Int = 20
    after: String
  ): PlateConnection!
}

type Mutation {
  registerPlate(input: RegisterPlateInput!): RegisterPlatePayload!
}
```

### 2. Relay-Style Pagination

Use connection pattern for all list fields.

```graphql
type PlateConnection {
  edges: [PlateEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PlateEdge {
  cursor: String!
  node: LicensePlate!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

### 3. Input Types for Mutations

```graphql
input RegisterPlateInput {
  plateNumber: String!
  region: String!
  vehicleId: ID
}

type RegisterPlatePayload {
  plate: LicensePlate
  errors: [UserError!]!
}

type UserError {
  field: String!
  message: String!
}
```

## DataLoader Pattern (N+1 Prevention)

```python
from dataloader import DataLoader

# Batch function: receives list of keys, returns list of results
async def batch_load_vehicles(vehicle_ids: list[str]) -> list[Vehicle | None]:
    vehicles = await db.query(Vehicle).filter(Vehicle.id.in_(vehicle_ids)).all()
    vehicle_map = {v.id: v for v in vehicles}
    return [vehicle_map.get(vid) for vid in vehicle_ids]

# Create per-request DataLoaders
def create_loaders():
    return {
        "vehicle": DataLoader(batch_load_vehicles),
        "detections": DataLoader(batch_load_detections),
    }

# In resolver
async def resolve_vehicle(plate, info):
    return await info.context["loaders"]["vehicle"].load(plate.vehicle_id)
```

## Security Patterns

### Query Depth & Complexity Limiting

```python
from graphql import validation

# Limit query depth to prevent deeply nested attacks
depth_limit = 10

# Complexity analysis: assign cost to each field
complexity_rules = {
    "Query.searchPlates": 10,        # List query base cost
    "LicensePlate.detections": 5,    # Nested list cost
    "default": 1,                     # Default field cost
}
max_complexity = 500
```

### Authorization

```python
from functools import wraps

def requires_permission(permission: str):
    def decorator(resolver_fn):
        @wraps(resolver_fn)
        async def wrapper(root, info, **kwargs):
            user = info.context.get("user")
            if not user or not user.has_permission(permission):
                raise PermissionError(f"Missing permission: {permission}")
            return await resolver_fn(root, info, **kwargs)
        return wrapper
    return decorator

@requires_permission("plates:read")
async def resolve_license_plate(root, info, id: str):
    return await plate_service.get_by_id(id)
```

## Error Handling

```python
# Structured errors in mutation payloads
async def resolve_register_plate(root, info, input: dict):
    errors = validate_plate_input(input)
    if errors:
        return {"plate": None, "errors": errors}

    try:
        plate = await plate_service.register(input)
        return {"plate": plate, "errors": []}
    except DuplicatePlateError:
        return {
            "plate": None,
            "errors": [{"field": "plateNumber", "message": "Plate already registered"}],
        }
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| No DataLoader | Always batch nested field resolution |
| No depth limiting | Set max depth (10) and complexity (500) |
| REST-style mutations (`createX`, `updateX`) | Use domain verbs (`registerPlate`, `markAsStolen`) |
| Exposing DB schema directly | Design schema from client perspective |
| No error types in mutations | Return structured error payloads |
| Single monolithic schema | Modularize by domain (federation or stitching) |
