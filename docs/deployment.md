# Deployment Guide

## Overview

This guide covers deploying the Codea Blog application to various platforms, from local development to production environments.

## Deployment Options

### 1. Local Development

#### Prerequisites
- Python 3.11+
- Virtual environment
- SQLite (included with Python)

#### Steps

```bash
# Clone repository
git clone <repo-url>
cd codea_blog

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: .\venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Run migrations
python manage.py migrate

# Create superuser
python manage.py create_dev_superuser

# Run application
python codea_blog/main.py
```

**Access Points:**
- Wagtail Admin: http://127.0.0.1:8000/cms/
- API Docs: http://127.0.0.1:8000/fastapi/docs
- Blog: http://127.0.0.1:8000/cms/blog/

### 2. Docker Deployment

#### Build Docker Image

```bash
docker build -t codea-blog .
```

#### Run Container

```bash
docker run -p 8000:8000 \
  -e DJANGO_SETTINGS_MODULE=codea_blog.settings.production \
  -e SECRET_KEY=your-secret-key \
  -e DATABASE_URL=postgresql://user:pass@host:5432/db \
  codea-blog
```

#### Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DJANGO_SETTINGS_MODULE=codea_blog.settings.production
      - SECRET_KEY=${SECRET_KEY}
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/codea_blog
      - ALLOWED_HOSTS=localhost,127.0.0.1
    depends_on:
      - db
    volumes:
      - ./media:/app/media
      - ./staticfiles:/app/staticfiles

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=codea_blog
      - POSTGRES_USER=postgres_user
      - POSTGRES_PASSWORD=postgres_pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Run with:
```bash
docker-compose up -d
```

### 3. Render.com Deployment

#### Prerequisites
- Render.com account
- GitHub repository

#### Steps

1. **Connect Repository**
   - Go to Render Dashboard
   - Click "New" → "Web Service"
   - Connect your GitHub repository

2. **Configure Service**
   - **Name**: `codea-blog`
   - **Environment**: `Python 3`
   - **Build Command**: `pip install -r requirements.txt`
   - **Start Command**: `uvicorn codea_blog.main:app --host 0.0.0.0 --port $PORT`

3. **Environment Variables**
   ```
   DJANGO_SETTINGS_MODULE=codea_blog.settings.production
   SECRET_KEY=<generate-secret-key>
   DEBUG=False
   ALLOWED_HOSTS=your-app.onrender.com
   DATABASE_URL=<render-postgres-url>
   EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
   EMAIL_HOST=smtp.gmail.com
   EMAIL_PORT=587
   EMAIL_USE_TLS=True
   EMAIL_HOST_USER=your-email@gmail.com
   EMAIL_HOST_PASSWORD=your-app-password
   DEFAULT_FROM_EMAIL=noreply@yourdomain.com
   CONTACT_FORM_RECIPIENT=admin@yourdomain.com
   ```

4. **PostgreSQL Database**
   - Create PostgreSQL database in Render
   - Copy connection string to `DATABASE_URL`

5. **Deploy**
   - Click "Create Web Service"
   - Render will build and deploy automatically

### 4. Heroku Deployment

#### Prerequisites
- Heroku CLI installed
- Heroku account

#### Steps

1. **Login to Heroku**
   ```bash
   heroku login
   ```

2. **Create App**
   ```bash
   heroku create codea-blog
   ```

3. **Add PostgreSQL**
   ```bash
   heroku addons:create heroku-postgresql:hobby-dev
   ```

4. **Set Environment Variables**
   ```bash
   heroku config:set DJANGO_SETTINGS_MODULE=codea_blog.settings.production
   heroku config:set SECRET_KEY=$(python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())')
   heroku config:set DEBUG=False
   heroku config:set ALLOWED_HOSTS=codea-blog.herokuapp.com
   ```

5. **Create Procfile**
   ```
   web: uvicorn codea_blog.main:app --host 0.0.0.0 --port $PORT
   ```

6. **Deploy**
   ```bash
   git push heroku main
   heroku run python manage.py migrate
   heroku run python manage.py createsuperuser
   ```

### 5. AWS EC2 Deployment

#### Prerequisites
- AWS account
- EC2 instance (Ubuntu 22.04 recommended)
- SSH access

#### Steps

1. **Connect to Instance**
   ```bash
   ssh -i your-key.pem ubuntu@your-ec2-ip
   ```

2. **Install Dependencies**
   ```bash
   sudo apt update
   sudo apt install python3-pip python3-venv nginx postgresql postgresql-contrib
   ```

3. **Setup PostgreSQL**
   ```bash
   sudo -u postgres psql
   CREATE DATABASE codea_blog;
   CREATE USER codea_user WITH PASSWORD 'your-password';
   GRANT ALL PRIVILEGES ON DATABASE codea_blog TO codea_user;
   \q
   ```

4. **Clone and Setup Application**
   ```bash
   git clone <repo-url>
   cd codea_blog
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```

5. **Configure Environment**
   ```bash
   export DJANGO_SETTINGS_MODULE=codea_blog.settings.production
   export SECRET_KEY=your-secret-key
   export DATABASE_URL=postgresql://codea_user:your-password@localhost:5432/codea_blog
   export ALLOWED_HOSTS=your-domain.com
   ```

6. **Run Migrations**
   ```bash
   python manage.py migrate
   python manage.py collectstatic --noinput
   python manage.py createsuperuser
   ```

7. **Setup Systemd Service**
   Create `/etc/systemd/system/codea-blog.service`:
   ```ini
   [Unit]
   Description=Codea Blog ASGI Application
   After=network.target

   [Service]
   User=ubuntu
   WorkingDirectory=/home/ubuntu/codea_blog
   Environment="PATH=/home/ubuntu/codea_blog/venv/bin"
   Environment="DJANGO_SETTINGS_MODULE=codea_blog.settings.production"
   Environment="SECRET_KEY=your-secret-key"
   Environment="DATABASE_URL=postgresql://codea_user:password@localhost:5432/codea_blog"
   ExecStart=/home/ubuntu/codea_blog/venv/bin/uvicorn codea_blog.main:app --host 0.0.0.0 --port 8000

   [Install]
   WantedBy=multi-user.target
   ```

   Enable and start:
   ```bash
   sudo systemctl enable codea-blog
   sudo systemctl start codea-blog
   ```

8. **Configure Nginx**
   Create `/etc/nginx/sites-available/codea-blog`:
   ```nginx
   server {
       listen 80;
       server_name your-domain.com;

       location / {
           proxy_pass http://127.0.0.1:8000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }

       location /static/ {
           alias /home/ubuntu/codea_blog/staticfiles/;
       }

       location /media/ {
           alias /home/ubuntu/codea_blog/media/;
       }
   }
   ```

   Enable site:
   ```bash
   sudo ln -s /etc/nginx/sites-available/codea-blog /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl restart nginx
   ```

9. **Setup SSL (Let's Encrypt)**
   ```bash
   sudo apt install certbot python3-certbot-nginx
   sudo certbot --nginx -d your-domain.com
   ```

## Environment Configuration

### Development Settings

```python
# settings/dev.py
DEBUG = True
ALLOWED_HOSTS = ["*"]
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"
```

### Production Settings

```python
# settings/production.py
DEBUG = False
ALLOWED_HOSTS = os.environ.get("ALLOWED_HOSTS", "").split(",")
SECRET_KEY = os.environ.get("SECRET_KEY")

DATABASES = {
    "default": dj_database_url.parse(os.environ["DATABASE_URL"])
}

# Email
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = os.environ.get("EMAIL_HOST")
EMAIL_PORT = int(os.environ.get("EMAIL_PORT", 587))
EMAIL_USE_TLS = True
EMAIL_HOST_USER = os.environ.get("EMAIL_HOST_USER")
EMAIL_HOST_PASSWORD = os.environ.get("EMAIL_HOST_PASSWORD")
DEFAULT_FROM_EMAIL = os.environ.get("DEFAULT_FROM_EMAIL")
CONTACT_FORM_RECIPIENT = os.environ.get("CONTACT_FORM_RECIPIENT")

# Security
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

## Required Environment Variables

### Core Settings
- `DJANGO_SETTINGS_MODULE`: Settings module to use
- `SECRET_KEY`: Django secret key (generate with `python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'`)
- `DEBUG`: `True` or `False`
- `ALLOWED_HOSTS`: Comma-separated list of allowed hostnames

### Database
- `DATABASE_URL`: PostgreSQL connection string (format: `postgresql://user:password@host:port/database`)

### Email
- `EMAIL_HOST`: SMTP server hostname
- `EMAIL_PORT`: SMTP port (usually 587)
- `EMAIL_HOST_USER`: SMTP username
- `EMAIL_HOST_PASSWORD`: SMTP password
- `DEFAULT_FROM_EMAIL`: Default sender email
- `CONTACT_FORM_RECIPIENT`: Contact form recipient email

### Optional
- `ENABLE_RATE_LIMITING`: `true` or `false`
- `RATE_LIMITER_URL`: Rate limiter service URL
- `RATE_LIMITER_TIMEOUT`: Timeout in seconds
- `RATE_LIMITER_FAIL_OPEN`: `true` or `false`

## Pre-Deployment Checklist

- [ ] Set `DEBUG=False` in production settings
- [ ] Generate and set `SECRET_KEY`
- [ ] Configure `ALLOWED_HOSTS`
- [ ] Set up PostgreSQL database
- [ ] Configure email settings
- [ ] Run migrations: `python manage.py migrate`
- [ ] Collect static files: `python manage.py collectstatic --noinput`
- [ ] Create superuser: `python manage.py createsuperuser`
- [ ] Test all endpoints
- [ ] Configure rate limiting (if needed)
- [ ] Set up SSL/HTTPS
- [ ] Configure backup strategy
- [ ] Set up monitoring/logging
- [ ] Test contact form email sending

## Post-Deployment Tasks

1. **Verify Deployment**
   - Check health endpoint: `https://your-domain.com/fastapi/health`
   - Test API endpoints
   - Verify Wagtail admin access
   - Test contact form submission

2. **Create Initial Content**
   - Log into Wagtail admin
   - Create blog index page
   - Create sample blog posts
   - Upload media files

3. **Monitor Performance**
   - Check application logs
   - Monitor database performance
   - Check static file serving
   - Monitor API response times

4. **Setup Backups**
   - Database backups (daily recommended)
   - Media file backups
   - Configuration backups

## Troubleshooting

### Database Connection Issues
```bash
# Test connection
python manage.py dbshell

# Check database URL format
echo $DATABASE_URL
```

### Static Files Not Loading
```bash
# Recollect static files
python manage.py collectstatic --noinput

# Check STATIC_ROOT and STATIC_URL settings
```

### Migration Issues
```bash
# Show migration status
python manage.py showmigrations

# Fake migrations if needed (use with caution)
python manage.py migrate --fake
```

### Port Already in Use
```bash
# Find process using port
lsof -i :8000  # macOS/Linux
netstat -ano | findstr :8000  # Windows

# Kill process
kill -9 <PID>  # macOS/Linux
taskkill /PID <PID> /F  # Windows
```

## Scaling Considerations

### Horizontal Scaling
- Deploy multiple instances behind load balancer
- Use shared database (PostgreSQL)
- Use shared media storage (S3, etc.)
- Configure session storage (Redis, database)

### Vertical Scaling
- Increase server resources (CPU, RAM)
- Optimize database queries
- Add caching layer (Redis)
- Use CDN for static/media files

### Database Optimization
- Use connection pooling
- Add read replicas
- Optimize queries with indexes
- Regular database maintenance

## Security Best Practices

1. **Never commit secrets**
   - Use environment variables
   - Use secret management services

2. **Enable HTTPS**
   - Use Let's Encrypt for free SSL
   - Configure secure cookies

3. **Regular Updates**
   - Keep dependencies updated
   - Monitor security advisories

4. **Access Control**
   - Use strong passwords
   - Enable two-factor authentication (if available)
   - Limit admin access

5. **Monitoring**
   - Monitor for suspicious activity
   - Set up alerts for errors
   - Regular security audits
