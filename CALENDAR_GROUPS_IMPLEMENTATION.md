# Calendar Groups Implementation Plan

## Overview
Implement cross-account calendar groups with JSON persistence, split into independent, testable packages.

## Package Architecture

### Package 1: Storage Layer (`storage.py`)
**Purpose:** Handle JSON file persistence with atomic operations
**Dependencies:** None (stdlib only)
**Testing:** Unit tests with temporary files

#### Components
- `GroupStorage` class for file I/O operations
- Atomic write pattern (temp file + rename)
- Schema validation and migration
- Error handling for file corruption

#### Interface
```python
class GroupStorage:
    def load_groups() -> dict
    def save_groups(groups: dict) -> None
    def backup_file() -> str
    def migrate_schema(data: dict) -> dict
```

#### Test Coverage
- File creation/read/write operations
- Atomic write safety (interruption simulation)
- Schema migration between versions
- Corruption recovery scenarios
- Permission error handling

---

### Package 2: Group Manager (`group_manager.py`)
**Purpose:** Business logic for calendar group CRUD operations
**Dependencies:** storage.py, models.py
**Testing:** Unit tests with mocked storage

#### Components
- `CalendarGroupManager` class for group operations
- Group validation and conflict resolution
- Calendar ID existence checking
- Group membership management

#### Interface
```python
class CalendarGroupManager:
    def create_group(name: str, calendar_ids: list[str], description: str = "") -> CalendarGroup
    def update_group(name: str, updates: dict) -> CalendarGroup
    def delete_group(name: str) -> bool
    def get_group(name: str) -> CalendarGroup | None
    def list_groups() -> list[CalendarGroup]
    def add_calendar_to_group(group_name: str, calendar_id: str) -> bool
    def remove_calendar_from_group(group_name: str, calendar_id: str) -> bool
    def get_calendars_in_group(group_name: str) -> list[str]
    def validate_calendar_exists(calendar_id: str) -> bool
```

#### Test Coverage
- CRUD operations for groups
- Validation logic (duplicate names, missing calendars)
- Group membership operations
- Edge cases (empty groups, invalid IDs)
- Concurrent access scenarios

---

### Package 3: Data Models (`group_models.py`)
**Purpose:** Pydantic models for group data structures
**Dependencies:** None (pydantic only)
**Testing:** Unit tests for validation and serialization

#### Components
- `CalendarGroup` model with validation
- JSON serialization/deserialization
- Schema versioning support
- Input validation and sanitization

#### Interface
```python
class CalendarGroup(BaseModel):
    name: str
    calendar_ids: list[str]
    description: str = ""
    created: datetime
    modified: datetime
    
    @validator('name')
    def validate_name(cls, v): ...
    
    @validator('calendar_ids')
    def validate_calendar_ids(cls, v): ...

class GroupsSchema(BaseModel):
    version: str = "1.0"
    groups: dict[str, CalendarGroup]
    
    def to_json() -> str
    @classmethod
    def from_json(cls, json_str: str) -> 'GroupsSchema'
```

#### Test Coverage
- Model validation rules
- JSON serialization round-trips
- Schema version handling
- Invalid input rejection
- Field constraints and defaults

---

### Package 4: Calendar Integration (`calendar_integration.py`)
**Purpose:** Bridge between group manager and existing CalendarManager
**Dependencies:** group_manager.py, existing ical.py
**Testing:** Integration tests with mock CalendarManager

#### Components
- `GroupCalendarBridge` class for calendar operations
- Calendar ID resolution and validation
- Group-based event filtering
- Error mapping between systems

#### Interface
```python
class GroupCalendarBridge:
    def __init__(self, calendar_manager: CalendarManager, group_manager: CalendarGroupManager)
    
    def get_events_by_group(group_name: str, start_date: datetime, end_date: datetime) -> list[Event]
    def validate_group_calendars(group_name: str) -> dict[str, bool]
    def resolve_calendar_names_to_ids(calendar_names: list[str]) -> list[str]
    def get_calendar_info_for_group(group_name: str) -> list[dict]
```

#### Test Coverage
- Group-based event retrieval
- Calendar validation integration
- Error handling from both systems
- Name-to-ID resolution accuracy
- Multi-source calendar handling

---

### Package 5: MCP Tools (`group_tools.py`)
**Purpose:** MCP server endpoints for group operations
**Dependencies:** calendar_integration.py, models.py from main codebase
**Testing:** Integration tests with FastMCP test framework

#### Components
- MCP tool implementations using FastMCP decorators
- Input validation and error response formatting
- Natural language response generation
- Integration with existing MCP server

#### Interface
```python
@mcp.tool()
async def create_calendar_group(name: str, calendar_names: list[str], description: str = "") -> str

@mcp.tool()
async def update_calendar_group(group_name: str, calendar_names: list[str] = None, description: str = None) -> str

@mcp.tool()
async def delete_calendar_group(group_name: str) -> str

@mcp.tool()
async def list_calendar_groups() -> str

@mcp.tool()
async def list_calendars_in_group(group_name: str) -> str

@mcp.tool()
async def list_events_by_group(start_date: datetime, end_date: datetime, group_name: str) -> str

@mcp.tool()
async def add_calendar_to_group(group_name: str, calendar_name: str) -> str

@mcp.tool()
async def remove_calendar_from_group(group_name: str, calendar_name: str) -> str
```

#### Test Coverage
- MCP tool parameter validation
- Response format consistency
- Error message clarity
- Integration with existing tools
- Natural language output quality

---

## Implementation Order & Git Commit Strategy

### Phase 1: Foundation (Packages 1-3)

#### Package 1: Storage Layer
- **Commit 1:** `feat(groups): add storage layer foundation`
  - Create `groups/` directory structure
  - Implement `GroupStorage` class with basic file I/O
  - Add atomic write operations
  - Include unit tests for storage operations

#### Package 2: Data Models  
- **Commit 2:** `feat(groups): add Pydantic models for calendar groups`
  - Implement `CalendarGroup` and `GroupsSchema` models
  - Add validation rules and serialization
  - Include unit tests for model validation

#### Package 3: Group Manager
- **Commit 3:** `feat(groups): implement group CRUD operations`
  - Create `CalendarGroupManager` class
  - Add group creation, update, delete operations
  - Include unit tests with mocked storage

### Phase 2: Integration (Package 4)

#### Package 4: Calendar Integration
- **Commit 4:** `feat(groups): add calendar system integration bridge`
  - Implement `GroupCalendarBridge` class
  - Add calendar validation and event filtering
  - Include integration tests with existing CalendarManager

### Phase 3: Interface (Package 5)

#### Package 5: MCP Tools
- **Commit 5:** `feat(groups): add MCP tools for group operations`
  - Implement all MCP tool endpoints
  - Add natural language response formatting
  - Include FastMCP integration tests

### Phase 4: Enhancement

#### Existing Tools Update
- **Commit 6:** `feat(groups): extend existing tools with group support`
  - Update existing MCP tools to accept group parameters
  - Maintain backward compatibility

#### Documentation & Polish
- **Commit 7:** `docs(groups): add comprehensive group documentation`
  - Add usage examples and API documentation
  - Update README with group functionality

## Testing Strategy

### Unit Tests (Per Package)
- **Isolated testing** with mocked dependencies
- **High coverage** (>90%) for each package
- **Fast execution** (<1s per package)
- **Deterministic** test outcomes

### Integration Tests
- **Cross-package** functionality testing
- **File system** operations with temporary directories
- **Calendar system** integration with test calendars
- **MCP protocol** testing with FastMCP framework

### Test Data
- **Sample calendar groups** for consistent testing
- **Mock calendar data** across different sources
- **Edge case scenarios** (corrupted files, missing calendars)

## File Structure
```
src/mcp_ical/
├── groups/
│   ├── __init__.py
│   ├── storage.py              # Package 1
│   ├── group_models.py         # Package 3  
│   ├── group_manager.py        # Package 2
│   ├── calendar_integration.py # Package 4
│   └── group_tools.py          # Package 5
├── ical.py                     # Existing
├── models.py                   # Existing
└── server.py                   # Existing (updated)

tests/
├── groups/
│   ├── test_storage.py
│   ├── test_group_models.py
│   ├── test_group_manager.py
│   ├── test_calendar_integration.py
│   └── test_group_tools.py
└── test_integration.py
```

## Commit Guidelines

### Commit Message Format
Follow conventional commits: `type(scope): description`

### Pre-Commit Checklist (Per Package)
- [ ] All unit tests pass for the package
- [ ] Code follows existing style conventions
- [ ] No breaking changes to existing functionality  
- [ ] Documentation updated for new features
- [ ] Integration tests pass (where applicable)

### Testing Before Each Commit
```bash
# Run tests for specific package
python -m pytest tests/groups/test_{package_name}.py -v

# Run full test suite (before final commits)
python -m pytest tests/ -v

# Check code style (if available)
ruff check src/mcp_ical/groups/
```

## Success Criteria
- [ ] All packages pass unit tests independently
- [ ] Integration tests pass with real calendar data
- [ ] Groups persist across application restarts
- [ ] Cross-account calendars work in single groups
- [ ] Natural language queries work with groups
- [ ] Existing functionality remains unaffected
- [ ] Performance: <100ms for group operations
- [ ] Error recovery: Graceful handling of missing calendars/files
- [ ] Clean git history with logical, atomic commits

## Agent Assignment Guidelines
- **One agent per package** for parallel development
- **Clear interfaces** defined upfront for integration
- **Minimal dependencies** between packages
- **Test-first development** approach
- **Documentation** included with each package