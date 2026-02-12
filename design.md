# AgriConnect: Technical Design Document

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-12 | Engineering Team | Initial design document |

## 1. System Architecture

### 1.1 High-Level Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     User Access Channels                     │
├──────────────┬──────────────┬──────────────┬────────────────┤
│  Voice IVR   │  WhatsApp    │     SMS      │   Web Portal   │
│  (Twilio)    │   (WhatsApp  │   (Twilio)   │   (React)      │
│              │    Business) │              │                │
└──────────────┴──────────────┴──────────────┴────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  API Gateway      │
                    │  (Rate Limiting,  │
                    │   Auth, Routing)  │
                    └─────────┬─────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   ┌────▼────┐          ┌────▼────┐          ┌────▼────┐
   │ Market  │          │ Weather │          │  Buyer  │
   │ Price   │          │ & Crop  │          │Directory│
   │Service  │          │Advisory │          │Service  │
   │         │          │Service  │          │         │
   └────┬────┘          └────┬────┘          └────┬────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Core Database  │
                    │  (PostgreSQL)   │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼────┐          ┌────▼────┐          ┌────▼────┐
   │AGMARKNET│          │Weather  │          │Govt     │
   │e-NAM    │          │API      │          │Schemes  │
   │         │          │(IMD)    │          │Database │
   └─────────┘          └─────────┘          └─────────┘
```

### 1.2 Technology Stack

#### Backend Services

**Application Runtime**
- Primary: Node.js 18+ LTS / Python 3.11+ (FastAPI framework)
- API Framework: Express.js / FastAPI with OpenAPI 3.0 specification
- Authentication: JWT-based authentication with refresh token rotation
- Session Management: Redis-backed session store

**Data Layer**
- Primary Database: PostgreSQL 15+ with PostGIS extension for geospatial queries
- Caching Layer: Redis 7+ (in-memory data store)
- Message Queue: RabbitMQ 3.12+ for asynchronous task processing
- Search Engine: Elasticsearch 8+ for full-text search capabilities

**AI/ML Infrastructure**
- LLM Integration: OpenAI GPT-4 API / Self-hosted Llama 2 (70B parameter model)
- NLP Framework: Hugging Face Transformers (mBERT, XLM-RoBERTa)
- ML Pipeline: Scikit-learn, TensorFlow 2.x for price prediction models
- Model Serving: TensorFlow Serving / TorchServe

#### Frontend Applications

**Web Application**
- Framework: React 18+ with TypeScript
- State Management: Redux Toolkit / Zustand
- UI Components: Material-UI / Tailwind CSS
- Build Tool: Vite / Webpack 5
- Progressive Web App (PWA) capabilities for offline functionality

**Mobile Strategy**
- Progressive Web App (PWA) for cross-platform compatibility
- Responsive design optimized for low-bandwidth scenarios
- Future consideration: React Native for native mobile applications

#### Communication Infrastructure

**Telephony Services**
- Voice IVR: Twilio Programmable Voice / Exotel
- Speech-to-Text: Google Cloud Speech-to-Text API / Azure Speech Services
- Text-to-Speech: Google Cloud Text-to-Speech / Azure TTS (regional language support)
- Call Recording: Twilio Recording API for quality assurance

**Messaging Platforms**
- WhatsApp: WhatsApp Business API (Cloud API or On-Premises)
- SMS Gateway: Twilio SMS / AWS SNS
- Push Notifications: Firebase Cloud Messaging (FCM)

#### Cloud Infrastructure

**Platform**
- Primary: AWS / Google Cloud Platform (multi-region deployment)
- Compute: EC2 / Compute Engine with auto-scaling groups
- Storage: S3 / Cloud Storage for static assets and backups
- CDN: CloudFront / Cloud CDN for content delivery

**DevOps & Orchestration**
- Containerization: Docker with multi-stage builds
- Orchestration: Kubernetes (EKS / GKE)
- Service Mesh: Istio for microservices communication
- CI/CD: GitHub Actions / GitLab CI with automated testing

**Monitoring & Observability**
- Metrics: Prometheus with Grafana dashboards
- Logging: ELK Stack (Elasticsearch, Logstash, Kibana)
- Tracing: Jaeger for distributed tracing
- Error Tracking: Sentry for real-time error monitoring
- APM: New Relic / Datadog for application performance monitoring

#### Security Infrastructure

**Authentication & Authorization**
- Identity Management: Custom JWT implementation with refresh tokens
- API Security: OAuth 2.0 for third-party integrations
- Secrets Management: AWS Secrets Manager / HashiCorp Vault
- Certificate Management: Let's Encrypt with automated renewal

**Network Security**
- Web Application Firewall (WAF): AWS WAF / Cloudflare
- DDoS Protection: AWS Shield / Cloudflare
- API Gateway: Kong / AWS API Gateway with rate limiting
- VPN: Site-to-site VPN for secure backend communication

## 2. Data Architecture

### 2.1 Database Schema Design

#### Entity Relationship Overview

The data model follows a normalized relational design with the following core entities and relationships:

- Users (Farmers) → Many-to-Many → Crops (via user_crops junction table)
- Market Prices → Many-to-One → Markets
- Market Prices → Many-to-One → Crops
- Buyers → Many-to-Many → Crops (via buyer_crops junction table)
- Government Schemes → Many-to-Many → States, Crops
- Crop Advisories → Many-to-Many → Regions, Crops
- Quote Requests → Many-to-One → Users, Buyers

#### Core Entity Definitions

**users (Farmer Profiles)**

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number VARCHAR(15) UNIQUE NOT NULL,
    phone_verified BOOLEAN DEFAULT FALSE,
    name VARCHAR(255) NOT NULL,
    state VARCHAR(100) NOT NULL,
    district VARCHAR(100) NOT NULL,
    village VARCHAR(100),
    language_preference VARCHAR(10) DEFAULT 'hi', -- ISO 639-1 codes
    land_size_acres DECIMAL(10,2),
    registration_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active TIMESTAMP,
    notification_frequency VARCHAR(20) DEFAULT 'daily', -- daily, weekly, none
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT valid_phone CHECK (phone_number ~ '^\+?[1-9]\d{1,14}$')
);

CREATE INDEX idx_users_phone ON users(phone_number);
CREATE INDEX idx_users_location ON users(state, district);
CREATE INDEX idx_users_last_active ON users(last_active);
```

**crops (Master Crop Data)**

```sql
CREATE TABLE crops (
    crop_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    crop_name VARCHAR(100) UNIQUE NOT NULL,
    crop_name_local JSONB, -- {"hi": "चावल", "ta": "அரிசி", ...}
    category VARCHAR(50), -- cereals, pulses, vegetables, fruits, etc.
    typical_season VARCHAR(50), -- kharif, rabi, zaid
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_crops_category ON crops(category);
```

**user_crops (User-Crop Association)**

```sql
CREATE TABLE user_crops (
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    crop_id UUID REFERENCES crops(crop_id) ON DELETE CASCADE,
    is_primary BOOLEAN DEFAULT FALSE,
    added_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, crop_id)
);
```

**markets (Market Master Data)**

```sql
CREATE TABLE markets (
    market_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    market_name VARCHAR(255) NOT NULL,
    market_code VARCHAR(50) UNIQUE, -- AGMARKNET code
    state VARCHAR(100) NOT NULL,
    district VARCHAR(100) NOT NULL,
    location_coordinates GEOGRAPHY(POINT, 4326), -- PostGIS geospatial
    market_type VARCHAR(50), -- wholesale, retail, mandi
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_markets_location ON markets USING GIST(location_coordinates);
CREATE INDEX idx_markets_state_district ON markets(state, district);
```

**market_prices (Price Data)**

```sql
CREATE TABLE market_prices (
    price_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    crop_id UUID REFERENCES crops(crop_id) NOT NULL,
    market_id UUID REFERENCES markets(market_id) NOT NULL,
    price_per_unit DECIMAL(10,2) NOT NULL,
    unit_type VARCHAR(20) DEFAULT 'quintal', -- quintal, kg, ton
    price_date DATE NOT NULL,
    data_source VARCHAR(50) NOT NULL, -- AGMARKNET, e-NAM, manual
    data_quality_score DECIMAL(3,2) DEFAULT 1.0, -- 0.0 to 1.0
    arrival_quantity DECIMAL(12,2), -- quantity arrived in market
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT valid_price CHECK (price_per_unit > 0),
    CONSTRAINT valid_quality_score CHECK (data_quality_score BETWEEN 0 AND 1)
);

CREATE INDEX idx_prices_crop_market_date ON market_prices(crop_id, market_id, price_date DESC);
CREATE INDEX idx_prices_date ON market_prices(price_date DESC);
CREATE INDEX idx_prices_source ON market_prices(data_source);
```

**buyers (Buyer Directory)**

```sql
CREATE TABLE buyers (
    buyer_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    contact_phone VARCHAR(15) NOT NULL,
    email VARCHAR(255),
    business_name VARCHAR(255),
    state VARCHAR(100) NOT NULL,
    district VARCHAR(100) NOT NULL,
    address TEXT,
    verification_status VARCHAR(20) DEFAULT 'pending', -- pending, verified, rejected
    verification_documents JSONB, -- document references
    rating DECIMAL(2,1) DEFAULT 0.0, -- 0.0 to 5.0
    review_count INTEGER DEFAULT 0,
    transaction_count INTEGER DEFAULT 0,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_verified_date TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    CONSTRAINT valid_rating CHECK (rating BETWEEN 0 AND 5)
);

CREATE INDEX idx_buyers_location ON buyers(state, district);
CREATE INDEX idx_buyers_verification ON buyers(verification_status);
CREATE INDEX idx_buyers_rating ON buyers(rating DESC);
```

**buyer_crops (Buyer-Crop Interest)**

```sql
CREATE TABLE buyer_crops (
    buyer_id UUID REFERENCES buyers(buyer_id) ON DELETE CASCADE,
    crop_id UUID REFERENCES crops(crop_id) ON DELETE CASCADE,
    min_quantity DECIMAL(10,2), -- minimum purchase quantity
    preferred_unit VARCHAR(20),
    added_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (buyer_id, crop_id)
);
```

**government_schemes (Subsidy Programs)**

```sql
CREATE TABLE government_schemes (
    scheme_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scheme_name VARCHAR(255) NOT NULL,
    scheme_code VARCHAR(50) UNIQUE,
    description TEXT NOT NULL,
    eligibility_criteria JSONB NOT NULL, -- structured eligibility rules
    applicable_states TEXT[], -- array of state codes
    applicable_crops UUID[], -- array of crop_ids
    subsidy_amount DECIMAL(12,2),
    subsidy_type VARCHAR(50), -- fixed, percentage, tiered
    application_deadline DATE,
    official_link VARCHAR(500),
    ministry VARCHAR(255),
    scheme_status VARCHAR(20) DEFAULT 'active', -- active, expired, suspended
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_schemes_status ON government_schemes(scheme_status);
CREATE INDEX idx_schemes_deadline ON government_schemes(application_deadline);
CREATE INDEX idx_schemes_states ON government_schemes USING GIN(applicable_states);
```

**crop_advisories (Agricultural Guidance)**

```sql
CREATE TABLE crop_advisories (
    advisory_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    crop_id UUID REFERENCES crops(crop_id),
    advisory_type VARCHAR(50) NOT NULL, -- pest, disease, weather, planting, harvesting
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    severity_level VARCHAR(20), -- low, medium, high, critical
    applicable_states TEXT[],
    applicable_districts TEXT[],
    valid_from DATE NOT NULL,
    valid_to DATE NOT NULL,
    recommended_action TEXT,
    source VARCHAR(100), -- IMD, ICAR, state agriculture dept
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT valid_date_range CHECK (valid_to >= valid_from)
);

CREATE INDEX idx_advisories_crop ON crop_advisories(crop_id);
CREATE INDEX idx_advisories_validity ON crop_advisories(valid_from, valid_to);
CREATE INDEX idx_advisories_type ON crop_advisories(advisory_type);
CREATE INDEX idx_advisories_states ON crop_advisories USING GIN(applicable_states);
```

**quote_requests (Buyer-Farmer Connections)**

```sql
CREATE TABLE quote_requests (
    request_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id) NOT NULL,
    buyer_id UUID REFERENCES buyers(buyer_id) NOT NULL,
    crop_id UUID REFERENCES crops(crop_id) NOT NULL,
    quantity DECIMAL(10,2) NOT NULL,
    unit_type VARCHAR(20) NOT NULL,
    request_status VARCHAR(20) DEFAULT 'pending', -- pending, responded, accepted, rejected, closed
    farmer_notes TEXT,
    buyer_response TEXT,
    quoted_price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    responded_at TIMESTAMP,
    closed_at TIMESTAMP
);

CREATE INDEX idx_quote_requests_user ON quote_requests(user_id);
CREATE INDEX idx_quote_requests_buyer ON quote_requests(buyer_id);
CREATE INDEX idx_quote_requests_status ON quote_requests(request_status);
```

**user_sessions (Authentication)**

```sql
CREATE TABLE user_sessions (
    session_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    access_token VARCHAR(500) NOT NULL,
    refresh_token VARCHAR(500) NOT NULL,
    device_info JSONB, -- device type, OS, app version
    ip_address INET,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    last_activity TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sessions_user ON user_sessions(user_id);
CREATE INDEX idx_sessions_expiry ON user_sessions(expires_at);
```

### 2.2 Data Partitioning Strategy

**Time-Series Data Partitioning**

Market prices table will be partitioned by month to optimize query performance:

```sql
CREATE TABLE market_prices_2026_02 PARTITION OF market_prices
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

Automated partition management via cron jobs or pg_partman extension.

### 2.3 Data Retention Policies

- Market prices: Retain 5 years, archive older data to cold storage
- User activity logs: Retain 2 years, then delete
- Crop advisories: Retain 3 years after expiry
- Quote requests: Retain 1 year after closure
- User sessions: Delete after expiration (24 hours)

## 3. API Architecture

### 3.1 API Design Principles

- RESTful architecture following OpenAPI 3.0 specification
- Versioned APIs (v1, v2) with backward compatibility guarantees
- Consistent error response format across all endpoints
- Pagination for list endpoints (cursor-based for large datasets)
- Rate limiting per user and API key
- Comprehensive request/response validation

### 3.2 Authentication & Authorization

**Authentication Flow**

```
1. User initiates registration/login via phone number
2. System sends OTP via SMS
3. User submits OTP for verification
4. System issues JWT access token (15 min expiry) and refresh token (7 day expiry)
5. Client includes access token in Authorization header for subsequent requests
6. Client uses refresh token to obtain new access token when expired
```

**Authorization Headers**

```
Authorization: Bearer <access_token>
X-API-Key: <api_key> (for service-to-service communication)
```

### 3.3 Core API Endpoints

#### Market Price APIs

**GET /api/v1/prices**

Retrieve current market prices for specified crop and location.

```
Query Parameters:
  - crop_id (UUID, required): Crop identifier
  - market_id (UUID, optional): Specific market, defaults to user's district
  - state (string, optional): State code for broader search
  - date (date, optional): Specific date, defaults to today

Response: 200 OK
{
  "status": "success",
  "data": {
    "prices": [
      {
        "price_id": "uuid",
        "crop": {
          "crop_id": "uuid",
          "name": "Rice",
          "name_local": "चावल"
        },
        "market": {
          "market_id": "uuid",
          "name": "Chennai Koyambedu Market",
          "district": "Chennai",
          "state": "Tamil Nadu"
        },
        "price_per_unit": 2100.00,
        "unit_type": "quintal",
        "date": "2026-02-12",
        "trend": "up", // up, down, stable
        "trend_percentage": 2.5,
        "data_source": "AGMARKNET",
        "data_quality_score": 0.95
      }
    ],
    "metadata": {
      "total_results": 1,
      "query_date": "2026-02-12"
    }
  }
}

Error Responses:
  - 400: Invalid parameters
  - 404: No price data found
  - 429: Rate limit exceeded
  - 500: Internal server error
```

**GET /api/v1/prices/trends**

Retrieve historical price trends for analysis.

```
Query Parameters:
  - crop_id (UUID, required)
  - market_id (UUID, optional)
  - days (integer, default: 7, max: 90): Number of days for trend analysis
  - aggregation (string, optional): daily, weekly, monthly

Response: 200 OK
{
  "status": "success",
  "data": {
    "crop": { "crop_id": "uuid", "name": "Rice" },
    "market": { "market_id": "uuid", "name": "Chennai Market" },
    "period": {
      "start_date": "2026-02-05",
      "end_date": "2026-02-12",
      "days": 7
    },
    "prices": [
      {
        "date": "2026-02-12",
        "price": 2100.00,
        "arrival_quantity": 1500.00
      },
      // ... more daily prices
    ],
    "statistics": {
      "avg_price": 2050.00,
      "min_price": 2000.00,
      "max_price": 2100.00,
      "std_deviation": 35.50,
      "trend_direction": "increasing",
      "volatility": "low" // low, medium, high
    }
  }
}
```

**GET /api/v1/prices/compare**

Compare prices across multiple markets.

```
Query Parameters:
  - crop_id (UUID, required)
  - market_ids (UUID[], required): Comma-separated market IDs
  - date (date, optional)

Response: 200 OK
{
  "status": "success",
  "data": {
    "crop": { "crop_id": "uuid", "name": "Rice" },
    "date": "2026-02-12",
    "comparisons": [
      {
        "market": { "market_id": "uuid", "name": "Chennai Market" },
        "price": 2100.00,
        "rank": 1
      },
      {
        "market": { "market_id": "uuid", "name": "Coimbatore Market" },
        "price": 2050.00,
        "rank": 2
      }
    ],
    "price_range": {
      "highest": 2100.00,
      "lowest": 2050.00,
      "difference": 50.00,
      "difference_percentage": 2.44
    }
  }
}
```

#### Weather & Advisory APIs

**GET /api/v1/weather**

Retrieve weather forecast for specified location.

```
Query Parameters:
  - latitude (float, optional): GPS coordinates
  - longitude (float, optional): GPS coordinates
  - district (string, optional): District name
  - state (string, optional): State name
  - days (integer, default: 5, max: 10): Forecast days

Response: 200 OK
{
  "status": "success",
  "data": {
    "location": {
      "district": "Chennai",
      "state": "Tamil Nadu",
      "coordinates": { "lat": 13.0827, "lon": 80.2707 }
    },
    "current": {
      "temperature": 32.5,
      "humidity": 75,
      "conditions": "Partly Cloudy",
      "wind_speed": 15.5,
      "updated_at": "2026-02-12T10:00:00Z"
    },
    "forecast": [
      {
        "date": "2026-02-13",
        "temp_min": 24.0,
        "temp_max": 33.0,
        "rainfall_mm": 0.0,
        "conditions": "Sunny",
        "humidity": 70
      },
      // ... more forecast days
    ],
    "alerts": [
      {
        "alert_id": "uuid",
        "type": "heavy_rain",
        "severity": "medium",
        "description": "Heavy rainfall expected on 2026-02-15",
        "valid_from": "2026-02-15T00:00:00Z",
        "valid_to": "2026-02-15T23:59:59Z"
      }
    ],
    "data_source": "IMD",
    "last_updated": "2026-02-12T10:00:00Z"
  }
}
```

**GET /api/v1/advisories**

Retrieve crop-specific advisories.

```
Query Parameters:
  - crop_id (UUID, optional): Specific crop
  - user_id (UUID, optional): Get advisories for user's crops
  - state (string, optional)
  - district (string, optional)
  - advisory_type (string, optional): pest, disease, weather, planting, harvesting
  - active_only (boolean, default: true): Only return currently valid advisories

Response: 200 OK
{
  "status": "success",
  "data": {
    "advisories": [
      {
        "advisory_id": "uuid",
        "crop": { "crop_id": "uuid", "name": "Rice" },
        "type": "pest",
        "title": "Brown Planthopper Alert",
        "description": "Increased brown planthopper activity detected...",
        "severity": "high",
        "applicable_regions": {
          "states": ["Tamil Nadu", "Karnataka"],
          "districts": ["Chennai", "Coimbatore"]
        },
        "valid_from": "2026-02-10",
        "valid_to": "2026-02-20",
        "recommended_action": "Apply recommended pesticides...",
        "source": "ICAR",
        "created_at": "2026-02-10T08:00:00Z"
      }
    ],
    "metadata": {
      "total_results": 1,
      "active_advisories": 1
    }
  }
}
```

#### Buyer Directory APIs

**GET /api/v1/buyers**

Search for verified buyers.

```
Query Parameters:
  - crop_id (UUID, optional): Buyers interested in specific crop
  - state (string, optional)
  - district (string, optional)
  - min_rating (float, optional): Minimum buyer rating (0-5)
  - verification_status (string, default: verified): verified, pending, all
  - page (integer, default: 1)
  - limit (integer, default: 20, max: 100)

Response: 200 OK
{
  "status": "success",
  "data": {
    "buyers": [
      {
        "buyer_id": "uuid",
        "name": "ABC Traders",
        "business_name": "ABC Agricultural Commodities Pvt Ltd",
        "location": {
          "district": "Chennai",
          "state": "Tamil Nadu"
        },
        "crops_interested": [
          { "crop_id": "uuid", "name": "Rice", "min_quantity": 100.0, "unit": "quintal" }
        ],
        "rating": 4.5,
        "review_count": 120,
        "transaction_count": 450,
        "verification_status": "verified",
        "last_verified": "2026-01-15"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total_results": 45,
      "total_pages": 3
    }
  }
}
```

**POST /api/v1/buyers/quote-request**

Submit a quote request to a buyer.

```
Request Body:
{
  "buyer_id": "uuid",
  "crop_id": "uuid",
  "quantity": 50.0,
  "unit_type": "quintal",
  "notes": "Organic rice, ready for immediate delivery"
}

Response: 201 Created
{
  "status": "success",
  "data": {
    "request_id": "uuid",
    "status": "pending",
    "buyer": {
      "buyer_id": "uuid",
      "name": "ABC Traders"
    },
    "crop": { "crop_id": "uuid", "name": "Rice" },
    "quantity": 50.0,
    "unit_type": "quintal",
    "created_at": "2026-02-12T14:30:00Z",
    "estimated_response_time": "24-48 hours"
  }
}

Error Responses:
  - 400: Invalid request data
  - 404: Buyer not found
  - 409: Duplicate quote request
  - 429: Rate limit exceeded (max 10 requests per day)
```

**GET /api/v1/buyers/quote-requests**

Retrieve user's quote request history.

```
Query Parameters:
  - status (string, optional): pending, responded, accepted, rejected, closed
  - page (integer, default: 1)
  - limit (integer, default: 20)

Response: 200 OK
{
  "status": "success",
  "data": {
    "quote_requests": [
      {
        "request_id": "uuid",
        "buyer": { "buyer_id": "uuid", "name": "ABC Traders" },
        "crop": { "crop_id": "uuid", "name": "Rice" },
        "quantity": 50.0,
        "unit_type": "quintal",
        "status": "responded",
        "quoted_price": 2050.00,
        "buyer_response": "We can offer Rs 2050 per quintal...",
        "created_at": "2026-02-10T10:00:00Z",
        "responded_at": "2026-02-11T15:30:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total_results": 5
    }
  }
}
```

#### Government Schemes APIs

**GET /api/v1/schemes**

Search for government agricultural schemes.

```
Query Parameters:
  - state (string, optional): State code
  - crop_id (UUID, optional): Schemes applicable to specific crop
  - scheme_status (string, default: active): active, expired, all
  - keyword (string, optional): Search in scheme name/description
  - page (integer, default: 1)
  - limit (integer, default: 20)

Response: 200 OK
{
  "status": "success",
  "data": {
    "schemes": [
      {
        "scheme_id": "uuid",
        "scheme_name": "Pradhan Mantri Fasal Bima Yojana",
        "scheme_code": "PMFBY-2024",
        "description": "Crop insurance scheme providing financial support...",
        "eligibility_criteria": {
          "farmer_type": ["small", "marginal"],
          "land_ownership": ["owned", "leased"],
          "crops": ["all"]
        },
        "applicable_states": ["all"],
        "applicable_crops": ["all"],
        "subsidy_amount": null,
        "subsidy_type": "insurance",
        "application_deadline": "2026-06-30",
        "official_link": "https://pmfby.gov.in",
        "ministry": "Ministry of Agriculture & Farmers Welfare",
        "scheme_status": "active",
        "last_updated": "2026-01-15"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total_results": 12
    }
  }
}
```

**POST /api/v1/schemes/check-eligibility**

Check user eligibility for a specific scheme.

```
Request Body:
{
  "scheme_id": "uuid",
  "user_profile": {
    "land_size_acres": 3.5,
    "crops": ["uuid1", "uuid2"],
    "state": "Tamil Nadu",
    "farmer_type": "small"
  }
}

Response: 200 OK
{
  "status": "success",
  "data": {
    "scheme": {
      "scheme_id": "uuid",
      "scheme_name": "Pradhan Mantri Fasal Bima Yojana"
    },
    "eligible": true,
    "eligibility_details": {
      "criteria_met": [
        "Farmer type: small (eligible)",
        "Land size: 3.5 acres (within limit)",
        "State: Tamil Nadu (applicable)"
      ],
      "criteria_not_met": [],
      "additional_requirements": [
        "Valid land ownership documents required",
        "Aadhaar card mandatory"
      ]
    },
    "next_steps": [
      "Visit nearest agriculture office",
      "Submit application before 2026-06-30",
      "Required documents: Land records, Aadhaar, Bank details"
    ]
  }
}
```

#### User Management APIs

**POST /api/v1/users/register**

Register a new farmer user.

```
Request Body:
{
  "phone_number": "+919876543210",
  "name": "Ravi Kumar",
  "state": "Tamil Nadu",
  "district": "Chennai",
  "village": "Perungudi",
  "language_preference": "ta",
  "crops_interested": ["uuid1", "uuid2"],
  "land_size_acres": 2.5
}

Response: 201 Created
{
  "status": "success",
  "data": {
    "user_id": "uuid",
    "phone_number": "+919876543210",
    "name": "Ravi Kumar",
    "verification_required": true,
    "otp_sent": true,
    "message": "OTP sent to your phone number. Please verify to complete registration."
  }
}
```

**POST /api/v1/users/verify-otp**

Verify phone number with OTP.

```
Request Body:
{
  "phone_number": "+919876543210",
  "otp": "123456"
}

Response: 200 OK
{
  "status": "success",
  "data": {
    "user_id": "uuid",
    "phone_verified": true,
    "access_token": "jwt_token",
    "refresh_token": "refresh_token",
    "expires_in": 900
  }
}
```

**GET /api/v1/users/profile**

Retrieve user profile information.

```
Headers:
  Authorization: Bearer <access_token>

Response: 200 OK
{
  "status": "success",
  "data": {
    "user_id": "uuid",
    "phone_number": "+919876543210",
    "name": "Ravi Kumar",
    "location": {
      "state": "Tamil Nadu",
      "district": "Chennai",
      "village": "Perungudi"
    },
    "language_preference": "ta",
    "crops_interested": [
      { "crop_id": "uuid", "name": "Rice", "name_local": "அரிசி" }
    ],
    "land_size_acres": 2.5,
    "registration_date": "2026-01-15T10:00:00Z",
    "last_active": "2026-02-12T14:30:00Z",
    "notification_frequency": "daily"
  }
}
```

**PUT /api/v1/users/preferences**

Update user preferences.

```
Headers:
  Authorization: Bearer <access_token>

Request Body:
{
  "language_preference": "hi",
  "crops_interested": ["uuid1", "uuid2", "uuid3"],
  "notification_frequency": "weekly"
}

Response: 200 OK
{
  "status": "success",
  "data": {
    "user_id": "uuid",
    "preferences_updated": true,
    "updated_fields": ["language_preference", "crops_interested", "notification_frequency"]
  }
}
```

### 3.4 Error Response Format

All API errors follow a consistent format:

```json
{
  "status": "error",
  "error": {
    "code": "INVALID_PARAMETERS",
    "message": "Invalid crop_id provided",
    "details": {
      "field": "crop_id",
      "reason": "UUID format expected"
    },
    "request_id": "uuid",
    "timestamp": "2026-02-12T14:30:00Z"
  }
}
```

### 3.5 Rate Limiting

- Authenticated users: 100 requests per hour
- Anonymous users: 20 requests per hour
- Quote requests: 10 per day per user
- Bulk operations: 5 per hour per user

Rate limit headers included in all responses:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1644678000
```

## 4. Voice IVR System Design

### 4.1 IVR Architecture

**Call Flow Management**
- Twilio Studio / Exotel Studio for visual call flow design
- State machine-based conversation management
- Context preservation across menu navigation
- Fallback to human operator for complex queries

**Speech Recognition Pipeline**
1. Audio capture from telephony network
2. Speech-to-Text conversion (Google Cloud Speech-to-Text / Azure)
3. Language detection and transcription
4. Intent recognition via NLP model
5. Entity extraction (crop names, locations, dates)
6. Response generation
7. Text-to-Speech conversion
8. Audio playback to caller

### 4.2 Main Menu Structure

**Level 1: Primary Menu**

```
[Welcome Message in User's Language]
"नमस्ते, AgriConnect में आपका स्वागत है।"
"Welcome to AgriConnect."

[Menu Options]
"कृपया चुनें:"
"Please select:"

1. फसल की कीमतें जानने के लिए 1 दबाएं
   Press 1 for crop prices

2. मौसम और सलाह के लिए 2 दबाएं
   Press 2 for weather and advisory

3. खरीदार खोजने के लिए 3 दबाएं
   Press 3 to find buyers

4. सरकारी योजनाओं के लिए 4 दबाएं
   Press 4 for government schemes

5. फसल योजना के लिए 5 दबाएं
   Press 5 for crop planning

6. सहायता के लिए 6 दबाएं
   Press 6 for help

0. मुख्य मेनू पर वापस जाने के लिए 0 दबाएं
   Press 0 to return to main menu
```

### 4.3 Price Lookup Flow (Option 1)

**Step 1: Crop Selection**

```
[Voice Prompt]
"आप किस फसल की कीमत जानना चाहते हैं?"
"Which crop price would you like to know?"

[Options]
"बोलें या दबाएं:"
"Say or press:"

1. धान / चावल (Rice)
2. गेहूं (Wheat)
3. कपास (Cotton)
4. मक्का (Maize)
5. अन्य फसल के लिए बोलें (Say other crop name)

[Natural Language Input]
User can speak: "धान की कीमत" or "Rice price"
System extracts: crop = "Rice"
```

**Step 2: Market Selection**

```
[Voice Prompt]
"किस बाजार की कीमत चाहिए?"
"Which market price do you need?"

[Options]
1. स्थानीय बाजार (Local market - based on user's district)
2. राज्य की राजधानी (State capital market)
3. राष्ट्रीय बाजार (National markets)
4. विशिष्ट बाजार का नाम बोलें (Say specific market name)

[Context-Aware Default]
If user profile has location: "आपके जिले Chennai के लिए कीमत दिखा रहे हैं"
"Showing prices for your district Chennai"
```

**Step 3: Price Information Delivery**

```
[Response Format]
"आज की तारीख 12 फरवरी 2026"
"Today's date: 12th February 2026"

"धान की कीमत Chennai Koyambedu बाजार में:"
"Rice price in Chennai Koyambedu market:"

"2,100 रुपये प्रति क्विंटल"
"Rs 2,100 per quintal"

"कल से 2% ऊपर"
"Up 2% from yesterday"

[Follow-up Options]
"अधिक जानकारी के लिए:"
"For more information:"

1. 7 दिन का रुझान सुनने के लिए 1 दबाएं
   Press 1 to hear 7-day trend

2. अन्य बाजारों की कीमत के लिए 2 दबाएं
   Press 2 for prices in other markets

3. खरीदार खोजने के लिए 3 दबाएं
   Press 3 to find buyers

0. मुख्य मेनू पर वापस जाने के लिए 0 दबाएं
   Press 0 to return to main menu
```

### 4.4 Natural Language Processing Flow

**Intent Recognition**

User input: "मुझे चेन्नई में धान की कीमत बताओ"
(Tell me rice price in Chennai)

NLP Processing:
```json
{
  "intent": "get_price",
  "entities": {
    "crop": {
      "value": "rice",
      "confidence": 0.95,
      "local_name": "धान"
    },
    "location": {
      "value": "Chennai",
      "type": "city",
      "confidence": 0.98
    }
  },
  "language": "hi",
  "confidence": 0.92
}
```

**Contextual Understanding**

Conversation history maintained:
```json
{
  "session_id": "uuid",
  "user_id": "uuid",
  "conversation_context": {
    "last_crop_queried": "rice",
    "last_market_queried": "Chennai",
    "user_location": {
      "state": "Tamil Nadu",
      "district": "Chennai"
    },
    "language": "hi",
    "turn_count": 3
  }
}
```

Follow-up query: "कल की कीमत क्या थी?"
(What was yesterday's price?)

System infers: Get price for "rice" in "Chennai" for "yesterday"

### 4.5 Error Handling & Fallbacks

**Speech Recognition Failure**

```
[Low Confidence Detection]
If confidence < 0.7:
  "क्षमा करें, मैं समझ नहीं पाया। कृपया फिर से बोलें।"
  "Sorry, I didn't understand. Please say again."

[After 2 Failed Attempts]
  "मैं आपको मेनू विकल्प बता रहा हूं..."
  "Let me give you menu options..."
  [Switch to DTMF menu]
```

**Data Unavailability**

```
[No Price Data Found]
"क्षमा करें, इस समय [crop] की कीमत [market] बाजार के लिए उपलब्ध नहीं है।"
"Sorry, price for [crop] in [market] market is not available at this time."

"क्या आप:"
"Would you like to:"
1. अन्य बाजार की कीमत जानना चाहते हैं? (Check other markets?)
2. कल की कीमत सुनना चाहते हैं? (Hear yesterday's price?)
3. मुख्य मेनू पर वापस जाना चाहते हैं? (Return to main menu?)
```

**Network Issues**

```
[API Timeout]
"कृपया प्रतीक्षा करें, जानकारी प्राप्त की जा रही है..."
"Please wait, fetching information..."

[After 10 seconds]
"नेटवर्क में समस्या है। कृपया कुछ समय बाद पुनः प्रयास करें।"
"There is a network issue. Please try again later."

[Offer Callback]
"क्या आप चाहते हैं कि हम आपको वापस कॉल करें?"
"Would you like us to call you back?"
```

### 4.6 Call Quality & Performance

**Audio Quality Standards**
- Minimum bitrate: 8 kbps (G.711 codec)
- Maximum latency: 200ms for speech recognition
- Echo cancellation enabled
- Background noise suppression

**Performance Metrics**
- Average call duration: 2-3 minutes
- Speech recognition accuracy: >90% for agricultural vocabulary
- Menu navigation success rate: >95%
- Call completion rate: >85%

### 4.7 Accessibility Features

**Support for Feature Phones**
- DTMF (touch-tone) input as primary navigation
- Voice input as enhancement, not requirement
- Clear audio prompts with appropriate pauses
- Option to repeat information

**Low Literacy Considerations**
- Simple, conversational language
- Avoid technical jargon
- Use local crop names and terminology
- Provide examples in prompts

**Network Resilience**
- Optimized for 2G networks
- Compressed audio formats
- Graceful degradation on poor connections
- Automatic retry mechanisms

## 5. WhatsApp & SMS Integration

### 5.1 WhatsApp Business API Architecture

**Platform Setup**
- WhatsApp Business API (Cloud API or On-Premises)
- Webhook integration for message handling
- Message template management for notifications
- Media handling for images (future: crop disease identification)

**Conversation Flow Management**
- Stateful conversation tracking
- Context preservation across messages
- Session timeout: 24 hours of inactivity
- Conversation history stored for 30 days

### 5.2 WhatsApp Bot Interaction Patterns

**Natural Language Query**

```
User: "Rice prices in Chennai"

Bot Response:
🌾 *Rice Prices in Chennai*
📍 Market: Koyambedu Wholesale Market
💰 Price: ₹2,100 per quintal
📈 Trend: ↑ 2% from yesterday
📅 Date: 12 Feb 2026

What would you like to do next?
1️⃣ See 7-day trend
2️⃣ Compare with other markets
3️⃣ Find buyers
4️⃣ Get crop advisory
🏠 Main menu

[Quick Reply Buttons]
```

**Interactive Buttons**

WhatsApp supports interactive buttons for better UX:

```json
{
  "type": "interactive",
  "interactive": {
    "type": "button",
    "body": {
      "text": "Rice price in Chennai: ₹2,100/quintal (↑2%)"
    },
    "action": {
      "buttons": [
        {
          "type": "reply",
          "reply": {
            "id": "trend_7day",
            "title": "7-Day Trend"
          }
        },
        {
          "type": "reply",
          "reply": {
            "id": "find_buyers",
            "title": "Find Buyers"
          }
        },
        {
          "type": "reply",
          "reply": {
            "id": "main_menu",
            "title": "Main Menu"
          }
        }
      ]
    }
  }
}
```

**List Messages for Multiple Options**

```
User: "Show me buyers for rice"

Bot Response:
🤝 *Verified Buyers for Rice*

Select a buyer to view details:

[List Message with up to 10 buyers]
1. ABC Traders - Chennai (⭐ 4.5)
2. XYZ Commodities - Coimbatore (⭐ 4.8)
3. PQR Exports - Madurai (⭐ 4.2)
...

[User selects from list]

Bot Response:
📋 *ABC Traders*
🏢 Business: ABC Agricultural Commodities Pvt Ltd
📍 Location: Chennai, Tamil Nadu
⭐ Rating: 4.5/5 (120 reviews)
📦 Interested in: Rice, Wheat, Pulses
✅ Verified: Yes
📞 Transactions: 450+

Would you like to:
1️⃣ Request a quote
2️⃣ View contact details
3️⃣ See other buyers
🔙 Back

[Quick Reply Buttons]
```

**Voice Message Support**

```
User: [Sends voice message in Tamil]
"சென்னையில் நெல் விலை என்ன?"

Bot Processing:
1. Download voice message
2. Speech-to-Text conversion
3. Language detection: Tamil
4. Intent extraction: get_price
5. Entity extraction: crop=rice, location=Chennai
6. Generate response

Bot Response:
🎤 *Voice message received*

🌾 *நெல் விலை சென்னை*
📍 சந்தை: கோயம்பேடு மொத்த சந்தை
💰 விலை: ₹2,100 ஒரு குவிண்டால்
📈 போக்கு: நேற்றிலிருந்து ↑ 2%
📅 தேதி: 12 பிப் 2026

[Response in Tamil with same interactive options]
```

### 5.3 SMS Gateway Integration

**SMS Command Structure**

AgriConnect uses structured SMS commands for feature phone users:

**Command Format:**
```
<COMMAND> <CROP> <LOCATION>
```

**Supported Commands:**

```
PRICE <crop> <location>
Example: PRICE RICE CHENNAI
Response: "Rice Chennai: Rs 2100/qt. Trend: +2%. Reply TREND for 7-day, BUYERS for contacts, HELP for menu"

TREND <crop> <location> [days]
Example: TREND RICE CHENNAI 7
Response: "Rice Chennai 7-day: Avg Rs2050, Min Rs2000, Max Rs2100. Trend: Rising. Reply BUYERS to find buyers"

WEATHER <location>
Example: WEATHER CHENNAI
Response: "Chennai: 32°C, Partly cloudy. Tomorrow: 28-33°C, Light rain. Reply ADVISORY for crop tips"

BUYERS <crop> <location>
Example: BUYERS RICE CHENNAI
Response: "3 buyers found. 1)ABC Traders 4.5★ 2)XYZ Commodities 4.8★. Reply BUYER1 for details"

SCHEME <state> [crop]
Example: SCHEME TN RICE
Response: "2 schemes: 1)PMFBY Insurance 2)MSP Support. Reply SCHEME1 for details"

HELP
Response: "Commands: PRICE, TREND, WEATHER, BUYERS, SCHEME. Format: PRICE RICE CHENNAI. Call 1800-XXX-XXXX for voice"

REGISTER <name> <location>
Example: REGISTER RAVI CHENNAI
Response: "Welcome Ravi! OTP sent. Reply OTP <code> to verify. Call 1800-XXX-XXXX for help"
```

**SMS Response Optimization**

- Maximum 160 characters per SMS (single message preferred)
- Use abbreviations: Rs (Rupees), qt (quintal), ★ (rating)
- Split into multiple SMS only when necessary
- Include clear next-step instructions
- Provide toll-free number for complex queries

**SMS Delivery Tracking**

```json
{
  "message_id": "uuid",
  "user_phone": "+919876543210",
  "command": "PRICE RICE CHENNAI",
  "response": "Rice Chennai: Rs 2100/qt...",
  "sent_at": "2026-02-12T14:30:00Z",
  "delivery_status": "delivered",
  "delivery_time": "2026-02-12T14:30:05Z",
  "carrier": "Airtel"
}
```

### 5.4 Notification System

**Proactive Notifications via WhatsApp**

**Price Alerts**
```
🔔 *Price Alert*

🌾 Rice price in Chennai has increased!

Yesterday: ₹2,050/quintal
Today: ₹2,100/quintal
Change: ↑ ₹50 (+2.4%)

This might be a good time to sell.

[Button: View Details]
[Button: Find Buyers]
```

**Weather Alerts**
```
⚠️ *Weather Alert*

Heavy rainfall expected in Chennai district

📅 Date: 15 Feb 2026
🌧️ Rainfall: 50-80mm
⚡ Thunderstorms likely

Recommended actions:
• Postpone harvesting if possible
• Ensure proper drainage
• Protect stored crops

[Button: Full Forecast]
[Button: Crop Advisory]
```

**Scheme Deadlines**
```
⏰ *Scheme Reminder*

PMFBY Crop Insurance application deadline approaching!

📅 Deadline: 28 Feb 2026 (16 days left)
💰 Coverage: Up to ₹50,000 per acre
✅ You are eligible

[Button: Check Eligibility]
[Button: Application Steps]
```

**Message Template Management**

WhatsApp requires pre-approved templates for notifications:

```json
{
  "name": "price_alert",
  "language": "hi",
  "category": "UTILITY",
  "components": [
    {
      "type": "BODY",
      "text": "{{crop}} की कीमत {{location}} में बदल गई है!\n\nकल: ₹{{yesterday_price}}\nआज: ₹{{today_price}}\nबदलाव: {{change_percentage}}%"
    },
    {
      "type": "BUTTONS",
      "buttons": [
        {
          "type": "QUICK_REPLY",
          "text": "विवरण देखें"
        },
        {
          "type": "QUICK_REPLY",
          "text": "खरीदार खोजें"
        }
      ]
    }
  ]
}
```

### 5.5 Multi-Channel Consistency

**Unified User Experience**

- Same information across all channels (voice, WhatsApp, SMS, web)
- Conversation context shared across channels
- User can start on SMS, continue on WhatsApp
- Preferences (language, crops) apply to all channels

**Channel Selection Logic**

```
User has smartphone + WhatsApp → Prefer WhatsApp
User has feature phone only → SMS + Voice IVR
User explicitly requests SMS → Honor preference
Emergency alerts → Send via all available channels
```

## 6. Data Integration Strategy

### 6.1 External Data Sources

#### AGMARKNET Integration

**API Details**
- Source: Agricultural Marketing Information Network
- URL: https://agmarknet.gov.in/
- Data: Daily commodity prices from 3,000+ markets across India
- Update Frequency: Daily at 6 AM, 12 PM, 6 PM IST
- Format: JSON/XML REST API
- Authentication: API key-based

**Data Sync Process**

```python
# Pseudocode for AGMARKNET sync
def sync_agmarknet_prices():
    markets = get_active_markets()
    crops = get_active_crops()
    
    for market in markets:
        for crop in crops:
            try:
                price_data = agmarknet_api.get_price(
                    market_code=market.code,
                    crop_code=crop.code,
                    date=today()
                )
                
                # Validate data quality
                quality_score = validate_price_data(price_data)
                
                # Store in database
                store_market_price(
                    crop_id=crop.id,
                    market_id=market.id,
                    price=price_data.modal_price,
                    unit=price_data.unit,
                    date=price_data.date,
                    source='AGMARKNET',
                    quality_score=quality_score,
                    arrival_quantity=price_data.arrivals
                )
                
                # Cache for quick access
                cache.set(f"price:{crop.id}:{market.id}", price_data, ttl=3600)
                
            except APIError as e:
                log_error(f"AGMARKNET sync failed: {e}")
                send_alert_to_ops_team(e)
```

**Data Quality Validation**

```python
def validate_price_data(price_data):
    quality_score = 1.0
    
    # Check for missing fields
    if not price_data.modal_price:
        quality_score -= 0.3
    
    # Check for outliers (price deviation > 50% from historical avg)
    historical_avg = get_historical_average(crop, market, days=30)
    deviation = abs(price_data.modal_price - historical_avg) / historical_avg
    if deviation > 0.5:
        quality_score -= 0.2
        flag_for_manual_review(price_data)
    
    # Check data freshness
    if (today() - price_data.date).days > 1:
        quality_score -= 0.2
    
    return max(quality_score, 0.0)
```

#### e-NAM Integration

**API Details**
- Source: National Agriculture Market
- URL: https://www.enam.gov.in/
- Data: Real-time trading prices from 1,000+ integrated mandis
- Update Frequency: Real-time during market hours (6 AM - 8 PM)
- Format: REST API with JSON responses
- Authentication: OAuth 2.0

**Integration Approach**
- Webhook subscriptions for real-time price updates
- Fallback to polling every 30 minutes during market hours
- Data enrichment with AGMARKNET for comprehensive coverage

#### Indian Meteorological Department (IMD)

**API Details**
- Source: India Meteorological Department
- URL: https://mausam.imd.gov.in/
- Data: Weather forecasts, alerts, historical data
- Update Frequency: 4 times daily (00:00, 06:00, 12:00, 18:00 IST)
- Format: REST API with JSON/XML
- Authentication: API key

**Data Retrieved**
- 5-day weather forecast (temperature, rainfall, humidity, wind)
- Severe weather alerts (cyclones, heavy rain, heatwaves)
- District-level granularity
- Agricultural weather advisories

**Sync Schedule**
```
00:30 IST - Sync morning forecast
06:30 IST - Sync updated forecast
12:30 IST - Sync afternoon forecast
18:30 IST - Sync evening forecast
```

#### Government Schemes Database

**Data Sources**
- Ministry of Agriculture & Farmers Welfare portal
- State agriculture department websites
- Direct government API integrations (where available)
- Web scraping for portals without APIs (with proper authorization)

**Update Frequency**
- Weekly automated sync
- Manual verification by data team
- Quarterly deep audit for accuracy

**Data Structure**
```json
{
  "scheme_id": "uuid",
  "source_url": "https://agricoop.gov.in/scheme/...",
  "last_scraped": "2026-02-12T10:00:00Z",
  "verification_status": "verified",
  "verified_by": "data_team_member_id",
  "verified_at": "2026-02-10T15:00:00Z",
  "next_verification_due": "2026-05-10"
}
```

### 6.2 Data Synchronization Architecture

**Sync Job Orchestration**

```yaml
# Airflow DAG for data synchronization
dag_id: agricultural_data_sync
schedule: "0 */6 * * *"  # Every 6 hours

tasks:
  - task_id: sync_agmarknet_prices
    retry: 3
    timeout: 600
    on_failure: alert_ops_team
    
  - task_id: sync_enam_prices
    retry: 3
    timeout: 300
    depends_on: sync_agmarknet_prices
    
  - task_id: sync_weather_data
    retry: 3
    timeout: 300
    parallel: true
    
  - task_id: validate_data_quality
    depends_on: [sync_agmarknet_prices, sync_enam_prices]
    
  - task_id: update_cache
    depends_on: validate_data_quality
    
  - task_id: send_price_alerts
    depends_on: update_cache
```

**Message Queue for Async Processing**

```
RabbitMQ Queues:
- price_sync_queue: High priority, processes price updates
- weather_sync_queue: Medium priority, processes weather data
- notification_queue: High priority, sends user notifications
- analytics_queue: Low priority, processes analytics events

Dead Letter Queue (DLQ):
- failed_sync_queue: Stores failed sync jobs for manual review
```

**Caching Strategy**

```
Redis Cache Layers:

L1 - Hot Data (TTL: 1 hour)
- Current day prices for popular crops
- User session data
- Active conversation contexts

L2 - Warm Data (TTL: 6 hours)
- Historical price trends (7-day, 30-day)
- Weather forecasts
- Buyer directory

L3 - Cold Data (TTL: 24 hours)
- Government schemes
- Crop advisories
- Market master data

Cache Invalidation:
- On data sync completion
- On manual data updates
- On cache expiry
```

### 6.3 Data Quality Management

**Quality Metrics**

```sql
-- Data quality dashboard query
SELECT 
    data_source,
    COUNT(*) as total_records,
    AVG(data_quality_score) as avg_quality,
    COUNT(CASE WHEN data_quality_score < 0.7 THEN 1 END) as low_quality_count,
    MAX(created_at) as last_update,
    EXTRACT(EPOCH FROM (NOW() - MAX(created_at)))/3600 as hours_since_update
FROM market_prices
WHERE price_date >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY data_source;
```

**Automated Quality Checks**

1. **Completeness Check**: All required fields populated
2. **Consistency Check**: Price within expected range for crop/season
3. **Timeliness Check**: Data freshness < 24 hours
4. **Accuracy Check**: Cross-validation with multiple sources
5. **Uniqueness Check**: No duplicate records

**Quality Score Calculation**

```
Quality Score = (
    0.3 * completeness_score +
    0.3 * consistency_score +
    0.2 * timeliness_score +
    0.1 * accuracy_score +
    0.1 * uniqueness_score
)

Thresholds:
- Score >= 0.9: Excellent
- Score >= 0.7: Good
- Score >= 0.5: Acceptable
- Score < 0.5: Poor (flag for review)
```

### 6.4 Fallback Mechanisms

**Data Source Unavailability**

```python
def get_crop_price(crop_id, market_id, date=today()):
    # Try primary source (AGMARKNET)
    try:
        price = agmarknet_api.get_price(crop_id, market_id, date)
        if price:
            return price
    except APIError:
        log_warning("AGMARKNET unavailable, trying fallback")
    
    # Try secondary source (e-NAM)
    try:
        price = enam_api.get_price(crop_id, market_id, date)
        if price:
            return price
    except APIError:
        log_warning("e-NAM unavailable, using cached data")
    
    # Use cached data
    cached_price = cache.get(f"price:{crop_id}:{market_id}")
    if cached_price:
        return cached_price
    
    # Use historical average as last resort
    historical_price = get_historical_average(crop_id, market_id, days=7)
    if historical_price:
        return {
            "price": historical_price,
            "source": "historical_average",
            "note": "Real-time data unavailable"
        }
    
    # No data available
    raise DataUnavailableError("Price data not available")
```

**Graceful Degradation**

- If real-time prices unavailable → Show yesterday's prices with disclaimer
- If weather API down → Show cached forecast with timestamp
- If buyer directory unavailable → Show cached list with "last updated" info
- If LLM API down → Fall back to rule-based intent recognition

## 7. Multilingual NLP Architecture

### 7.1 Language Support Strategy

**Phase 1 Languages (Launch)**
- Hindi (hi): 43% of rural population
- Tamil (ta): 6% of rural population
- Telugu (te): 7% of rural population
- Marathi (mr): 7% of rural population
- Kannada (kn): 4% of rural population
- English (en): Administrative and urban users

**Phase 2 Languages (Months 6-12)**
- Bengali (bn)
- Gujarati (gu)
- Punjabi (pa)
- Malayalam (ml)
- Odia (or)

### 7.2 NLP Model Architecture

**Multilingual Transformer Models**

```
Base Model: XLM-RoBERTa (Cross-lingual Language Model)
- Pre-trained on 100 languages
- Fine-tuned on agricultural domain corpus
- Supports zero-shot transfer to new languages

Domain-Specific Fine-tuning:
- Agricultural vocabulary corpus (50,000+ terms)
- Crop names, market terminology, weather terms
- Regional variations and dialects
- Transliteration patterns (e.g., "chawal" → "चावल")
```

**Intent Classification**

```python
# Intent categories
INTENTS = {
    "get_price": ["price", "rate", "cost", "value", "कीमत", "விலை"],
    "get_weather": ["weather", "forecast", "rain", "मौसम", "வானிலை"],
    "find_buyers": ["buyer", "purchase", "sell", "खरीदार", "வாங்குபவர்"],
    "get_advisory": ["advisory", "tips", "guidance", "सलाह", "ஆலோசனை"],
    "get_schemes": ["scheme", "subsidy", "योजना", "திட்டம்"]
}

# Multi-label classification with confidence scores
def classify_intent(text, language):
    # Tokenize and encode
    inputs = tokenizer(text, return_tensors="pt", language=language)
    
    # Get model predictions
    outputs = model(**inputs)
    probabilities = softmax(outputs.logits)
    
    # Return top intents with confidence
    return {
        "primary_intent": get_top_intent(probabilities),
        "confidence": float(probabilities.max()),
        "all_intents": get_all_intents_with_scores(probabilities)
    }
```

**Entity Extraction**

```python
# Named Entity Recognition for agricultural domain
ENTITY_TYPES = {
    "CROP": ["rice", "wheat", "cotton", "धान", "गेहूं", "அரிசி"],
    "LOCATION": ["Chennai", "Mumbai", "चेन्नई", "சென்னை"],
    "DATE": ["today", "yesterday", "आज", "कल", "இன்று"],
    "QUANTITY": ["50 quintal", "100 kg", "50 क्विंटल"],
    "PRICE": ["2000 rupees", "Rs 2000", "2000 रुपये"]
}

def extract_entities(text, language):
    # Use NER model
    entities = ner_model(text, language=language)
    
    # Post-process and normalize
    normalized_entities = {
        "crop": normalize_crop_name(entities.get("CROP")),
        "location": normalize_location(entities.get("LOCATION")),
        "date": parse_date(entities.get("DATE")),
        "quantity": parse_quantity(entities.get("QUANTITY"))
    }
    
    return normalized_entities
```

### 7.3 Context Management

**Conversation State Tracking**

```python
class ConversationContext:
    def __init__(self, user_id, session_id):
        self.user_id = user_id
        self.session_id = session_id
        self.language = self.get_user_language()
        self.history = []
        self.entities = {}
        self.last_intent = None
        
    def update(self, user_input, bot_response):
        self.history.append({
            "turn": len(self.history) + 1,
            "user_input": user_input,
            "bot_response": bot_response,
            "timestamp": datetime.now()
        })
        
        # Extract and merge entities
        new_entities = extract_entities(user_input, self.language)
        self.entities.update(new_entities)
        
    def get_context_for_query(self):
        return {
            "user_profile": get_user_profile(self.user_id),
            "recent_history": self.history[-3:],  # Last 3 turns
            "accumulated_entities": self.entities,
            "language": self.language
        }
```

**Contextual Query Resolution**

```
User Turn 1: "मुझे धान की कीमत बताओ" (Tell me rice price)
Entities: {crop: "rice"}
Missing: location

System: Uses user profile location (Chennai)
Response: "Chennai में धान की कीमत ₹2,100 प्रति क्विंटल है"

User Turn 2: "कल की कीमत क्या थी?" (What was yesterday's price?)
Entities: {date: "yesterday"}
Context: {crop: "rice", location: "Chennai"} from Turn 1

System: Resolves full query using context
Response: "कल Chennai में धान की कीमत ₹2,050 प्रति क्विंटल थी"
```

### 7.4 Transliteration Support

**Input Transliteration**

Many farmers type in Roman script for local languages:

```python
# Transliteration examples
TRANSLITERATION_MAP = {
    "chawal": "चावल",  # Rice in Hindi
    "gehu": "गेहूं",    # Wheat in Hindi
    "arisi": "அரிசி",   # Rice in Tamil
    "godhi": "ಗೋಧಿ"     # Wheat in Kannada
}

def handle_transliteration(text, target_language):
    # Detect if input is in Roman script
    if is_roman_script(text):
        # Transliterate to target language script
        transliterated = transliterate(text, target_language)
        
        # Process both original and transliterated
        results_original = process_query(text, target_language)
        results_transliterated = process_query(transliterated, target_language)
        
        # Return best match
        return max(results_original, results_transliterated, 
                   key=lambda x: x.confidence)
    
    return process_query(text, target_language)
```

### 7.5 Cultural Adaptation

**Localized Responses**

```python
# Response templates by language and culture
RESPONSE_TEMPLATES = {
    "hi": {
        "greeting": "नमस्ते {name}जी",
        "price_response": "{crop} की कीमत {location} में ₹{price} प्रति {unit} है",
        "trend_up": "कीमत बढ़ रही है ↑",
        "trend_down": "कीमत घट रही है ↓"
    },
    "ta": {
        "greeting": "வணக்கம் {name}",
        "price_response": "{location} இல் {crop} விலை ₹{price} ஒரு {unit}",
        "trend_up": "விலை அதிகரித்து வருகிறது ↑",
        "trend_down": "விலை குறைந்து வருகிறது ↓"
    }
}

def generate_response(intent, entities, language):
    template = RESPONSE_TEMPLATES[language][intent]
    
    # Fill template with entities
    response = template.format(**entities)
    
    # Add cultural elements
    if language == "hi":
        response = add_respectful_suffix(response)  # Add "जी"
    
    return response
```

**Regional Crop Names**

```python
# Crop name variations by region
CROP_NAMES = {
    "rice": {
        "en": "Rice",
        "hi": "चावल / धान",
        "ta": "அரிசி / நெல்",
        "te": "బియ్యం / వరి",
        "mr": "तांदूळ / भात",
        "kn": "ಅಕ್ಕಿ / ಭತ್ತ"
    },
    "wheat": {
        "en": "Wheat",
        "hi": "गेहूं",
        "ta": "கோதுமை",
        "te": "గోధుమ",
        "mr": "गहू",
        "kn": "ಗೋಧಿ"
    }
}
```

### 7.6 Voice-Specific Optimizations

**Speech Recognition Tuning**

```python
# Custom vocabulary for agricultural terms
CUSTOM_VOCABULARY = [
    "quintal", "क्विंटल", "குவிண்டால்",
    "AGMARKNET", "e-NAM",
    "Koyambedu", "Azadpur", "Vashi"  # Market names
]

# Speech recognition configuration
speech_config = {
    "language": user_language,
    "custom_vocabulary": CUSTOM_VOCABULARY,
    "profanity_filter": False,  # Agricultural terms may be flagged
    "enable_automatic_punctuation": True,
    "speech_context": {
        "phrases": CUSTOM_VOCABULARY,
        "boost": 20  # Increase recognition confidence
    }
}
```

**Text-to-Speech Optimization**

```python
# TTS configuration for natural speech
tts_config = {
    "language": user_language,
    "voice_gender": "NEUTRAL",
    "speaking_rate": 0.9,  # Slightly slower for clarity
    "pitch": 0.0,
    "volume_gain_db": 0.0,
    "effects_profile": ["telephony-class-application"]
}

# Pronunciation corrections
PRONUNCIATION_FIXES = {
    "₹": "rupees",
    "Rs": "rupees",
    "qt": "quintal",
    "kg": "kilogram"
}
```

## 8. Security & Privacy

### 8.1 Data Protection

- Encrypt all data in transit (TLS 1.3)
- Encrypt sensitive data at rest (AES-256)
- Use environment variables for API keys
- Implement role-based access control (RBAC)

### 8.2 Privacy Measures

- Anonymize buyer ratings and transaction data
- Don't share farmer data with third parties
- Implement data retention policies (delete after 2 years)
- Provide data export and deletion options

### 8.3 Authentication

- Phone number verification via OTP
- Session tokens with 24-hour expiry
- Rate limiting (100 requests/hour per user)

## 9. Deployment Strategy

### 9.1 Infrastructure

- Multi-region deployment for redundancy
- Load balancing across multiple instances
- CDN for static content
- Database replication for disaster recovery

### 9.2 Monitoring & Logging

- Real-time monitoring with Prometheus/Grafana
- Centralized logging with ELK stack
- Error tracking with Sentry
- Performance monitoring with New Relic

### 9.3 Rollout Plan

- Phase 1: Pilot in 1 state (10,000 farmers)
- Phase 2: Expand to 3 states (100,000 farmers)
- Phase 3: National scale (1M+ farmers)

## 10. Success Metrics & KPIs

### 10.1 User Adoption

- Monthly active users (MAU)
- Daily active users (DAU)
- User retention rate (30-day, 90-day)
- Churn rate

### 10.2 Engagement

- Average session duration
- Queries per user per month
- Feature usage distribution
- Voice vs. text vs. web usage ratio

### 10.3 Impact

- Average income increase for users
- Price accuracy vs. official sources
- Buyer satisfaction rating
- Government scheme applications submitted

### 10.4 Technical

- System uptime percentage
- API response time (p95, p99)
- Error rate
- Data freshness (% of data < 24 hours old)

## 11. Risk Mitigation

| Risk                    | Impact | Mitigation                                                 |
| ----------------------- | ------ | ---------------------------------------------------------- |
| Government API downtime | High   | Cache data, fallback to historical data                    |
| Low farmer adoption     | High   | Community partnerships, field testing, incentives          |
| Data quality issues     | Medium | Verification process, user feedback, manual audits         |
| Language model errors   | Medium | Human review, continuous fine-tuning, fallback to menu     |
| Scalability issues      | Medium | Load testing, auto-scaling, database optimization          |
| Privacy concerns        | High   | Clear privacy policy, data minimization, compliance audits |

## 12. Future Enhancements (Phase 2+)

- Direct payment integration for buyer transactions
- Crop insurance marketplace
- Agricultural loan facilitation
- Video content for crop advisory
- Offline functionality for low-connectivity areas
- Integration with farm equipment suppliers
- Blockchain for buyer verification
- Predictive analytics for crop yield and pricing
