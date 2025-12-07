# Code Examples

## 1. Main Application Setup

### Starlette ASGI Application

```python
# codea_blog/main.py
import os, sys, django
from django.core.asgi import get_asgi_application
from django.conf import settings
from starlette.routing import Mount
from starlette.applications import Starlette

# Setup Django
THIS_DIR = os.path.dirname(__file__)
PARENT_DIR = os.path.dirname(THIS_DIR)
if PARENT_DIR not in sys.path:
    sys.path.insert(0, PARENT_DIR)

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "codea_blog.settings.dev")
django.setup()

# Import FastAPI app AFTER Django setup
from fastapi_app.api import app as fastapi_app

django_app = get_asgi_application()

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

app = Starlette(
    routes=[
        Mount("/fastapi", app=fastapi_app),
        Mount("/api", app=DjangoAPIMount(django_app)),
        Mount("/", app=django_app),
    ]
)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("codea_blog.main:app", host="127.0.0.1", port=8000, reload=True)
```

## 2. FastAPI Endpoints

### Blog Post Endpoints

```python
# fastapi_app/api.py
from fastapi import FastAPI, HTTPException, Request, Depends
from pydantic import BaseModel, EmailStr, Field
from typing import Optional
from blog.models import BlogPostPage

app = FastAPI(
    title="Codea Blog API",
    description="API for managing blog posts and content",
    version="1.0.0",
)

class BlogPost(BaseModel):
    id: int
    title: str
    slug: str
    date: str
    body: str  # HTML

def render_body_to_html(p: BlogPostPage) -> str:
    """Safely render StreamField body to HTML"""
    try:
        if p.body:
            return "".join(block.render() for block in p.body)
        return ""
    except Exception:
        return f"<p>Content unavailable</p>"

@app.get("/health")
async def health():
    """Health check endpoint"""
    return {"status": "ok", "service": "Codea Blog API"}

@app.get("/blogs", response_model=list[BlogPost])
async def get_all_posts():
    """Get all published blog posts"""
    posts = BlogPostPage.objects.live().public().order_by("-date")
    return [
        BlogPost(
            id=p.id,
            title=p.title,
            slug=p.slug,
            date=str(p.date),
            body=render_body_to_html(p),
        ) for p in posts
    ]

@app.get("/blogs/slug/{slug}", response_model=BlogPost)
async def get_post_by_slug(slug: str):
    """Get a blog post by slug"""
    p = BlogPostPage.objects.live().public().filter(slug=slug).first()
    if not p:
        raise HTTPException(status_code=404, detail="Post not found")
    return BlogPost(
        id=p.id,
        title=p.title,
        slug=p.slug,
        date=str(p.date),
        body=render_body_to_html(p)
    )
```

## 3. Contact Form Endpoint

```python
# fastapi_app/api.py
from django.core.mail import send_mail, get_connection
from django.conf import settings

class ContactFormRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: EmailStr
    subject: str = Field(..., min_length=1, max_length=200)
    message: str = Field(..., min_length=10, max_length=5000)
    phone: Optional[str] = Field(None, max_length=20)

class ContactFormResponse(BaseModel):
    success: bool
    message: str

@app.post("/contact/", response_model=ContactFormResponse)
async def submit_contact_form(form: ContactFormRequest):
    """Submit a contact form and send an email"""
    try:
        email_subject = f"[Contact Form] {form.subject}"
        email_body = f"""
New contact form submission:

Name: {form.name}
Email: {form.email}
Phone: {form.phone or 'Not provided'}

Subject: {form.subject}

Message:
{form.message}
"""
        recipient_email = getattr(
            settings, 'CONTACT_FORM_RECIPIENT',
            settings.EMAIL_HOST_USER or 'admin@example.com'
        )
        
        email_backend = getattr(
            settings, 'EMAIL_BACKEND',
            'django.core.mail.backends.console.EmailBackend'
        )
        
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
                    fail_silently=False,
                    connection=connection,
                )
            finally:
                connection.close()
        else:
            send_mail(
                subject=email_subject,
                message=email_body,
                from_email=settings.DEFAULT_FROM_EMAIL,
                recipient_list=[recipient_email],
                fail_silently=False,
            )
        
        return ContactFormResponse(
            success=True,
            message="Thank you for your message! We will get back to you soon."
        )
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Failed to send contact form. Error: {str(e)}"
        )
```

## 4. Rate Limiting Dependency

```python
# fastapi_app/api.py
import os
import httpx

ENABLE_RATE_LIMITING = os.getenv("ENABLE_RATE_LIMITING", "false").lower() == "true"
RATE_LIMITER_URL = os.getenv("RATE_LIMITER_URL", "https://codea-auth-server.onrender.com/api/limiter/")
RATE_LIMITER_TIMEOUT = float(os.getenv("RATE_LIMITER_TIMEOUT", "5.0"))
RATE_LIMITER_FAIL_OPEN = os.getenv("RATE_LIMITER_FAIL_OPEN", "true").lower() == "true"

async def check_rate_limit(request: Request):
    """Dependency function to check rate limits"""
    if not ENABLE_RATE_LIMITING:
        return
    
    try:
        client_ip = request.client.host if request.client else "127.0.0.1"
        forwarded_for = request.headers.get("X-Forwarded-For")
        if forwarded_for:
            client_ip = forwarded_for.split(",")[0].strip()
        
        async with httpx.AsyncClient(timeout=RATE_LIMITER_TIMEOUT) as client:
            response = await client.get(
                RATE_LIMITER_URL,
                headers={
                    "X-Forwarded-For": client_ip,
                    "X-Real-IP": client_ip,
                    "accept": "application/json"
                },
                params={"ip": client_ip} if client_ip else None
            )
            
            if response.status_code == 200:
                data = response.json()
                if data.get("rate_limit_enabled", False):
                    current_usage = data.get("current_usage", {})
                    remaining = current_usage.get("remaining", "unlimited")
                    
                    if remaining != "unlimited":
                        try:
                            remaining_int = int(remaining)
                            if remaining_int <= 0:
                                raise HTTPException(
                                    status_code=429,
                                    detail="Rate limit exceeded"
                                )
                        except (ValueError, TypeError):
                            pass
    except httpx.TimeoutException:
        if not RATE_LIMITER_FAIL_OPEN:
            raise HTTPException(status_code=503, detail="Rate limiter timeout")
    except HTTPException:
        raise
    except Exception as e:
        if not RATE_LIMITER_FAIL_OPEN:
            raise HTTPException(status_code=503, detail="Rate limiter error")

# Use in endpoints
@app.get("/blogs")
async def get_all_posts(rate_limit: None = Depends(check_rate_limit)):
    # ... endpoint code
```

## 5. Wagtail Models

### Blog Post Model

```python
# blog/models.py
from wagtail.models import Page
from wagtail.fields import StreamField
from wagtail import blocks
from wagtail.images.blocks import ImageChooserBlock
from wagtail.embeds.blocks import EmbedBlock
from wagtailmedia.blocks import VideoChooserBlock
from modelcluster.contrib.taggit import ClusterTaggableManager
from modelcluster.fields import ParentalManyToManyField

BODY_BLOCKS = [
    ("html", blocks.RawHTMLBlock()),
    ("image", FigureBlock()),
    ("embed", EmbedBlock()),
    ("video", VideoBlock()),
]

class BlogPostPage(Page):
    date = models.DateField("Post date")
    hero_image = models.ForeignKey(
        Image, null=True, blank=True, on_delete=models.SET_NULL, related_name="+"
    )
    body = StreamField(BODY_BLOCKS, use_json_field=True)
    categories = ParentalManyToManyField("blog.BlogCategory", blank=True)
    tags = ClusterTaggableManager(through=BlogPostTag, blank=True)
    
    parent_page_types = ["blog.BlogIndexPage"]
    subpage_types = []
    
    content_panels = Page.content_panels + [
        MultiFieldPanel(
            [FieldPanel("date"), FieldPanel("hero_image")],
            heading="Post info",
        ),
        FieldPanel("body"),
        MultiFieldPanel(
            [FieldPanel("categories"), FieldPanel("tags")],
            heading="Taxonomy",
        ),
    ]
```

### Custom Blocks

```python
# blog/models.py
class FigureBlock(blocks.StructBlock):
    image = ImageChooserBlock(required=True)
    caption = blocks.CharBlock(required=False, max_length=200)
    alt_text = blocks.CharBlock(required=False, max_length=160)
    
    class Meta:
        icon = "image"
        label = "Image"

class VideoBlock(blocks.StructBlock):
    video = VideoChooserBlock(required=True)
    
    class Meta:
        icon = "media"
        label = "Video"
```

## 6. Middleware Examples

### Request Logging Middleware

```python
# fastapi_app/api.py
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request as StarletteRequest
import time
import logging

logger = logging.getLogger("fastapi")

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

app.add_middleware(RequestLoggingMiddleware)
```

### CSRF Exemption Middleware

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

## 7. Settings Configuration

### Base Settings

```python
# codea_blog/settings/base.py
from pathlib import Path

PROJECT_DIR = Path(__file__).resolve().parent.parent
BASE_DIR = PROJECT_DIR.parent

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "wagtail.contrib.forms",
    "wagtail.embeds",
    "wagtail.sites",
    "wagtail.users",
    "wagtail.snippets",
    "wagtail.documents",
    "wagtail.images",
    "wagtail.search",
    "wagtail.admin",
    "wagtail",
    "rest_framework",
    "wagtail.api.v2",
    "wagtailmedia",
    "taggit",
    "modelcluster",
    "corsheaders",
    "home",
    "blog",
    "authentication",
]

MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "codea_blog.middleware.CsrfExemptApiMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "authentication.middleware.JWTAuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "wagtail.contrib.redirects.middleware.RedirectMiddleware",
]

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}
```

### Development Settings

```python
# codea_blog/settings/dev.py
from .base import *

DEBUG = True
ALLOWED_HOSTS = ["*"]

# Use SQLite for development
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}

# Console email backend for development
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"
```

### Production Settings

```python
# codea_blog/settings/production.py
from .base import *
import dj_database_url
import os

DEBUG = False
ALLOWED_HOSTS = os.environ.get("ALLOWED_HOSTS", "").split(",")

# Use PostgreSQL in production
DATABASES = {
    "default": dj_database_url.parse(os.environ["DATABASE_URL"])
}

# SMTP email backend
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = os.environ.get("EMAIL_HOST")
EMAIL_PORT = int(os.environ.get("EMAIL_PORT", 587))
EMAIL_USE_TLS = True
EMAIL_HOST_USER = os.environ.get("EMAIL_HOST_USER")
EMAIL_HOST_PASSWORD = os.environ.get("EMAIL_HOST_PASSWORD")
DEFAULT_FROM_EMAIL = os.environ.get("DEFAULT_FROM_EMAIL")
CONTACT_FORM_RECIPIENT = os.environ.get("CONTACT_FORM_RECIPIENT")
```

## 8. Docker Configuration

### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create necessary directories
RUN mkdir -p /app/staticfiles /app/media

# Set environment variables
ENV PYTHONPATH=/app
ENV PYTHONUNBUFFERED=1

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:${PORT:-8000}/fastapi/health || exit 1

# Use entrypoint script
ENTRYPOINT ["/app/docker-entrypoint.sh"]

# Default command
CMD ["uvicorn", "codea_blog.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Entrypoint Script

```bash
#!/bin/bash
# docker-entrypoint.sh

set -e

echo "Running migrations..."
python manage.py migrate --noinput

echo "Collecting static files..."
python manage.py collectstatic --noinput

echo "Starting server..."
exec "$@"
```

## 9. Management Commands

### Create Dev Superuser

```python
# home/management/commands/create_dev_superuser.py
from django.core.management.base import BaseCommand
from django.contrib.auth import get_user_model

User = get_user_model()

class Command(BaseCommand):
    help = 'Creates a development superuser (admin/admin)'

    def handle(self, *args, **options):
        if User.objects.filter(username='admin').exists():
            self.stdout.write(self.style.WARNING('Superuser "admin" already exists.'))
            return
        
        User.objects.create_superuser('admin', 'admin@example.com', 'admin')
        self.stdout.write(self.style.SUCCESS('Successfully created superuser "admin"'))
```

## 10. Testing Examples

### Django Test

```python
# blog/tests.py
from django.test import TestCase
from blog.models import BlogPostPage, BlogIndexPage
from home.models import HomePage

class BlogPostTestCase(TestCase):
    def setUp(self):
        home = HomePage.objects.create(title="Home", slug="home")
        blog_index = BlogIndexPage.objects.create(
            title="Blog",
            slug="blog",
            parent=home
        )
        self.post = BlogPostPage.objects.create(
            title="Test Post",
            slug="test-post",
            date="2024-01-01",
            parent=blog_index
        )
    
    def test_post_creation(self):
        self.assertEqual(self.post.title, "Test Post")
        self.assertEqual(self.post.slug, "test-post")
    
    def test_post_is_live(self):
        self.assertTrue(self.post.live)
```

### FastAPI Test

```python
# tests/test_api.py
from fastapi.testclient import TestClient
from fastapi_app.api import app

client = TestClient(app)

def test_health_endpoint():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok", "service": "Codea Blog API"}

def test_get_all_posts():
    response = client.get("/blogs")
    assert response.status_code == 200
    assert isinstance(response.json(), list)
```

