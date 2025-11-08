# URL Shortener System Design

## Overview

A URL shortening service that converts long URLs into short, manageable links, similar to bit.ly, TinyURL, or goo.gl. The system provides custom short URLs, analytics, expiration, and high-performance redirection.

## Requirements

### Functional Requirements
- Shorten long URLs to short URLs
- Redirect short URLs to original long URLs
- Custom short URL aliases (optional)
- URL expiration (time-based)
- Analytics and click tracking
- User accounts and URL management
- QR code generation
- URL preview before redirect
- API for programmatic access
- Bulk URL shortening

### Non-Functional Requirements
- **High Availability**: 99.99% uptime
- **Low Latency**: Redirect within 10-50ms
- **Scalability**: Handle 1B+ URLs, 10K+ requests per second
- **Durability**: URLs never lost
- **Read-Heavy**: 100:1 read-to-write ratio
- **Global**: Low latency worldwide

## System Architecture

### Core Components

#### 1. Client Interface
- **Web Application**: User interface for URL shortening
- **Browser Extension**: Quick shortening from browser
- **Mobile Apps**: iOS and Android apps
- **API**: RESTful API for developers

#### 2. API Gateway
- Entry point for all requests
- Authentication and authorization
- Rate limiting per user/IP
- Request validation
- Load balancing

#### 3. URL Shortening Service
- Generates short URL codes
- Validates long URLs
- Checks for duplicates (optional)
- Stores URL mappings
- Implements encoding algorithm

#### 4. Redirection Service
- Resolves short URLs to long URLs
- Performs HTTP 301/302 redirect
- Handles cache lookups
- Tracks analytics asynchronously
- Highly optimized for read performance

#### 5. Analytics Service
- Tracks click events
- Records metadata (timestamp, IP, user agent, referrer)
- Aggregates statistics
- Geo-location tracking
- Device and browser analytics

#### 6. User Service
- User registration and authentication
- User URL management (list, delete, edit)
- Custom domain support
- Usage quotas and limits
- API key management

#### 7. Custom Alias Service
- Validates custom short URLs
- Checks availability
- Reserves custom aliases
- Prevents collision with generated codes

#### 8. Expiration Service
- Manages URL expiration
- Background job to cleanup expired URLs
- Soft delete for recovery
- Configurable retention policies

## URL Encoding Algorithm

### Base62 Encoding
- **Characters**: [a-z, A-Z, 0-9] = 62 characters
- **Length**: 7 characters = 62^7 = 3.5 trillion unique URLs
- **Approach**: Convert numeric ID to base62 string

### Algorithm Options

#### Option 1: Counter-Based (Recommended)
1. Use auto-incrementing counter in database
2. When URL created, get next counter value
3. Convert counter to base62 string
4. Result: `abc1234`
5. **Pros**: Deterministic, no collisions, predictable
6. **Cons**: Sequential (predictable), requires coordination

#### Option 2: Hash-Based
1. Generate MD5/SHA256 hash of long URL
2. Take first 7 characters (base62 encoded)
3. Check for collision in database
4. If collision, append suffix or retry
5. **Pros**: Distributed, no central counter
6. **Cons**: Collision possible, needs checking

#### Option 3: Random Generation
1. Generate random 7-character base62 string
2. Check if exists in database
3. If exists, regenerate (rare with 7 chars)
4. **Pros**: Non-sequential, distributed
5. **Cons**: Rare collisions, requires checking

### Chosen Approach: Counter-Based with Zookeeper
- Use distributed counter (Zookeeper, Redis)
- Each app server gets range of IDs (e.g., 1000 IDs)
- Convert ID to base62 locally
- No collision, no coordination overhead
- Scalable and efficient

## Data Storage

### Primary Database
- **Technology**: PostgreSQL or MySQL
- **Schema**:
  ```sql
  url_mappings (
    id BIGINT PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE,
    long_url TEXT,
    user_id BIGINT,
    created_at TIMESTAMP,
    expires_at TIMESTAMP,
    is_active BOOLEAN
  )
  ```
- Indexed on `short_code` for fast lookups
- Sharded by `short_code` for horizontal scaling

### NoSQL Database (Alternative)
- **Technology**: Cassandra, DynamoDB
- Key-value store with `short_code` as key
- High write and read throughput
- Eventual consistency acceptable
- Lower operational complexity

### Cache Layer (Critical)
- **Technology**: Redis or Memcached
- Cache short_code → long_url mappings
- Cache hot URLs (80-20 rule: 20% URLs = 80% traffic)
- TTL based on URL popularity
- Cache hit rate target: 90%+
- **Reduces database load by 90%+**

### Analytics Database
- **Technology**: Cassandra, ClickHouse, or TimescaleDB
- Time-series data for clicks
- **Schema**:
  ```sql
  click_events (
    short_code VARCHAR(10),
    timestamp TIMESTAMP,
    ip_address VARCHAR(45),
    user_agent TEXT,
    referrer TEXT,
    country VARCHAR(2),
    city VARCHAR(100)
  )
  ```
- Partitioned by `short_code` and `timestamp`
- Write-heavy workload
- Aggregated for analytics dashboards

### Object Storage
- **Technology**: AWS S3
- QR code images
- Analytics reports (CSV exports)
- Backup and archival

### Distributed Counter
- **Technology**: Zookeeper or Redis
- Manages ID generation
- Provides ranges to app servers
- Ensures uniqueness across instances

## Data Flow

### URL Shortening Flow
1. User submits long URL via web/API
2. API Gateway validates and authenticates request
3. URL Shortening Service checks if URL already shortened (optional):
   - Query database for existing mapping
   - If found, return existing short URL (deduplication)
4. Get next ID from distributed counter
5. Convert ID to base62 short code (e.g., `abc1234`)
6. Store mapping in database:
   - `short_code`: `abc1234`
   - `long_url`: original URL
   - `user_id`, `created_at`, `expires_at`
7. Cache mapping in Redis
8. Generate QR code (optional, async)
9. Return short URL to user: `https://short.ly/abc1234`

### URL Redirection Flow (Critical Path)
1. User clicks short URL: `https://short.ly/abc1234`
2. Request hits Load Balancer → Redirection Service
3. Extract short code: `abc1234`
4. Check Redis cache for mapping:
   - **Cache Hit (90%)**: Get long URL from cache
   - **Cache Miss (10%)**: Query database, update cache
5. Validate URL is active and not expired
6. Perform HTTP 302 redirect to long URL
7. Asynchronously log click event to message queue (Kafka)
8. Analytics Service consumes event and stores in analytics DB
9. User lands on original long URL

**Optimization**: Serve redirect directly from CDN edge (for very hot URLs)

### Analytics Query Flow
1. User requests analytics for short URL
2. Analytics Service queries analytics database
3. Aggregate clicks by time period (hourly, daily, monthly)
4. Generate charts and statistics
5. Cache aggregated results (5-minute TTL)
6. Return analytics to user

## Scalability Considerations

### Horizontal Scaling
- **Stateless Services**: All services can scale independently
- **Load Balancing**: Distribute traffic across instances
- **Auto-Scaling**: Scale based on CPU/memory/request rate

### Database Scaling
- **Sharding**: Shard by `short_code` (hash-based or range-based)
- **Read Replicas**: Scale read-heavy redirection queries
- **Caching**: Redis cluster for distributed caching
- **Partitioning**: Partition analytics data by time

### Caching Strategy
- **Cache Hot URLs**: Use LRU eviction policy
- **Cache Warming**: Pre-load popular URLs
- **Multi-Level Cache**:
  - L1: CDN edge cache (geo-distributed)
  - L2: Application-level cache (Redis)
  - L3: Database query cache
- **Cache Invalidation**: On URL deletion or expiration

### Geographic Distribution
- Deploy in multiple regions (US, EU, Asia)
- Use geo-based DNS routing (Route53, Cloudflare)
- Replicate read-only data to regions
- Write to primary region, replicate asynchronously

### CDN Integration
- Cache redirection responses at CDN edge
- Reduce origin server load
- Lower latency for users globally
- Cloudflare, Fastly, AWS CloudFront

## Technology Stack Recommendations

### Backend
- **Languages**: Go (high performance), Java, Python, Node.js
- **Framework**: Gin (Go), Spring Boot (Java), FastAPI (Python)
- **Protocol**: HTTP/REST, gRPC for inter-service

### Databases
- **Relational**: PostgreSQL, MySQL
- **NoSQL**: Cassandra, DynamoDB, MongoDB
- **Cache**: Redis, Memcached
- **Analytics**: ClickHouse, TimescaleDB, Cassandra

### Infrastructure
- **Container Orchestration**: Kubernetes
- **Service Mesh**: Istio (optional)
- **Message Queue**: Apache Kafka, RabbitMQ, AWS SQS
- **Monitoring**: Prometheus, Grafana, Datadog
- **Logging**: ELK Stack, Loki

### Frontend
- **Web**: React, Vue.js, Svelte
- **Mobile**: Swift (iOS), Kotlin (Android), Flutter

### CDN & DNS
- **CDN**: Cloudflare, Fastly, AWS CloudFront
- **DNS**: Route53, Cloudflare DNS
- **DDoS Protection**: Cloudflare, AWS Shield

## Security Best Practices

### Malicious URL Protection
- **URL Validation**: Check against blacklists
- **Virus Scanning**: Integrate with VirusTotal API
- **Phishing Detection**: Machine learning models
- **User Reporting**: Allow users to report malicious links
- **Automatic Blocking**: Disable reported URLs

### Abuse Prevention
- **Rate Limiting**: Per IP and per user
- **CAPTCHA**: For anonymous users (high volume)
- **API Keys**: Throttle API usage
- **Monitoring**: Detect spam patterns

### Authentication & Authorization
- **JWT Tokens**: Secure API access
- **OAuth 2.0**: Third-party integrations
- **API Keys**: For programmatic access
- **Role-Based Access**: Free, premium, enterprise tiers

### Data Security
- **HTTPS Only**: Enforce TLS for all connections
- **Input Validation**: Sanitize URLs (prevent XSS)
- **SQL Injection Protection**: Parameterized queries
- **DDoS Protection**: Rate limiting and CDN

### Privacy
- **Anonymized Analytics**: Hash IP addresses
- **GDPR Compliance**: Allow data deletion
- **Data Retention**: Auto-delete old analytics
- **Opt-Out**: Allow users to disable tracking

## Monitoring & Observability

### Key Metrics
- **Performance**:
  - Redirection latency (p50, p95, p99)
  - API response time
  - Cache hit rate
  - Database query latency
- **Business**:
  - URLs shortened per day
  - Total clicks per day
  - Active users
  - Most popular short URLs
- **System**:
  - Service uptime (SLA: 99.99%)
  - Error rate per endpoint
  - Database connection pool utilization
  - Cache memory usage

### Alerting
- High error rate on redirection
- Cache hit rate drops below threshold
- Database replication lag
- High API latency
- Rate limit breaches

### Real-Time Dashboards
- Live click heatmap
- Top short URLs by clicks
- Geographic distribution of clicks
- Device and browser breakdown

## Trade-offs and Considerations

### URL Encoding Strategy
- **Counter-Based**: Simple, fast, but sequential (predictable)
- **Hash-Based**: Non-sequential, but collision possible
- **Trade-off**: Use counter for performance, add randomness if needed

### Redirect Type (301 vs 302)
- **301 (Permanent)**: Browsers cache, faster, but analytics lost
- **302 (Temporary)**: No browser cache, track every click
- **Trade-off**: Use 302 for analytics, 301 for SEO (configurable)

### Consistency vs Availability
- **Strong Consistency**: Ensure URL exists before returning
- **Eventual Consistency**: Faster writes, rare edge cases
- **Trade-off**: Strong for writes, eventual for analytics

### Deduplication
- **Enable**: Save storage, same short URL for same long URL
- **Disable**: Each request gets unique short URL (more flexible)
- **Trade-off**: Enable for public service, disable for user-specific URLs

### Data Retention
- **Keep Forever**: Simple, but storage grows indefinitely
- **Expire Inactive**: Save storage, but URLs may break
- **Trade-off**: Configurable expiration, default to long retention

## Future Enhancements

1. **Link-in-Bio**: Single page with multiple links
2. **A/B Testing**: Split traffic between URLs
3. **Password Protection**: Private short URLs
4. **Branded Domains**: Custom domains for enterprise
5. **UTM Parameter Injection**: Auto-add tracking parameters
6. **Retargeting Pixels**: Add marketing pixels
7. **Link Preview**: Open Graph metadata display
8. **API Rate Limit Tiers**: Free, pro, enterprise
9. **Blockchain**: Decentralized URL shortening
10. **AI-Powered Analytics**: Predict click patterns

## References

- [TinyURL System Design](https://www.youtube.com/watch?v=fMZMm_0ZhK4)
- [Bitly Engineering Blog](https://bitly.com/blog/)
- [System Design Interview: URL Shortener](https://www.educative.io/courses/grokking-the-system-design-interview)
- [Base62 Encoding](https://en.wikipedia.org/wiki/Base62)
- [Zookeeper for Distributed Coordination](https://zookeeper.apache.org/)
