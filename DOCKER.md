# 🐳 Docker Deployment Guide

This guide explains how to run the entire VANTAGE platform using Docker Compose.

## 📋 Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- 4GB+ RAM allocated to Docker
- API Keys (watsonx, Clerk, Google Patents)

## 🚀 Quick Start

### 1. Configure Environment Variables

```bash
# Copy the environment template
cp .env.example .env

# Edit .env with your actual credentials
nano .env  # or use your preferred editor
```

Required variables:
- `WATSONX_API_KEY` - IBM watsonx API key
- `WATSONX_PROJECT_ID` - IBM watsonx project ID
- `WATSONX_NLU_API_KEY` - watsonx NLU API key
- `WATSONX_NLU_URL` - watsonx NLU endpoint URL
- `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` - Clerk publishable key
- `CLERK_SECRET_KEY` - Clerk secret key

Optional:
- `GOOGLE_PATENTS_API_KEY` - Google Patents API key
- `WATSONX_ORCHESTRATE_*` - watsonx Orchestrate credentials

### 2. Build and Start Services

```bash
# Build images and start all services
docker-compose up -d

# View logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f backend
docker-compose logs -f frontend
```

### 3. Access the Application

Once containers are running:

- **Frontend UI**: http://localhost:3000
- **Backend API**: http://localhost:8000
- **API Docs**: http://localhost:8000/docs
- **Health Check**: http://localhost:8000/api/v1/health

## 📦 Service Architecture

```
┌─────────────────────────────────────────┐
│          Frontend (Next.js)             │
│          Port: 3000                     │
│  ┌────────────────────────────────────┐ │
│  │  - React 19 UI                     │ │
│  │  - Clerk Authentication            │ │
│  │  - Zustand State Management        │ │
│  └────────────────────────────────────┘ │
└────────────────┬────────────────────────┘
                 │ HTTP (Internal Network)
                 ▼
┌─────────────────────────────────────────┐
│          Backend (FastAPI)              │
│          Port: 8000                     │
│  ┌────────────────────────────────────┐ │
│  │  - FastAPI REST API                │ │
│  │  - SQLAlchemy ORM                  │ │
│  │  - watsonx Integration             │ │
│  │  - SQLite Database                 │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

## 🔧 Common Commands

### Start Services
```bash
# Start in background
docker-compose up -d

# Start with build
docker-compose up --build -d

# Start and follow logs
docker-compose up
```

### Stop Services
```bash
# Stop containers (keeps data)
docker-compose stop

# Stop and remove containers (keeps volumes)
docker-compose down

# Stop and remove everything including volumes
docker-compose down -v
```

### Rebuild Services
```bash
# Rebuild all services
docker-compose build

# Rebuild specific service
docker-compose build backend
docker-compose build frontend

# Rebuild without cache
docker-compose build --no-cache
```

### View Logs
```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f backend

# Last 100 lines
docker-compose logs --tail=100
```

### Execute Commands in Containers
```bash
# Backend shell
docker-compose exec backend /bin/sh

# Frontend shell
docker-compose exec frontend /bin/sh

# Run backend tests
docker-compose exec backend pytest

# Check database
docker-compose exec backend python -c "from app.database import engine; print(engine.url)"
```

### Restart Services
```bash
# Restart all
docker-compose restart

# Restart specific service
docker-compose restart backend
```

## 📊 Health Checks

Both services have health checks configured:

### Backend Health Check
```bash
curl http://localhost:8000/api/v1/health
```

Expected response:
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "database": "connected"
}
```

### Frontend Health Check
```bash
curl http://localhost:3000
```

Should return HTTP 200.

## 💾 Data Persistence

Docker volumes are configured to persist important data:

### Backend Data
- **Database**: `./backend/priorai.db` - SQLite database
- **Uploads**: `./backend/uploads/` - User-uploaded disclosure files
- **Reports**: `./backend/uploads/reports/` - Generated PDF reports

All data persists on the host machine even if containers are removed.

## 🔒 Security Notes

### Environment Variables
- Never commit `.env` file to version control
- Use strong, unique API keys
- Rotate credentials regularly

### Production Deployment
For production, consider:
1. Use PostgreSQL instead of SQLite
2. Enable HTTPS with reverse proxy (nginx/traefik)
3. Set `DEBUG=False` in backend environment
4. Use production-grade secrets management
5. Configure proper CORS origins
6. Enable rate limiting
7. Set up log aggregation

## 🐛 Troubleshooting

### Issue: Port already in use
```bash
# Check what's using the port
lsof -i :3000
lsof -i :8000

# Stop conflicting services
docker-compose down

# Use different ports
docker-compose -f docker-compose.yml -p vantage up -d
```

### Issue: Database locked
```bash
# Stop all containers
docker-compose down

# Remove database lock
rm backend/priorai.db-journal

# Restart
docker-compose up -d
```

### Issue: Frontend can't reach backend
```bash
# Check network
docker network ls
docker network inspect prior-ai_vantage-network

# Check backend health
docker-compose exec frontend curl http://backend:8000/api/v1/health
```

### Issue: Container keeps restarting
```bash
# View logs to identify issue
docker-compose logs backend

# Check container status
docker-compose ps

# Inspect specific container
docker inspect vantage-backend
```

### Issue: Out of memory
```bash
# Check Docker resource usage
docker stats

# Increase Docker memory limit in Docker Desktop settings
# Recommended: 4GB minimum
```

### Issue: Build fails
```bash
# Clean build cache
docker-compose build --no-cache

# Remove old images
docker image prune -a

# Check disk space
docker system df
```

## 🔄 Updating the Application

### Update Code
```bash
# Pull latest changes
git pull origin main

# Rebuild and restart
docker-compose up --build -d
```

### Update Dependencies
```bash
# Backend
cd backend
# Update requirements.txt
docker-compose build backend
docker-compose up -d backend

# Frontend
cd frontend
# Update package.json
docker-compose build frontend
docker-compose up -d frontend
```

## 🧪 Development vs Production

### Development Mode
The default `docker-compose.yml` is configured for development:
- Code mounted as volumes for hot-reload
- Debug mode enabled
- Detailed logging
- CORS allows localhost

### Production Mode
For production, create `docker-compose.prod.yml`:
```yaml
version: '3.8'
services:
  backend:
    environment:
      - DEBUG=False
      - DATABASE_URL=postgresql://user:pass@db:5432/vantage
    # Remove volume mounts
    restart: always

  frontend:
    environment:
      - NODE_ENV=production
    restart: always
```

Run with:
```bash
docker-compose -f docker-compose.prod.yml up -d
```

## 📈 Monitoring

### Container Metrics
```bash
# Real-time resource usage
docker stats

# Container logs
docker-compose logs -f --tail=100
```

### Application Metrics
- Backend API: http://localhost:8000/docs
- Health endpoint: http://localhost:8000/api/v1/health

## 🛑 Stopping and Cleanup

### Stop Services
```bash
# Stop (keeps containers)
docker-compose stop

# Stop and remove containers
docker-compose down

# Stop, remove containers and networks
docker-compose down --remove-orphans
```

### Complete Cleanup
```bash
# Remove everything including volumes (DELETES DATA!)
docker-compose down -v

# Clean up Docker system
docker system prune -a --volumes
```

## 📚 Additional Resources

- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Next.js Docker Guide](https://nextjs.org/docs/deployment#docker-image)
- [FastAPI Docker Guide](https://fastapi.tiangolo.com/deployment/docker/)

## 🆘 Support

If you encounter issues:
1. Check logs: `docker-compose logs -f`
2. Verify environment variables in `.env`
3. Ensure all required API keys are valid
4. Check [Backend README](backend/README.md)
5. Open an issue on GitHub

---

**Last Updated**: December 2025
