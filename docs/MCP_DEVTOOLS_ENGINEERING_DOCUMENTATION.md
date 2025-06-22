# **MCP DevTools: Engineering Documentation**

## **Project Overview**

MCP DevTools is a real-time inspection and modification platform for Model Context Protocol (MCP) tool calls. It transforms the existing MCP proxy into a powerful debugging and development tool that allows engineers to:

- Monitor all MCP tool calls in real-time
- Pause and modify tool parameters on-the-fly
- Mock responses for testing scenarios
- Analyze tool usage patterns
- Debug MCP integrations interactively

## **System Architecture**

### **High-Level Architecture**

```mermaid
graph TB
    subgraph "Client Layer"
        A[Claude Desktop] --> B[MCP DevTools Proxy]
        C[Custom MCP Clients] --> B
    end

    subgraph "MCP DevTools Core"
        B --> D[Message Interceptor]
        D --> E[Protocol Parser]
        E --> F[Modification Engine]
        F --> G[Rule Engine]
        G --> H[Response Handler]
        H --> I[MCP Server Pool]
    end

    subgraph "Real-Time Interface"
        E --> J[WebSocket Manager]
        J --> K[Web Dashboard]
        K --> L[Parameter Editor]
        L --> M[Response Mocker]
        M --> N[Modification Queue]
        N --> F
    end

    subgraph "Data & Analytics"
        E --> O[Message Store]
        H --> P[Response Store]
        O --> Q[Analytics Engine]
        P --> Q
        Q --> R[Pattern Detection]
    end

    subgraph "Backend Services"
        I --> S[MCP Server: fetch]
        I --> T[MCP Server: github]
        I --> U[MCP Server: custom]
    end
```

### **Component Responsibilities**

| Component | Responsibility | Key Interfaces |
|-----------|---------------|----------------|
| **Message Interceptor** | Capture all MCP JSON-RPC messages | `intercept_request()`, `intercept_response()` |
| **Protocol Parser** | Parse and structure MCP messages | `parse_message()`, `extract_tool_calls()` |
| **Modification Engine** | Handle pause/resume and modifications | `pause_message()`, `apply_modification()` |
| **WebSocket Manager** | Real-time communication with web UI | `broadcast_tool_call()`, `handle_client_message()` |
| **Web Dashboard** | User interface for inspection/modification | REST API + WebSocket endpoints |
| **Message Store** | Persistent storage of tool calls | `store_message()`, `query_history()` |
| **Analytics Engine** | Pattern detection and metrics | `analyze_patterns()`, `generate_metrics()` |

## **Technical Specifications**

### **Technology Stack**

| Layer | Technology | Version | Purpose |
|-------|------------|---------|---------|
| **Backend** | Python | 3.10+ | Core application runtime |
| **Web Framework** | FastAPI | 0.104+ | REST API and WebSocket server |
| **Async Runtime** | AsyncIO | Built-in | Concurrent message handling |
| **Message Queue** | Redis | 7.0+ | Real-time message broadcasting |
| **Database** | SQLite/PostgreSQL | Latest | Message and analytics storage |
| **Frontend** | Vue.js | 3.0+ | Real-time web dashboard |
| **WebSockets** | FastAPI WebSockets | Built-in | Real-time client communication |
| **Containerization** | Docker + Compose | Latest | Deployment and orchestration |

### **Performance Requirements**

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Message Latency** | < 10ms | Time from intercept to forward |
| **WebSocket Latency** | < 50ms | Time from intercept to UI display |
| **Concurrent Connections** | 100+ | Simultaneous WebSocket clients |
| **Message Throughput** | 1000/sec | MCP messages processed |
| **Memory Usage** | < 512MB | Base application footprint |
| **Storage Growth** | < 1GB/day | Message history retention |

### **Security Requirements**

| Requirement | Implementation | Priority |
|-------------|----------------|----------|
| **Authentication** | JWT-based session management | High |
| **Authorization** | Role-based access control | High |
| **Data Encryption** | TLS 1.3 for all communications | High |
| **Input Validation** | JSON schema validation | High |
| **Audit Logging** | All modifications logged | Medium |
| **Rate Limiting** | Per-client request throttling | Medium |

## **Detailed Component Design**

### **1. Message Interceptor**

**File:** `src/mcp_proxy/inspector/interceptor.py`

```python
class MCPMessageInterceptor:
    """
    Core component responsible for intercepting all MCP messages
    flowing through the proxy and triggering appropriate handlers.
    """

    def __init__(self, websocket_manager, modification_engine, storage):
        self.websocket_manager = websocket_manager
        self.modification_engine = modification_engine
        self.storage = storage
        self.active_sessions = {}
        self.message_correlation = {}

    async def intercept_request(self, message: dict, server_name: str, client_id: str) -> dict:
        """
        Intercept outgoing MCP request and apply any modifications.

        Args:
            message: Raw MCP JSON-RPC message
            server_name: Target MCP server identifier
            client_id: Originating client identifier

        Returns:
            Modified message (or original if no modifications)

        Raises:
            MessageBlockedException: If message is blocked by rules
        """

    async def intercept_response(self, response: dict, request_id: str, server_name: str) -> dict:
        """
        Intercept MCP response and apply any modifications.

        Args:
            response: Raw MCP JSON-RPC response
            request_id: Correlation ID from original request
            server_name: Source MCP server identifier

        Returns:
            Modified response (or original if no modifications)
        """
```

**Key Methods:**
- `_parse_message()`: Extract structured data from JSON-RPC
- `_correlate_request_response()`: Link requests with responses
- `_apply_modifications()`: Execute pending modifications
- `_broadcast_to_ui()`: Send updates to web interface

### **2. Modification Engine**

**File:** `src/mcp_proxy/inspector/modifier.py`

```python
class ModificationEngine:
    """
    Handles pausing, modifying, and resuming MCP messages.
    Supports both manual modifications from UI and automated rules.
    """

    def __init__(self):
        self.paused_messages = {}  # message_id -> asyncio.Event
        self.pending_modifications = {}  # message_id -> Modification
        self.modification_rules = []  # List[ModificationRule]
        self.response_mocks = {}  # message_id -> mock_response

    async def pause_message(self, message_id: str) -> None:
        """
        Pause a message for manual modification.
        Message will wait until resume_message() is called.
        """

    async def resume_message(self, message_id: str, modification: Optional[Modification] = None) -> None:
        """
        Resume a paused message with optional modifications.
        """

    def apply_modification(self, original_message: dict, modification: Modification) -> dict:
        """
        Apply a modification to a message.

        Supported modification types:
        - PARAMETER_CHANGE: Modify existing parameters
        - PARAMETER_ADD: Add new parameters
        - PARAMETER_REMOVE: Remove parameters
        - MOCK_RESPONSE: Return mock response instead
        - BLOCK_REQUEST: Block the request entirely
        - ROUTE_TO_SERVER: Route to different server
        """
```

**Modification Types:**
```python
class ModificationType(Enum):
    PARAMETER_CHANGE = "parameter_change"
    PARAMETER_ADD = "parameter_add"
    PARAMETER_REMOVE = "parameter_remove"
    MOCK_RESPONSE = "mock_response"
    BLOCK_REQUEST = "block_request"
    ROUTE_TO_SERVER = "route_to_server"
```

### **3. WebSocket Manager**

**File:** `src/mcp_proxy/inspector/websocket.py`

```python
class WebSocketManager:
    """
    Manages real-time communication between the backend and web UI.
    Handles client connections, message broadcasting, and UI interactions.
    """

    def __init__(self):
        self.active_connections = set()  # Set[WebSocket]
        self.client_subscriptions = {}  # WebSocket -> Set[str]
        self.message_queue = asyncio.Queue()

    async def connect(self, websocket: WebSocket, client_id: str = None) -> None:
        """Accept new WebSocket connection and initialize client."""

    async def disconnect(self, websocket: WebSocket) -> None:
        """Clean up disconnected WebSocket client."""

    async def broadcast_tool_call(self, intercepted_message: InterceptedMessage) -> None:
        """Broadcast tool call to all connected clients."""

    async def broadcast_tool_response(self, request_id: str, response: InterceptedMessage) -> None:
        """Broadcast tool response to all connected clients."""

    async def handle_client_message(self, websocket: WebSocket, message: dict) -> None:
        """Process incoming messages from web UI clients."""
```

**WebSocket Message Types:**
```python
# Client -> Server
{
    "type": "pause_message",
    "message_id": "uuid",
    "reason": "manual_inspection"
}

{
    "type": "modify_message",
    "message_id": "uuid",
    "modification_type": "parameter_change",
    "changes": {"param1": "new_value"}
}

{
    "type": "resume_message",
    "message_id": "uuid"
}

# Server -> Client
{
    "type": "tool_call",
    "data": {
        "id": "uuid",
        "timestamp": "2024-01-01T00:00:00Z",
        "method": "tools/call",
        "params": {...},
        "server_name": "fetch"
    }
}

{
    "type": "tool_response",
    "request_id": "uuid",
    "data": {...}
}
```

### **4. Web Dashboard**

**File:** `src/mcp_proxy/web/app.py`

```python
app = FastAPI(
    title="MCP DevTools",
    description="Real-time MCP inspection and modification platform",
    version="1.0.0"
)

# REST API Endpoints
@app.get("/api/messages/history")
async def get_message_history(
    limit: int = 100,
    offset: int = 0,
    server_filter: Optional[str] = None,
    method_filter: Optional[str] = None
) -> MessageHistoryResponse:
    """Retrieve paginated message history with optional filters."""

@app.post("/api/messages/{message_id}/pause")
async def pause_message(message_id: str) -> StatusResponse:
    """Pause a message for manual modification."""

@app.post("/api/messages/{message_id}/modify")
async def modify_message(
    message_id: str,
    modification: ModificationRequest
) -> ModificationResponse:
    """Apply modifications to a paused message."""

@app.get("/api/analytics/summary")
async def get_analytics_summary() -> AnalyticsSummary:
    """Get high-level analytics and metrics."""

@app.get("/api/servers")
async def list_servers() -> ServerListResponse:
    """List all configured MCP servers."""

# WebSocket Endpoint
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    """Real-time communication endpoint for web UI."""
```

### **5. Data Models**

**File:** `src/mcp_proxy/inspector/models.py`

```python
@dataclass
class InterceptedMessage:
    """Represents a captured MCP message with metadata."""
    id: str
    timestamp: datetime
    message_type: MessageType
    method: Optional[str]
    params: Dict[str, Any]
    raw_message: Dict[str, Any]
    server_name: Optional[str] = None
    client_id: Optional[str] = None
    correlation_id: Optional[str] = None
    status: MessageStatus = MessageStatus.PROCESSED

@dataclass
class Modification:
    """Represents a modification to be applied to a message."""
    id: str
    message_id: str
    modification_type: ModificationType
    changes: Dict[str, Any]
    created_at: datetime
    applied_at: Optional[datetime] = None
    created_by: Optional[str] = None

@dataclass
class ModificationRule:
    """Automated modification rule."""
    id: str
    name: str
    description: str
    condition: str  # JSON path expression
    modification: Modification
    enabled: bool = True
    priority: int = 0

class MessageStatus(Enum):
    PENDING = "pending"
    PAUSED = "paused"
    MODIFIED = "modified"
    PROCESSED = "processed"
    BLOCKED = "blocked"
    ERROR = "error"
```

## **Database Schema**

### **Messages Table**
```sql
CREATE TABLE messages (
    id VARCHAR(36) PRIMARY KEY,
    timestamp TIMESTAMP NOT NULL,
    message_type VARCHAR(20) NOT NULL,
    method VARCHAR(50),
    server_name VARCHAR(50),
    client_id VARCHAR(50),
    correlation_id VARCHAR(36),
    status VARCHAR(20) DEFAULT 'processed',
    raw_message JSON NOT NULL,
    params JSON,
    response JSON,
    processing_time_ms INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_timestamp (timestamp),
    INDEX idx_server_method (server_name, method),
    INDEX idx_correlation (correlation_id)
);
```

### **Modifications Table**
```sql
CREATE TABLE modifications (
    id VARCHAR(36) PRIMARY KEY,
    message_id VARCHAR(36) NOT NULL,
    modification_type VARCHAR(30) NOT NULL,
    changes JSON NOT NULL,
    created_at TIMESTAMP NOT NULL,
    applied_at TIMESTAMP,
    created_by VARCHAR(50),
    FOREIGN KEY (message_id) REFERENCES messages(id),
    INDEX idx_message_id (message_id),
    INDEX idx_created_at (created_at)
);
```

### **Rules Table**
```sql
CREATE TABLE modification_rules (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    condition_json JSON NOT NULL,
    modification_json JSON NOT NULL,
    enabled BOOLEAN DEFAULT true,
    priority INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_enabled_priority (enabled, priority)
);
```

## **API Documentation**

### **REST API Endpoints**

#### **Message Management**

**GET /api/messages/history**
```json
{
  "messages": [
    {
      "id": "uuid",
      "timestamp": "2024-01-01T00:00:00Z",
      "message_type": "tool_call",
      "method": "tools/call",
      "server_name": "fetch",
      "params": {...},
      "status": "processed",
      "processing_time_ms": 45
    }
  ],
  "total": 1500,
  "page": 1,
  "per_page": 100
}
```

**POST /api/messages/{message_id}/pause**
```json
{
  "status": "success",
  "message": "Message paused successfully",
  "message_id": "uuid",
  "paused_at": "2024-01-01T00:00:00Z"
}
```

**POST /api/messages/{message_id}/modify**
```json
// Request
{
  "modification_type": "parameter_change",
  "changes": {
    "arguments.url": "https://new-url.com",
    "arguments.timeout": 30
  }
}

// Response
{
  "status": "success",
  "modification_id": "uuid",
  "applied_at": "2024-01-01T00:00:00Z"
}
```

#### **Analytics**

**GET /api/analytics/summary**
```json
{
  "total_messages": 15000,
  "messages_today": 450,
  "most_used_tools": [
    {"name": "fetch", "count": 5000, "percentage": 33.3},
    {"name": "github_search", "count": 3000, "percentage": 20.0}
  ],
  "error_rate": 2.5,
  "average_response_time_ms": 125,
  "servers": [
    {
      "name": "fetch",
      "status": "healthy",
      "uptime_percentage": 99.9,
      "last_seen": "2024-01-01T00:00:00Z"
    }
  ]
}
```

#### **Server Management**

**GET /api/servers**
```json
{
  "servers": [
    {
      "name": "fetch",
      "type": "stdio",
      "command": "uvx mcp-server-fetch",
      "status": "running",
      "uptime": "2d 5h 30m",
      "message_count": 5000,
      "error_count": 12,
      "last_activity": "2024-01-01T00:00:00Z"
    }
  ]
}
```

### **WebSocket Events**

#### **Client -> Server Events**

```javascript
// Subscribe to specific message types
{
  "type": "subscribe",
  "filters": {
    "servers": ["fetch", "github"],
    "methods": ["tools/call"],
    "message_types": ["tool_call", "tool_response"]
  }
}

// Pause a message
{
  "type": "pause_message",
  "message_id": "uuid",
  "reason": "manual_inspection"
}

// Modify a message
{
  "type": "modify_message",
  "message_id": "uuid",
  "modification": {
    "type": "parameter_change",
    "changes": {"param1": "new_value"}
  }
}

// Resume a message
{
  "type": "resume_message",
  "message_id": "uuid"
}
```

#### **Server -> Client Events**

```javascript
// New tool call
{
  "type": "tool_call",
  "data": {
    "id": "uuid",
    "timestamp": "2024-01-01T00:00:00Z",
    "method": "tools/call",
    "params": {...},
    "server_name": "fetch",
    "client_id": "claude_desktop"
  }
}

// Tool response
{
  "type": "tool_response",
  "request_id": "uuid",
  "data": {
    "result": {...},
    "processing_time_ms": 150
  }
}

// Message paused
{
  "type": "message_paused",
  "message_id": "uuid",
  "requires_action": true
}

// Modification applied
{
  "type": "modification_applied",
  "original_id": "uuid",
  "modification_id": "uuid",
  "changes": {...}
}
```

## **Frontend Architecture**

### **Vue.js Component Structure**

```
src/web/static/
├── components/
│   ├── MessageList.vue          # Live message feed
│   ├── MessageDetail.vue        # Individual message inspector
│   ├── ModificationPanel.vue    # Parameter editor
│   ├── ServerStatus.vue         # Server health indicators
│   ├── Analytics.vue            # Usage analytics dashboard
│   └── RuleEditor.vue           # Modification rule management
├── stores/
│   ├── messages.js              # Message state management
│   ├── websocket.js             # WebSocket connection
│   └── modifications.js         # Modification state
├── utils/
│   ├── api.js                   # REST API client
│   ├── websocket.js             # WebSocket client
│   └── formatters.js            # Data formatting utilities
└── App.vue                      # Main application component
```

### **Key Frontend Features**

1. **Real-Time Message Feed**
   - Auto-scrolling list of tool calls
   - Syntax-highlighted JSON display
   - Filtering and search capabilities
   - Status indicators (paused, modified, error)

2. **Interactive Parameter Editor**
   - JSON editor with validation
   - Diff view showing changes
   - Template system for common modifications
   - Undo/redo functionality

3. **Response Mocking Interface**
   - Mock response editor
   - Response template library
   - A/B testing capabilities
   - Performance simulation

4. **Analytics Dashboard**
   - Tool usage charts
   - Performance metrics
   - Error rate trends
   - Server health monitoring

## **Deployment Guide**

### **Docker Compose Configuration**

```yaml
# docker-compose.yml
version: '3.8'

services:
  mcp-devtools:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"  # MCP proxy port
      - "3000:3000"  # Web UI port
    environment:
      - REDIS_URL=redis://redis:6379
      - DATABASE_URL=postgresql://user:pass@postgres:5432/mcpdevtools
      - LOG_LEVEL=INFO
      - ENABLE_INSPECTOR=true
    depends_on:
      - redis
      - postgres
    volumes:
      - ./config:/app/config
      - ./logs:/app/logs

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=mcpdevtools
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - mcp-devtools

volumes:
  redis_data:
  postgres_data:
```

### **Environment Configuration**

```bash
# .env
# Application
LOG_LEVEL=INFO
DEBUG=false
ENABLE_INSPECTOR=true
WEB_UI_ENABLED=true

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/mcpdevtools
REDIS_URL=redis://localhost:6379

# Security
JWT_SECRET_KEY=your-secret-key
CORS_ORIGINS=http://localhost:3000,https://your-domain.com

# Performance
MAX_CONCURRENT_CONNECTIONS=100
MESSAGE_RETENTION_DAYS=30
WEBSOCKET_PING_INTERVAL=30

# MCP Servers
MCP_SERVER_CONFIG_PATH=/app/config/servers.json
```

### **Production Deployment Checklist**

- [ ] **Security**
  - [ ] JWT authentication configured
  - [ ] HTTPS/TLS certificates installed
  - [ ] CORS origins restricted
  - [ ] Input validation enabled
  - [ ] Rate limiting configured

- [ ] **Performance**
  - [ ] Redis cluster for high availability
  - [ ] Database connection pooling
  - [ ] Message retention policies
  - [ ] Log rotation configured

- [ ] **Monitoring**
  - [ ] Health check endpoints
  - [ ] Application metrics collection
  - [ ] Error tracking (Sentry/similar)
  - [ ] Log aggregation (ELK/similar)

- [ ] **Backup & Recovery**
  - [ ] Database backup strategy
  - [ ] Configuration backup
  - [ ] Disaster recovery plan

## **Development Workflow**

### **Local Development Setup**

```bash
# 1. Clone repository
git clone <repository-url>
cd mcp-devtools

# 2. Set up Python environment
python -m venv venv
source venv/bin/activate  # or `venv\Scripts\activate` on Windows
pip install -r requirements-dev.txt

# 3. Set up database
createdb mcpdevtools
python -m alembic upgrade head

# 4. Start Redis
docker run -d -p 6379:6379 redis:alpine

# 5. Start development server
python -m uvicorn src.mcp_proxy.web.app:app --reload --port 3000

# 6. Start frontend development server
cd src/web/static
npm install
npm run dev
```

### **Testing Strategy**

```bash
# Unit tests
pytest tests/unit/ -v

# Integration tests
pytest tests/integration/ -v

# End-to-end tests
pytest tests/e2e/ -v

# Performance tests
pytest tests/performance/ -v

# Coverage report
pytest --cov=src/mcp_proxy --cov-report=html
```

### **Code Quality Tools**

```bash
# Linting
ruff check src/
black src/
mypy src/

# Security scanning
bandit -r src/

# Dependency checking
safety check
```

## **Monitoring & Observability**

### **Application Metrics**

```python
# Key metrics to track
- mcp_messages_total{server, method, status}
- mcp_message_duration_seconds{server, method}
- mcp_active_connections{client_type}
- mcp_modifications_total{type, source}
- mcp_websocket_connections_active
- mcp_errors_total{error_type, server}
```

### **Health Checks**

```python
# Health check endpoints
GET /health              # Basic health check
GET /health/detailed     # Detailed component status
GET /health/ready        # Readiness probe
GET /health/live         # Liveness probe
```

### **Logging Configuration**

```python
# Structured logging format
{
  "timestamp": "2024-01-01T00:00:00Z",
  "level": "INFO",
  "logger": "mcp_proxy.inspector",
  "message": "Tool call intercepted",
  "context": {
    "message_id": "uuid",
    "server_name": "fetch",
    "method": "tools/call",
    "client_id": "claude_desktop",
    "processing_time_ms": 45
  }
}
```

## **Security Considerations**

### **Authentication & Authorization**

1. **JWT-based Authentication**
   - Token-based session management
   - Configurable token expiration
   - Refresh token support

2. **Role-Based Access Control**
   - Admin: Full access to all features
   - Developer: Read/write access to modifications
   - Viewer: Read-only access to messages

3. **API Security**
   - Rate limiting per client
   - Input validation and sanitization
   - SQL injection prevention
   - XSS protection

### **Data Protection**

1. **Sensitive Data Handling**
   - Configurable parameter masking
   - PII detection and redaction
   - Audit logging of all modifications

2. **Network Security**
   - TLS 1.3 for all communications
   - WebSocket security headers
   - CORS policy enforcement

## **Performance Optimization**

### **Backend Optimizations**

1. **Message Processing**
   - Async/await throughout
   - Connection pooling
   - Message batching for high throughput

2. **Database Optimization**
   - Proper indexing strategy
   - Query optimization
   - Connection pooling

3. **Caching Strategy**
   - Redis for real-time data
   - In-memory caching for frequently accessed data
   - CDN for static assets

### **Frontend Optimizations**

1. **Real-Time Updates**
   - Virtual scrolling for large message lists
   - Debounced search and filtering
   - Lazy loading of message details

2. **Bundle Optimization**
   - Code splitting
   - Tree shaking
   - Asset compression

## **Troubleshooting Guide**

### **Common Issues**

1. **WebSocket Connection Failures**
   - Check CORS configuration
   - Verify network connectivity
   - Review proxy settings

2. **Message Processing Delays**
   - Monitor Redis performance
   - Check database query performance
   - Review async task queue

3. **High Memory Usage**
   - Implement message retention policies
   - Monitor connection leaks
   - Review caching strategies

### **Debug Tools**

1. **Logging**
   - Enable debug logging: `LOG_LEVEL=DEBUG`
   - Message tracing: `ENABLE_MESSAGE_TRACING=true`
   - Performance profiling: `ENABLE_PROFILING=true`

2. **Monitoring**
   - Health check endpoints
   - Metrics dashboard
   - Error tracking integration

This comprehensive engineering documentation provides your team with all the technical details needed to implement the MCP DevTools platform successfully.