# Notification Service

Event-driven webhook notification microservice for the Video Converter platform.

## Overview

The Notification Service consumes video processing events from RabbitMQ and sends HTTP webhook notifications to configured endpoints. It handles both completion and error events for processed videos.

**Service Details:**
- **Framework**: FastAPI 0.104.1 (health check endpoint only)
- **Message Broker**: RabbitMQ (consumes events)
- **Database**: PostgreSQL (shared - for user lookup)
- **Event Consumer**: RabbitMQ event listener

## Quick Start

### Prerequisites
- Docker & Docker Compose
- OR Python 3.11+ with pip

### Option 1: Docker (Recommended)

```bash
# Start with docker-compose (from monorepo root)
docker-compose -f docker/docker-compose.yml up notification-service
```

### Option 2: Local Development

```bash
# Install dependencies
pip install -r requirements.txt
pip install -r ../shared/requirements.txt

# Set environment variables
export DATABASE_URL=postgresql://user:password@localhost:5432/videoconverter
export RABBITMQ_URL=amqp://guest:guest@localhost:5672/
export WEBHOOK_URL=http://localhost:3000/webhook

# Start the service
python -m app.listeners
```

## API Endpoints

### Health Check
```bash
curl http://localhost:8003/health
```

Response:
```json
{"status": "ok", "service": "notification-service"}
```

## Webhook Integration

The service sends HTTP POST requests to your configured webhook URL with the following payloads:

### Video Completed Event

**When**: Video processing successfully completes

```json
{
  "event": "video.completed",
  "video_id": 123,
  "username": "john_doe",
  "timestamp": "2026-01-21T10:05:30Z",
  "download_url": "http://localhost:8002/videos/download/123"
}
```

### Video Error Event

**When**: Video processing fails

```json
{
  "event": "video.error",
  "video_id": 123,
  "username": "john_doe",
  "timestamp": "2026-01-21T10:05:30Z",
  "error": "Unsupported video format or FFmpeg error"
}
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `RABBITMQ_URL` | amqp://guest:guest@localhost:5672/ | RabbitMQ connection URL |
| `DATABASE_URL` | - | PostgreSQL connection string |
| `WEBHOOK_URL` | http://localhost:3000/webhook | Webhook destination URL |

### Setting Webhook URL

The webhook URL tells the service where to send notifications:

```bash
export WEBHOOK_URL=https://your-api.example.com/webhooks/videos
```

## Testing Webhooks Locally

### Option 1: Simple Python Server

```bash
# From monorepo root
python webhook_test_server.py
```

This starts a test server on `http://localhost:3000/webhook` that logs all received notifications.

Set environment variable:
```bash
export WEBHOOK_URL=http://localhost:3000/webhook
```

### Option 2: curl for Manual Testing

```bash
# Simulate a video.completed event
curl -X POST http://localhost:3000/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "event": "video.completed",
    "video_id": 1,
    "username": "testuser",
    "timestamp": "2026-01-21T10:00:00Z",
    "download_url": "http://localhost:8002/videos/download/1"
  }'
```

### Option 3: ngrok for Remote Testing

Expose your local webhook receiver to the internet:

```bash
ngrok http 3000
# Output: https://abc123.ngrok.io

export WEBHOOK_URL=https://abc123.ngrok.io/webhook
```

## Project Structure

```
notification-service/
├── app/
│   ├── main.py           # FastAPI health check endpoint
│   ├── listeners.py      # RabbitMQ event consumer
│   └── webhook.py        # HTTP webhook delivery logic
├── Dockerfile            # Container configuration
├── requirements.txt      # Python dependencies
└── README.md            # This file
```

## Key Dependencies

- **fastapi** - Health check endpoint
- **uvicorn** - ASGI server
- **pika** - RabbitMQ client
- **requests** - HTTP webhook delivery
- **sqlalchemy** - Database access
- **psycopg2-binary** - PostgreSQL adapter

See [requirements.txt](requirements.txt) for full list.

## Architecture

Event processing flow:

```
┌──────────────────────┐
│  Video Worker        │
│  (Celery)            │
└──────┬───────────────┘
       │
       │ Publishes events
       ▼
┌──────────────────────────┐
│  RabbitMQ                │
│  Exchange: videos        │
│  Routing keys:           │
│  - video.completed       │
│  - video.error           │
└──────┬───────────────────┘
       │
       │ Consumes events
       ▼
┌──────────────────────────┐
│  Notification Service    │
│  Event Listener          │
└──────┬───────────────────┘
       │
       │ Looks up user
       │ from PostgreSQL
       │
       ▼
┌──────────────────────────┐
│  Your Webhook URL        │
│  (HTTP POST)             │
│  https://your-api.../    │
│  webhooks/videos         │
└──────────────────────────┘
```

## Development

### View Logs

```bash
# Docker
docker-compose logs -f notification-service

# Local (Terminal 1)
python -m app.listeners
```

### Check Event Queue

```bash
# Access RabbitMQ management UI
# http://localhost:15672
# Username: guest
# Password: guest

# Or use command line
docker exec rabbitmq rabbitmqctl list_queues
```

### Lint Code

```bash
flake8 notification-service/ --max-line-length=127
```

### Security Scanning

```bash
safety check -r requirements.txt
bandit -r app/
```

## CI/CD

This service has automated workflows:

- **CI Pipeline** (`.github/workflows/notification-ci.yml`):
  - ✅ Lint (flake8)
  - ✅ Security scan (safety, bandit)
  - ✅ Tests (pytest)
  - Triggered on: Changes to `notification-service/` or `shared/`

- **Docker Build** (`.github/workflows/notification-docker.yml`):
  - ✅ Build image
  - ✅ Push to ghcr.io (on main/develop)
  - Image: `ghcr.io/Ericatici/video-converter-notification`

**Badge:** ![Notification CI](https://github.com/Ericatici/notification-service/actions/workflows/notification-ci.yml/badge.svg)

## Integration with Other Services

**Video Worker**: Publishes events via RabbitMQ
- `video.completed` - Sent when video conversion succeeds
- `video.error` - Sent when video conversion fails

**PostgreSQL**: Looks up user webhook URL configuration
- Queries to find webhook endpoints
- Retrieves user contact information

## Webhook Delivery Guarantees

- **Retry Logic**: Failed deliveries are retried up to 3 times
- **Timeout**: 10-second timeout per webhook call
- **Logging**: All events logged for audit trail
- **Error Handling**: Failed deliveries logged but don't block event processing

## Troubleshooting

**Issue**: "Connection refused" when connecting to RabbitMQ
- Verify RabbitMQ running: `docker-compose ps rabbitmq`
- Check `RABBITMQ_URL` environment variable
- Ensure RabbitMQ port 5672 is accessible

**Issue**: "Webhooks not being sent"
- Check service running: `docker-compose ps notification-service`
- Verify `WEBHOOK_URL` configured correctly
- Check logs: `docker-compose logs notification-service`
- Test webhook manually: `curl -X POST $WEBHOOK_URL -d '...'`

**Issue**: "Webhook receiver returns error"
- Check HTTP status codes in logs
- Verify webhook URL is correct
- Ensure webhook endpoint accepts POST requests
- Test with curl first: `curl -X POST http://localhost:3000/webhook -d '{}'`

**Issue**: "Can't find user in database"
- Verify PostgreSQL running: `docker-compose ps db`
- Check `DATABASE_URL` environment variable
- Ensure shared models loaded correctly

**Issue**: Events stuck in RabbitMQ queue
- Check for consumer: `docker-compose logs notification-service`
- Verify no errors in logs
- Manually purge queue (RabbitMQ UI or `rabbitmqctl`)

## Security Considerations

- **Validate webhook URL**: Ensure it's HTTPS in production
- **Retry Limit**: Set to 3 to avoid DDoS scenarios
- **Timeout**: 10 seconds prevents hanging connections
- **Authentication**: Consider adding webhook signature verification
- **Rate Limiting**: Implement on webhook endpoint if needed

## Production Deployment

### Best Practices

1. **Use HTTPS** for webhook URLs in production
2. **Add webhook authentication** (API key, signature, HMAC)
3. **Implement webhook retry logic** on receiver side
4. **Monitor webhook deliveries** in your application logs
5. **Set up alerts** for failed webhook deliveries
6. **Test webhook receiver** before going live

### Example: Webhook with Authentication

```bash
# Configure webhook with auth token
export WEBHOOK_URL=https://api.example.com/webhooks/videos?token=secret123

# Or use custom headers (if supported)
# Authorization: Bearer <token>
```

## License

MIT

## Resources

- **Main Project**: [video-converter-prod](https://github.com/Ericatici/video-converter-prod)
- **Auth Service**: [auth-service](https://github.com/Ericatici/auth-service)
- **Video Service**: [video-service](https://github.com/Ericatici/video-service)
- **API Documentation**: See main project README for full architecture
- **Webhook Test Server**: Use `webhook_test_server.py` from main project
