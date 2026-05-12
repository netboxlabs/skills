# Authentication Backends Guide

NetBox supports multiple auth backends tried in order. First success wins.

## Local Authentication (Default)

Built-in Django authentication with NetBox's `ObjectPermissionMixin`. Configure password policy via `AUTH_PASSWORD_VALIDATORS` in `configuration.py`.

Default (4.5): 12-character minimum + alphanumeric requirement.

## LDAP

Requires `django-auth-ldap`:

```bash
echo "django-auth-ldap" >> /opt/netbox/local_requirements.txt
pip install django-auth-ldap
```

Configure in a **separate file** — `ldap_config.py` alongside `configuration.py`:

```python
import ldap
from django_auth_ldap.config import LDAPSearch, GroupOfNamesType

AUTH_LDAP_SERVER_URI = "ldaps://ldap.example.com"
AUTH_LDAP_BIND_DN = "cn=netbox,ou=services,dc=example,dc=com"
AUTH_LDAP_BIND_PASSWORD = "secret"
AUTH_LDAP_USER_SEARCH = LDAPSearch(
    "ou=users,dc=example,dc=com", ldap.SCOPE_SUBTREE, "(uid=%(user)s)"
)
AUTH_LDAP_USER_ATTR_MAP = {
    "first_name": "givenName",
    "last_name": "sn",
    "email": "mail",
}

# Group mapping
AUTH_LDAP_GROUP_SEARCH = LDAPSearch(
    "ou=groups,dc=example,dc=com", ldap.SCOPE_SUBTREE, "(objectClass=groupOfNames)"
)
AUTH_LDAP_GROUP_TYPE = GroupOfNamesType()
AUTH_LDAP_MIRROR_GROUPS = True
```

Enable in `configuration.py`:

```python
REMOTE_AUTH_ENABLED = True
REMOTE_AUTH_BACKEND = 'netbox.authentication.LDAPBackend'
```

**TLS options**: `LDAP_IGNORE_CERT_ERRORS`, `LDAP_CA_CERT_DIR`, `LDAP_CA_CERT_FILE`

## Remote User (Header-Based / Proxy Auth)

For reverse proxy authentication (nginx, Apache, Authelia, etc.):

```python
REMOTE_AUTH_ENABLED = True
REMOTE_AUTH_BACKEND = 'netbox.authentication.RemoteUserBackend'
REMOTE_AUTH_HEADER = 'HTTP_REMOTE_USER'
REMOTE_AUTH_AUTO_CREATE_USER = True
REMOTE_AUTH_DEFAULT_GROUPS = ['NetworkTeam']
```

Features:
- Group sync via `REMOTE_AUTH_GROUP_SYNC_ENABLED` + `REMOTE_AUTH_GROUP_HEADER`
- Superuser promotion via `REMOTE_AUTH_SUPERUSER_GROUPS`
- Email/name population via `REMOTE_AUTH_USER_EMAIL`, `_FIRST_NAME`, `_LAST_NAME`

> **⚠️ Gunicorn v22.0+** silently drops HTTP headers containing underscores. Add to `gunicorn.py`:
> ```python
> header_map = "dangerous"
> ```

## Social Auth (SAML / OIDC / OAuth2)

Uses `python-social-auth`. All `SOCIAL_AUTH_*` settings go in `configuration.py`.

### OIDC Example (Keycloak)

```python
REMOTE_AUTH_ENABLED = True
REMOTE_AUTH_BACKEND = 'social_core.backends.keycloak.KeycloakOAuth2'

SOCIAL_AUTH_KEYCLOAK_KEY = 'netbox'
SOCIAL_AUTH_KEYCLOAK_SECRET = 'your-client-secret'
SOCIAL_AUTH_KEYCLOAK_PUBLIC_KEY = 'your-realm-public-key'
SOCIAL_AUTH_KEYCLOAK_AUTHORIZATION_URL = 'https://keycloak.example.com/realms/myrealm/protocol/openid-connect/auth'
SOCIAL_AUTH_KEYCLOAK_ACCESS_TOKEN_URL = 'https://keycloak.example.com/realms/myrealm/protocol/openid-connect/token'
```

### Supported Providers (Built-in)

Azure AD (Entra ID), Google, GitHub (+ Enterprise/Team), GitLab, Okta, Keycloak, Auth0, Amazon, Apple, Salesforce, SAML (generic), OIDC (generic), and more.

### SAML Configuration

```python
REMOTE_AUTH_BACKEND = 'social_core.backends.saml.SAMLAuth'
SOCIAL_AUTH_SAML_SP_ENTITY_ID = 'https://netbox.example.com'
SOCIAL_AUTH_SAML_SP_PUBLIC_CERT = '...'
SOCIAL_AUTH_SAML_SP_PRIVATE_KEY = '...'
SOCIAL_AUTH_SAML_ENABLED_IDPS = {
    'corporate': {
        'entity_id': 'https://idp.example.com',
        'url': 'https://idp.example.com/sso',
        'x509cert': '...',
        'attr_user_permanent_id': 'name_id',
        'attr_email': 'email',
    }
}
```

### Custom Login Button Appearance

```python
SOCIAL_AUTH_BACKEND_ATTRS = {
    'keycloak': {
        'display_name': 'Corporate SSO',
        'icon_url': '/static/img/sso-icon.png',
    }
}
```

## Multiple Backends

Chain backends by passing a list:

```python
REMOTE_AUTH_BACKEND = [
    'netbox.authentication.LDAPBackend',
    'social_core.backends.keycloak.KeycloakOAuth2',
]
```

Backends are tried in order. Order matters — put the most common backend first.
