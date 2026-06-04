# Authentication

Detailed reference for NetBox API authentication, token management, and access control.

## Token Formats

### v2 Tokens (Recommended — NetBox 4.5+)

```
Authorization: Bearer nbt_<key>.<token>
```

- `nbt_` prefix identifies NetBox tokens
- `<key>` is the public key identifier (used for lookup)
- `<token>` is the secret portion
- Only HMAC-SHA256 hash stored in database (with pepper)
- Requires `API_TOKEN_PEPPERS` in server config

### v1 Tokens (Legacy)

```
Authorization: Token 0123456789abcdef0123456789abcdef01234567
```

- 40-character hex string stored **in plaintext** in the database
- **Deprecated as of NetBox 4.6; removed in 5.0** — migrate before 5.0

### Migration Timeline

| NetBox Version | Status |
|----------------|--------|
| < 4.5.0 | v1 tokens only |
| 4.5.0 | v2 introduced, v1 fully supported |
| 4.6.0 | v1 deprecated |
| 5.0.0 | v1 removed |

## Server Configuration for v2 Tokens

v2 tokens require `API_TOKEN_PEPPERS` in `configuration.py`:

```python
API_TOKEN_PEPPERS = {
    'default': 'your-secret-pepper-value-minimum-32-chars',
}
```

Without this, v2 tokens cannot be validated.

## Migration Steps

1. Verify NetBox is on 4.5+
2. Confirm `API_TOKEN_PEPPERS` is configured
3. Generate new v2 token in NetBox UI
4. Update integration with new token and `Bearer` prefix
5. Test thoroughly
6. Revoke old v1 token

## pynetbox

pynetbox handles both formats automatically:

```python
import pynetbox
nb = pynetbox.api("https://netbox.example.com",
                  token="nbt_abc123.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx")
```

## Token Provisioning Endpoint

Bootstrap tokens programmatically without pre-existing API access:

```python
import requests

def provision_token(netbox_url, username, password, description=None):
    """Provision a new API token using credentials."""
    payload = {"username": username, "password": password}
    if description:
        payload["description"] = description

    response = requests.post(
        f"{netbox_url}/api/users/tokens/provision/",
        json=payload,
        headers={"Content-Type": "application/json"}
    )
    if response.status_code == 201:
        return response.json()["key"]
    else:
        raise Exception(f"Token provisioning failed: {response.text}")
```

Use cases: CI/CD bootstrapping, dynamic environment provisioning, token rotation automation.

> **NetBox 4.6.1+:** the plaintext token `key` is returned by the REST API **only once**, in the creation/provision response. Only a hash is stored afterward, so the full token is never retrievable again — capture and store `response.json()["key"]` immediately.

## IP Restrictions

Tokens can be restricted to specific IPs or CIDR ranges. Set `allowed_ips` on the token (e.g., `["10.0.0.0/8", "192.168.1.100/32"]`). Requests from other IPs receive 403.

## Read-Only Tokens

Create a user with read-only permissions, then generate a token for that user. Use for dashboards, monitoring, and exports.

## Best Practices Summary

| Practice | Priority |
|----------|----------|
| Use v2 tokens on NetBox 4.5+ | CRITICAL |
| Migrate v1 → v2 before 5.0 (v1 deprecated 4.6) | CRITICAL |
| Never store tokens in code | CRITICAL |
| Use environment variables | HIGH |
| Implement token rotation | HIGH |
| Apply IP restrictions in production | MEDIUM |
| Use read-only tokens when possible | MEDIUM |
