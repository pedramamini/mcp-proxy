# **MCP DevTools: Implementation Guide**

## **Project Structure**

```
mcp-devtools/
├── src/
│   └── mcp_proxy/
│       ├── __init__.py
│       ├── __main__.py                 # Enhanced entry point
│       ├── config_loader.py            # Existing config loader
│       ├── mcp_server.py               # Enhanced MCP server
│       ├── proxy_server.py             # Existing proxy server
│       ├── sse_client.py               # Existing SSE client
│       ├── streamablehttp_client.py    # Existing HTTP client
│       ├── inspector/                  # NEW: Inspector components
│       │   ├── __init__.py
│       │   ├── interceptor.py          # Message interception
│       │   ├── modifier.py             # Modification engine
│       │   ├── websocket.py            # WebSocket manager
│       │   ├── storage.py              # Data persistence
│       │   ├── analytics.py            # Usage analytics
│       │   └── models.py               # Data models
│       └── web/                        # NEW: Web interface
│           ├── __init__.py
│           ├── app.py                  # FastAPI application
│           ├── auth.py                 # Authentication
│           ├── api/                    # REST API endpoints
│           │   ├── __init__.py
│           │   ├── messages.py
│           │   ├── modifications.py
│           │   ├── analytics.py
│           │   └── servers.py
│           ├── static/                 # Frontend assets
│           │   ├── index.html
│           │   ├── css/
│           │   ├── js/
│           │   └── components/
│           └── templates/              # Jinja2 templates
│               └── dashboard.html
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── config/
│   ├── servers.json                    # MCP server configurations
│   ├── rules.json                      # Modification rules
│   └── settings.yaml                   # Application settings
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── docker-compose.dev.yml
│   └── nginx.conf
├── docs/
│   ├── api.md
│   ├── deployment.md
│   └── user-guide.md
├── scripts/
│   ├── setup.sh
│   ├── migrate.py
│   └── seed-data.py
├── requirements.txt
├── requirements-dev.txt
├── pyproject.toml
├── alembic.ini                         # Database migrations
├── alembic/
│   └── versions/
└── README.md
```

## **Implementation Phases**

### **Phase 1: Core Infrastructure (Week 1-2)**

#### **1.1 Enhanced Dependencies**

Update `pyproject.toml`:

```toml
[project]
name = "mcp-proxy"
version = "1.0.0"
dependencies = [
    "mcp>=1.8.0,<2.0.0",
    "uvicorn>=0.34.0",
    "fastapi>=0.104.0",
    "websockets>=12.0",
    "redis>=5.0.0",
    "sqlalchemy>=2.0.0",
    "alembic>=1.13.0",
    "pydantic>=2.5.0",
    "structlog>=23.0.0",
    "jinja2>=3.1.0",
    "python-multipart>=0.0.6",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "psutil>=5.9.0",
]

[tool.uv]
dev-dependencies = [
    "pytest>=8.3.3",
    "pytest-asyncio>=0.25.0",
    "pytest-mock>=3.12.0",
    "httpx>=0.25.0",
    "coverage>=7.6.0",
    "mypy>=1.0.0",
    "ruff>=0.1.0",
    "black>=23.0.0",
    "bandit>=1.7.0",
    "safety>=2.3.0",
]
```

#### **1.2 Database Schema Setup**

Create `alembic/versions/001_initial_schema.py`:

```python
"""Initial schema

Revision ID: 001
Revises:
Create Date: 2024-01-01 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# revision identifiers
revision = '001'
down_revision = None
branch_labels = None
depends_on = None

def upgrade() -> None:
    # Messages table
    op.create_table('messages',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('timestamp', sa.DateTime(timezone=True), nullable=False),
        sa.Column('message_type', sa.String(20), nullable=False),
        sa.Column('method', sa.String(50), nullable=True),
        sa.Column('server_name', sa.String(50), nullable=True),
        sa.Column('client_id', sa.String(50), nullable=True),
        sa.Column('correlation_id', sa.String(36), nullable=True),
        sa.Column('status', sa.String(20), nullable=False, default='processed'),
        sa.Column('raw_message', sa.JSON(), nullable=False),
        sa.Column('params', sa.JSON(), nullable=True),
        sa.Column('response', sa.JSON(), nullable=True),
        sa.Column('processing_time_ms', sa.Integer(), nullable=True),
        sa.Column('created_at', sa.DateTime(timezone=True), server_default=sa.text('now()')),
        sa.PrimaryKeyConstraint('id')
    )

    # Create indexes
    op.create_index('idx_messages_timestamp', 'messages', ['timestamp'])
    op.create_index('idx_messages_server_method', 'messages', ['server_name', 'method'])
    op.create_index('idx_messages_correlation', 'messages', ['correlation_id'])

    # Modifications table
    op.create_table('modifications',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('message_id', sa.String(36), nullable=False),
        sa.Column('modification_type', sa.String(30), nullable=False),
        sa.Column('changes', sa.JSON(), nullable=False),
        sa.Column('created_at', sa.DateTime(timezone=True), nullable=False),
        sa.Column('applied_at', sa.DateTime(timezone=True), nullable=True),
        sa.Column('created_by', sa.String(50), nullable=True),
        sa.ForeignKeyConstraint(['message_id'], ['messages.id'], ),
        sa.PrimaryKeyConstraint('id')
    )

    op.create_index('idx_modifications_message_id', 'modifications', ['message_id'])
    op.create_index('idx_modifications_created_at', 'modifications', ['created_at'])

    # Rules table
    op.create_table('modification_rules',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('name', sa.String(100), nullable=False),
        sa.Column('description', sa.Text(), nullable=True),
        sa.Column('condition_json', sa.JSON(), nullable=False),
        sa.Column('modification_json', sa.JSON(), nullable=False),
        sa.Column('enabled', sa.Boolean(), nullable=False, default=True),
        sa.Column('priority', sa.Integer(), nullable=False, default=0),
        sa.Column('created_at', sa.DateTime(timezone=True), server_default=sa.text('now()')),
        sa.Column('updated_at', sa.DateTime(timezone=True), server_default=sa.text('now()')),
        sa.PrimaryKeyConstraint('id')
    )

    op.create_index('idx_rules_enabled_priority', 'modification_rules', ['enabled', 'priority'])

def downgrade() -> None:
    op.drop_table('modification_rules')
    op.drop_table('modifications')
    op.drop_table('messages')
```

#### **1.3 Core Data Models**

Create `src/mcp_proxy/inspector/models.py`:

```python
from datetime import datetime
from enum import Enum
from typing import Any, Dict, Optional
from dataclasses import dataclass, asdict
from sqlalchemy import Column, String, DateTime, JSON, Integer, Boolean, Text, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship

Base = declarative_base()

class MessageType(Enum):
    TOOL_CALL = "tool_call"
    TOOL_RESPONSE = "tool_response"
    RESOURCE_REQUEST = "resource_request"
    RESOURCE_RESPONSE = "resource_response"

class MessageStatus(Enum):
    PENDING = "pending"
    PAUSED = "paused"
    MODIFIED = "modified"
    PROCESSED = "processed"
    BLOCKED = "blocked"
    ERROR = "error"

class ModificationType(Enum):
    PARAMETER_CHANGE = "parameter_change"
    PARAMETER_ADD = "parameter_add"
    PARAMETER_REMOVE = "parameter_remove"
    MOCK_RESPONSE = "mock_response"
    BLOCK_REQUEST = "block_request"
    ROUTE_TO_SERVER = "route_to_server"

# SQLAlchemy Models
class MessageModel(Base):
    __tablename__ = 'messages'

    id = Column(String(36), primary_key=True)
    timestamp = Column(DateTime(timezone=True), nullable=False)
    message_type = Column(String(20), nullable=False)
    method = Column(String(50))
    server_name = Column(String(50))
    client_id = Column(String(50))
    correlation_id = Column(String(36))
    status = Column(String(20), nullable=False, default='processed')
    raw_message = Column(JSON, nullable=False)
    params = Column(JSON)
    response = Column(JSON)
    processing_time_ms = Column(Integer)
    created_at = Column(DateTime(timezone=True))

    modifications = relationship("ModificationModel", back_populates="message")

class ModificationModel(Base):
    __tablename__ = 'modifications'

    id = Column(String(36), primary_key=True)
    message_id = Column(String(36), ForeignKey('messages.id'), nullable=False)
    modification_type = Column(String(30), nullable=False)
    changes = Column(JSON, nullable=False)
    created_at = Column(DateTime(timezone=True), nullable=False)
    applied_at = Column(DateTime(timezone=True))
    created_by = Column(String(50))

    message = relationship("MessageModel", back_populates="modifications")

class ModificationRuleModel(Base):
    __tablename__ = 'modification_rules'

    id = Column(String(36), primary_key=True)
    name = Column(String(100), nullable=False)
    description = Column(Text)
    condition_json = Column(JSON, nullable=False)
    modification_json = Column(JSON, nullable=False)
    enabled = Column(Boolean, nullable=False, default=True)
    priority = Column(Integer, nullable=False, default=0)
    created_at = Column(DateTime(timezone=True))
    updated_at = Column(DateTime(timezone=True))

# Pydantic Models for API
@dataclass
class InterceptedMessage:
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
    processing_time_ms: Optional[int] = None

    def to_dict(self) -> Dict[str, Any]:
        return {
            **asdict(self),
            'timestamp': self.timestamp.isoformat(),
            'message_type': self.message_type.value,
            'status': self.status.value
        }

@dataclass
class Modification:
    id: str
    message_id: str
    modification_type: ModificationType
    changes: Dict[str, Any]
    created_at: datetime
    applied_at: Optional[datetime] = None
    created_by: Optional[str] = None

@dataclass
class ModificationRule:
    id: str
    name: str
    description: str
    condition: Dict[str, Any]
    modification: Dict[str, Any]
    enabled: bool = True
    priority: int = 0
    created_at: Optional[datetime] = None
    updated_at: Optional[datetime] = None
```

### **Phase 2: Message Interception (Week 2-3)**

#### **2.1 Message Interceptor Implementation**

Create `src/mcp_proxy/inspector/interceptor.py`:

```python
import asyncio
import json
import uuid
import logging
from datetime import datetime, timezone
from typing import Any, Dict, Optional, Callable
from .models import InterceptedMessage, MessageType, MessageStatus
from .storage import MessageStorage
from .websocket import WebSocketManager
from .modifier import ModificationEngine

logger = logging.getLogger(__name__)

class MCPMessageInterceptor:
    """Core message interception and processing engine."""

    def __init__(
        self,
        websocket_manager: WebSocketManager,
        modification_engine: ModificationEngine,
        storage: MessageStorage
    ):
        self.websocket_manager = websocket_manager
        self.modification_engine = modification_engine
        self.storage = storage
        self.active_sessions: Dict[str, Dict] = {}
        self.message_correlation: Dict[str, str] = {}

    async def intercept_request(
        self,
        message: Dict[str, Any],
        server_name: str,
        client_id: str
    ) -> Dict[str, Any]:
        """
        Intercept and potentially modify outgoing MCP requests.

        This is the main entry point for all outgoing MCP messages.
        """
        start_time = datetime.now(timezone.utc)

        try:
            # Parse the MCP message
            intercepted = self._parse_message(message, server_name, client_id)

            # Store original message
            await self.storage.store_message(intercepted)

            # Broadcast to web UI immediately
            await self.websocket_manager.broadcast_tool_call(intercepted)

            # Apply any modifications (this may pause execution)
            modified_message = await self._apply_modifications(intercepted, message)

            # Log processing time
            processing_time = (datetime.now(timezone.utc) - start_time).total_seconds() * 1000
            await self.storage.update_processing_time(intercepted.id, int(processing_time))

            # If message was modified, log the modification
            if modified_message != message:
                modified_intercepted = self._parse_message(modified_message, server_name, client_id)
                modified_intercepted.id = f"{intercepted.id}_modified"
                modified_intercepted.status = MessageStatus.MODIFIED

                await self.storage.store_message(modified_intercepted)
                await self.websocket_manager.broadcast_modification(
                    intercepted.id, modified_intercepted
                )

                logger.info(
                    "Message modified",
                    extra={
                        "original_id": intercepted.id,
                        "modified_id": modified_intercepted.id,
                        "server_name": server_name,
                        "method": intercepted.method
                    }
                )

            return modified_message

        except Exception as e:
            logger.error(
                "Error intercepting request",
                extra={
                    "error": str(e),
                    "server_name": server_name,
                    "client_id": client_id,
                    "message_preview": str(message)[:200]
                }
            )
            # Return original message on error to avoid breaking the flow
            return message

    async def intercept_response(
        self,
        response: Dict[str, Any],
        original_request_id: str,
        server_name: str
    ) -> Dict[str, Any]:
        """
        Intercept and potentially modify MCP responses.
        """
        try:
            # Create response record
            response_record = InterceptedMessage(
                id=f"{original_request_id}_response",
                timestamp=datetime.now(timezone.utc),
                message_type=MessageType.TOOL_RESPONSE,
                method=None,
                params=response,
                raw_message=response,
                server_name=server_name,
                correlation_id=original_request_id,
                status=MessageStatus.PROCESSED
            )

            # Store response
            await self.storage.store_message(response_record)

            # Check for response modifications
            modified_response = await self._apply_response_modifications(
                response_record, response
            )

            # Broadcast to web UI
            await self.websocket_manager.broadcast_tool_response(
                original_request_id, response_record
            )

            return modified_response

        except Exception as e:
            logger.error(
                "Error intercepting response",
                extra={
                    "error": str(e),
                    "request_id": original_request_id,
                    "server_name": server_name
                }
            )
            return response

    def _parse_message(
        self,
        message: Dict[str, Any],
        server_name: str,
        client_id: str
    ) -> InterceptedMessage:
        """Parse MCP JSON-RPC message into structured format."""

        message_id = str(uuid.uuid4())
        method = message.get('method', '')
        params = message.get('params', {})

        # Determine message type based on method
        if method == 'tools/call':
            msg_type = MessageType.TOOL_CALL
        elif method == 'resources/read':
            msg_type = MessageType.RESOURCE_REQUEST
        elif method == 'resources/list':
            msg_type = MessageType.RESOURCE_REQUEST
        else:
            msg_type = MessageType.TOOL_CALL  # Default fallback

        return InterceptedMessage(
            id=message_id,
            timestamp=datetime.now(timezone.utc),
            message_type=msg_type,
            method=method,
            params=params,
            raw_message=message,
            server_name=server_name,
            client_id=client_id,
            status=MessageStatus.PENDING
        )

    async def _apply_modifications(
        self,
        intercepted: InterceptedMessage,
        original_message: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Apply any pending modifications to the message."""

        # Check for manual modifications (this may pause execution)
        modification = await self.modification_engine.get_pending_modification(
            intercepted.id
        )

        if modification:
            logger.info(
                "Applying manual modification",
                extra={
                    "message_id": intercepted.id,
                    "modification_type": modification.modification_type.value
                }
            )
            modified_message = self.modification_engine.apply_modification(
                original_message, modification
            )
            return modified_message

        # Check for rule-based modifications
        rule_modification = await self.modification_engine.apply_rules(
            intercepted, original_message
        )

        if rule_modification != original_message:
            logger.info(
                "Applied rule-based modification",
                extra={
                    "message_id": intercepted.id,
                    "server_name": intercepted.server_name,
                    "method": intercepted.method
                }
            )

        return rule_modification

    async def _apply_response_modifications(
        self,
        response_record: InterceptedMessage,
        original_response: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Apply modifications to responses."""

        modification = await self.modification_engine.get_response_modification(
            response_record.correlation_id
        )

        if modification:
            logger.info(
                "Applying response modification",
                extra={
                    "response_id": response_record.id,
                    "correlation_id": response_record.correlation_id
                }
            )
            return self.modification_engine.apply_response_modification(
                original_response, modification
            )

        return original_response
```

#### **2.2 Storage Layer Implementation**

Create `src/mcp_proxy/inspector/storage.py`:

```python
import asyncio
import logging
from datetime import datetime, timezone
from typing import List, Optional, Dict, Any
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy import select, update, desc, and_, or_
from .models import MessageModel, ModificationModel, ModificationRuleModel, InterceptedMessage

logger = logging.getLogger(__name__)

class MessageStorage:
    """Handles persistent storage of intercepted messages and modifications."""

    def __init__(self, database_url: str):
        self.engine = create_async_engine(database_url, echo=False)
        self.async_session = sessionmaker(
            self.engine, class_=AsyncSession, expire_on_commit=False
        )

    async def store_message(self, message: InterceptedMessage) -> None:
        """Store an intercepted message in the database."""
        try:
            async with self.async_session() as session:
                db_message = MessageModel(
                    id=message.id,
                    timestamp=message.timestamp,
                    message_type=message.message_type.value,
                    method=message.method,
                    server_name=message.server_name,
                    client_id=message.client_id,
                    correlation_id=message.correlation_id,
                    status=message.status.value,
                    raw_message=message.raw_message,
                    params=message.params,
                    processing_time_ms=message.processing_time_ms,
                    created_at=datetime.now(timezone.utc)
                )

                session.add(db_message)
                await session.commit()

        except Exception as e:
            logger.error(f"Error storing message: {e}", extra={"message_id": message.id})

    async def update_processing_time(self, message_id: str, processing_time_ms: int) -> None:
        """Update the processing time for a message."""
        try:
            async with self.async_session() as session:
                await session.execute(
                    update(MessageModel)
                    .where(MessageModel.id == message_id)
                    .values(processing_time_ms=processing_time_ms)
                )
                await session.commit()
        except Exception as e:
            logger.error(f"Error updating processing time: {e}")

    async def get_message_history(
        self,
        limit: int = 100,
        offset: int = 0,
        server_filter: Optional[str] = None,
        method_filter: Optional[str] = None,
        start_time: Optional[datetime] = None,
        end_time: Optional[datetime] = None
    ) -> List[Dict[str, Any]]:
        """Retrieve paginated message history with optional filters."""
        try:
            async with self.async_session() as session:
                query = select(MessageModel).order_by(desc(MessageModel.timestamp))

                # Apply filters
                conditions = []
                if server_filter:
                    conditions.append(MessageModel.server_name == server_filter)
                if method_filter:
                    conditions.append(MessageModel.method == method_filter)
                if start_time:
                    conditions.append(MessageModel.timestamp >= start_time)
                if end_time:
                    conditions.append(MessageModel.timestamp <= end_time)

                if conditions:
                    query = query.where(and_(*conditions))

                query = query.offset(offset).limit(limit)

                result = await session.execute(query)
                messages = result.scalars().all()

                return [
                    {
                        "id": msg.id,
                        "timestamp": msg.timestamp.isoformat(),
                        "message_type": msg.message_type,
                        "method": msg.method,
                        "server_name": msg.server_name,
                        "client_id": msg.client_id,
                        "status": msg.status,
                        "params": msg.params,
                        "processing_time_ms": msg.processing_time_ms
                    }
                    for msg in messages
                ]

        except Exception as e:
            logger.error(f"Error retrieving message history: {e}")
            return []

    async def get_analytics_summary(self) -> Dict[str, Any]:
        """Get high-level analytics and metrics."""
        try:
            async with self.async_session() as session:
                # Total message count
                total_result = await session.execute(
                    select(MessageModel.id).count()
                )
                total_messages = total_result.scalar()

                # Messages today
                today_start = datetime.now(timezone.utc).replace(
                    hour=0, minute=0, second=0, microsecond=0
                )
                today_result = await session.execute(
                    select(MessageModel.id)
                    .where(MessageModel.timestamp >= today_start)
                    .count()
                )
                messages_today = today_result.scalar()

                # Most used tools (simplified query)
                # In production, you'd want more sophisticated analytics

                return {
                    "total_messages": total_messages,
                    "messages_today": messages_today,
                    "most_used_tools": [],  # TODO: Implement
                    "error_rate": 0.0,  # TODO: Calculate
                    "average_response_time_ms": 0,  # TODO: Calculate
                    "servers": []  # TODO: Get server status
                }

        except Exception as e:
            logger.error(f"Error getting analytics summary: {e}")
            return {
                "total_messages": 0,
                "messages_today": 0,
                "most_used_tools": [],
                "error_rate": 0.0,
                "average_response_time_ms": 0,
                "servers": []
            }
```

### **Phase 3: Web Interface (Week 3-4)**

#### **3.1 FastAPI Application Setup**

Create `src/mcp_proxy/web/app.py`:

```python
import logging
from contextlib import asynccontextmanager
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Request, Depends, HTTPException
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from fastapi.responses import HTMLResponse
from fastapi.middleware.cors import CORSMiddleware
import json
from typing import Dict, Any, Optional, List
from datetime import datetime

from ..inspector.websocket import WebSocketManager
from ..inspector.modifier import ModificationEngine
from ..inspector.storage import MessageStorage
from .api.messages import router as messages_router
from .api.modifications import router as modifications_router
from .api.analytics import router as analytics_router
from .api.servers import router as servers_router

logger = logging.getLogger(__name__)

# Global instances (will be properly injected in production)
websocket_manager = WebSocketManager()
modification_engine = ModificationEngine()
storage = None  # Will be initialized in lifespan

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan management."""
    logger.info("Starting MCP DevTools web application...")

    # Initialize storage
    global storage
    database_url = "sqlite+aiosqlite:///./mcpdevtools.db"  # TODO: Get from config
    storage = MessageStorage(database_url)

    # Set up dependencies
    app.state.websocket_manager = websocket_manager
    app.state.modification_engine = modification_engine
    app.state.storage = storage

    yield

    logger.info("Shutting down MCP DevTools web application...")

# Create FastAPI app
app = FastAPI(
    title="MCP DevTools",
    description="Real-time MCP inspection and modification platform",
    version="1.0.0",
    lifespan=lifespan
)

# Add CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # TODO: Restrict in production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Mount static files and templates
app.mount("/static", StaticFiles(directory="src/mcp_proxy/web/static"), name="static")
templates = Jinja2Templates(directory="src/mcp_proxy/web/templates")

# Include API routers
app.include_router(messages_router, prefix="/api/messages", tags=["messages"])
app.include_router(modifications_router, prefix="/api/modifications", tags=["modifications"])
app.include_router(analytics_router, prefix="/api/analytics", tags=["analytics"])
app.include_router(servers_router, prefix="/api/servers", tags=["servers"])

@app.get("/", response_class=HTMLResponse)
async def dashboard(request: Request):
    """Main dashboard page."""
    return templates.TemplateResponse("dashboard.html", {"request": request})

@app.get("/health")
async def health_check():
    """Basic health check endpoint."""
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    """WebSocket endpoint for real-time communication."""
    client_id = websocket.query_params.get("client_id", "anonymous")

    await websocket_manager.connect(websocket, client_id)
    try:
        while True:
            data = await websocket.receive_text()
            try:
                message = json.loads(data)
                await websocket_manager.handle_client_message(websocket, message)
            except json.JSONDecodeError:
                await websocket.send_json({
                    "type": "error",
                    "message": "Invalid JSON format"
                })
    except WebSocketDisconnect:
        websocket_manager.disconnect(websocket)
    except Exception as e:
        logger.error(f"WebSocket error: {e}")
        websocket_manager.disconnect(websocket)

# Dependency injection helpers
def get_websocket_manager() -> WebSocketManager:
    return websocket_manager

def get_modification_engine() -> ModificationEngine:
    return modification_engine

def get_storage() -> MessageStorage:
    if storage is None:
        raise HTTPException(status_code=500, detail="Storage not initialized")
    return storage
```

#### **3.2 API Endpoints**

Create `src/mcp_proxy/web/api/messages.py`:

```python
from fastapi import APIRouter, Depends, Query, HTTPException
from typing import List, Optional
from datetime import datetime
from ..app import get_storage
from ...inspector.storage import MessageStorage

router = APIRouter()

@router.get("/history")
async def get_message_history(
    limit: int = Query(100, ge=1, le=1000),
    offset: int = Query(0, ge=0),
    server_filter: Optional[str