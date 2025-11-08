# Messaging Service System Design

## Overview

A real-time messaging service that enables users to send and receive messages instantly, similar to WhatsApp, Telegram, or Slack. The system supports one-on-one chats, group conversations, message delivery status, and multimedia content.

## Requirements

### Functional Requirements
- User registration and authentication
- One-on-one messaging
- Group chat functionality
- Message delivery and read receipts
- Online/offline status indicators
- Support for text, images, videos, and files
- Message history and search
- Push notifications

### Non-Functional Requirements
- **High Availability**: 99.99% uptime
- **Low Latency**: Messages delivered within 100ms
- **Scalability**: Support millions of concurrent users
- **Consistency**: Messages delivered in order
- **Security**: End-to-end encryption support

## System Architecture

### Core Components

#### 1. Client Applications
- **Web Client**: Browser-based interface
- **Mobile Apps**: iOS and Android native apps
- **Desktop Apps**: Cross-platform desktop applications
- Uses WebSocket for real-time bidirectional communication

#### 2. API Gateway
- Entry point for all client requests
- Handles authentication and authorization
- Rate limiting and request validation
- Routes requests to appropriate microservices
- Load balances incoming traffic

#### 3. Authentication Service
- User registration and login
- Token generation (JWT)
- Session management
- OAuth integration for social login

#### 4. User Service
- User profile management
- Contact list management
- User presence (online/offline status)
- User search functionality

#### 5. Message Service
- Core messaging logic
- Message validation and processing
- Message routing to recipients
- Temporary message storage

#### 6. WebSocket/Connection Service
- Maintains persistent connections with clients
- Real-time message delivery
- Connection state management
- Handles reconnection logic
- Scales horizontally with consistent hashing

#### 7. Group Service
- Group creation and management
- Member management (add/remove)
- Group metadata storage
- Group permissions and roles

#### 8. Notification Service
- Push notifications for offline users
- Email notifications
- Integration with FCM (Firebase Cloud Messaging) and APNs
- Notification preferences management

#### 9. Media Service
- Handles file uploads (images, videos, documents)
- Media compression and optimization
- Thumbnail generation
- Integration with CDN for media delivery

#### 10. Message Queue (Message Broker)
- **Technology**: Apache Kafka or RabbitMQ
- Ensures reliable message delivery
- Decouples services
- Handles message persistence
- Supports pub-sub patterns for group messages

## Data Storage

### Primary Database (User & Group Data)
- **Technology**: PostgreSQL or MySQL
- Stores user profiles, contacts, group metadata
- Master-slave replication for read scalability
- Sharded by user_id for horizontal scaling

### Message Store
- **Technology**: Cassandra or MongoDB
- Stores message history
- Wide-column store optimized for write-heavy workloads
- Partitioned by conversation_id
- TTL for automatic message cleanup (optional)

### Cache Layer
- **Technology**: Redis
- Caches user sessions
- Online/offline status
- Recent messages for quick retrieval
- Group member lists
- Reduces database load

### Object Storage
- **Technology**: Amazon S3, Google Cloud Storage, or MinIO
- Stores media files (images, videos, documents)
- Integrated with CDN for fast global delivery

## Data Flow

### Sending a Message (One-on-One)
1. User A sends message via WebSocket connection
2. Connection Service receives and forwards to Message Service
3. Message Service validates and enriches message (timestamp, message_id)
4. Message published to Message Queue
5. Message Service stores message in Message Store (Cassandra)
6. Message Queue consumer checks User B's connection status:
   - **Online**: Delivers via WebSocket immediately
   - **Offline**: Stores in pending messages queue
7. Notification Service sends push notification if User B is offline
8. Delivery receipt sent back to User A

### Sending a Message (Group Chat)
1. User A sends message to group
2. Message Service validates group membership
3. Message stored in Message Store with group_id
4. Message published to group topic in Message Queue
5. Multiple consumers process message for each group member
6. Online members receive via WebSocket
7. Offline members get push notifications
8. Message delivery tracked per member

### User Coming Online
1. Client establishes WebSocket connection
2. User Service updates presence status in Redis
3. Connection Service retrieves pending messages
4. Pending messages delivered to client
5. Delivery receipts sent to original senders

## Scalability Considerations

### Horizontal Scaling
- **Stateless Services**: All services designed to be stateless
- **Connection Service**: Use consistent hashing to map users to connection servers
- **Database Sharding**: Partition data by user_id or conversation_id
- **Message Queue Partitioning**: Distribute load across multiple brokers

### Load Balancing
- **Layer 7 Load Balancer**: For HTTP/HTTPS traffic (API Gateway)
- **Layer 4 Load Balancer**: For WebSocket connections
- Sticky sessions for WebSocket connections
- Health checks and automatic failover

### Caching Strategy
- **Cache user sessions**: Reduce authentication overhead
- **Cache recent messages**: Speed up message retrieval
- **Cache user presence**: Quick online/offline status checks
- **Cache-aside pattern**: Read-through caching for frequently accessed data

### Geographic Distribution
- Deploy services in multiple regions
- Use CDN for media delivery
- Geographic routing for low latency
- Cross-region data replication for disaster recovery

## Technology Stack Recommendations

### Backend
- **Languages**: Go, Java (Spring Boot), or Node.js
- **Message Queue**: Apache Kafka or RabbitMQ
- **WebSocket Framework**: Socket.io, WebSocket API

### Databases
- **Relational DB**: PostgreSQL or MySQL
- **NoSQL DB**: Cassandra, MongoDB, or ScyllaDB
- **Cache**: Redis or Memcached

### Infrastructure
- **Container Orchestration**: Kubernetes
- **Service Mesh**: Istio (optional, for advanced routing)
- **Monitoring**: Prometheus, Grafana
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)

### Cloud Services
- **Object Storage**: AWS S3, Google Cloud Storage
- **CDN**: CloudFlare, AWS CloudFront
- **Push Notifications**: Firebase Cloud Messaging, Apple Push Notification Service

## Security Best Practices

### Data Security
- **End-to-End Encryption**: Signal Protocol or similar
- **TLS/SSL**: All communications encrypted in transit
- **Data at Rest**: Encrypted storage for sensitive data
- **Key Management**: Hardware Security Modules (HSM)

### Authentication & Authorization
- **JWT Tokens**: Short-lived access tokens
- **Refresh Tokens**: Long-lived, stored securely
- **OAuth 2.0**: For third-party integrations
- **Multi-Factor Authentication**: Optional security layer

### API Security
- **Rate Limiting**: Prevent abuse and DDoS
- **Input Validation**: Sanitize all user inputs
- **CORS**: Properly configured for web clients
- **API Keys**: For service-to-service communication

## Monitoring & Observability

### Key Metrics
- Message delivery latency (p50, p95, p99)
- WebSocket connection count
- Message throughput (messages/second)
- Error rates per service
- Database query performance
- Cache hit ratio

### Alerting
- Service downtime alerts
- High error rate alerts
- Database replication lag
- Message queue backlog
- Resource utilization (CPU, memory)

### Logging
- Structured logging (JSON format)
- Correlation IDs for tracing requests
- Log aggregation and search
- Audit logs for security events

## Trade-offs and Considerations

### Consistency vs Availability
- **Choice**: Eventual consistency for message delivery
- **Reason**: Prioritize availability; messages can be delivered with slight delay
- **Implementation**: Use message queue for reliable async delivery

### Read vs Write Optimization
- **Challenge**: Write-heavy workload (new messages)
- **Solution**: Use Cassandra (write-optimized) for message storage
- **Trade-off**: Query flexibility limited; use separate read models if needed

### Cost vs Performance
- **Caching**: Reduces database load but increases infrastructure cost
- **CDN**: Fast media delivery but adds cost for bandwidth
- **Multi-region**: Lower latency but higher operational complexity

### Stateful Connections
- **Challenge**: WebSocket connections are stateful
- **Solution**: Use consistent hashing and session affinity
- **Trade-off**: Harder to scale, but necessary for real-time delivery

## Future Enhancements

1. **Voice and Video Calls**: WebRTC integration
2. **Message Reactions**: Emoji reactions and polls
3. **Chatbots**: Integration with AI-powered bots
4. **Message Translation**: Real-time language translation
5. **Stories/Status**: Temporary content sharing
6. **Desktop Screen Sharing**: For collaboration
7. **Advanced Search**: Full-text search with filters
8. **Message Scheduling**: Send messages at specified times

## References

- [WhatsApp System Design](https://www.youtube.com/watch?v=vvhC64hQZMk)
- [Building Scalable Messaging Systems](https://engineering.fb.com/2018/06/26/core-data/scaling-the-instagram-explore-page/)
- [Signal Protocol for E2E Encryption](https://signal.org/docs/)
- [Apache Kafka for Messaging](https://kafka.apache.org/documentation/)
