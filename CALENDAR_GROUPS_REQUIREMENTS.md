# Calendar Groups Support Requirements

## Overview
Add cross-account calendar group support to enable organizing calendars from different sources (iCloud, Google, Office365, etc.) into logical collections with persistent storage.

## Research Findings
**EKCalendarGroup does not exist** in Apple's EventKit framework. EventKit only supports account-based organization via `EKSource` objects, which cannot span multiple accounts. Custom calendar grouping requires implementing our own persistence layer.

## Core Requirements

### Cross-Account Support
- Groups must work across all calendar sources (iCloud, Google, Exchange, etc.)
- Calendar identification using `calendarIdentifier` for persistence
- Handle calendar source changes gracefully

### Persistent Storage
- Groups persist between app launches
- Local storage implementation (no cloud dependency)
- Support for group metadata and configuration

### New MCP Tools
- `create_calendar_group(name, calendar_names)` - Create new custom group
- `update_calendar_group(group_name, calendar_names)` - Modify group membership
- `delete_calendar_group(group_name)` - Remove custom group
- `list_calendar_groups()` - List all custom groups
- `list_calendars_in_group(group_name)` - Get calendars within a specific group
- `list_events_by_group(start_date, end_date, group_name)` - Get events from all calendars in a group

### Enhanced Existing Tools
- `list_events()` - Add optional `group_name` parameter
- `list_calendars()` - Show group membership in output

### Data Models
- `CalendarGroup` model with name, calendar identifiers, and metadata
- Update `Event` model to include group context
- `GroupManager` class for persistence operations

## Technical Implementation

### Custom Storage Layer
- Use JSON file for group persistence (`~/.mcp-ical-groups.json`)
- Store calendar identifiers, not names (names can change)
- Validate calendar existence on group operations

### Code Changes
- Create `GroupManager` class for JSON persistence
- Extend `CalendarManager` with group lookup methods
- Add group filtering to event queries
- Handle calendar ID resolution and validation

### Storage Format
```json
{
  "groups": {
    "Work": {
      "calendar_ids": ["cal-123", "cal-456"],
      "created": "2025-01-22T10:00:00Z",
      "description": "Work-related calendars"
    },
    "Personal": {
      "calendar_ids": ["cal-789"],
      "created": "2025-01-22T10:00:00Z", 
      "description": "Personal calendars"
    }
  }
}
```

## User Experience

### Natural Language Examples
```
"Create a Work group with my Office365 and Google work calendars"
"What meetings do I have in my Work calendars this week?"
"Add my Project calendar to the Work group"
"Show me events from Family calendars next month"
```

### CRUD Operations
```
"Create group 'Fitness' with my Gym and Sports calendars"
"Remove Personal calendar from the Work group"
"Delete the Old Work group"
"List all my calendar groups"
```

## Edge Cases
- Calendars deleted from system but still in groups
- Duplicate calendar names across sources
- Group names conflicting with calendar names
- Storage file corruption or migration
- Calendar identifiers changing (rare but possible)

## Success Criteria
- Groups work across all calendar account types
- Groups persist between app restarts
- Full CRUD operations available via MCP tools
- Maintains backward compatibility
- Natural language group queries work
- Graceful handling of missing calendars