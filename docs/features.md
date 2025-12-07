# Key Features

## 1. Hybrid CMS + API Architecture

### Description
The project seamlessly combines Django/Wagtail CMS with FastAPI, providing both a powerful content management interface and a modern REST API.

### Benefits
- **Single Codebase**: One codebase serves both CMS and API needs
- **Shared Database**: No data synchronization issues
- **Flexible Access**: Content editors use Wagtail, developers use API
- **Cost Effective**: Single server deployment

### Implementation
- Starlette ASGI app mounts both frameworks
- Path-based routing (`/fastapi` for FastAPI, `/` for Django)
- Shared Django models accessible from FastAPI

## 2. Wagtail Content Management

### Rich Content Editing
- **StreamField**: Flexible content blocks (text, images, videos, embeds, HTML)
- **Rich Text Editor**: WYSIWYG editing with formatting options
- **Media Management**: Built-in image and video upload/management
- **Page Hierarchy**: Parent-child page relationships

### Content Organization
- **Categories**: Reusable category snippets
- **Tags**: Flexible tagging system using django-taggit
- **Draft/Published**: Content workflow with draft and published states
- **Revision History**: Track changes to content over time

### Admin Interface
- **Intuitive UI**: User-friendly admin interface
- **Page Tree**: Visual page hierarchy
- **Search**: Built-in search functionality
- **Permissions**: Role-based access control

## 3. FastAPI REST API

### Modern API Framework
- **Async Support**: Non-blocking request handling
- **Type Safety**: Pydantic models for request/response validation
- **Auto Documentation**: OpenAPI/Swagger and ReDoc documentation
- **High Performance**: Fast request processing

### API Endpoints

#### Blog Endpoints
- `GET /fastapi/blogs` - List all published posts
- `GET /fastapi/blogs/{post_id}` - Get post by ID
- `GET /fastapi/blogs/slug/{slug}` - Get post by slug

#### Utility Endpoints
- `GET /fastapi/health` - Health check
- `GET /fastapi/schema` - OpenAPI schema
- `POST /fastapi/contact/` - Contact form submission

### Features
- **Automatic Validation**: Request/response validation via Pydantic
- **Error Handling**: Comprehensive error responses
- **Rate Limiting**: Optional external rate limiting
- **CORS Support**: Cross-origin resource sharing

## 4. Flexible Content Blocks

### StreamField Blocks

#### Rich Text Block
- Formatted text with headings, lists, links
- Inline formatting (bold, italic, underline)
- Code blocks and quotes

#### HTML Block
- Raw HTML insertion (for trusted authors)
- Custom styling and scripts
- Embed third-party widgets

#### Image Block
- Image upload and selection
- Caption and alt text support
- Responsive image handling

#### Video Block
- Video upload via Wagtail Media
- Support for multiple video formats
- Embedded video playback

#### Embed Block
- YouTube/Vimeo video embeds
- Twitter/X embeds
- Any oEmbed-compatible service

### Benefits
- **Flexibility**: Editors can create diverse content layouts
- **Reusability**: Blocks can be reused across posts
- **Type Safety**: Each block has defined structure
- **Extensibility**: Easy to add new block types

## 5. Authentication & Security

### Multi-Backend Authentication

#### JWT Authentication
- Token-based authentication
- Integration with external auth server
- Stateless authentication

#### Remote Authentication
- Username/password validation via external server
- Fallback to local Django users
- Session management

#### Google OAuth
- OAuth 2.0 integration
- Social login support
- Token validation

### Security Features
- **CSRF Protection**: Enabled for forms, exempted for API
- **CORS Configuration**: Configurable cross-origin policies
- **Rate Limiting**: Optional external rate limiting service
- **Input Validation**: Pydantic and Django form validation

## 6. Rate Limiting

### External Rate Limiter
- Integration with Codea Auth Server rate limiter
- Per-IP rate limiting
- Configurable limits and timeouts

### Configuration Options
- **Enable/Disable**: Toggle rate limiting via environment variable
- **Fail-Open/Fail-Closed**: Behavior when service unavailable
- **Timeout**: Configurable request timeout
- **Per-Endpoint**: Rate limiting applied via FastAPI dependencies

### Benefits
- **API Protection**: Prevent abuse and DDoS attacks
- **Flexible**: Can be enabled/disabled per environment
- **Resilient**: Fail-open mode prevents service disruption

## 7. Contact Form API

### Features
- **Validation**: Email format, field length validation
- **Email Sending**: Automatic email notifications
- **Error Handling**: Comprehensive error responses
- **Rate Limited**: Protected by rate limiting

### Request Model
```python
{
    "name": "John Doe",
    "email": "john@example.com",
    "subject": "Question",
    "message": "Hello, I have a question...",
    "phone": "+1234567890"  # Optional
}
```

### Email Configuration
- Configurable recipient email
- Customizable email template
- Support for SMTP and console backends

## 8. Environment-Based Configuration

### Settings Structure
- **base.py**: Shared settings
- **dev.py**: Development overrides
- **production.py**: Production overrides
- **stag.py**: Staging overrides

### Environment Variables
- `DJANGO_SETTINGS_MODULE`: Selects settings module
- `DATABASE_URL`: Database connection string
- `SECRET_KEY`: Django secret key
- `DEBUG`: Debug mode toggle
- `ALLOWED_HOSTS`: Allowed hostnames

### Benefits
- **Separation**: Different configs for different environments
- **Security**: Sensitive data via environment variables
- **Flexibility**: Easy to add new environments

## 9. Docker Support

### Dockerfile Features
- Multi-stage build support
- Health check configuration
- Optimized layer caching
- Production-ready setup

### Docker Compose
- Service orchestration
- Database service (optional)
- Volume management
- Network configuration

### Benefits
- **Consistency**: Same environment across dev/staging/prod
- **Isolation**: Containerized dependencies
- **Deployment**: Easy deployment to cloud platforms

## 10. API Documentation

### Swagger UI
- Interactive API documentation
- Try-it-out functionality
- Request/response examples
- Schema definitions

### ReDoc
- Alternative documentation format
- Clean, readable interface
- Search functionality
- Mobile-friendly

### Auto-Generation
- Generated from code
- Always up-to-date
- Type information included
- No manual documentation needed

## 11. Media Management

### Image Management
- Upload and organize images
- Automatic thumbnail generation
- Responsive image serving
- Alt text and captions

### Video Management
- Video upload via Wagtail Media
- Multiple format support
- Streaming support
- Thumbnail generation

### File Organization
- Folder structure
- Search and filter
- Bulk operations
- Usage tracking

## 12. Search Functionality

### Built-in Search
- Full-text search
- Page content indexing
- Search results page
- Query highlighting

### Search Features
- Search across all content
- Filter by content type
- Sort results
- Pagination support

## 13. Development Tools

### Management Commands
- `create_dev_superuser`: Auto-create admin user for development
- `migrate`: Database migrations
- `collectstatic`: Collect static files
- `runserver`: Development server

### Hot Reload
- Automatic code reloading
- Fast development cycle
- Error reporting
- Debug toolbar (optional)

### Testing Support
- Django test framework
- Pytest integration
- Test fixtures
- Coverage reporting

## 14. Production Features

### Static File Serving
- WhiteNoise integration
- Compressed static files
- Cache headers
- CDN support

### Database Support
- SQLite (development)
- PostgreSQL (production)
- Migration management
- Backup support

### Logging
- Request logging
- Error logging
- Performance monitoring
- Log rotation

## 15. Extensibility

### Custom Blocks
Easy to add new StreamField blocks:
```python
class CustomBlock(blocks.StructBlock):
    field = blocks.CharBlock()
    
BODY_BLOCKS.append(("custom", CustomBlock()))
```

### Custom Endpoints
Easy to add new FastAPI endpoints:
```python
@app.get("/custom")
async def custom_endpoint():
    return {"message": "Custom endpoint"}
```

### Plugin Architecture
- Django app structure
- Wagtail hooks
- FastAPI routers
- Middleware extensions

