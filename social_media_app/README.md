# Social Media App System Design

## Overview

A social networking platform that enables users to connect, share content, and interact with each other, similar to Facebook, Instagram, or Twitter. The system supports user profiles, posts, feeds, comments, likes, follows, and real-time notifications.

## Requirements

### Functional Requirements
- User registration and authentication
- User profiles with photos and bio
- Post creation (text, images, videos)
- News feed generation
- Follow/unfollow users
- Like, comment, and share posts
- Real-time notifications
- Direct messaging
- Search users and content
- Trending topics and hashtags
- Stories (temporary content)
- Groups and communities

### Non-Functional Requirements
- **High Availability**: 99.99% uptime
- **Low Latency**: Feed loads within 200ms
- **Scalability**: Support 1B+ users, 500M+ daily active users
- **Consistency**: Eventual consistency acceptable for feeds
- **Performance**: Handle 100K+ posts per second
- **Reliability**: No data loss for posts and messages

## System Architecture

### Core Components

#### 1. Client Applications
- **Web Application**: Responsive web interface
- **Mobile Apps**: iOS and Android native apps
- **PWA**: Progressive Web App for offline support
- Uses REST APIs and WebSocket for real-time updates

#### 2. API Gateway
- Single entry point for all client requests
- Authentication and authorization
- Rate limiting per user/IP
- Request validation and sanitization
- API versioning and routing
- Load balancing across services

#### 3. User Service
- User registration and login
- Profile management (name, bio, profile picture)
- Account settings and privacy controls
- Email/phone verification
- Password reset functionality
- OAuth integration (Google, Facebook login)

#### 4. Post Service
- Create, read, update, delete posts
- Post metadata (timestamp, location, tags)
- Post visibility settings (public, friends, private)
- Rich media handling (text, images, videos, links)
- Post editing and deletion
- Post scheduling (future posts)

#### 5. Feed Generation Service
- Generates personalized news feed for each user
- Implements feed ranking algorithms
- Pull-based (on-demand) and push-based (pre-computed) approaches
- Fan-out on write for small follower counts
- Fan-out on read for celebrities
- Hybrid approach for optimal performance

#### 6. Timeline Service
- Stores and retrieves user timelines
- Chronological and algorithmic sorting
- Pagination support
- Cache invalidation strategies

#### 7. Social Graph Service
- Manages follower/following relationships
- Friend requests and connections
- Block and mute functionality
- Suggests friends/users to follow
- Graph traversal for connections

#### 8. Interaction Service
- Handles likes, comments, shares
- Reaction types (like, love, angry, etc.)
- Comment threading and nesting
- Edit and delete interactions
- Interaction counts and aggregations

#### 9. Notification Service
- Real-time notifications (new followers, likes, comments)
- Push notifications to mobile devices
- In-app notification center
- Email and SMS notifications
- Notification preferences and settings
- Batching to reduce noise

#### 10. Search Service
- Full-text search for users, posts, hashtags
- Autocomplete and suggestions
- Advanced filters (date, location, media type)
- Trending topics detection
- Elasticsearch or similar for indexing

#### 11. Media Service
- Image and video uploads
- Media processing (resize, compress, thumbnail)
- Content moderation (detect inappropriate content)
- CDN integration for fast delivery
- Supports multiple formats and resolutions

#### 12. Messaging Service
- One-on-one and group messaging
- Real-time message delivery
- Message read receipts
- Typing indicators
- Message history and search

#### 13. Analytics Service
- User engagement metrics
- Content performance analytics
- A/B testing framework
- Real-time dashboards
- Business intelligence reporting

## Data Storage

### Primary Database (Relational)
- **Technology**: PostgreSQL or MySQL
- User accounts and profiles
- User settings and preferences
- Relationship data (followers, friends)
- Master-slave replication
- Sharded by user_id

### Post Database (NoSQL)
- **Technology**: Cassandra or DynamoDB
- Stores posts and their metadata
- Wide-column store for high write throughput
- Partitioned by user_id or post_id
- Optimized for time-series data

### Social Graph Database
- **Technology**: Neo4j or JanusGraph
- Stores user relationships (following, followers)
- Efficient graph traversal
- Friend suggestions
- Degrees of separation queries

### Timeline Storage
- **Technology**: Redis or Cassandra
- Stores pre-computed timelines for users
- Fast read access for feed generation
- TTL for automatic cleanup
- Stores post_ids, not full post content

### Interaction Database
- **Technology**: Cassandra or MongoDB
- Stores likes, comments, shares
- High write volume
- Aggregation for counts
- Partitioned by post_id

### Cache Layer
- **Technology**: Redis or Memcached
- Caches user profiles
- Recent posts and hot content
- Feed results
- Interaction counts
- Reduces database load by 70-80%

### Object Storage
- **Technology**: AWS S3, Google Cloud Storage
- Stores media files (images, videos)
- CDN integration for fast delivery
- Different storage tiers for cost optimization

### Search Index
- **Technology**: Elasticsearch
- Indexes posts, users, hashtags
- Full-text search capabilities
- Near real-time indexing
- Supports complex queries

## Data Flow

### Post Creation
1. User creates post via client app
2. API Gateway authenticates and validates request
3. Post Service receives and validates post content
4. Media Service processes any attached media:
   - Upload to object storage
   - Generate thumbnails
   - Run content moderation
5. Post stored in Post Database with unique post_id
6. Post indexed in Search Service for discoverability
7. Fan-out begins (based on follower count):
   - **Small following**: Fan-out on write (push to followers' timelines)
   - **Large following**: No immediate fan-out (pull on demand)
8. Notification Service notifies mentioned users
9. Analytics Service records post creation event

### Feed Generation (Hybrid Approach)

**Fan-out on Write (for users with < 10K followers)**:
1. When user creates post, post_id added to all followers' timelines
2. Timeline Service pushes to Redis (one entry per follower)
3. Followers' feed reads from pre-computed timeline
4. Fast read, expensive write

**Fan-out on Read (for celebrities with > 10K followers)**:
1. Post stored in Post Database only
2. When follower requests feed, celebrity posts fetched on-demand
3. Merged with regular timeline
4. Slow read, cheap write

**Feed Retrieval**:
1. User opens app and requests feed
2. Timeline Service retrieves post_ids from cache/database
3. Post Service batch-fetches full post details
4. Ranking algorithm applies:
   - Chronological vs algorithmic
   - Factors: recency, engagement, user affinity
5. Feed returned to client with pagination
6. Client renders posts with media from CDN

### Interaction (Like/Comment)
1. User likes or comments on a post
2. Interaction Service records in Interaction Database
3. Asynchronous update to interaction counts (eventual consistency)
4. Notification Service notifies post author
5. Cache updated for interaction counts
6. Client shows optimistic update immediately

### Real-Time Notifications
1. Event occurs (new like, comment, follower)
2. Notification Service receives event from message queue
3. Check user's notification preferences
4. Format notification message
5. Push via WebSocket to connected clients
6. Send push notification to mobile devices (FCM, APNs)
7. Store in notification database for later retrieval
8. Badge count updated

## Scalability Considerations

### Database Sharding
- **User Service**: Shard by user_id
- **Post Service**: Shard by user_id or post_id
- **Timeline Service**: Shard by user_id
- Consistent hashing for shard assignment

### Caching Strategy
- **Multi-Level Caching**:
  - L1: Client-side cache (app memory)
  - L2: CDN for static content
  - L3: Redis for hot data
  - L4: Database read replicas
- **Cache Invalidation**: Time-based TTL and event-driven invalidation

### Feed Generation Optimization
- Pre-compute feeds for active users
- On-demand computation for inactive users
- Machine learning models for ranking
- A/B test different algorithms

### Horizontal Scaling
- Stateless services for easy scaling
- Auto-scaling based on CPU/memory/request metrics
- Geographic distribution (multi-region deployment)

### Asynchronous Processing
- Message queues (Kafka, RabbitMQ) for:
  - Post fan-out
  - Notification delivery
  - Analytics events
  - Search indexing
- Batch processing for non-critical tasks

### CDN for Media Delivery
- Serve images and videos from edge locations
- Reduce origin server load
- Faster load times globally
- Image optimization and lazy loading

## Technology Stack Recommendations

### Backend
- **Languages**: Java (Spring Boot), Go, Python, Node.js
- **API Style**: REST, GraphQL
- **Message Queue**: Apache Kafka, RabbitMQ, AWS SQS
- **Task Queue**: Celery, Bull (for Node.js)

### Databases
- **Relational**: PostgreSQL, MySQL
- **NoSQL**: Cassandra, DynamoDB, MongoDB
- **Graph**: Neo4j, JanusGraph
- **Cache**: Redis, Memcached
- **Search**: Elasticsearch

### Frontend
- **Web**: React, Vue.js, Angular
- **Mobile**: Swift (iOS), Kotlin (Android), React Native, Flutter
- **State Management**: Redux, MobX, Vuex

### Infrastructure
- **Container Orchestration**: Kubernetes
- **Service Mesh**: Istio
- **Monitoring**: Prometheus, Grafana, Datadog
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Tracing**: Jaeger, Zipkin

### CDN & Storage
- **CDN**: Cloudflare, AWS CloudFront, Fastly
- **Object Storage**: AWS S3, Google Cloud Storage
- **Image Processing**: Cloudinary, Imgix

### Machine Learning
- **Recommendation**: TensorFlow, PyTorch
- **Content Moderation**: AWS Rekognition, Google Cloud Vision
- **NLP**: spaCy, BERT for text analysis

## Security Best Practices

### Authentication & Authorization
- **JWT Tokens**: Secure session management
- **OAuth 2.0**: Third-party login
- **Two-Factor Authentication**: SMS or app-based
- **Rate Limiting**: Prevent brute force attacks

### Data Security
- **Encryption in Transit**: TLS 1.3
- **Encryption at Rest**: Sensitive data encrypted
- **Data Masking**: PII masked in logs
- **GDPR Compliance**: User data protection and privacy

### Content Moderation
- **Automated**: AI/ML for detecting inappropriate content
- **Manual Review**: Human moderators for flagged content
- **User Reporting**: Allow users to report violations
- **Spam Detection**: Filter spam posts and comments

### API Security
- **Input Validation**: Sanitize all user inputs (prevent XSS, SQL injection)
- **CORS**: Properly configured
- **API Keys**: For service-to-service communication
- **DDoS Protection**: CloudFlare, AWS Shield

### Privacy Controls
- **Profile Privacy**: Public, friends-only, private
- **Block and Mute**: User-controlled blocking
- **Data Export**: Allow users to download their data
- **Account Deletion**: Permanent data removal option

## Monitoring & Observability

### Key Metrics
- **Performance**:
  - Feed load time (p50, p95, p99)
  - API response times
  - Database query latency
  - Cache hit rate
- **Business**:
  - Daily/Monthly Active Users (DAU/MAU)
  - Post creation rate
  - Engagement rate (likes, comments, shares)
  - Time spent on platform
- **System**:
  - Service uptime
  - Error rates per service
  - Message queue lag
  - CPU/Memory utilization

### Alerting
- High error rates
- Service downtime
- Feed generation latency spikes
- Database replication lag
- Cache failures

### A/B Testing
- Test feed algorithms
- UI/UX changes
- Notification strategies
- Feature rollouts

## Trade-offs and Considerations

### Consistency vs Availability
- **Choice**: Eventual consistency for most features
- **Reason**: Prioritize availability; slight delays acceptable
- **Implementation**: Message queues for async updates

### Fan-out Strategy
- **Fan-out on Write**: Fast reads, slow writes, high storage
- **Fan-out on Read**: Fast writes, slow reads, low storage
- **Hybrid**: Balance based on follower count

### Real-Time vs Batch Processing
- **Real-Time**: Notifications, messaging (critical user experience)
- **Batch**: Analytics, recommendations (can tolerate delay)
- **Cost**: Real-time more expensive, batch more cost-effective

### Algorithmic vs Chronological Feed
- **Chronological**: Simple, transparent, may miss important content
- **Algorithmic**: Personalized, higher engagement, can feel manipulative
- **Solution**: Offer user choice or hybrid approach

## Future Enhancements

1. **Live Streaming**: Broadcast video to followers
2. **Stories**: Temporary 24-hour content
3. **Augmented Reality Filters**: Face filters and AR effects
4. **Marketplace**: Buy/sell within the platform
5. **Events**: Create and manage events
6. **Fundraising**: Crowdfunding campaigns
7. **Professional Profiles**: LinkedIn-style networking
8. **AI-Generated Content**: Auto-complete posts, captions
9. **Blockchain**: Decentralized social media
10. **Metaverse Integration**: VR social spaces

## References

- [Facebook Engineering Blog](https://engineering.fb.com/)
- [Instagram Engineering](https://instagram-engineering.com/)
- [Twitter System Design](https://blog.twitter.com/engineering)
- [Designing Data-Intensive Applications](https://dataintensive.net/)
- [System Design Interview Book](https://www.amazon.com/System-Design-Interview-insiders-Second/dp/B08CMF2CQF)
