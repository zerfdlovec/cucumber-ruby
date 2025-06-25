
# Django Workspace Isolation Package ![](https://github.com/django-saas-tools/django-tenant-schemas/workflows/ci/badge.svg)

![Workspace Isolation Architecture](https://github.com/django-saas-tools/django-tenant-schemas/assets/8234567/f7a3e2b1-9c2f-4d8e-a6f2-1e5c6d8e9f0a)

The Workspace Isolation package for Django provides tenant-aware database routing for SaaS applications. Simplifies PostgreSQL schema-based multi-tenancy with transparent connection management and per-tenant migrations.

Core capabilities include runtime schema switching via middleware, separate model namespaces for shared and tenant-specific data, custom management commands for tenant operations, and independent migration handling per workspace.

Roadmap includes bulk migration execution across all tenant schemas and automatic schema provisioning on tenant creation.

### Supported Databases:
- PostgreSQL
- CockroachDB
- YugabyteDB

### Installation

Requirements:
- [Django](https://www.djangoproject.com/) v4.2+ 
- [psycopg](https://github.com/psycopg/psycopg) v3+

Install via pip:

```sh
$ pip install django-workspace-isolation
```

### Configuration Guide

###### Architecture: Separate PostgreSQL schemas for shared infrastructure and isolated tenant workspaces.

1. Define tenant configuration model implementing `WorkspaceConfigInterface` in shared schema.
2. Leverage `WorkspaceConfigMixin` for standard configuration fields.
3. Apply `TimestampMixin` for automatic timestamp tracking, requires `@receiver` decorators.
4. Organize models into `shared/` and `tenant/` directories.
5. Split migrations into `shared/` and `tenant/` directories.
6. Update Django settings for schema-aware model discovery:

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django_workspace_isolation.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        # ... other settings
    }
}

WORKSPACE_ISOLATION = {
    'TENANT_MODEL': 'shared.WorkspaceConfig',
    'TENANT_IDENTIFIER_FIELD': 'schema_name',
}

INSTALLED_APPS = [
    'django_workspace_isolation',
    'apps.shared',
    'apps.tenant',
    # ...
]
```

7. Inject `WorkspaceRouter` into views or services.
8. Trigger schema switch via `WorkspaceSwitchSignal` with tenant identifier.
9. `WorkspaceRouter` automatically routes queries to active schema.
10. Separate Django apps recommended for shared vs tenant models.
11. Execute schema-specific operations via custom management commands:

```bash
python manage.py create_tenant_schema <identifier>  # c:t:s for short

python manage.py makemigrations_tenant <identifier>  # m:m:t for short

python manage.py migrate_tenant <identifier>  # m:t for short
```

### Implementation Example

Reference implementation: [Django Multi-Workspace Demo](https://github.com/django-saas-tools/demo-project)

### Technical Notes

All tenant migrations execute independently from shared schema migrations via Django's migration framework database routing.

Tenant migration files stored in configured tenant app directory.

All tenant schemas inherit connection credentials from default database configuration. Per-tenant credential support planned for future release.

### Getting Started Workflow

1. Configure package per above instructions.
2. Generate shared schema migration: `python manage.py makemigrations`
3. Apply shared migrations: `python manage.py migrate`
4. Create tenant configuration record in shared schema.
5. Provision tenant schema: `python manage.py create_tenant_schema <identifier>`
6. Switch active schema via signal: `workspace_switch.send(sender=self.__class__, identifier=tenant.schema_name)`
7. Query tenant data through `WorkspaceRouter`.
8. Establish foreign key from user model to `WorkspaceConfig` for session-based routing.

```python
from django.views import View
from django_workspace_isolation.signals import workspace_switch
from django_workspace_isolation.routing import WorkspaceRouter

class WorkspaceAwareView(View):
    def __init__(self):
        self.shared_db = connections['default']
        self.tenant_router = WorkspaceRouter()
    
    def dispatch(self, request, *args, **kwargs):
        tenant = request.user.workspace_config
        workspace_switch.send(
            sender=self.__class__,
            identifier=tenant.schema_name
        )
        return super().dispatch(request, *args, **kwargs)
```

9. Execute schema-specific management commands with tenant identifier.
10. Maintain separate model evolution for shared and tenant schemas.

```python
from django.db import models
from django_workspace_isolation.models import WorkspaceConfigInterface, WorkspaceConfigMixin

class WorkspaceConfig(WorkspaceConfigMixin, models.Model):
    # Inherits schema_name, created_at, etc.
    pass

# Tenant model example
class TenantData(models.Model):
    # Automatically routed to active schema
    value = models.CharField(max_length=255)
    
    class Meta:
        app_label = 'tenant'
```

### Configuration Reference

Complete configuration in `settings.py`:

```python
WORKSPACE_ISOLATION = {
    'TENANT_MODEL': 'shared.WorkspaceConfig',
    'TENANT_IDENTIFIER_FIELD': 'schema_name',
    'PUBLIC_SCHEMA_NAME': 'public',  # Schema for shared tables
    'TENANT_APPS': [
        'apps.tenant',
    ],
    'SHARED_APPS': [
        'django.contrib.admin',
        'django.contrib.auth',
        'apps.shared',
    ],
}
```

### Design Patterns

#### Session-Based Routing

Store active tenant in session or User model for middleware-based automatic switching.

### Tenant Interface

Implement marker interface on tenant models for automated detection:

```python
from django_workspace_isolation.models import TenantModelInterface

class TenantResource(TenantModelInterface, models.Model):
    name = models.CharField(max_length=100)
    
    class Meta:
        app_label = 'tenant'
```

#### Custom Middleware

Override Django middleware for automatic schema resolution:

```python
from django_workspace_isolation.middleware import WorkspaceMiddleware

class AutoSwitchMiddleware(WorkspaceMiddleware):
    def process_request(self, request):
        if request.user.is_authenticated:
            tenant = request.user.workspace_config
            self.switch_schema(tenant.schema_name)
        return super().process_request(request)
```

#### Schema-Aware Serializers

Extend DRF serializers with tenant context validation:

```python
from rest_framework import serializers
from django_workspace_isolation.context import get_current_schema

class TenantSerializer(serializers.ModelSerializer):
    def validate(self, attrs):
        if not get_current_schema():
            raise serializers.ValidationError("No active workspace")
        return attrs
```

### Contribution Guidelines

- Fork repository
- Implement features with test coverage
- Submit pull request with description

## License

MIT License

**Open Source Infrastructure**

