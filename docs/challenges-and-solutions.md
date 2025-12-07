# Challenges & Solutions

## Challenge 1: Integrating Django and FastAPI on a Single Server

### Problem
Django and FastAPI are typically used separately. Integrating them requires:
- Sharing the same database and models
- Routing requests to the correct framework
- Initializing Django before FastAPI can use Django models
- Handling path prefixes correctly

### Solution
**Starlette ASGI Application as Router**

We created a Starlette application that mounts both frameworks:

```python
app = Starlette(
    routes=[
        Mount("/fastapi", app=fastapi_app),
        Mount("/api", app=DjangoAPIMount(django_app)),
        Mount("/", app=django_app),
    ]
)
```

**Key Implementation Details:**
1. Django initialized before FastAPI imports models
2. Custom `DjangoAPIMount` wrapper to restore path prefixes
3. Shared database connection via Django ORM
4. Separate URL namespaces prevent conflicts

### Result
- Both frameworks work seamlessly together
- Single server deployment
- Shared data model
- No data synchronization issues

## Challenge 2: Path Prefix Handling

### Problem
When Starlette mounts an app at `/api`, it strips the prefix from the request path. Django expects the full path including `/api` for its URL routing.

### Solution
**Custom ASGI Wrapper**

Created a wrapper that restores the prefix:

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

### Result
- Django receives requests with correct paths
- URL routing works as expected
- No need to modify Django URL patterns

## Challenge 3: Django Model Access from FastAPI

### Problem
FastAPI needs to access Django models, but Django must be initialized first. Importing models before Django setup causes errors.

### Solution
**Explicit Django Setup**

Initialize Django before importing models in FastAPI:

```python
# fastapi_app/api.py
import os
import django

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "codea_blog.settings.dev")
django.setup()  # Initialize Django

# Now safe to import Django models
from blog.models import BlogPostPage
```

### Result
- FastAPI can query Django models
- No import errors
- Shared database connection
- Consistent data access

## Challenge 4: StreamField Rendering in API

### Problem
Wagtail's StreamField stores content as structured data. FastAPI endpoints need to return HTML, requiring conversion from StreamField blocks to HTML.

### Solution
**Block Rendering Function**

Created a function that renders StreamField blocks to HTML:

```python
def render_body_to_html(p: BlogPostPage) -> str:
    """Safely render StreamField body to HTML"""
    try:
        if p.body:
            return "".join(block.render() for block in p.body)
        return ""
    except Exception:
        return f"<p>Content unavailable</p>"
```

### Result
- API returns properly formatted HTML
- Error handling for edge cases
- Consistent rendering across endpoints

## Challenge 5: CSRF Protection for API Endpoints

### Problem
Django's CSRF middleware blocks API requests that don't include CSRF tokens. API endpoints should be exempt from CSRF protection.

### Solution
**Custom CSRF Middleware**

Created middleware that exempts `/api` paths:

```python
class CsrfExemptApiMiddleware:
    def __call__(self, request):
        if request.path.startswith('/api/'):
            setattr(request, '_dont_enforce_csrf_checks', True)
        return self.get_response(request)
```

### Result
- API endpoints work without CSRF tokens
- Forms still protected by CSRF
- Clear separation between API and form endpoints

## Challenge 6: Rate Limiting Integration

### Problem
Need to integrate external rate limiting service without blocking requests when the service is unavailable.

### Solution
**Fail-Open Rate Limiting**

Implemented configurable rate limiting with fail-open mode:

```python
async def check_rate_limit(request: Request):
    if not ENABLE_RATE_LIMITING:
        return
    
    try:
        # Call rate limiter service
        response = await client.get(RATE_LIMITER_URL, ...)
        # Check rate limit
    except Exception:
        if not RATE_LIMITER_FAIL_OPEN:
            raise HTTPException(status_code=503)
        # Allow request if fail-open mode
```

**Configuration:**
- `ENABLE_RATE_LIMITING`: Toggle on/off
- `RATE_LIMITER_FAIL_OPEN`: Allow requests if service unavailable
- `RATE_LIMITER_TIMEOUT`: Request timeout

### Result
- API protection when rate limiter available
- Service continues if rate limiter unavailable (fail-open)
- Configurable per environment

## Challenge 7: Email Sending in Async Context

### Problem
Django's email sending requires explicit connection management in async FastAPI endpoints. Without proper connection handling, emails fail with "please run connect() first" errors.

### Solution
**Explicit Connection Management**

Open and close SMTP connections explicitly:

```python
if 'smtp' in email_backend.lower():
    connection = get_connection()
    connection.open()  # Explicitly open
    try:
        send_mail(..., connection=connection)
    finally:
        connection.close()  # Always close
```

### Result
- Emails send reliably
- Proper resource cleanup
- Works with both SMTP and console backends

## Challenge 8: Environment-Based Configuration

### Problem
Need different settings for development, staging, and production without code duplication.

### Solution
**Split Settings Structure**

Created separate settings files:

```
settings/
├── base.py      # Shared settings
├── dev.py       # Development (imports base, overrides)
├── production.py # Production (imports base, overrides)
└── stag.py      # Staging (imports base, overrides)
```

**Usage:**
```python
# Set via environment variable
DJANGO_SETTINGS_MODULE=codea_blog.settings.production
```

### Result
- Clean separation of concerns
- Easy to add new environments
- No code duplication
- Environment-specific overrides

## Challenge 9: Tagging System Integration

### Problem
Wagtail pages need tagging support, but django-taggit requires special handling for modelcluster (Wagtail's page system).

### Solution
**ClusterTaggableManager**

Used modelcluster's ClusterTaggableManager instead of standard taggit:

```python
from modelcluster.contrib.taggit import ClusterTaggableManager

class BlogPostPage(Page):
    tags = ClusterTaggableManager(through=BlogPostTag, blank=True)
```

### Result
- Tags work correctly with Wagtail pages
- Proper parent-child relationships
- Tag management in admin interface

## Challenge 10: Static Files in Production

### Problem
Django's development server doesn't serve static files efficiently in production. Need a production-ready solution.

### Solution
**WhiteNoise Integration**

Added WhiteNoise middleware for static file serving:

```python
MIDDLEWARE = [
    # ...
    'whitenoise.middleware.WhiteNoiseMiddleware',
    # ...
]

STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

### Result
- Efficient static file serving
- Compression and caching
- No need for separate web server
- Works with CDN

## Challenge 11: Docker Deployment

### Problem
Need consistent deployment across environments with proper initialization (migrations, static files).

### Solution
**Docker Entrypoint Script**

Created entrypoint script that runs initialization:

```bash
#!/bin/bash
# docker-entrypoint.sh

# Run migrations
python manage.py migrate --noinput

# Collect static files
python manage.py collectstatic --noinput

# Start server
exec "$@"
```

### Result
- Automatic migrations on container start
- Static files collected automatically
- Consistent deployment process
- Health checks configured

## Challenge 12: Request Logging

### Problem
Need to log all API requests for debugging and monitoring without cluttering code.

### Solution
**FastAPI Middleware**

Created custom logging middleware:

```python
class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start_time = time.time()
        logger.info(f"→ {request.method} {request.url.path}")
        
        response = await call_next(request)
        
        process_time = time.time() - start_time
        logger.info(f"← {request.method} {request.url.path} - Status: {response.status_code} - Time: {process_time:.3f}s")
        
        return response
```

### Result
- All requests logged automatically
- Response times tracked
- Easy debugging
- Performance monitoring

## Challenge 13: CORS Configuration

### Problem
API needs to be accessible from frontend applications on different domains.

### Solution
**CORS Middleware**

Configured CORS for both Django and FastAPI:

```python
# Django
CORS_ALLOW_ALL_ORIGINS = True  # Dev only

# FastAPI
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Restrict in production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Result
- Frontend can access API
- Configurable per environment
- Secure defaults for production

## Challenge 14: Health Check Endpoints

### Problem
Deployment platforms need health check endpoints to verify service availability.

### Solution
**Dedicated Health Check Endpoint**

Created simple health check endpoint:

```python
@app.get("/health")
async def health():
    return {"status": "ok", "service": "Codea Blog API"}
```

**Docker Health Check:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=30s \
    CMD curl -f http://localhost:${PORT:-8000}/fastapi/health || exit 1
```

### Result
- Deployment platforms can verify health
- Automatic container restart on failure
- Monitoring integration

## Challenge 15: Development Superuser Creation

### Problem
Creating superusers manually is tedious during development.

### Solution
**Management Command**

Created custom management command:

```python
# management/commands/create_dev_superuser.py
class Command(BaseCommand):
    def handle(self, *args, **options):
        User.objects.create_superuser('admin', 'admin@example.com', 'admin')
```

### Result
- Quick development setup
- Consistent test credentials
- One command to create admin user

