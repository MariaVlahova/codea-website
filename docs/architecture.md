# Architecture Overview

## System Architecture

The Codea Blog project implements a **hybrid architecture** that combines Django/Wagtail CMS with FastAPI, running on a single ASGI server. This approach provides the best of both worlds: powerful content management capabilities and modern, high-performance API endpoints.

## Core Components

### 1. ASGI Application Layer

The entry point is `codea_blog/main.py`, which creates a Starlette application that mounts three sub-applications:

```python
app = Starlette(
    routes=[
        Mount("/fastapi", app=fastapi_app),           # FastAPI at /fastapi/*
        Mount("/api", app=DjangoAPIMount(django_app)), # Django/Wagtail API at /api/*
        Mount("/", app=django_app),                    # Django at root
    ]
)
```

**Key Design Decisions:**
- **Single Server**: Both frameworks run on the same server, reducing infrastructure complexity
- **Path-based Routing**: Different URL prefixes route to different frameworks
- **Shared Database**: Both frameworks access the same Django models and database

### 2. Django/Wagtail Layer

**Purpose**: Content management and admin interface

**Components:**
- **Wagtail CMS**: Provides the admin interface for content editors
- **Django ORM**: Manages database models and relationships
- **Django REST Framework**: Provides additional API capabilities (Wagtail API v2)
- **Django Apps**:
  - `home`: Homepage and welcome pages
  - `blog`: Blog post models and templates
  - `authentication`: JWT and external auth integration
  - `search`: Search functionality

**URL Structure:**
- `/cms/`: Wagtail admin interface
- `/cms/blog/`: Public blog index page
- `/api/`: Wagtail REST API endpoints

### 3. FastAPI Layer

**Purpose**: Modern REST API with automatic documentation

**Components:**
- **FastAPI Application**: High-performance async API framework
- **Pydantic Models**: Request/response validation
- **Dependency Injection**: Rate limiting, authentication
- **OpenAPI/Swagger**: Auto-generated API documentation

**URL Structure:**
- `/fastapi/docs`: Swagger UI documentation
- `/fastapi/redoc`: ReDoc documentation
- `/fastapi/health`: Health check endpoint
- `/fastapi/blogs`: Blog post endpoints
- `/fastapi/contact/`: Contact form submission

### 4. Database Layer

**Development**: SQLite (default)
**Production**: PostgreSQL (recommended)

**Models:**
- `BlogPostPage`: Individual blog posts with StreamField body
- `BlogIndexPage`: Blog listing page
- `BlogCategory`: Categorization snippets
- `BlogPostTag`: Tagging system using django-taggit

## Data Flow

### Content Creation Flow

```
1. Editor logs into Wagtail Admin (/cms/)
2. Creates/edits blog post using StreamField
3. Publishes post (saves to database)
4. Post becomes available via:
   - Wagtail templates (Django views)
   - Wagtail API (/api/)
   - FastAPI endpoints (/fastapi/blogs)
```

### API Request Flow

```
1. Client sends request to /fastapi/blogs
2. Starlette routes to FastAPI app
3. FastAPI dependency checks rate limit (if enabled)
4. FastAPI endpoint queries Django ORM (BlogPostPage.objects)
5. Response serialized using Pydantic models
6. JSON response returned to client
```

## Integration Points

### Django Setup in FastAPI

FastAPI needs to access Django models, so Django must be initialized before FastAPI imports:

```python
# In fastapi_app/api.py
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "codea_blog.settings.dev")
django.setup()  # Initialize Django

from blog.models import BlogPostPage  # Now safe to import
```

### Path Prefix Handling

When mounting Django at `/api`, Starlette strips the prefix. A custom wrapper restores it:

```python
class DjangoAPIMount:
    """Wrapper to restore /api prefix for Django when mounted at /api"""
    async def __call__(self, scope, receive, send):
        if scope["type"] == "http":
            path = scope["path"]
            full_path = f"/api{path}"
            new_scope = {**scope, "path": full_path, "raw_path": full_path.encode()}
            await self.app(new_scope, receive, send)
```

## Middleware Stack

### Django Middleware
1. CORS Middleware
2. Security Middleware
3. Session Middleware
4. CSRF Middleware (exempts `/api` paths)
5. Authentication Middleware (JWT support)
6. Message Middleware
7. Redirect Middleware

### FastAPI Middleware
1. Request Logging Middleware
2. CORS Middleware

## Authentication Architecture

### Multi-Backend Support

Django supports multiple authentication backends:

1. **JWT Auth Backend**: Validates JWT tokens from external auth server
2. **Remote Auth Backend**: Username/password validation via external server
3. **Model Backend**: Fallback for local Django users

### REST Framework Authentication

- JWT Authentication (token-based)
- Google OAuth Authentication
- Session Authentication (for browsable API)

## Static Files & Media

### Static Files
- **Development**: Served by Django's development server
- **Production**: Collected via `collectstatic` and served by WhiteNoise

### Media Files
- **Storage**: FileSystemStorage (configurable)
- **URL**: `/media/`
- **Root**: `BASE_DIR/media/`

## Environment Configuration

### Settings Structure

```
codea_blog/settings/
├── base.py      # Shared settings
├── dev.py       # Development overrides
├── production.py # Production overrides
└── stag.py      # Staging overrides
```

**Key Settings:**
- `DJANGO_SETTINGS_MODULE`: Environment variable determines which settings to use
- Database configuration: SQLite (dev) vs PostgreSQL (prod)
- Debug mode: Enabled in dev, disabled in prod
- Allowed hosts: Restricted in production

## Scalability Considerations

### Current Architecture
- **Single Server**: All components run on one server
- **Synchronous Django**: Traditional Django request handling
- **Async FastAPI**: FastAPI endpoints are async-capable

### Future Scalability Options

1. **Horizontal Scaling**: Deploy multiple instances behind a load balancer
2. **Database Scaling**: Use read replicas for read-heavy operations
3. **Caching**: Add Redis for caching frequently accessed content
4. **CDN**: Use CDN for static files and media
5. **Microservices**: Split FastAPI and Django into separate services (if needed)

## Security Architecture

### CSRF Protection
- Enabled for Django forms
- Exempted for `/api` paths (API endpoints)

### CORS Configuration
- Development: Allow all origins
- Production: Restrict to specific domains

### Rate Limiting
- Optional external rate limiting service
- Fail-open mode (allows requests if service unavailable)
- Configurable per endpoint via FastAPI dependencies

### Authentication
- JWT token validation
- External auth server integration
- Session-based auth for admin interface

## Deployment Architecture

### Docker Deployment

```
┌─────────────────────────────────┐
│      Docker Container           │
│  ┌───────────────────────────┐  │
│  │   Uvicorn ASGI Server     │  │
│  │  ┌─────────────────────┐  │  │
│  │  │  Starlette App       │  │  │
│  │  │  (Django + FastAPI)  │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │   PostgreSQL Database     │  │
│  │   (External or Internal)  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### Production Considerations

- **Database**: External PostgreSQL instance
- **Static Files**: Collected and served via WhiteNoise or CDN
- **Media Files**: Stored in persistent volume or cloud storage
- **Environment Variables**: Configured via deployment platform
- **Health Checks**: Built-in health check endpoints

## Performance Characteristics

### Django/Wagtail
- **Synchronous**: Traditional request/response cycle
- **ORM**: Efficient database queries with select_related/prefetch_related
- **Caching**: Wagtail's built-in caching for page rendering

### FastAPI
- **Async**: Non-blocking request handling
- **Pydantic**: Fast serialization/deserialization
- **Type Hints**: Better performance and developer experience

### Combined
- **Shared Database**: Both frameworks query the same database
- **No Duplication**: Single source of truth for content
- **Efficient**: Minimal overhead from framework integration

