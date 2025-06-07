# Job Application Delegation System
## Technical Flow & Architecture Document

---

## 🏗️ **System Architecture Overview**

### **High-Level Architecture**

```mermaid
graph TB
    subgraph "Client Layer"
        WEB[Web Dashboard]
        EXT[Chrome Extension]
        MOB[Mobile Browser]
    end
    
    subgraph "API Gateway"
        API[REST API Gateway]
        AUTH[Authentication Service]
        RATE[Rate Limiting]
    end
    
    subgraph "Application Layer"
        DJANGO[Django Backend]
        ADMIN[Django Admin Panel]
        WORKER[Background Workers]
    end
    
    subgraph "Data Layer"
        DB[(PostgreSQL Database)]
        REDIS[(Redis Cache)]
        FILES[File Storage]
    end
    
    subgraph "External Services"
        EMAIL[Email Service]
        NOTIF[Push Notifications]
        ANALYTICS[Analytics Service]
    end
    
    WEB --> API
    EXT --> API
    MOB --> API
    
    API --> AUTH
    API --> RATE
    API --> DJANGO
    
    DJANGO --> ADMIN
    DJANGO --> WORKER
    DJANGO --> DB
    DJANGO --> REDIS
    DJANGO --> FILES
    
    WORKER --> EMAIL
    WORKER --> NOTIF
    DJANGO --> ANALYTICS
```

### **Technology Stack**

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Backend Framework** | Django 4.2+ | Web framework with built-in admin |
| **Database** | PostgreSQL 15+ | Primary data storage |
| **Cache** | Redis 7+ | Session management, job queues |
| **API** | Django REST Framework | RESTful API endpoints |
| **Task Queue** | Celery + Redis | Background job processing |
| **File Storage** | AWS S3 / Local Storage | Resume PDFs, screenshots |
| **Frontend** | HTML5 + JavaScript | Responsive web dashboard |
| **Extension** | Chrome Extension (MV3) | Browser integration |
| **Authentication** | JWT + Session Auth | Secure user authentication |

---

## 🗄️ **Database Schema Design**

### **Entity Relationship Diagram**

```mermaid
erDiagram
    User {
        uuid id PK
        string email UK
        string password_hash
        string first_name
        string last_name
        enum user_type
        boolean is_active
        datetime created_at
        datetime updated_at
    }
    
    UserProfile {
        uuid id PK
        uuid user_id FK
        text bio
        string phone
        string location
        json metadata
        datetime created_at
        datetime updated_at
    }
    
    Client {
        uuid id PK
        uuid user_id FK
        boolean is_activated
        datetime activated_at
        uuid activated_by FK
        string payment_status
        json subscription_data
    }
    
    Employee {
        uuid id PK
        uuid user_id FK
        uuid created_by FK
        json performance_metrics
        boolean is_available
        datetime last_active
    }
    
    Admin {
        uuid id PK
        uuid user_id FK
        enum admin_level
        json permissions
        uuid created_by FK
    }
    
    ResumeProfile {
        uuid id PK
        uuid client_id FK
        string professional_summary
        json experience
        json education
        json skills
        json contact_info
        datetime created_at
        datetime updated_at
    }
    
    ResumeType {
        uuid id PK
        uuid resume_profile_id FK
        string name
        string file_path
        string file_name
        integer file_size
        datetime uploaded_at
    }
    
    ClientEmployeeAssignment {
        uuid id PK
        uuid client_id FK
        uuid employee_id FK
        uuid assigned_by FK
        datetime assigned_at
        boolean is_active
    }
    
    JobDelegation {
        uuid id PK
        uuid client_id FK
        uuid assigned_employee_id FK
        uuid resume_type_id FK
        string job_url
        string company_name
        string job_title
        enum status
        enum priority
        datetime deadline
        datetime delegated_at
        datetime completed_at
        json metadata
    }
    
    JobCompletion {
        uuid id PK
        uuid job_delegation_id FK
        uuid completed_by FK
        string proof_screenshot_path
        text completion_notes
        enum completion_status
        datetime completed_at
    }
    
    ActivityLog {
        uuid id PK
        uuid user_id FK
        string action_type
        string entity_type
        uuid entity_id
        json action_data
        string ip_address
        string user_agent
        datetime created_at
    }
    
    Notification {
        uuid id PK
        uuid user_id FK
        string notification_type
        string title
        text message
        boolean is_read
        json metadata
        datetime created_at
        datetime read_at
    }
    
    User ||--|| UserProfile : "has"
    User ||--o| Client : "can be"
    User ||--o| Employee : "can be"
    User ||--o| Admin : "can be"
    
    Client ||--|| ResumeProfile : "has"
    ResumeProfile ||--o{ ResumeType : "contains"
    
    Client ||--o{ ClientEmployeeAssignment : "assigned to"
    Employee ||--o{ ClientEmployeeAssignment : "works for"
    
    Client ||--o{ JobDelegation : "delegates"
    Employee ||--o{ JobDelegation : "receives"
    ResumeType ||--o{ JobDelegation : "used in"
    
    JobDelegation ||--o| JobCompletion : "completed as"
    
    User ||--o{ ActivityLog : "performs"
    User ||--o{ Notification : "receives"
```

### **Key Database Constraints & Indexes**

```sql
-- Primary indexes for performance
CREATE INDEX idx_job_delegation_client_status ON job_delegation(client_id, status);
CREATE INDEX idx_job_delegation_employee_status ON job_delegation(assigned_employee_id, status);
CREATE INDEX idx_activity_log_user_date ON activity_log(user_id, created_at DESC);

-- Unique constraints
ALTER TABLE client_employee_assignment 
ADD CONSTRAINT unique_active_assignment 
UNIQUE(client_id, employee_id, is_active) 
WHERE is_active = true;

-- Check constraints
ALTER TABLE job_delegation 
ADD CONSTRAINT valid_deadline 
CHECK (deadline > delegated_at);
```

---

## 🔄 **API Flow Architecture**

### **Authentication Flow**

```mermaid
sequenceDiagram
    participant C as Client/Employee
    participant API as API Gateway
    participant AUTH as Auth Service
    participant DB as Database
    participant REDIS as Redis Cache
    
    C->>API: POST /auth/login {email, password}
    API->>AUTH: Validate credentials
    AUTH->>DB: Query user details
    DB-->>AUTH: User data + permissions
    AUTH->>AUTH: Generate JWT token
    AUTH->>REDIS: Store session data
    AUTH-->>API: {token, user_type, permissions}
    API-->>C: {access_token, refresh_token, user_data}
    
    Note over C,REDIS: Subsequent API calls include JWT token
    
    C->>API: GET /dashboard (with JWT)
    API->>AUTH: Validate JWT
    AUTH->>REDIS: Check session
    AUTH-->>API: Valid + user context
    API->>DB: Fetch dashboard data
    DB-->>API: Dashboard response
    API-->>C: Dashboard data
```

### **Job Delegation Flow**

```mermaid
sequenceDiagram
    participant CLIENT as Client
    participant EXT as Chrome Extension
    participant API as API Gateway
    participant DJANGO as Django Backend
    participant WORKER as Background Worker
    participant DB as Database
    participant EMAIL as Email Service
    
    Note over CLIENT,EXT: Method 1: Browser Extension
    CLIENT->>EXT: Click delegate job button
    EXT->>EXT: Auto-capture job URL
    EXT->>API: POST /jobs/delegate
    
    Note over CLIENT,API: Method 2: Web Dashboard
    CLIENT->>API: POST /jobs/delegate {url, company, resume_type}
    
    API->>DJANGO: Process job delegation
    DJANGO->>DB: Fetch assigned employees
    DB-->>DJANGO: Employee list
    DJANGO->>DJANGO: Apply distribution algorithm
    DJANGO->>DB: Create job delegation record
    
    DJANGO->>WORKER: Queue notification task
    WORKER->>EMAIL: Send email to assigned employee
    WORKER->>DB: Create notification record
    
    DJANGO-->>API: Job created successfully
    API-->>CLIENT: {job_id, assigned_employee, status}
```

### **Job Completion Flow**

```mermaid
sequenceDiagram
    participant EMP as Employee
    participant EXT as Chrome Extension
    participant API as API Gateway
    participant DJANGO as Django Backend
    participant FILES as File Storage
    participant DB as Database
    participant NOTIF as Notification Service
    
    EMP->>EXT: Click "Mark as Completed"
    EXT->>EXT: Capture screenshot
    EXT->>API: POST /jobs/{job_id}/complete
    Note over EXT,API: Includes screenshot data
    
    API->>DJANGO: Process completion
    DJANGO->>FILES: Upload screenshot
    FILES-->>DJANGO: File URL
    
    DJANGO->>DB: Update job status to completed
    DJANGO->>DB: Create completion record
    
    DJANGO->>NOTIF: Send completion notification
    NOTIF->>DB: Notify client
    
    DJANGO-->>API: Completion successful
    API-->>EXT: {status: "completed", proof_url}
    EXT-->>EMP: Job marked as completed
```

---

## 🔧 **Technical Components**

### **Django Backend Structure**

```
project/
├── apps/
│   ├── authentication/          # User management & auth
│   │   ├── models.py           # User, Profile models
│   │   ├── views.py            # Auth endpoints
│   │   ├── serializers.py      # API serializers
│   │   └── permissions.py      # Role-based permissions
│   │
│   ├── users/                  # User roles & profiles
│   │   ├── models.py           # Client, Employee, Admin
│   │   ├── views.py            # User management
│   │   └── admin.py            # Django admin config
│   │
│   ├── jobs/                   # Job delegation system
│   │   ├── models.py           # JobDelegation, Completion
│   │   ├── views.py            # Job CRUD operations
│   │   ├── tasks.py            # Celery background tasks
│   │   └── algorithms.py       # Distribution algorithms
│   │
│   ├── resumes/                # Resume management
│   │   ├── models.py           # ResumeProfile, ResumeType
│   │   ├── views.py            # Resume CRUD
│   │   └── file_handlers.py    # PDF upload/processing
│   │
│   ├── notifications/          # Real-time notifications
│   │   ├── models.py           # Notification models
│   │   ├── views.py            # Notification API
│   │   └── channels.py         # WebSocket handlers
│   │
│   └── analytics/              # Performance metrics
│       ├── models.py           # ActivityLog, Metrics
│       ├── views.py            # Analytics API
│       └── aggregators.py      # Data aggregation
│
├── api/
│   ├── v1/                     # API version 1
│   │   ├── urls.py            # URL routing
│   │   └── routers.py         # DRF routers
│   │
│   └── middleware/             # Custom middleware
│       ├── auth.py            # JWT middleware
│       ├── logging.py         # Request logging
│       └── rate_limiting.py   # API rate limiting
│
├── chrome_extension/
│   ├── manifest.json          # Extension manifest
│   ├── content_scripts/       # Page interaction
│   ├── popup/                 # Extension popup UI
│   ├── background/            # Background service worker
│   └── api/                   # API communication
│
└── frontend/
    ├── static/                # Static assets
    ├── templates/             # HTML templates
    └── js/                    # Dashboard JavaScript
```

### **Chrome Extension Architecture**

```mermaid
graph TB
    subgraph "Chrome Extension"
        MANIFEST[manifest.json]
        
        subgraph "Content Scripts"
            DETECT[Job URL Detection]
            INJECT[UI Injection]
            CAPTURE[Screenshot Capture]
        end
        
        subgraph "Popup Interface"
            LOGIN[Login Form]
            DELEGATE[Job Delegation]
            STATUS[Status Display]
        end
        
        subgraph "Background Service"
            API_COMM[API Communication]
            STORAGE[Local Storage]
            SYNC[Data Synchronization]
        end
        
        subgraph "Permissions"
            TABS[Active Tab Access]
            HOST[Host Permissions]
            STORAGE_PERM[Storage Permission]
        end
    end
    
    DETECT --> API_COMM
    DELEGATE --> API_COMM
    CAPTURE --> API_COMM
    API_COMM --> STORAGE
    STORAGE --> SYNC
```

### **Background Task Processing**

```mermaid
graph LR
    subgraph "Task Types"
        EMAIL_TASK[Email Notifications]
        ANALYTICS_TASK[Analytics Processing]
        CLEANUP_TASK[Data Cleanup]
        REMINDER_TASK[Deadline Reminders]
    end
    
    subgraph "Celery Workers"
        WORKER1[Worker 1<br/>High Priority]
        WORKER2[Worker 2<br/>Normal Priority]
        WORKER3[Worker 3<br/>Low Priority]
    end
    
    subgraph "Queues"
        HIGH_Q[(High Priority Queue)]
        NORMAL_Q[(Normal Queue)]
        LOW_Q[(Low Priority Queue)]
    end
    
    EMAIL_TASK --> HIGH_Q
    REMINDER_TASK --> HIGH_Q
    ANALYTICS_TASK --> NORMAL_Q
    CLEANUP_TASK --> LOW_Q
    
    HIGH_Q --> WORKER1
    NORMAL_Q --> WORKER2
    LOW_Q --> WORKER3
```

---

## 🔐 **Security Architecture**

### **Authentication & Authorization**

```mermaid
graph TB
    subgraph "Security Layers"
        
        subgraph "Network Security"
            HTTPS[HTTPS/TLS 1.3]
            CORS[CORS Policy]
            CSP[Content Security Policy]
        end
        
        subgraph "Authentication"
            JWT[JWT Tokens]
            REFRESH[Refresh Token Rotation]
            SESSION[Session Management]
            MFA[Multi-Factor Auth]
        end
        
        subgraph "Authorization"
            RBAC[Role-Based Access Control]
            PERMS[Granular Permissions]
            SCOPE[API Scope Validation]
        end
        
        subgraph "Data Security"
            ENCRYPT[Data Encryption at Rest]
            HASH[Password Hashing]
            SANITIZE[Input Sanitization]
            AUDIT[Audit Logging]
        end
        
    end
    
    CLIENT[Client Request] --> HTTPS
    HTTPS --> CORS
    CORS --> JWT
    JWT --> RBAC
    RBAC --> ENCRYPT
    ENCRYPT --> AUDIT
```

### **Role-Based Permissions Matrix**

| Resource | Super Admin | Admin | Client | Employee |
|----------|-------------|-------|--------|----------|
| **User Management** | ✅ Full | ✅ Limited | ❌ | ❌ |
| **Client Activation** | ✅ | ✅ | ❌ | ❌ |
| **Employee Assignment** | ✅ | ✅ | ❌ | ❌ |
| **Job Delegation** | ✅ View | ✅ View | ✅ Create | ❌ |
| **Job Processing** | ✅ View | ✅ View | ❌ | ✅ Process |
| **Resume Management** | ✅ View | ✅ View | ✅ Full | ✅ View |
| **Analytics** | ✅ Full | ✅ Basic | ✅ Personal | ✅ Personal |
| **System Settings** | ✅ | ❌ | ❌ | ❌ |

---

## 📊 **Performance & Scalability**

### **Caching Strategy**

```mermaid
graph TB
    subgraph "Caching Layers"
        
        subgraph "Browser Cache"
            BROWSER[Static Assets<br/>24h TTL]
        end
        
        subgraph "CDN Cache"
            CDN[Images, CSS, JS<br/>30d TTL]
        end
        
        subgraph "Application Cache"
            REDIS_SESSION[User Sessions<br/>24h TTL]
            REDIS_DATA[Frequently Accessed Data<br/>1h TTL]
            REDIS_COUNTER[Analytics Counters<br/>5m TTL]
        end
        
        subgraph "Database Cache"
            DB_QUERY[Query Result Cache<br/>15m TTL]
        end
        
    end
    
    USER[User Request] --> BROWSER
    BROWSER --> CDN
    CDN --> REDIS_SESSION
    REDIS_SESSION --> REDIS_DATA
    REDIS_DATA --> DB_QUERY
    DB_QUERY --> DATABASE[(Database)]
```

### **Database Optimization**

```sql
-- Performance optimization strategies

-- Partitioning for large tables
CREATE TABLE activity_log_2024_q1 PARTITION OF activity_log
FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

-- Materialized views for analytics
CREATE MATERIALIZED VIEW job_completion_stats AS
SELECT 
    client_id,
    COUNT(*) as total_jobs,
    COUNT(*) FILTER (WHERE status = 'completed') as completed_jobs,
    AVG(EXTRACT(EPOCH FROM (completed_at - delegated_at))/3600) as avg_completion_hours
FROM job_delegation 
WHERE delegated_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY client_id;

-- Indexes for performance
CREATE INDEX CONCURRENTLY idx_job_delegation_composite 
ON job_delegation(client_id, status, delegated_at DESC);
```

---

## 🚀 **Deployment Architecture**

### **Production Infrastructure**

```mermaid
graph TB
    subgraph "Load Balancer"
        LB[Nginx Load Balancer]
    end
    
    subgraph "Application Servers"
        APP1[Django App Server 1]
        APP2[Django App Server 2]
        APP3[Django App Server 3]
    end
    
    subgraph "Background Workers"
        WORKER1[Celery Worker 1]
        WORKER2[Celery Worker 2]
    end
    
    subgraph "Data Layer"
        DB_MASTER[(PostgreSQL Master)]
        DB_REPLICA[(PostgreSQL Replica)]
        REDIS_CLUSTER[(Redis Cluster)]
    end
    
    subgraph "File Storage"
        S3[AWS S3 / File Storage]
    end
    
    subgraph "Monitoring"
        LOGS[Centralized Logging]
        METRICS[Application Metrics]
        ALERTS[Alert Manager]
    end
    
    LB --> APP1
    LB --> APP2
    LB --> APP3
    
    APP1 --> DB_MASTER
    APP2 --> DB_REPLICA
    APP3 --> DB_REPLICA
    
    APP1 --> REDIS_CLUSTER
    APP2 --> REDIS_CLUSTER
    APP3 --> REDIS_CLUSTER
    
    WORKER1 --> DB_MASTER
    WORKER2 --> DB_MASTER
    WORKER1 --> REDIS_CLUSTER
    WORKER2 --> REDIS_CLUSTER
    
    APP1 --> S3
    APP2 --> S3
    APP3 --> S3
    
    APP1 --> LOGS
    APP2 --> LOGS
    APP3 --> LOGS
    
    METRICS --> ALERTS
```

### **Environment Configuration**

| Environment | Purpose | Resources | Scaling |
|-------------|---------|-----------|---------|
| **Development** | Local development | 1 app server, SQLite | Manual |
| **Staging** | Testing & QA | 1 app server, PostgreSQL | Manual |
| **Production** | Live system | 3+ app servers, HA database | Auto-scaling |

---

## 📈 **Monitoring & Analytics**

### **Key Metrics to Track**

```mermaid
graph TB
    subgraph "Business Metrics"
        JOBS_DELEGATED[Jobs Delegated per Day]
        COMPLETION_RATE[Job Completion Rate]
        AVG_COMPLETION_TIME[Average Completion Time]
        CLIENT_SATISFACTION[Client Satisfaction Score]
    end
    
    subgraph "Technical Metrics"
        API_RESPONSE_TIME[API Response Time]
        ERROR_RATE[Error Rate]
        DB_PERFORMANCE[Database Performance]
        CACHE_HIT_RATIO[Cache Hit Ratio]
    end
    
    subgraph "User Metrics"
        ACTIVE_USERS[Daily Active Users]
        USER_RETENTION[User Retention Rate]
        FEATURE_USAGE[Feature Usage Statistics]
        LOGIN_SUCCESS_RATE[Login Success Rate]
    end
    
    subgraph "System Metrics"
        CPU_USAGE[CPU Usage]
        MEMORY_USAGE[Memory Usage]
        DISK_USAGE[Disk Usage]
        NETWORK_TRAFFIC[Network Traffic]
    end
```

### **Alert Conditions**

| Metric | Warning Threshold | Critical Threshold | Action |
|--------|------------------|-------------------|---------|
| **API Response Time** | > 500ms | > 2000ms | Scale up servers |
| **Error Rate** | > 2% | > 5% | Immediate investigation |
| **Job Completion Rate** | < 85% | < 70% | Review assignments |
| **Database Connections** | > 80% | > 95% | Scale database |

---

## 🔄 **Integration Points**

### **External Service Integrations**

```mermaid
graph LR
    subgraph "Core System"
        DJANGO[Django Backend]
    end
    
    subgraph "Email Services"
        SENDGRID[SendGrid]
        SES[AWS SES]
    end
    
    subgraph "File Storage"
        S3[AWS S3]
        CLOUDINARY[Cloudinary]
    end
    
    subgraph "Analytics"
        GA[Google Analytics]
        MIXPANEL[Mixpanel]
    end
    
    subgraph "Monitoring"
        SENTRY[Sentry]
        DATADOG[DataDog]
    end
    
    DJANGO --> SENDGRID
    DJANGO --> S3
    DJANGO --> GA
    DJANGO --> SENTRY
    
    DJANGO -.-> SES
    DJANGO -.-> CLOUDINARY
    DJANGO -.-> MIXPANEL
    DJANGO -.-> DATADOG
```

---

## 🔧 **Development Workflow**

### **CI/CD Pipeline**

```mermaid
graph LR
    DEV[Developer Push] --> GIT[Git Repository]
    GIT --> BUILD[Build & Test]
    BUILD --> QUALITY[Code Quality Checks]
    QUALITY --> STAGING[Deploy to Staging]
    STAGING --> TEST[Automated Testing]
    TEST --> APPROVE[Manual Approval]
    APPROVE --> PROD[Deploy to Production]
    PROD --> MONITOR[Post-deployment Monitoring]
```

### **Testing Strategy**

| Test Type | Coverage | Tools | Frequency |
|-----------|----------|-------|-----------|
| **Unit Tests** | 90%+ | pytest, unittest | Every commit |
| **Integration Tests** | Key flows | pytest-django | Every PR |
| **API Tests** | All endpoints | Postman/pytest | Daily |
| **E2E Tests** | Critical paths | Selenium/Playwright | Weekly |
| **Performance Tests** | Load testing | Locust | Monthly |

---

**This technical architecture provides a scalable, secure, and maintainable foundation for the Job Application Delegation System with clear separation of concerns and robust performance characteristics.**
