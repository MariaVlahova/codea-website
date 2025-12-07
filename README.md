# Codea Blog - Case Study

## Project Overview

**Codea Blog** is a modern, hybrid content management system that seamlessly combines the power of Django/Wagtail CMS with FastAPI's high-performance REST API capabilities. This project demonstrates how to build a production-ready blog platform that serves both content editors (via Wagtail's intuitive admin interface) and developers (via a modern REST API).

### Key Highlights

- **Hybrid Architecture**: Django/Wagtail CMS and FastAPI running on a single ASGI server
- **Modern API**: FastAPI with automatic OpenAPI/Swagger documentation
- **Content Management**: Full-featured Wagtail CMS with rich text editing, media management, and flexible content blocks
- **Production Ready**: Docker support, environment-based configuration, and deployment-ready setup
- **Authentication Integration**: JWT-based authentication with external auth server support
- **Rate Limiting**: Optional rate limiting via external service with fail-open/fail-closed modes

## Table of Contents

1. [Architecture Overview](./docs/architecture.md)
2. [Technical Implementation](./docs/technical-implementation.md)
3. [Key Features](./docs/features.md)
4. [Challenges & Solutions](./docs/challenges-and-solutions.md)
5. [Deployment Guide](./docs/deployment.md)
6. [Code Examples](./docs/code-examples.md)
7. [Lessons Learned](./docs/lessons-learned.md)

## Quick Start

### Prerequisites

- Python 3.11+
- pip
- Virtual environment (recommended)

### Installation

```bash
# Clone the repository
git clone <repo-url>
cd codea_blog

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: .\venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Run migrations
python manage.py migrate

# Create superuser
python manage.py create_dev_superuser

# Run the application
python codea_blog/main.py
```

The application will be available at:
- **Wagtail Admin**: http://127.0.0.1:8000/cms/
- **API Documentation**: http://127.0.0.1:8000/fastapi/docs
- **Blog Index**: http://127.0.0.1:8000/cms/blog/

## Technology Stack

### Backend
- **Django 5.2+**: Web framework and ORM
- **Wagtail 7.0+**: Content management system
- **FastAPI**: Modern, fast API framework
- **Uvicorn**: ASGI server
- **Django REST Framework**: Additional API capabilities

### Database
- **SQLite**: Development (default)
- **PostgreSQL**: Production (recommended)

### Authentication & Security
- **JWT Authentication**: Token-based authentication
- **External Auth Server**: Integration with Codea Auth Server
- **CORS**: Cross-origin resource sharing support
- **Rate Limiting**: Optional external rate limiting service

### Media & Content
- **Wagtail Media**: Video and image management
- **django-taggit**: Tagging system
- **StreamField**: Flexible content blocks

### Deployment
- **Docker**: Containerization
- **WhiteNoise**: Static file serving
- **Environment-based Configuration**: Separate dev/staging/production settings

## Project Statistics

- **Total Lines of Code**: ~3,000+
- **Django Apps**: 4 (home, blog, authentication, search)
- **API Endpoints**: 6+ FastAPI endpoints
- **Content Models**: 3 (BlogPostPage, BlogIndexPage, BlogCategory)
- **Content Blocks**: 4 (Rich Text, HTML, Image, Video, Embed)

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Starlette ASGI App                   │
│                  (codea_blog/main.py)                   │
└─────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   FastAPI    │  │    Django    │  │    Django    │
│   /fastapi   │  │     /api     │  │      /       │
│              │  │              │  │              │
│  - /health   │  │  Wagtail API │  │  Wagtail CMS │
│  - /blogs    │  │  (REST API)  │  │  (Admin UI)  │
│  - /contact  │  │              │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                          ▼
                  ┌──────────────┐
                  │   Database   │
                  │  (SQLite/    │
                  │  PostgreSQL)  │
                  └──────────────┘
```

## Key Achievements

1. **Seamless Integration**: Successfully integrated Django and FastAPI on a single ASGI server
2. **Dual API Support**: Both Wagtail's built-in API and FastAPI endpoints available
3. **Flexible Content Management**: StreamField allows editors to create rich, flexible content
4. **Production Ready**: Docker support, environment-based configs, and deployment scripts
5. **Developer Experience**: Auto-generated API docs, hot reload, and comprehensive error handling

## Use Cases

- **Content Editors**: Use Wagtail CMS to create and manage blog posts with rich media
- **Frontend Developers**: Consume FastAPI endpoints to build modern web/mobile applications
- **API Consumers**: Access blog content via REST API with automatic documentation
- **System Integrators**: Integrate blog content into existing systems via API

## Contributing

This is a case study repository. For contributions to the main project, please refer to the main repository's contributing guidelines.

## License

This project is licensed under the MIT License.

---

**Built with ❤️ using Django, Wagtail, and FastAPI**

