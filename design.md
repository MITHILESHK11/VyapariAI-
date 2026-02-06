# VyapariAI – System Design Document

## 1. System Architecture Overview

### 1.1 High-Level Description

VyapariAI is a cloud-native, microservices-based platform built on AWS infrastructure that delivers AI-powered business intelligence to small retailers across India. The system follows a modern three-tier architecture with clear separation of concerns:

- **Presentation Layer**: Progressive Web App (PWA) with responsive design for mobile and web access
- **Application Layer**: RESTful API services built with microservices architecture
- **Data Layer**: Multi-database strategy combining relational, NoSQL, and time-series databases
- **Intelligence Layer**: ML/AI models for forecasting, optimization, and recommendations

### 1.2 Modular and Scalable Architecture

The platform is designed with modularity and scalability as core principles:

**Microservices Architecture:**
- Each business capability is implemented as an independent, loosely-coupled service
- Services communicate via REST APIs and asynchronous message queues
- Independent scaling based on service-specific load patterns
- Fault isolation prevents cascading failures

**Key Service Modules:**
- **User Service**: Authentication, authorization, profile management
- **Business Service**: Business registration, compliance tracking, onboarding
- **Inventory Service**: Product catalog, stock management, batch tracking
- **Sales Service**: Transaction recording, sales analytics, revenue tracking
- **Forecasting Service**: Demand prediction, trend analysis, seasonality detection
- **Pricing Service**: Price optimization, competitor analysis, margin calculation
- **Customer Service**: Customer analytics, segmentation, retention insights
- **Notification Service**: Multi-channel alerts (SMS, WhatsApp, email, push)
- **Reporting Service**: Dashboard data aggregation, report generation
- **AI Orchestration Service**: Model training, inference coordination, feedback loop

### 1.3 Cloud-Native Design Using AWS

The platform leverages AWS managed services to minimize operational overhead and maximize reliability:

**Compute**: Containerized services on Amazon ECS with Fargate for serverless container orchestration
**Storage**: Multi-tier storage strategy using RDS, DynamoDB, S3, and ElastiCache
**AI/ML**: Amazon SageMaker for model training and hosting, AWS Forecast for time-series predictions
**Integration**: API Gateway for API management, EventBridge for event-driven workflows
**Monitoring**: CloudWatch for logging and metrics, X-Ray for distributed tracing
**Security**: Cognito for user management, Secrets Manager for credentials, KMS for encryption

---

## 2. Architecture Diagram Description (Textual)

### 2.1 Component Overview and Data Flow

```
[Mobile/Web Client (PWA)]
         |
         | HTTPS/TLS 1.3
         v
[Amazon CloudFront CDN] --> [S3: Static Assets]
         |
         v
[AWS WAF - Security Layer]
         |
         v
[Amazon API Gateway]
         |
         +---> [AWS Cognito] (Authentication)
         |
         v
[Application Load Balancer]
         |
         +------------------+------------------+------------------+
         |                  |                  |                  |
         v                  v                  v                  v
   [ECS: User Service]  [ECS: Sales Service]  [ECS: Inventory]  [ECS: Forecasting]
         |                  |                  |                  |
         v                  v                  v                  v
[Amazon RDS PostgreSQL] - Primary relational database
         |
         +---> [Read Replicas] (for analytics queries)
         |
[Amazon DynamoDB] - NoSQL for high-velocity writes (sales transactions)
         |
[Amazon S3] - Object storage (reports, backups, ML training data)
         |
[Amazon ElastiCache Redis] - Session management, caching
         |
         v
[Amazon SageMaker] - ML model training and hosting
         |
         +---> [AWS Forecast] - Time-series demand prediction
         |
[AWS Lambda] - Serverless functions for:
         - Data preprocessing
         - Model inference triggers
         - Notification dispatch
         - Scheduled jobs
         |
         v
[Amazon SNS] --> [SMS Gateway, Email (SES), WhatsApp Business API]
         |
[Amazon EventBridge] - Event-driven orchestration
         |
[Amazon QuickSight] - BI dashboards and analytics
         |
[Amazon CloudWatch] - Monitoring, logging, alerting
```

### 2.2 Detailed Data Flow

**Step 1: User Authentication**
- User accesses PWA via CloudFront CDN
- Login request routed through API Gateway to Cognito
- JWT token issued and cached in Redis for session management

**Step 2: Sales Data Ingestion**
- Retailer enters sales transaction via mobile interface
- API Gateway validates request and routes to Sales Service
- Sales Service writes to DynamoDB (low-latency write)
- Asynchronous Lambda function syncs data to RDS for analytics
- Inventory Service receives event via EventBridge and updates stock levels

**Step 3: AI Model Inference**
- Scheduled Lambda function triggers daily forecasting job
- Forecasting Service fetches historical data from RDS
- Data preprocessed and sent to SageMaker endpoint
- AWS Forecast generates 7-14 day demand predictions
- Results stored in RDS and cached in Redis

**Step 4: Decision Engine**
- Pricing Service analyzes forecast + current inventory + competitor data
- Generates price optimization recommendations
- Inventory Service evaluates stock levels against forecasts
- Triggers alerts for low stock, overstock, or expiry warnings
- Customer Service analyzes purchase patterns and generates retention insights

**Step 5: Notification Delivery**
- Notification Service receives alert events from EventBridge
- Routes notifications based on user preferences (SMS/WhatsApp/Email/Push)
- SNS publishes messages to appropriate channels
- Delivery status tracked and logged in CloudWatch

**Step 6: Dashboard and Reporting**
- User requests dashboard data via API Gateway
- Reporting Service aggregates data from Redis cache (hot data) and RDS (historical)
- QuickSight embedded dashboards for advanced analytics
- Real-time updates via WebSocket connections (API Gateway WebSocket API)

---

## 3. AI & ML Design

### 3.1 Time-Series Forecasting Model for Demand Prediction

**Model Architecture:**
- **Primary Model**: AWS Forecast with DeepAR+ algorithm (probabilistic forecasting)
- **Fallback Model**: Custom LSTM neural network trained on SageMaker for edge cases

**Model Inputs:**
- Historical sales data (product-level, daily granularity, minimum 30 days)
- Temporal features: day of week, month, season, holidays, local festivals
- Product attributes: category, price point, perishability
- External factors: weather data (via API), local events, competitor promotions
- Store metadata: location, size, customer demographics

**Model Outputs:**
- Point forecasts (mean predicted demand) for next 7-14 days
- Prediction intervals (P10, P50, P90 quantiles) for uncertainty quantification
- Confidence scores for each prediction

**Training Approach:**
- Initial training: Batch training on historical data (minimum 90 days)
- Retraining frequency: Weekly for active users, monthly for low-activity users
- Incremental learning: Daily updates using online learning for high-volume stores
- Cold start handling: Category-level models for new products, collaborative filtering

**Model Evaluation:**
- Metrics: MAPE (Mean Absolute Percentage Error), RMSE, Weighted Quantile Loss
- Target: MAPE < 20% for 7-day forecasts, < 30% for 14-day forecasts
- A/B testing: Compare model versions on holdout set before deployment

### 3.2 Regression Models for Price Optimization

**Model Architecture:**
- **Price Elasticity Model**: XGBoost regression to estimate demand sensitivity to price changes
- **Competitive Pricing Model**: Random Forest to predict optimal price based on competitor analysis
- **Margin Optimization Model**: Multi-objective optimization (maximize margin while maintaining volume)

**Model Inputs:**
- Historical pricing data and corresponding sales volumes
- Competitor prices (scraped from online sources, user-reported)
- Product cost price and current inventory levels
- Demand forecast from time-series model
- Market segment and customer price sensitivity
- Seasonality and promotional calendar

**Model Outputs:**
- Recommended selling price with expected impact on sales volume
- Predicted profit margin at recommended price
- Price elasticity coefficient (% change in demand per 1% price change)
- Competitive positioning (percentile rank vs. market)
- Alternative pricing scenarios (aggressive, moderate, conservative)

**Training Approach:**
- Training data: Historical price-volume pairs with minimum 60 days of data
- Feature engineering: Price ratios, moving averages, lag features
- Retraining: Bi-weekly to capture market dynamics
- Personalization: Store-specific models for high-volume retailers

**Optimization Strategy:**
- Constraint-based optimization: Minimum margin thresholds, maximum price deviation
- Multi-armed bandit approach: Explore-exploit tradeoff for price testing
- Feedback loop: Track adoption rate of recommendations and actual outcomes

### 3.3 Customer Clustering and Recommendation Logic

**Clustering Model:**
- **Algorithm**: K-Means clustering with optimal K selection via Elbow method and Silhouette score
- **Alternative**: DBSCAN for density-based clustering when customer base is heterogeneous

**Clustering Features:**
- RFM metrics: Recency (days since last purchase), Frequency (purchase count), Monetary (total spend)
- Product preferences: Category affinity, brand loyalty, price sensitivity
- Purchase patterns: Basket size, time of day, day of week preferences
- Lifecycle stage: New, active, at-risk, churned

**Customer Segments:**
- **VIP Customers**: High frequency, high value, recent purchases
- **Regular Customers**: Moderate frequency and value, consistent patterns
- **Occasional Customers**: Low frequency, sporadic purchases
- **At-Risk Customers**: Previously active but declining engagement
- **Churned Customers**: No purchases in last 90 days

**Recommendation Logic:**
- **Collaborative Filtering**: Item-item similarity based on co-purchase patterns
- **Content-Based Filtering**: Recommend products similar to customer's purchase history
- **Hybrid Approach**: Combine both methods with weighted scoring

**Personalization Strategies:**
- VIP customers: Exclusive offers, early access to new products
- At-risk customers: Win-back campaigns, personalized discounts
- New customers: Onboarding offers, popular product recommendations
- Occasional customers: Reminder notifications, bundle deals

**Model Retraining:**
- Clustering: Monthly reclustering to capture evolving customer behavior
- Recommendations: Weekly updates based on latest purchase data
- Real-time personalization: Session-based recommendations using cached profiles

---

## 4. AWS Services Used

### 4.1 Amazon SageMaker

**Role**: End-to-end machine learning platform for model development, training, and deployment

**Usage in VyapariAI:**
- **SageMaker Notebooks**: Data exploration, feature engineering, model experimentation
- **SageMaker Training Jobs**: Distributed training of custom LSTM and XGBoost models
- **SageMaker Endpoints**: Real-time inference for price optimization and customer segmentation
- **SageMaker Batch Transform**: Bulk inference for nightly forecasting jobs
- **SageMaker Model Monitor**: Track model performance drift and data quality issues
- **SageMaker Pipelines**: Automate ML workflows from data prep to deployment

**Configuration:**
- Instance types: ml.m5.xlarge for training, ml.t3.medium for inference (cost-optimized)
- Auto-scaling: Scale inference endpoints based on request volume
- Multi-model endpoints: Host multiple models on single endpoint to reduce costs

### 4.2 AWS Forecast

**Role**: Fully managed time-series forecasting service using deep learning

**Usage in VyapariAI:**
- Primary demand forecasting engine for product-level predictions
- Handles seasonality, trends, and holiday effects automatically
- Generates probabilistic forecasts with confidence intervals

**Configuration:**
- Predictor: DeepAR+ algorithm for complex patterns, Prophet for simpler cases
- Forecast horizon: 7-14 days with daily granularity
- Forecast dimensions: Product ID, Store ID, Category
- Related time series: Price, promotions, weather data
- Automated retraining: Weekly schedule via Lambda triggers

**Data Requirements:**
- Target time series: Historical sales data (minimum 30 days, recommended 90+ days)
- Related time series: Price changes, promotional flags
- Item metadata: Product category, attributes, cost price

### 4.3 Amazon DynamoDB

**Role**: Fully managed NoSQL database for high-velocity, low-latency operations

**Usage in VyapariAI:**
- **Sales Transactions Table**: Real-time sales data ingestion with single-digit millisecond latency
- **User Sessions Table**: Active session management and caching
- **Notification Queue Table**: Pending notifications with TTL for automatic cleanup
- **Audit Logs Table**: Immutable audit trail for compliance

**Table Design:**
- Partition key: User ID / Store ID for even distribution
- Sort key: Timestamp for time-based queries
- Global Secondary Indexes (GSI): Product ID, Category for cross-user analytics
- DynamoDB Streams: Trigger Lambda functions for downstream processing

**Capacity Planning:**
- On-demand billing for unpredictable workloads (initial launch)
- Provisioned capacity with auto-scaling for steady-state operations
- DAX (DynamoDB Accelerator) for read-heavy workloads (dashboard queries)

### 4.4 Amazon RDS (PostgreSQL)

**Role**: Managed relational database for structured data and complex queries

**Usage in VyapariAI:**
- **Master Database**: User profiles, business metadata, product catalog
- **Analytics Database**: Aggregated sales data, historical trends, reporting
- **Compliance Database**: License tracking, audit logs, regulatory data

**Schema Design:**
- Normalized schema for transactional data (3NF)
- Denormalized views for analytics and reporting
- Partitioning: Time-based partitioning for sales data (monthly partitions)
- Indexes: B-tree indexes on frequently queried columns, GiST indexes for full-text search

**High Availability:**
- Multi-AZ deployment for automatic failover
- Read replicas in same region for read scaling (analytics queries)
- Automated backups with 7-day retention, manual snapshots for long-term archival
- Point-in-time recovery enabled

**Performance Optimization:**
- Connection pooling via RDS Proxy
- Query optimization with EXPLAIN ANALYZE
- Materialized views for complex aggregations
- pg_stat_statements for query performance monitoring

### 4.5 Amazon S3

**Role**: Scalable object storage for unstructured data and archival

**Usage in VyapariAI:**
- **ML Training Data Bucket**: Historical datasets for model training (versioned)
- **Reports Bucket**: Generated PDF/Excel reports with lifecycle policies
- **Backup Bucket**: Database backups, disaster recovery snapshots
- **Static Assets Bucket**: PWA assets, images, videos (served via CloudFront)
- **Data Lake Bucket**: Raw data for future analytics (Parquet format)

**Storage Classes:**
- S3 Standard: Frequently accessed data (reports, active ML datasets)
- S3 Intelligent-Tiering: Automatic cost optimization for variable access patterns
- S3 Glacier: Long-term archival (compliance data, old backups)

**Security:**
- Server-side encryption (SSE-S3) for all buckets
- Bucket policies restricting access to specific IAM roles
- Versioning enabled for critical data buckets
- Cross-region replication for disaster recovery

### 4.6 AWS Lambda

**Role**: Serverless compute for event-driven processing and scheduled tasks

**Usage in VyapariAI:**
- **Data Sync Functions**: DynamoDB Streams → RDS synchronization
- **Notification Dispatcher**: EventBridge → SNS routing logic
- **Model Inference Triggers**: Scheduled forecasting jobs (daily at 2 AM IST)
- **Data Preprocessing**: ETL for ML training data preparation
- **API Backends**: Lightweight API endpoints for infrequent operations
- **Scheduled Jobs**: Daily reports, weekly model retraining, monthly billing

**Function Configuration:**
- Runtime: Python 3.11 for ML/data processing, Node.js 20 for API functions
- Memory: 512MB-3GB based on workload (ML inference requires higher memory)
- Timeout: 15 minutes for long-running jobs, 30 seconds for API functions
- Concurrency: Reserved concurrency for critical functions to prevent throttling
- VPC Integration: Access to RDS and ElastiCache within private subnets

**Cost Optimization:**
- Use Lambda Layers for shared dependencies (reduces deployment package size)
- Implement caching to reduce redundant invocations
- Monitor CloudWatch metrics to right-size memory allocation

### 4.7 Amazon QuickSight

**Role**: Business intelligence and embedded analytics platform

**Usage in VyapariAI:**
- **Retailer Dashboards**: Embedded interactive dashboards in PWA
- **Admin Analytics**: Platform-wide metrics, user behavior analysis
- **Custom Reports**: Ad-hoc analysis for business insights

**Dashboard Components:**
- Sales performance: Revenue trends, top products, category breakdown
- Inventory health: Stock levels, turnover rates, wastage metrics
- Customer insights: Segmentation, retention rates, lifetime value
- Forecast accuracy: Predicted vs. actual comparison, model performance

**Data Sources:**
- Direct connection to RDS for real-time data
- S3 data lake for historical analysis (via Athena)
- SPICE (in-memory engine) for fast query performance

**Embedding:**
- QuickSight Embedding SDK integrated into React PWA
- Row-level security (RLS) ensures users see only their own data
- Anonymous embedding for public dashboards (if needed)

### 4.8 Amazon Cognito

**Role**: User authentication, authorization, and identity management

**Usage in VyapariAI:**
- **User Pools**: Manage retailer and admin user accounts
- **Identity Pools**: Federated identities for social login (Google, Facebook)
- **MFA**: Multi-factor authentication for admin accounts
- **Custom Attributes**: Store business metadata (store ID, subscription tier)

**Authentication Flow:**
- User registers with mobile number → OTP sent via SNS
- User verifies OTP → Cognito creates user account
- User logs in → Cognito issues JWT tokens (ID token, access token, refresh token)
- API Gateway validates JWT on each request
- Token refresh handled automatically by client SDK

**Authorization:**
- User groups: Retailer, Admin, Support
- IAM roles mapped to Cognito groups for fine-grained access control
- Custom claims in JWT for role-based access (RBAC)

**Security Features:**
- Password policies: Minimum 8 characters, complexity requirements
- Account lockout after 5 failed login attempts
- Adaptive authentication: Risk-based MFA challenges
- Advanced security features: Compromised credentials detection

### 4.9 Additional AWS Services

**Amazon API Gateway:**
- RESTful API management with request/response transformation
- WebSocket API for real-time dashboard updates
- Throttling and rate limiting (100 requests/minute per user)
- API key management for third-party integrations

**Amazon CloudFront:**
- Global CDN for low-latency content delivery
- Edge caching for static assets (PWA, images)
- SSL/TLS termination with AWS Certificate Manager
- DDoS protection via AWS Shield Standard

**Amazon EventBridge:**
- Event-driven architecture for service decoupling
- Custom event buses for different domains (sales, inventory, notifications)
- Event replay for debugging and recovery
- Integration with SaaS providers (future: WhatsApp, payment gateways)

**Amazon ElastiCache (Redis):**
- Session management and user state caching
- API response caching for frequently accessed data
- Real-time leaderboards and counters
- Pub/Sub for real-time notifications

**Amazon CloudWatch:**
- Centralized logging for all services
- Custom metrics for business KPIs (sales volume, forecast accuracy)
- Alarms for operational issues (high error rates, latency spikes)
- Dashboards for system health monitoring

**AWS Secrets Manager:**
- Secure storage for database credentials, API keys, encryption keys
- Automatic rotation of secrets (RDS passwords, API tokens)
- Integration with Lambda and ECS for seamless secret retrieval

---

## 5. Data Flow & Processing

### 5.1 Sales Data Ingestion Pipeline

**Step 1: Data Entry**
- User enters sales transaction via PWA (product, quantity, price, timestamp)
- Client-side validation: Required fields, data types, reasonable ranges
- Offline support: Data stored in IndexedDB if no connectivity
- Background sync: Automatic upload when connection restored

**Step 2: API Request**
- HTTPS POST request to API Gateway endpoint `/api/v1/sales`
- JWT token in Authorization header validated by Cognito
- API Gateway applies rate limiting and request throttling
- Request logged to CloudWatch for audit trail

**Step 3: Sales Service Processing**
- Sales Service (ECS container) receives request
- Business logic validation: Product exists, price within bounds, duplicate detection
- Transaction ID generated (UUID v4)
- Write to DynamoDB Sales Transactions table (partition key: user_id, sort key: timestamp)
- Acknowledge success to client (HTTP 201 Created)

**Step 4: Asynchronous Processing**
- DynamoDB Stream triggers Lambda function
- Lambda enriches data (product details, category, margin calculation)
- Writes to RDS for analytics (batch insert every 5 minutes for efficiency)
- Publishes event to EventBridge: `sales.transaction.created`

**Step 5: Downstream Updates**
- Inventory Service subscribes to sales event
- Updates stock levels in RDS (decrement quantity)
- Checks reorder thresholds, triggers alert if low stock
- Customer Service updates purchase history for customer analytics

**Step 6: Real-Time Dashboard Update**
- Reporting Service subscribes to sales event
- Updates cached aggregates in Redis (daily sales total, product counts)
- Pushes update to connected clients via WebSocket (API Gateway WebSocket API)
- Dashboard reflects new sale within 2 seconds

### 5.2 Model Inference Pipeline

**Demand Forecasting (Daily Batch Job):**

**Step 1: Trigger**
- CloudWatch Events rule triggers Lambda at 2:00 AM IST daily
- Lambda queries RDS for users requiring forecast updates (active users with new data)

**Step 2: Data Preparation**
- Lambda fetches historical sales data (last 90 days) from RDS
- Aggregates data to daily granularity per product
- Adds temporal features (day of week, month, holidays)
- Fetches external data (weather API, local events calendar)
- Formats data as CSV and uploads to S3 input bucket

**Step 3: AWS Forecast Inference**
- Lambda invokes AWS Forecast CreateForecast API
- Forecast service loads predictor model (pre-trained weekly)
- Generates 7-day and 14-day forecasts with P10, P50, P90 quantiles
- Exports forecast to S3 output bucket

**Step 4: Post-Processing**
- Lambda triggered by S3 event (forecast file created)
- Parses forecast results, calculates confidence scores
- Writes forecasts to RDS Forecasts table
- Caches recent forecasts in Redis for fast dashboard access

**Step 5: Notification**
- Publishes event to EventBridge: `forecast.completed`
- Decision Engine evaluates forecasts against current inventory
- Generates actionable insights (reorder recommendations, demand alerts)

**Price Optimization (On-Demand):**

**Step 1: User Request**
- User clicks "Get Price Recommendation" for a product
- API Gateway routes request to Pricing Service

**Step 2: Feature Collection**
- Pricing Service fetches product data (cost, current price, sales history)
- Retrieves demand forecast from Redis/RDS
- Queries competitor pricing (cached data, refreshed daily via web scraping Lambda)
- Calculates current inventory level and turnover rate

**Step 3: Model Inference**
- Invokes SageMaker endpoint with feature vector
- XGBoost model predicts optimal price and expected sales volume
- Calculates profit margin at recommended price
- Generates alternative scenarios (±5%, ±10% price variations)

**Step 4: Response**
- Returns recommendation with explanation (e.g., "Increase price by 8% to improve margin while maintaining 95% of current sales")
- Logs recommendation to RDS for tracking adoption and outcomes
- Caches result in Redis (TTL: 24 hours)

### 5.3 Decision Engine (Pricing, Stock Alerts)

**Inventory Alert Engine:**

**Trigger**: Runs every 6 hours via CloudWatch Events

**Logic:**
1. Query RDS for all products with current stock levels
2. Fetch demand forecasts for next 7 days
3. Calculate days of inventory remaining (current stock / avg daily demand)
4. Apply business rules:
   - Low stock: Days remaining < 3 → Generate reorder alert
   - Overstock: Days remaining > 30 AND slow-moving → Suggest clearance
   - Expiry warning: Products with expiry date < 7 days → Urgent alert
   - Optimal reorder: Days remaining = 5-7 → Proactive reorder suggestion

5. Publish alerts to EventBridge: `inventory.alert.low_stock`, `inventory.alert.expiry`, etc.
6. Notification Service routes alerts based on severity and user preferences

**Pricing Decision Engine:**

**Trigger**: Daily at 6:00 AM IST for all active products

**Logic:**
1. Fetch all products with recent sales activity (last 30 days)
2. For each product:
   - Get current price and margin
   - Retrieve demand forecast and inventory level
   - Query competitor prices (if available)
   - Calculate price elasticity from historical data
3. Apply optimization algorithm:
   - If margin < target (e.g., 20%): Suggest price increase
   - If inventory high + demand low: Suggest price decrease (clearance)
   - If competitor price significantly lower: Alert user to competitive threat
   - If demand forecast high: Opportunity for slight price increase
4. Generate recommendations with confidence scores
5. Store in RDS, cache in Redis, notify user via dashboard

### 5.4 Dashboard and Notification Delivery

**Dashboard Data Flow:**

**Real-Time Metrics:**
- User opens dashboard → API request to Reporting Service
- Reporting Service checks Redis cache for today's metrics
- If cache hit: Return data immediately (< 100ms response time)
- If cache miss: Query RDS, aggregate data, cache result (TTL: 5 minutes), return to user
- WebSocket connection established for real-time updates

**Historical Analytics:**
- User selects date range for analysis
- API request to Reporting Service with query parameters
- Service queries RDS read replica (avoid impacting master)
- Complex aggregations use pre-computed materialized views
- Results cached in Redis with longer TTL (1 hour)
- QuickSight embedded dashboard for advanced visualizations

**Notification Delivery:**

**Multi-Channel Routing:**
1. Event published to EventBridge (e.g., `inventory.alert.low_stock`)
2. Notification Service Lambda triggered
3. Fetch user notification preferences from RDS/Redis
4. Determine delivery channels based on alert severity and user settings:
   - Critical alerts: SMS + WhatsApp + In-app
   - Important alerts: WhatsApp + In-app
   - Informational: In-app only

**Channel-Specific Delivery:**

**SMS (via Amazon SNS):**
- Format message for 160-character limit
- Publish to SNS topic with phone number
- SNS routes to telecom provider
- Delivery receipt logged to CloudWatch

**WhatsApp (via WhatsApp Business API):**
- Format rich message with buttons/quick replies
- POST request to WhatsApp API endpoint
- Message template pre-approved by WhatsApp
- Delivery status webhook updates DynamoDB

**Email (via Amazon SES):**
- Generate HTML email from template
- Attach reports/invoices if applicable
- Send via SES with tracking enabled
- Open/click tracking for engagement metrics

**In-App (Push Notifications):**
- Store notification in DynamoDB Notifications table
- Send push notification via Firebase Cloud Messaging (FCM)
- Badge count updated on app icon
- User can view notification history in app

**Delivery Tracking:**
- All notifications logged with status (sent, delivered, failed, read)
- Retry logic for failed deliveries (exponential backoff, max 3 attempts)
- User can configure quiet hours (no notifications 10 PM - 8 AM)

---

## 6. Security & Reliability Design

### 6.1 Authentication and Authorization

**Multi-Layered Security:**

**Layer 1: Network Security**
- AWS WAF (Web Application Firewall) at CloudFront edge
- Rules to block common attacks (SQL injection, XSS, DDoS)
- IP rate limiting: Max 1000 requests per 5 minutes per IP
- Geo-blocking for non-Indian traffic (optional, for cost optimization)

**Layer 2: API Security**
- API Gateway with AWS Signature Version 4 authentication
- JWT token validation via Cognito authorizer
- Token expiry: Access token (1 hour), Refresh token (30 days)
- CORS policies restricting origins to approved domains

**Layer 3: Application Security**
- Role-Based Access Control (RBAC) enforced in each microservice
- Service-to-service authentication via IAM roles (no hardcoded credentials)
- Input validation and sanitization at API layer
- Parameterized queries to prevent SQL injection

**Layer 4: Data Security**
- Encryption at rest: RDS (AES-256), DynamoDB (AWS-managed keys), S3 (SSE-S3)
- Encryption in transit: TLS 1.3 for all communications
- Secrets stored in AWS Secrets Manager, rotated automatically
- PII data masked in logs and monitoring tools

**Authorization Model:**
- **Retailer Role**: Access only to own business data, cannot view other users
- **Admin Role**: Read access to all data for support, write access to system config
- **System Role**: ML services can read anonymized data across users for model training

**Compliance:**
- GDPR-like principles: User consent, data portability, right to deletion
- Audit logs: All data access and modifications logged with user ID, timestamp, action
- Data retention: 7 years for financial data (Indian tax law), 3 years for operational data

### 6.2 Secure Data Storage

**Database Security:**

**RDS PostgreSQL:**
- Deployed in private subnets (no public internet access)
- Security groups allow traffic only from ECS services and Lambda functions
- Master user credentials stored in Secrets Manager
- Automated backups encrypted with AWS KMS
- SSL/TLS required for all connections
- Database activity monitoring via CloudWatch Logs

**DynamoDB:**
- Encryption at rest enabled by default (AWS-managed keys)
- Fine-grained access control via IAM policies
- VPC endpoints for private connectivity (no internet gateway)
- Point-in-time recovery (PITR) enabled for 35 days
- DynamoDB Streams encrypted in transit

**S3:**
- Bucket policies enforce encryption (deny unencrypted uploads)
- Versioning enabled for critical buckets (ML data, backups)
- MFA Delete for production buckets
- Access logging to separate audit bucket
- Lifecycle policies for automatic archival and deletion

**Data Classification:**
- **Public**: Product catalogs, marketing content (S3 public bucket)
- **Internal**: Aggregated analytics, system metrics (S3 private bucket)
- **Confidential**: User PII, financial data (RDS encrypted, access logged)
- **Restricted**: Admin credentials, API keys (Secrets Manager)

### 6.3 Fault Tolerance

**Service-Level Resilience:**

**Microservices:**
- Each service deployed across multiple Availability Zones (AZs)
- ECS tasks distributed via Application Load Balancer
- Health checks: HTTP endpoint `/health` checked every 30 seconds
- Unhealthy tasks automatically replaced by ECS
- Circuit breaker pattern: Fail fast when downstream service unavailable

**Databases:**
- RDS Multi-AZ: Synchronous replication to standby in different AZ
- Automatic failover in < 2 minutes (typically 60-120 seconds)
- DynamoDB: Multi-AZ by default, 99.99% availability SLA
- ElastiCache: Redis cluster mode with automatic failover

**Stateless Design:**
- Application servers are stateless (session data in Redis)
- Any ECS task can handle any request (no sticky sessions required)
- Enables horizontal scaling and seamless failover

**Data Resilience:**

**Backup Strategy:**
- RDS: Automated daily backups, 7-day retention, manual snapshots before major changes
- DynamoDB: Continuous backups with PITR, on-demand backups before schema changes
- S3: Versioning + Cross-Region Replication (CRR) to secondary region for DR
- Backup testing: Monthly restore drills to verify backup integrity

**Disaster Recovery:**
- RTO (Recovery Time Objective): 4 hours for full system recovery
- RPO (Recovery Point Objective): 1 hour (maximum acceptable data loss)
- DR region: Secondary AWS region (e.g., Mumbai primary, Singapore DR)
- Runbook: Documented procedures for failover and recovery

**Graceful Degradation:**
- If ML services unavailable: Fall back to rule-based recommendations
- If RDS read replica down: Route queries to master (with rate limiting)
- If external APIs fail (weather, competitor data): Use cached data or skip feature
- If notification service down: Queue messages in DynamoDB for retry

### 6.4 High Availability

**Architecture for 99.9% Uptime:**

**Multi-AZ Deployment:**
- All services deployed across 3 Availability Zones in primary region
- Load balancer distributes traffic evenly across AZs
- If one AZ fails, traffic automatically routed to healthy AZs
- Capacity planning: Each AZ can handle 50% of total load (N+1 redundancy)

**Auto-Scaling:**
- ECS Service Auto Scaling: Target tracking based on CPU (70%) and memory (80%)
- Scale-out: Add tasks when metrics exceed threshold for 2 consecutive minutes
- Scale-in: Remove tasks when metrics below threshold for 10 minutes (gradual)
- Scheduled scaling: Pre-scale before peak hours (8 AM - 10 AM, 6 PM - 8 PM IST)

**Database Scaling:**
- RDS: Vertical scaling (instance size) for master, horizontal scaling (read replicas) for reads
- Read replica lag monitoring: Alert if lag > 5 seconds
- DynamoDB: Auto-scaling for read/write capacity units
- ElastiCache: Cluster mode with sharding for horizontal scaling

**CDN and Caching:**
- CloudFront: 99.99% availability SLA, global edge locations
- Cache hit ratio target: > 80% for static assets
- ElastiCache: 99.9% availability with Multi-AZ replication
- Application-level caching: Reduce database load by 60-70%

**Monitoring and Alerting:**
- CloudWatch alarms for critical metrics:
  - API Gateway 5xx errors > 1% of requests
  - ECS CPU utilization > 85% for 5 minutes
  - RDS connections > 80% of max
  - Lambda throttling or errors
- PagerDuty integration for on-call engineer alerts
- Status page for user-facing service status

---

## 7. Scalability & Future Enhancements

### 7.1 Scalability Design

**Current Capacity (Launch):**
- Support 10,000 active users (retailers)
- Handle 100,000 transactions per day
- Store 1 TB of data (sales, inventory, analytics)
- Serve 1 million API requests per day

**Scaling Strategy (12-Month Horizon):**

**Horizontal Scaling:**
- Microservices: Add more ECS tasks (target: 100,000 users, 1M transactions/day)
- Database: Add read replicas (up to 5 replicas for read-heavy workloads)
- DynamoDB: Auto-scaling handles write throughput increases
- Lambda: Concurrent execution limit increase (default 1000 → 10,000)

**Vertical Scaling:**
- RDS: Upgrade instance class (db.t3.large → db.r5.xlarge for memory-intensive queries)
- ElastiCache: Larger node types for increased cache capacity
- SageMaker: More powerful instances for faster model training

**Data Partitioning:**
- Shard RDS by region/state for geographic distribution
- DynamoDB: Partition key design ensures even distribution (user_id hash)
- S3: Prefix-based partitioning for parallel processing

**Cost Optimization:**
- Reserved Instances for predictable workloads (RDS, ElastiCache) - 40% savings
- Savings Plans for compute (ECS, Lambda) - 30% savings
- S3 Intelligent-Tiering for automatic cost optimization
- Spot Instances for non-critical batch jobs (ML training) - 70% savings

### 7.2 Multi-Store Support

**Current State:** Single store per user account

**Future Enhancement:**
- User can manage multiple store locations from single account
- Store hierarchy: Chain → Region → Store
- Consolidated dashboard with drill-down by store
- Store-specific inventory and sales tracking
- Cross-store inventory transfer and balancing
- Centralized purchasing with store-level allocation

**Technical Changes:**
- Add `store_id` to all data models (sales, inventory, forecasts)
- Multi-tenancy: Partition key becomes `user_id#store_id`
- Aggregation queries across stores for chain-level insights
- Role-based access: Store manager (single store) vs. Chain owner (all stores)

**ML Enhancements:**
- Cross-store demand forecasting: Learn from similar stores
- Store clustering: Group stores by characteristics (location, size, customer base)
- Inventory optimization: Suggest transfers between stores to balance stock

### 7.3 Regional Expansion

**Phase 1: India (Current)**
- Languages: 10 major Indian languages
- Currency: INR
- Compliance: Indian regulations (GST, FSSAI, etc.)
- Payment: UPI, cards, cash

**Phase 2: South Asia (12-18 months)**
- Countries: Bangladesh, Sri Lanka, Nepal
- Languages: Bengali, Sinhala, Nepali
- Localization: Currency, tax systems, business regulations
- Infrastructure: AWS regions in Singapore/Mumbai

**Phase 3: Southeast Asia (18-24 months)**
- Countries: Indonesia, Philippines, Vietnam
- Languages: Bahasa, Tagalog, Vietnamese
- Challenges: Diverse regulatory environments, payment systems
- Partnerships: Local payment gateways, logistics providers

**Technical Considerations:**
- Multi-region deployment: Latency optimization via regional endpoints
- Data residency: Comply with local data protection laws
- Currency conversion: Real-time exchange rates, multi-currency support
- Internationalization (i18n): Externalize all strings, support for new locales

### 7.4 Additional AI Models

**Churn Prediction:**
- Predict which customers are likely to stop purchasing
- Features: Recency, frequency, monetary value, purchase trends
- Model: Gradient Boosting (XGBoost) with binary classification
- Action: Proactive retention campaigns (discounts, personalized offers)

**Product Recommendation:**
- Suggest new products for retailers to stock based on local demand
- Collaborative filtering: "Stores like yours also stock..."
- Market basket analysis: Identify complementary products
- Trend detection: Emerging product categories in region

**Fraud Detection:**
- Identify anomalous transactions (data entry errors or fraud)
- Unsupervised learning: Isolation Forest for anomaly detection
- Features: Transaction amount, time, frequency, product mix
- Alert: Flag suspicious patterns for manual review

**Sentiment Analysis:**
- Analyze customer feedback and reviews (if collected)
- NLP model: BERT-based sentiment classification
- Insights: Product quality issues, service improvements
- Integration: WhatsApp chatbot for customer feedback collection

**Computer Vision:**
- Receipt scanning: OCR to extract transaction data from paper receipts
- Shelf monitoring: Analyze store photos to track inventory levels
- Product recognition: Barcode-free product identification via image
- Technology: Amazon Rekognition, Textract

### 7.5 Integration with POS and Suppliers

**POS Integration:**

**Supported Systems:**
- Cloud-based POS: Shopify POS, Square, Lightspeed
- Traditional POS: Tally, Marg ERP, Busy Accounting

**Integration Methods:**
- API integration: Real-time sync via REST APIs
- File-based: Daily CSV/Excel export/import
- Webhook: POS pushes transaction data to VyapariAI

**Benefits:**
- Eliminate manual data entry (reduce errors, save time)
- Real-time inventory updates
- Unified view of online and offline sales
- Automatic reconciliation of cash and digital payments

**Technical Implementation:**
- Integration Service: New microservice for POS connectors
- Adapter pattern: Normalize data from different POS systems
- Idempotency: Handle duplicate transactions gracefully
- Error handling: Retry logic, dead-letter queue for failed syncs

**Supplier Integration:**

**B2B Marketplace:**
- Connect retailers with wholesalers and distributors
- Automated reordering based on forecasts and inventory levels
- Price comparison across suppliers
- Order tracking and delivery management

**Integration Features:**
- Supplier catalog: Browse products, prices, minimum order quantities
- Purchase orders: Generate and send POs directly from VyapariAI
- Invoice matching: Reconcile received goods with invoices
- Payment integration: UPI, NEFT, credit terms

**Technical Implementation:**
- Supplier API: RESTful API for supplier systems to integrate
- EDI support: Electronic Data Interchange for large distributors
- Blockchain: Explore for supply chain transparency (future)

**Benefits:**
- Reduce procurement time and costs
- Better negotiation with data-driven insights
- Reduce stock-outs with automated reordering
- Improve cash flow with optimized ordering

---

## 8. Technology Stack Summary

### 8.1 Frontend

- **Framework**: React 18 with TypeScript
- **State Management**: Redux Toolkit for global state, React Query for server state
- **UI Library**: Material-UI (MUI) with custom theme for Indian aesthetics
- **PWA**: Workbox for service workers, offline support, background sync
- **Internationalization**: i18next for multi-language support
- **Charts**: Recharts for data visualization
- **Build Tool**: Vite for fast development and optimized production builds
- **Hosting**: S3 + CloudFront for global CDN delivery

### 8.2 Backend

- **Language**: Python 3.11 (ML/data services), Node.js 20 (API services)
- **Framework**: FastAPI (Python), Express.js (Node.js)
- **API Style**: RESTful APIs with OpenAPI 3.0 documentation
- **Container**: Docker with multi-stage builds
- **Orchestration**: Amazon ECS with Fargate (serverless containers)
- **API Gateway**: AWS API Gateway with Lambda authorizers
- **Message Queue**: Amazon SQS for asynchronous processing
- **Event Bus**: Amazon EventBridge for event-driven architecture

### 8.3 Data Layer

- **Relational DB**: Amazon RDS PostgreSQL 15
- **NoSQL DB**: Amazon DynamoDB
- **Cache**: Amazon ElastiCache (Redis 7)
- **Object Storage**: Amazon S3
- **Data Warehouse**: Amazon Redshift (future, for advanced analytics)
- **Search**: Amazon OpenSearch (future, for full-text search)

### 8.4 AI/ML

- **Training**: Amazon SageMaker (Jupyter notebooks, training jobs)
- **Inference**: SageMaker endpoints, AWS Lambda for lightweight models
- **Forecasting**: AWS Forecast with DeepAR+ algorithm
- **ML Frameworks**: TensorFlow 2.x, PyTorch 2.x, Scikit-learn, XGBoost
- **Feature Store**: SageMaker Feature Store (future)
- **MLOps**: SageMaker Pipelines for CI/CD, Model Monitor for drift detection

### 8.5 DevOps & Monitoring

- **IaC**: AWS CloudFormation / Terraform for infrastructure as code
- **CI/CD**: GitHub Actions for build/test, AWS CodePipeline for deployment
- **Monitoring**: Amazon CloudWatch (metrics, logs, alarms)
- **Tracing**: AWS X-Ray for distributed tracing
- **Logging**: CloudWatch Logs with structured JSON logging
- **Alerting**: CloudWatch Alarms + PagerDuty for on-call
- **APM**: AWS X-Ray + CloudWatch ServiceLens for application performance

### 8.6 Security

- **Identity**: Amazon Cognito for user authentication
- **Secrets**: AWS Secrets Manager for credential management
- **Encryption**: AWS KMS for key management
- **WAF**: AWS WAF for web application firewall
- **DDoS**: AWS Shield Standard (free) + Shield Advanced (optional)
- **Compliance**: AWS Config for compliance monitoring, AWS Audit Manager

---

## 9. Performance Targets

### 9.1 Response Time SLAs

- **API Endpoints**: P95 < 500ms, P99 < 1000ms
- **Dashboard Load**: Initial load < 3 seconds on 3G, < 1 second on 4G/WiFi
- **ML Inference**: Price optimization < 2 seconds, demand forecast < 5 seconds
- **Database Queries**: Simple queries < 100ms, complex aggregations < 500ms
- **Notification Delivery**: SMS/WhatsApp < 10 seconds, Email < 30 seconds

### 9.2 Throughput Targets

- **API Requests**: 10,000 requests per second (peak)
- **Concurrent Users**: 50,000 simultaneous users
- **Database Writes**: 5,000 transactions per second (DynamoDB)
- **ML Predictions**: 1,000 inference requests per minute

### 9.3 Availability Targets

- **Overall System**: 99.9% uptime (8.76 hours downtime per year)
- **Critical Services**: 99.95% uptime (authentication, data entry)
- **ML Services**: 99.5% uptime (graceful degradation if unavailable)
- **Planned Maintenance**: < 4 hours per month, scheduled during low-traffic periods

---

## 10. Cost Estimation (Monthly, 10K Users)

### 10.1 Compute
- **ECS Fargate**: 20 tasks × $50 = $1,000
- **Lambda**: 10M invocations × $0.20 per 1M = $2
- **API Gateway**: 30M requests × $3.50 per 1M = $105

### 10.2 Storage
- **RDS**: db.r5.large Multi-AZ = $400
- **DynamoDB**: 10GB storage + 1M writes/day = $50
- **S3**: 500GB storage + 1TB transfer = $30
- **ElastiCache**: cache.r5.large = $200

### 10.3 AI/ML
- **SageMaker Endpoints**: 2 × ml.t3.medium = $100
- **AWS Forecast**: 10K forecasts/month = $150
- **SageMaker Training**: 20 hours/month = $50

### 10.4 Other Services
- **CloudFront**: 1TB data transfer = $85
- **Cognito**: 10K MAU (free tier)
- **SNS/SES**: 100K messages = $10
- **CloudWatch**: Logs + metrics = $50

**Total Estimated Cost**: ~$2,230/month for 10,000 users (~$0.22 per user/month)

**Scaling**: At 100K users, estimated cost ~$15,000/month (~$0.15 per user/month due to economies of scale)

---

## Document Control

**Version**: 1.0  
**Date**: February 6, 2026  
**Status**: Draft  
**Author**: Cloud Architecture Team  
**Reviewers**: Engineering, ML Team, Security, DevOps  

---

*This design document provides the technical blueprint for VyapariAI platform. All architectural decisions are aligned with AWS Well-Architected Framework principles: operational excellence, security, reliability, performance efficiency, and cost optimization.*
