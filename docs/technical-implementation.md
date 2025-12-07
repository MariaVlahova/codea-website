# Technical Implementation Details

## Hybrid Framework Integration

### The Challenge

Integrating Django and FastAPI on a single server requires careful orchestration because:
1. Django needs to be initialized before FastAPI can access Django models
2. Both frameworks need to share the same database connection
3. Path routing must correctly direct requests to the appropriate framework
4. Middleware must be configured for both frameworks

### Solution: Starlette ASGI Application

We use Starlette as the top-level ASGI application that mounts both Django and FastAPI:

```python
# codea_blog/main.py
from starlette.applications import Starlette
from starlette.routing import Mount

app = Starlette(
    routes=[
        Mount("/fastapi", app=fastapi_app),
        Mount("/api", app=DjangoAPIMount(django_app)),
        Mount("/", app=django_app),
    ]
)
```

### Django Initialization

Django must be set up before FastAPI can import Django models:

```python
# fastapi_app/api.py
import os
import django

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "codea_blog.settings.dev")
django.setup()  # Initialize Django

# Now safe to import Django models
from blog.models import BlogPostPage
```

### Path Prefix Restoration

When Starlette mounts an app at a path prefix, it strips the prefix from the request path. For Django mounted at `/api`, we need to restore the prefix:

```python
class DjangoAPIMount:
    """Wrapper to restore /api prefix for Django when mounted at /api"""
    def __init__(self, app):
        self.app = app
    
    async def __call__(self, scope, receive, send):
        if scope["type"] == "http":
            path = scope["path"]
            full_path = f"/api{path}"
            new_scope = {**scope, "path": full_path, "raw_path": full_path.encode()}
            await self.app(new_scope, receive, send)
        else:
            await self.app(scope, receive, send)
```

## Content Management with Wagtail

### StreamField Implementation

Wagtail's StreamField allows flexible content composition. We define custom blocks:

```python
# blog/models.py
BODY_BLOCKS = [
    ("html", HtmlBlock(help_text="Paste raw HTML")),
    ("image", FigureBlock()),
    ("embed", EmbedBlock(help_text="YouTube/Vimeo URL")),
    ("video", VideoBlock()),
]

class BlogPostPage(Page):
    body = StreamField(BODY_BLOCKS, use_json_field=True)
```

### Custom Blocks

**FigureBlock** - Image with caption:
```python
class FigureBlock(blocks.StructBlock):
    image = ImageChooserBlock(required=True)
    caption = blocks.CharBlock(required=False, max_length=200)
    alt_text = blocks.CharBlock(required=False, max_length=160)
```

**VideoBlock** - Video from Wagtail Media:
```python
class VideoBlock(blocks.StructBlock):
    video = VideoChooserBlock(required=True)
```

### Rendering StreamField to HTML

FastAPI endpoints need to render StreamField content to HTML:

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

## FastAPI Endpoints

### Health Check Endpoint

Simple endpoint to verify API availability:

```python
@app.get("/health")
async def health(rate_limit: None = Depends(check_rate_limit)):
    """Health check endpoint for the API"""
    return {"status": "ok", "service": "Codea Blog API"}
```

### Blog Post Endpoints

**Get All Posts:**
```python
@app.get("/blogs", response_model=list[BlogPost])
async def get_all_posts(rate_limit: None = Depends(check_rate_limit)):
    """Get all published blog posts"""
    posts = BlogPostPage.objects.live().public().order_by("-date")
    return [
        BlogPost(
            id=p.id, title=p.title, slug=p.slug, date=str(p.date),
            body=render_body_to_html(p),
        ) for p in posts
    ]
```

**Get Post by Slug:**
```python
@app.get("/blogs/slug/{slug}", response_model=BlogPost)
async def get_post_by_slug(slug: str, rate_limit: None = Depends(check_rate_limit)):
    """Get a blog post by slug"""
    p = BlogPostPage.objects.live().public().filter(slug=slug).first()
    if not p:
        raise HTTPException(status_code=404, detail="Post not found")
    return BlogPost(id=p.id, title=p.title, slug=p.slug, date=str(p.date), 
                    body=render_body_to_html(p))
```

### Pydantic Models

Request/response validation using Pydantic:

```python
class BlogPost(BaseModel):
    id: int
    title: str
    slug: str
    date: str
    body: str  # HTML

class ContactFormRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: EmailStr
    subject: str = Field(..., min_length=1, max_length=200)
    message: str = Field(..., min_length=10, max_length=5000)
    phone: Optional[str] = Field(None, max_length=20)
```

## Rate Limiting Implementation

### External Rate Limiter Integration

Rate limiting is handled by an external service (Codea Auth Server):

```python
async def check_rate_limit(request: Request):
    """Dependency function to check rate limits"""
    if not ENABLE_RATE_LIMITING:
        return
    
    client_ip = request.client.host
    async with httpx.AsyncClient(timeout=RATE_LIMITER_TIMEOUT) as client:
        response = await client.get(
            RATE_LIMITER_URL,
            headers={"X-Forwarded-For": client_ip},
            params={"ip": client_ip}
        )
        
        if response.status_code == 200:
            data = response.json()
            remaining = data.get("current_usage", {}).get("remaining", "unlimited")
            
            if remaining != "unlimited" and int(remaining) <= 0:
                raise HTTPException(status_code=429, detail="Rate limit exceeded")
```

### Fail-Open vs Fail-Closed

Configurable behavior when rate limiter is unavailable:

```python
RATE_LIMITER_FAIL_OPEN = os.getenv("RATE_LIMITER_FAIL_OPEN", "true").lower() == "true"

if not RATE_LIMITER_FAIL_OPEN:
    raise HTTPException(status_code=503, detail="Rate limiter unavailable")
else:
    # Allow request to proceed
    pass
```

## Authentication Integration

### JWT Authentication Backend

Custom Django authentication backend for JWT tokens:

```python
# authentication/backends.py
class JWTAuthBackend(BaseBackend):
    def authenticate(self, request, token=None):
        if not token:
            return None
        
        # Validate token with external auth server
        # Return user if valid, None otherwise
```

### REST Framework Authentication

Multiple authentication classes:

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "authentication.authentication.JWTAuthentication",
        "authentication.authentication.GoogleOAuthAuthentication",
        "rest_framework.authentication.SessionAuthentication",
    ],
}
```

## Contact Form Implementation

### Email Sending

Contact form submissions send emails via Django's email backend:

```python
@app.post("/contact/", response_model=ContactFormResponse)
async def submit_contact_form(form: ContactFormRequest):
    email_subject = f"[Contact Form] {form.subject}"
    email_body = f"""
    New contact form submission:
    
    Name: {form.name}
    Email: {form.email}
    Phone: {form.phone or 'Not provided'}
    
    Message:
    {form.message}
    """
    
    # Handle SMTP connection properly
    if 'smtp' in email_backend.lower():
        connection = get_connection()
        connection.open()
        try:
            send_mail(
                subject=email_subject,
                message=email_body,
                from_email=settings.DEFAULT_FROM_EMAIL,
                recipient_list=[recipient_email],
                connection=connection,
            )
        finally:
            connection.close()
```

## Middleware Implementation

### Request Logging Middleware

Custom FastAPI middleware for request/response logging:

```python
class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: StarletteRequest, call_next):
        start_time = time.time()
        client_ip = request.client.host if request.client else "unknown"
        logger.info(f"→ {request.method} {request.url.path} - Client: {client_ip}")
        
        response = await call_next(request)
        
        process_time = time.time() - start_time
        logger.info(
            f"← {request.method} {request.url.path} - "
            f"Status: {response.status_code} - Time: {process_time:.3f}s"
        )
        
        return response
```

### CSRF Exemption Middleware

Django middleware that exempts API paths from CSRF:

```python
# codea_blog/middleware.py
class CsrfExemptApiMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.path.startswith('/api/'):
            setattr(request, '_dont_enforce_csrf_checks', True)
        return self.get_response(request)
```

## Database Models

### Blog Post Model

```python
class BlogPostPage(Page):
    date = models.DateField("Post date")
    hero_image = models.ForeignKey(Image, null=True, blank=True, on_delete=models.SET_NULL)
    body = StreamField(BODY_BLOCKS, use_json_field=True)
    categories = ParentalManyToManyField("blog.BlogCategory", blank=True)
    tags = ClusterTaggableManager(through=BlogPostTag, blank=True)
    
    parent_page_types = ["blog.BlogIndexPage"]
    subpage_types = []
```

### Category Snippet

```python
@register_snippet
class BlogCategory(models.Model):
    name = models.CharField(max_length=80, unique=True)
    slug = models.SlugField(max_length=90, unique=True, blank=True)
    
    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)
```

## Settings Configuration

### Environment-Based Settings

Split settings for different environments:

```python
# settings/base.py - Shared settings
INSTALLED_APPS = [...]
DATABASES = {...}

# settings/dev.py - Development overrides
from .base import *
DEBUG = True
DATABASES = {"default": {"ENGINE": "django.db.backends.sqlite3", ...}}

# settings/production.py - Production overrides
from .base import *
DEBUG = False
ALLOWED_HOSTS = ["your-domain.com"]
DATABASES = {"default": dj_database_url.parse(os.environ["DATABASE_URL"])}
```

## Docker Configuration

### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --upgrade pip && pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Create directories
RUN mkdir -p /app/staticfiles /app/media

# Health check
HEALTHCHECK --interval=30s --timeout=30s \
    CMD curl -f http://localhost:${PORT:-8000}/fastapi/health || exit 1

CMD ["uvicorn", "codea_blog.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Entrypoint Script

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

## Testing Strategy

### Django Tests

```python
# blog/tests.py
from django.test import TestCase
from blog.models import BlogPostPage

class BlogPostTestCase(TestCase):
    def test_blog_post_creation(self):
        post = BlogPostPage.objects.create(
            title="Test Post",
            slug="test-post",
            date="2024-01-01"
        )
        self.assertEqual(post.title, "Test Post")
```

### FastAPI Tests

```python
from fastapi.testclient import TestClient
from fastapi_app.api import app

client = TestClient(app)

def test_health_endpoint():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok", "service": "Codea Blog API"}
```

## Performance Optimizations

### Database Queries

Use `select_related` and `prefetch_related` to reduce queries:

```python
posts = BlogPostPage.objects.live().public().select_related('hero_image').order_by("-date")
```

### Caching

Wagtail's built-in caching for page rendering:

```python
# In production settings
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}
```

### Static Files

WhiteNoise for efficient static file serving:

```python
MIDDLEWARE = [
    # ...
    'whitenoise.middleware.WhiteNoiseMiddleware',
    # ...
]

STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

