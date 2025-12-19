# Deployment Guide - Stellar Payment Gateway

## Prerequisites

- Python 3.10 or higher
- pip (Python package manager)
- PostgreSQL (for production) or SQLite (for development)
- Git

## Local Development Setup

### 1. Clone and Setup

```bash
cd d:\Hackthon\backend
```

### 2. Create Virtual Environment

```bash
python -m venv venv

# Windows
venv\Scripts\activate

# Linux/Mac
source venv/bin/activate
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure Environment

The `.env` file is already created with default settings. **Important**: Change these values for production:

```bash
# Edit .env file
JWT_SECRET=your-very-long-secure-random-string-here
ADMIN_PASSWORD=your-secure-admin-password
```

### 5. Initialize Database

```bash
python init_db.py
```

This will:
- Create all database tables
- Create an admin account
- Display admin credentials

### 6. Start the API Server

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Or:

```bash
python -m app.main
```

The API will be available at: `http://localhost:8000`

API Documentation:
- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`

### 7. Start the Stellar Listener (Separate Terminal)

**CRITICAL**: The Stellar payment listener must run separately!

```bash
# In a new terminal, activate the virtual environment first
venv\Scripts\activate  # Windows
# source venv/bin/activate  # Linux/Mac

# Then run the listener
python -m app.services.stellar_listener
```

This service:
- Monitors Stellar blockchain for USDC payments
- Validates payments against payment sessions
- Updates payment status automatically
- Triggers webhook notifications

### 8. Test the API

```bash
python test_api.py
```

This will:
- Register a test merchant
- Create a payment session
- Display checkout URL
- Test admin endpoints

## Production Deployment

### Option 1: Render

1. **Create a new Web Service** on Render
2. **Connect your repository**
3. **Configure build settings**:
   - Build Command: `pip install -r requirements.txt`
   - Start Command: `uvicorn app.main:app --host 0.0.0.0 --port $PORT`

4. **Add environment variables**:
   ```
   DATABASE_URL=your-postgresql-url
   JWT_SECRET=your-secret-key
   APP_BASE_URL=https://your-app.onrender.com
   ```

5. **Create a Background Worker** (for Stellar listener):
   - Same repository
   - Start Command: `python -m app.services.stellar_listener`

### Option 2: Railway

1. **Create a new project** on Railway
2. **Deploy from GitHub**
3. **Add PostgreSQL database**
4. **Set environment variables**
5. **Add worker service** for Stellar listener

### Option 3: Fly.io

1. **Install Fly CLI**
2. **Create fly.toml**:

```toml
app = "stellar-payment-gateway"

[build]
  builder = "paketobuildpacks/builder:base"

[env]
  PORT = "8000"

[[services]]
  internal_port = 8000
  protocol = "tcp"

  [[services.ports]]
    port = 80
    handlers = ["http"]

  [[services.ports]]
    port = 443
    handlers = ["tls", "http"]

[processes]
  web = "uvicorn app.main:app --host 0.0.0.0 --port 8000"
  worker = "python -m app.services.stellar_listener"
```

3. **Deploy**:
```bash
fly deploy
```

## Environment Variables Reference

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `DATABASE_URL` | Database connection string | Yes | sqlite:///./payment_gateway.db |
| `JWT_SECRET` | Secret key for JWT tokens | Yes | - |
| `JWT_ALGORITHM` | JWT algorithm | No | HS256 |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | Token expiry time | No | 1440 (24 hours) |
| `STELLAR_NETWORK` | Stellar network (testnet/public) | No | testnet |
| `STELLAR_HORIZON_URL` | Horizon API URL | No | https://horizon-testnet.stellar.org |
| `USDC_ASSET_CODE` | USDC asset code | No | USDC |
| `USDC_ASSET_ISSUER` | USDC issuer address | No | GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5 |
| `PAYMENT_EXPIRY_MINUTES` | Payment session timeout | No | 15 |
| `WEBHOOK_RETRY_LIMIT` | Webhook retry attempts | No | 3 |
| `WEBHOOK_TIMEOUT_SECONDS` | Webhook request timeout | No | 10 |
| `APP_BASE_URL` | Base URL of your app | Yes | http://localhost:8000 |
| `CORS_ORIGINS` | Allowed CORS origins | No | http://localhost:3000,http://localhost:8000 |
| `ADMIN_EMAIL` | Default admin email | No | admin@paymentgateway.com |
| `ADMIN_PASSWORD` | Default admin password | No | admin123456 |

## Database Migration (PostgreSQL)

### For Production:

1. **Update DATABASE_URL** in `.env`:
```bash
DATABASE_URL=postgresql://user:password@host:port/database
```

2. **Run initialization**:
```bash
python init_db.py
```

### Using Alembic (Advanced):

```bash
# Initialize Alembic
alembic init alembic

# Generate migration
alembic revision --autogenerate -m "Initial migration"

# Apply migration
alembic upgrade head
```

## Health Checks

### API Health
```bash
curl http://localhost:8000/health
```

### Admin Health (with auth)
```bash
curl -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
     http://localhost:8000/admin/health
```

## Monitoring

### Logs

The application logs:
- All HTTP requests and responses
- Payment detections
- Webhook delivery attempts
- Errors and exceptions

### Recommended Monitoring

For production, use:
- **Sentry** for error tracking
- **DataDog** or **New Relic** for APM
- **Prometheus + Grafana** for metrics

## Security Checklist

- [ ] Change `JWT_SECRET` to a strong random string (min 32 chars)
- [ ] Change `ADMIN_PASSWORD`
- [ ] Use HTTPS in production (`APP_BASE_URL` should use https://)
- [ ] Use PostgreSQL in production (not SQLite)
- [ ] Enable rate limiting (consider using nginx or FastAPI middleware)
- [ ] Set proper CORS origins
- [ ] Keep dependencies updated
- [ ] Use environment variables for all secrets
- [ ] Never commit `.env` file
- [ ] Enable database backups

## Troubleshooting

### Database Issues

**Problem**: `no such table: merchants`
**Solution**: Run `python init_db.py`

### Stellar Listener Not Working

**Problem**: Payments not being detected
**Solution**: 
1. Check if listener is running
2. Verify network settings (testnet vs mainnet)
3. Check Horizon URL is accessible
4. Verify merchant Stellar address is correct

### CORS Errors

**Problem**: Frontend can't connect to API
**Solution**: Add frontend URL to `CORS_ORIGINS` in `.env`

### Token Expired

**Problem**: 401 Unauthorized errors
**Solution**: Login again to get a new token

## Performance Optimization

### For High Traffic:

1. **Use connection pooling**:
```python
# In config.py
engine = create_engine(
    settings.DATABASE_URL,
    pool_size=20,
    max_overflow=40
)
```

2. **Add caching** (Redis):
```bash
pip install redis aioredis
```

3. **Use async database** (asyncpg):
```bash
pip install asyncpg databases
```

4. **Deploy multiple instances** behind a load balancer

## Backup and Recovery

### Database Backup (PostgreSQL)

```bash
# Backup
pg_dump -U username -d payment_gateway > backup.sql

# Restore
psql -U username -d payment_gateway < backup.sql
```

### SQLite Backup

```bash
# Simple file copy
cp payment_gateway.db payment_gateway_backup.db
```

## Support

For issues and questions:
- Check the [README.md](README.md)
- Review API documentation at `/docs`
- Check application logs
- Verify environment configuration

## Next Steps

1. Deploy to production
2. Set up monitoring
3. Configure backups
4. Implement rate limiting
5. Add logging aggregation
6. Set up CI/CD pipeline
