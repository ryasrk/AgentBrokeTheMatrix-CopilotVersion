---
name: auth-patterns
description: Authentication and authorization patterns for JWT, OAuth2, RBAC, session management, API keys, and SSO across Python, Node.js, and Go backends.
---

# Authentication & Authorization Patterns

Secure patterns for implementing auth flows, token management, and permission systems.

## When to Activate

- Implementing login/signup flows (password, OAuth2, magic link)
- Managing JWT access and refresh tokens
- Designing role-based or attribute-based access control
- Securing API endpoints with middleware/guards
- Implementing API key authentication for services
- Setting up SSO or identity federation

## Core Principles

### 1. Defense in Depth

Layer multiple security controls — never rely on a single check.

```python
# Good: Multiple layers
@app.route("/api/admin/users")
@require_auth           # 1. Authenticated?
@require_role("admin")  # 2. Authorized?
@rate_limit(100)        # 3. Rate limited?
@audit_log              # 4. Logged?
async def list_users():
    ...
```

### 2. Principle of Least Privilege

Grant minimum necessary permissions. Default deny.

```python
# Good: Explicit permission grants
class Permission(str, Enum):
    PLATES_READ = "plates:read"
    PLATES_WRITE = "plates:write"
    PLATES_DELETE = "plates:delete"
    ADMIN_USERS = "admin:users"

ROLE_PERMISSIONS = {
    "viewer":  {Permission.PLATES_READ},
    "editor":  {Permission.PLATES_READ, Permission.PLATES_WRITE},
    "admin":   {p for p in Permission},  # All permissions
}
```

### 3. Secure Token Lifecycle

Short-lived access, long-lived refresh, rotation on use.

```
┌─── Login ──────────────────────────────────────────┐
│ Client sends credentials                            │
│ Server validates → issues access_token (15min)      │
│                  → issues refresh_token (30 days)   │
│                  → stores refresh_token hash in DB   │
└────────────────────────────────────────────────────┘

┌─── API Request ────────────────────────────────────┐
│ Client: Authorization: Bearer <access_token>        │
│ Server: validate signature, expiry, audience        │
└────────────────────────────────────────────────────┘

┌─── Token Refresh ──────────────────────────────────┐
│ Client sends expired access + refresh_token         │
│ Server: validate refresh_token against DB           │
│       → revoke old refresh_token (one-time use)     │
│       → issue new access_token + refresh_token      │
└────────────────────────────────────────────────────┘
```

## JWT Patterns

### Token Creation

```python
import jwt
from datetime import datetime, timedelta, timezone

def create_tokens(user_id: str, roles: list[str]) -> dict:
    """Create access and refresh token pair."""
    now = datetime.now(timezone.utc)

    access_payload = {
        "sub": user_id,
        "roles": roles,
        "type": "access",
        "iat": now,
        "exp": now + timedelta(minutes=15),
        "aud": "api",
        "iss": "my-app",
    }

    refresh_payload = {
        "sub": user_id,
        "type": "refresh",
        "jti": secrets.token_urlsafe(32),  # Unique ID for revocation
        "iat": now,
        "exp": now + timedelta(days=30),
    }

    access_token = jwt.encode(access_payload, ACCESS_SECRET, algorithm="HS256")
    refresh_token = jwt.encode(refresh_payload, REFRESH_SECRET, algorithm="HS256")

    return {"access_token": access_token, "refresh_token": refresh_token}
```

### Token Validation Middleware

```python
def require_auth(func):
    """Validate JWT and attach user to request."""
    @wraps(func)
    async def wrapper(request, *args, **kwargs):
        token = extract_bearer_token(request)
        if not token:
            raise HTTPException(401, "Missing authorization token")

        try:
            payload = jwt.decode(
                token, ACCESS_SECRET,
                algorithms=["HS256"],
                audience="api",
                issuer="my-app",
            )
        except jwt.ExpiredSignatureError:
            raise HTTPException(401, "Token expired")
        except jwt.InvalidTokenError:
            raise HTTPException(401, "Invalid token")

        request.state.user = await get_user(payload["sub"])
        return await func(request, *args, **kwargs)
    return wrapper
```

## Password Handling

```python
import bcrypt

# NEVER store plaintext passwords
# Use bcrypt with cost factor ≥ 12
def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt(rounds=12)).decode()

def verify_password(password: str, hashed: str) -> bool:
    return bcrypt.checkpw(password.encode("utf-8"), hashed.encode("utf-8"))

# Password requirements
def validate_password(password: str) -> list[str]:
    errors = []
    if len(password) < 8:
        errors.append("Password must be at least 8 characters")
    if not any(c.isupper() for c in password):
        errors.append("Must contain an uppercase letter")
    if not any(c.isdigit() for c in password):
        errors.append("Must contain a digit")
    return errors
```

## OAuth2 Integration

```python
# OAuth2 Authorization Code Flow
async def oauth2_callback(code: str, provider: str) -> dict:
    """Exchange authorization code for tokens."""
    config = OAUTH_PROVIDERS[provider]

    # Exchange code for provider tokens
    token_response = await http_client.post(config["token_url"], data={
        "grant_type": "authorization_code",
        "code": code,
        "redirect_uri": config["redirect_uri"],
        "client_id": config["client_id"],
        "client_secret": config["client_secret"],
    })

    # Get user profile from provider
    provider_tokens = token_response.json()
    profile = await http_client.get(
        config["userinfo_url"],
        headers={"Authorization": f"Bearer {provider_tokens['access_token']}"},
    )

    # Find or create local user
    user = await find_or_create_user(provider, profile.json())

    # Issue our own tokens
    return create_tokens(user.id, user.roles)
```

## RBAC Middleware

```python
def require_permission(permission: str):
    """Check user has specific permission."""
    def decorator(func):
        @wraps(func)
        async def wrapper(request, *args, **kwargs):
            user = request.state.user
            user_permissions = set()
            for role in user.roles:
                user_permissions |= ROLE_PERMISSIONS.get(role, set())

            if permission not in user_permissions:
                raise HTTPException(403, "Insufficient permissions")

            return await func(request, *args, **kwargs)
        return wrapper
    return decorator
```

## Cookie Security

```python
# Secure cookie settings for auth tokens
response.set_cookie(
    key="refresh_token",
    value=refresh_token,
    httponly=True,       # Not accessible via JavaScript
    secure=True,         # HTTPS only
    samesite="lax",      # CSRF protection
    max_age=30 * 24 * 3600,  # 30 days
    path="/api/auth",    # Restrict to auth endpoints only
)
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Tokens in localStorage | Use HttpOnly cookies for refresh tokens |
| JWT with `none` algorithm | Always specify algorithm in verification |
| Single token (no refresh) | Separate short-lived access + long-lived refresh |
| Plaintext password storage | bcrypt with cost ≥ 12 or argon2 |
| No brute-force protection | Rate limit + account lockout after N failures |
| Permission checks only in UI | Always enforce on server side |
| Shared secrets across environments | Unique keys per environment |
| No token revocation | Store refresh tokens in DB, support revocation |
