# Analytics Implementation

## Files

- `analytics.py` - `capture(event, **properties)` async function, direct HTTP to PostHog
- `config.py` - `~/.claude/claudechic.yaml` management (user ID, opt-out flag)

## Adding New Events

1. Call `capture("event_name", prop1=value1, prop2=value2)` from app.py
2. Use `self.run_worker(capture(...))` for fire-and-forget from sync context
3. For shutdown events, use `await capture(...)` to ensure delivery

## Current Events

### App Lifecycle
- `app_installed` - first launch only, includes `claudechic_version`, `os`
- `app_started` - every launch, includes `claudechic_version`, `term_width`, `term_height`, `term_program`, `os`, `has_uv`, `has_conda`, `is_git_repo`, `resumed`
- `app_closed` - shutdown, includes `duration_seconds`, `term_width`, `term_height`

### Agent Lifecycle
- `agent_created` - new agent, includes `same_directory`, `model`
- `agent_closed` - agent closes, includes `message_count`, `duration_seconds`, `same_directory`

### User Actions
All include `agent_id` to link events to specific Claude sessions.

- `message_sent` - when user sends a message to Claude
- `command_used` - when user runs a slash command, includes `command` name
- `model_changed` - when user switches models, includes `from_model`, `to_model`
- `worktree_action` - when user runs worktree commands, includes `action` (create/finish/cleanup/discard)

### MCP Tools (Claude-initiated)
All include `agent_id`.

- `mcp_tool_used` - when Claude calls an MCP tool, includes `tool` (spawn_agent/spawn_worktree/ask_agent/tell_agent)

### Errors
- `error_occurred` - on errors, includes `error_type`, `status_code`, `agent_id`

## Design Decisions

- **No PostHog SDK** - direct HTTP keeps dependencies minimal
- **Context only on app_started** - other events are minimal (just `$session_id` + event-specific props)
- **Opt-out not opt-in** - check `get_analytics_enabled()` before sending
- **Silent failures** - analytics must never crash or slow the app

## Testing

Add debug logging temporarily:
```python
# In analytics.py before the try block
import json
with open("/tmp/posthog_debug.log", "a") as f:
    f.write(json.dumps(payload, indent=2) + "\n---\n")
```

Then restart app via remote: `curl -s -X POST localhost:9999/exit`
