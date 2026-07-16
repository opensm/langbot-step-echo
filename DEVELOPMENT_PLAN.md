# LangBot Dify Node Type Awareness Plugin - Development Plan

## 1. Project Overview

Based on the updated PRD (LangBot-Dify-StageAwareness-PRD-v1.1.md), we need to develop a LangBot plugin that provides **node type awareness** for Dify workflows: when a node type matches the configuration (e.g., `direct_output`), the plugin extracts the node’s content and triggers a message through LangBot's built-in functionality, enabling real-time visibility of specific node outputs.

## 2. Architecture Overview

Following the PRD's technical architecture section (Section 5), we'll implement a plugin with these core components:

```
LangBot Core → Dify API (Streaming) → [Node Type Awareness Plugin] 
                                    ↓
                           Event Parser → Node Type Resolver → Content Extractor → Message Trigger
                                                    ↓
                                         LangBot Built-in Messaging → Final Answer Output
```

### 2.1 Core Components

1. **Event Parser** - Parses Dify Streaming API events (`workflow_node_started` / `workflow_node_finished`) into typed objects.
2. **Node Type Resolver** - Determines whether a given node type (and optional workflow ID) matches the configured list.
3. **Content Extractor** - Extracts the displayable content from a matched node (prefers `outputs`, falls back to `inputs`).
4. **Message Trigger** - Triggers LangBot's built-in messaging functionality with the extracted content when a node type matches.
5. **Configuration Manager** - Loads and validates the YAML node‑type configuration.

## 3. Development Approach

Following Python best practices from the `python-patterns` skill:

### 3.1 Project Structure
```
langbot_stage_awareness/
├── src/
│   └── langbot_stage_awareness/
│       ├── __init__.py
│       ├── config.py
│       ├── event_parser.py
│       ├── node_type_resolver.py
│       ├── content_extractor.py
│       ├── message_trigger.py
│       └── plugin.py
├── tests/
│   ├── __init__.py
│   ├── test_config.py
│   ├── test_event_parser.py
│   ├── test_node_type_resolver.py
│   ├── test_content_extractor.py
│   ├── test_message_trigger.py
│   └── ...
├── configs/
│   └── node_type_config.yml.example
├── pyproject.toml
├── README.md
└── LICENSE
```

### 3.2 Development Phases

#### Phase 1: Project Setup & Configuration (Days 1-2)
- Initialize Python project with proper structure
- Create configuration loading mechanism (YAML)
- Implement configuration validation
- Set up development environment (pytest, black, ruff, mypy)

#### Phase 2: Core Event Processing & Resolution (Days 3-5)
- Implement Dify Streaming API event parser
- Implement Node Type Resolver logic
- Implement Content Extractor logic
- Implement Message Trigger to interface with LangBot's built-in messaging
- Add basic logging and metrics infrastructure

#### Phase 3: Testing & Refinement (Days 6-8)
- Write unit tests for all components (≥80% coverage)
- Implement integration tests (end‑to‑end flow)
- Performance testing and optimization
- Security review and code cleanup

## 4. Technical Implementation Details

### 4.1 Configuration Management
- Use `@dataclass` for configuration models with `__post_init__` validation (Python patterns: Data Classes).
- Use type hints extensively (Python 3.9+ built‑in types).
- Provide clear docstrings.

### 4.2 Error Handling
- Follow EAFP (try/except) rather than pre‑checking.
- Custom exception hierarchy where needed (e.g., ConfigError).
- Chain exceptions with `from` when re‑raising to preserve traceback.

### 4.3 Resource Management
- Use `requests.Session` with retry adapter for connection pooling to Dify API.
- Ensure proper cleanup (though session lives for plugin lifetime).

### 4.4 Concurrency
- The plugin is single‑threaded (event‑by‑event) but uses `requests` with internal retry threading; no additional concurrency needed.
- If high QPS required, could move Dify API calls to a thread pool, but not necessary for expected load.

### 4.5 Testing Strategy
- Unit tests for each component.
- Integration tests using mocked Dify SSE events and mocked LangBot messaging interface.
- Property‑based testing for edge cases (e.g., malformed JSON, missing fields).
- Mock external dependencies.

## 5. Key Implementation Files

### 5.1 config.py
```python
from dataclasses import dataclass, field
from typing import List, Optional
import yaml

@dataclass
class NodeTypeConfig:
    workflow_id: Optional[str] = None   # None => match any workflow
    node_types: List[str] = field(default_factory=list)

    def __post_init__(self):
        if self.node_types is None:
            self.node_types = []
        # Ensure all entries are strings
        self.node_types = [str(t) for t in self.node_types]

@dataclass
class PluginConfig:
    node_type_config: NodeTypeConfig
    # Removed Feishu-specific config as messaging uses LangBot's built-in functionality
    enable_metrics: bool = True

    def __post_init__(self):
        if not self.node_type_config.node_types:
            raise ValueError("node_types must not be empty")
```

### 5.2 event_parser.py
```python
import json
from typing import Dict, Any, Optional
from dataclasses import dataclass

@dataclass
class NodeStartedEvent:
    event: str
    workflow_run_id: str
    node_id: str
    node_title: str
    node_type: str
    inputs: Dict[str, Any]
    outputs: Dict[str, Any]
    created_at: int

@dataclass
class NodeFinishedEvent:
    event: str
    workflow_run_id: str
    node_id: str
    node_title: str
    node_type: str
    inputs: Dict[str, Any]
    outputs: Dict[str, Any]
    elapsed_time: Optional[float]
    status: str   # "succeeded" or "failed"

class EventParser:
    @staticmethod
    def parse_event(raw_event: str) -> Optional[NodeStartedEvent | NodeFinishedEvent]:
        try:
            data = json.loads(raw_event)
            event_type = data.get("event")
            if event_type not in ("workflow_node_started", "workflow_node_finished"):
                return None
            node_data = data.get("data", {})
            common = dict(
                event=event_type,
                workflow_run_id=data.get("workflow_run_id", ""),
                node_id=node_data.get("id", ""),
                node_title=node_data.get("title", ""),
                node_type=node_data.get("node_type", ""),
                inputs=node_data.get("inputs", {}),
                outputs=node_data.get("outputs", {}),
            )
            if event_type == "workflow_node_started":
                return NodeStartedEvent(
                    **common,
                    created_at=node_data.get("created_at", 0),
                )
            else:  # finished
                return NodeFinishedEvent(
                    **common,
                    elapsed_time=node_data.get("elapsed_time"),
                    status=node_data.get("status", "unknown"),
                )
        except (json.JSONDecodeError, KeyError, TypeError):
            return None
```

### 5.3 node_type_resolver.py
```python
from typing import Optional
from .config import NodeTypeConfig

class NodeTypeResolver:
    def __init__(self, config: NodeTypeConfig):
        self.config = config

    def is_matching(self, node_type: str, workflow_id: Optional[str] = None) -> bool:
        if self.config.workflow_id and workflow_id != self.config.workflow_id:
            return False
        return node_type in self.config.node_types
```

### 5.4 content_extractor.py
```python
from typing import Dict, Any, Optional
import json

class ContentExtractor:
    @staticmethod
    def extract(event_data: Dict[str, Any]) -> Optional[str]:
        data = event_data.get("data", {})
        content = data.get("outputs")
        if content is None:
            content = data.get("inputs")
        if content is None:
            return None
        try:
            return json.dumps(content, ensure_ascii=False, indent=2)
        except Exception:
            return str(content)
```

### 5.5 message_trigger.py
```python
import logging
from typing import Optional

logger = logging.getLogger(__name__)

class MessageTrigger:
    """Triggers messages through LangBot's built-in functionality."""
    
    def __init__(self):
        # In a real implementation, this would initialize connection to LangBot's messaging API
        pass
        
    def trigger_message(self, content: str, node_type: str, workflow_run_id: str) -> bool:
        """
        Trigger a message through LangBot's built-in functionality.
        
        Args:
            content: The extracted content to send
            node_type: The type of node that generated the content
            workflow_run_id: The workflow run ID for tracking
            
        Returns:
            True if triggered successfully, False otherwise
        """
        try:
            # In a real implementation, this would call into LangBot's messaging API
            # For demonstration, we log what would be sent
            logger.info(
                f"Triggering message for workflow {workflow_run_id}: "
                f"node_type={node_type}, content_length={len(content)}"
            )
            
            # Example of what the actual implementation might do:
            # langbot_messaging.send_message(
            #     content=f"[Node Type: {node_type}]\n{content}",
            #     metadata={"workflow_run_id": workflow_run_id, "node_type": node_type}
            # )
            
            return True
        except Exception as e:
            logger.error(f"Failed to trigger message: {e}", exc_info=True)
            return False
```

### 5.6 plugin.py
```python
"""LangBot Dify Node Type Awareness Plugin - Main Plugin Interface."""

import logging
import time
from typing import Optional
from .config import PluginConfig
from .event_parser import EventParser, NodeStartedEvent, NodeFinishedEvent
from .node_type_resolver import NodeTypeResolver
from .content_extractor import ContentExtractor
from .message_trigger import MessageTrigger

logger = logging.getLogger(__name__)


class NodeTypeAwarenessPlugin:
    """
    Main plugin class for LangBot Dify Node Type Awareness functionality.
    The plugin focuses on message content acquisition and triggering,
    while message sending is handled by LangBot's built-in functionality.
    """

    def __init__(self, config: PluginConfig):
        self.config = config
        self.event_parser = EventParser()
        self.resolver = NodeTypeResolver(config.node_type_config)
        self.extractor = ContentExtractor()
        self.message_trigger = MessageTrigger()
        self.logger = logging.getLogger(__name__ + ".NodeTypeAwarenessPlugin")
        self.is_enabled = True
        self.node_start_times: dict[str, float] = {}   # workflow_run_id -> start time

    # ------------------- Public API -------------------

    def process_dify_event(self, event_data: str) -> bool:
        """
        Process a single Dify streaming event.

        Returns:
            True if processed successfully, False on unrecoverable error.
        """
        if not self.is_enabled:
            return False

        try:
            event = self.event_parser.parse_event(event_data)
            if not event:
                return True   # ignore non‑relevant events

            if isinstance(event, NodeStartedEvent):
                self._handle_node_started(event)
                return True
            elif isinstance(event, NodeFinishedEvent):
                self._handle_node_finished(event)
                return True
            else:
                return True
        except Exception as e:
            self.logger.error(f"Error processing Dify event: {e}", exc_info=True)
            return False

    # ------------------- Internal handlers -------------------
    def _handle_node_started(self, event: NodeStartedEvent) -> None:
        """Record start time for timeout detection."""
        self.node_start_times[event.workflow_run_id] = time.time()
        self._maybe_trigger_message(event)

    def _handle_node_finished(self, event: NodeFinishedEvent) -> None:
        """Clear start time and possibly trigger message (for failure case)."""
        self.node_start_times.pop(event.workflow.workflow_id)event.workflow_run_id, None)
        self._maybe_trigger_message(event)

    # ------------------- Message trigger logic -------------------
    def _maybe_trigger_message(self, event) -> None:
        """Decide whether to trigger a message based on the event."""
        try:
            # Determine if we need to show timeout/failure first
            content_to_send = None
            wr_id = event.workflow_run_id

            # Timeout detection (only for started events; for finished we rely on status)
            if isinstance(event, NodeStartedEvent):
                start = self.node_start_times.get(wr_id)
                if start and (time.time() - start) > 30.0:
                    # Timeout case - send a special message
                    content_to_send = f"⏳ 节点处理中（耗时较长，请耐心等待）\n\n节点：{event.node_title}"

            # Failure detection (status == finished)
            if content_to_send is None and isinstance(event, NodeFinishedEvent):
                if event.status == "failed":
                    # Failure case - send a special message
                    content_to_send = f"⚠️ 节点处理异常\n\n节点：{event.node_title}\n状态：{event.status}"

            # Normal matching node type
            if content_to_send is None:
                if self.resolver.is_matching(event.node_type, wr_id):
                    content = self.extractor.extract(event.__dict__)
                    if content:
                        content_to_send = f"**节点类型**：{event.node_type}\n\n**内容**：\n{content}"

            # If still nothing, we do nothing (silent)
            if content_to_send is None:
                return

            # Send message through LangBot's built-in functionality
            if self.message_trigger.trigger_message(
                content=content_to_send,
                node_type=event.node_type,
                workflow_run_id=wr_id
            ):
                self.logger.info(f"Successfully triggered message for workflow {wr_id}")
            else:
                self.logger.warning(f"Failed to trigger message for workflow {wr_id}")

        except Exception as e:
            self.logger.error(f"Error triggering message: {e}", exc_info=True)

    # ------------------- Lifecycle -------------------
    def enable(self) -> None:
        self.is_enabled = True
        self.logger.info("Node type awareness plugin enabled")

    def disable(self) -> None:
        self.is_enabled = False
        self.logger.info("Node type awareness plugin disabled")

    def get_status(self) -> dict:
        return {
            "enabled": self.is_enabled,
            "workflow_id": self.config.node_type_config.workflow_id,
            "configured_node_types": self.config.node_type_config.node_types,
        }


# Factory function
def create_plugin_from_config(config_dict: dict) -> NodeTypeAwarenessPlugin:
    from .config import PluginConfig, NodeTypeConfig

    node_cfg = NodeTypeConfig(
        workflow_id=config_dict.get("workflow_id"),
        node_types=config_dict.get("node_types", []),
    )
    cfg = PluginConfig(
        node_type_config=node_cfg,
        enable_metrics=config_dict.get("enable_metrics", True),
    )
    return NodeTypeAwarenessPlugin(cfg)
```
## 6. Development Milestones

| Milestone | Deliverables | Estimated Effort | Target Date |
|-----------|--------------|------------------|-------------|
| **M1: Requirements Freeze** | PRD v1.1 finalized + technical design review | 2 days | T+2 |
| **M2: Core Framework** | Event parser, node‑type resolver, content extractor, config loading, basic logging | 2 days | T+4 |
| **M3: Message Trigger Implementation** | Message trigger to interface with LangBot's built-in messaging, content extraction logic | 3 days | T+7 |
| **M4: Testing & Refinement** | Unit test coverage ≥80%, integration tests, performance checks, documentation, linting, code review | 2 days | T+9 |
| **Total** | | **9 days** | |

## 7. Risk Management

| Risk | Impact | Mitigation |
|------|--------|------------|
| Dify Streaming API format changes | Event parsing fails | Validate required fields; ignore unknown fields; easy to adapt parser. |
| LangBot plugin interface incompatibility | Cannot hook into LangBot | Verify LangBot version and plugin entry point early in M1. |
| LangBot messaging API changes | Message triggering fails | Abstract messaging interface; update adapter as needed. |
| Message triggering failures | Node content not displayed | Implement retry mechanism; fallback to logging for debugging. |
| Configuration errors (YAML) | Plugin fails to start | Schema validation in `config.py`; provide clear error messages. |

## 8. Definition of Done

A feature is considered complete when:
1. Code follows Python best practices (PEP 8, type hints, dataclasses, EAFP).
2. Unit test coverage ≥80% for all modules.
3. Code passes `ruff`, `flake8`, and `mypy` checks.
4. Documentation (README, docstrings) is updated.
5. Changes are reviewed and merged into main branch.
6. Integration tests in a staging environment pass (end‑to‑end flow with mocked Dify and LangBot messaging interface).

## 9. Appendices

### 9.1 Open Questions
1. How does LangBot provide messaging functionality to plugins for triggering messages? (Expectation: an API or interface will be provided by the host.)  
2. Should the plugin support multiple concurrent workflows independently? (Yes – we keep per‑workflow timestamps and sequences.)  
3. Is there a need to limit the size of extracted content shown in messages? (We can truncate if excessively large; currently we send full JSON string.)
4. What formatting options does LangBot's built-in messaging support for node content display?

### 9.2 References
- [LangBot Plugin Documentation](https://docs.langbot.app/plugin)
- [Dify Streaming API Guide](https://docs.dify.ai/guides/application-publishing/developing-with-apis)
- [LangBot Messaging Interface Documentation](https://docs.langbot.app/plugin/messaging)
- Python Patterns Skill Guidelines (provided in context)