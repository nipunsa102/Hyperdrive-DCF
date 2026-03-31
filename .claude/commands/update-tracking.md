---
description: Update Tracking
model: claude-opus-4-6
---

switches: `-module`, `-status`

Switch Definitions:
- `-module` -> Specific module to update (e.g., M1, M2, M3)
- `-status` -> New status (not_started, in_progress, l1_pass, blocked, complete, deployed)

## Purpose
Updates module status in the tracking document. This command ensures that module status and progress remain accurate and up-to-date.

## Prerequisites
- Tracking document must exist: `/tracking/module-tracking.md`
- Architecture must exist: `/architecture/architecture.md`

## Process Flow

### Phase 1: Status Collection

```python
def collect_current_status():
    """Gather current status from module-tracking.md."""

    status_data = {
        'modules': {}
    }

    # Read module-tracking.md
    tracking = read_file("/tracking/module-tracking.md")

    # Parse module statuses from table
    for module in parse_module_table(tracking):
        status_data['modules'][module['id']] = {
            'name': module['name'],
            'status': module['status'],
            'l1_coverage': module['l1_coverage'],
            'l2_status': module['l2_status'],
            'dependencies': module['dependencies']
        }

    return status_data
```

### Phase 2: Status Update

```python
def update_tracking_status(module=None, new_status=None):
    """Update status in module-tracking.md."""

    if module and new_status:
        # Update specific module
        updates = {module: new_status}
    else:
        # Sync all modules (recalculate based on current state)
        updates = reconcile_statuses()

    # Update module-tracking.md
    update_module_tracking(updates)

    # Recalculate summary counts
    update_summary_counts()
```

### Phase 3: Progress Calculation

```python
def update_summary_counts():
    """Calculate and update summary section."""

    # Read current module statuses
    statuses = get_all_module_statuses()

    # Count by status
    total = len(statuses)
    complete = sum(1 for s in statuses.values() if s == 'complete')
    in_progress = sum(1 for s in statuses.values() if s == 'in_progress')
    l1_pass = sum(1 for s in statuses.values() if s == 'l1_pass')
    blocked = sum(1 for s in statuses.values() if s == 'blocked')
    not_started = sum(1 for s in statuses.values() if s == 'not_started')

    # Update summary section in module-tracking.md
    update_summary({
        'total': total,
        'complete': complete,
        'in_progress': in_progress + l1_pass,
        'blocked': blocked,
        'not_started': not_started
    })
```

## Usage Examples

### Automatic Sync (Called by generate-code)

```bash
# Sync all module tracking
update-tracking

# Output:
Synchronizing module tracking...
✓ Reading /tracking/module-tracking.md
✓ Found 5 modules to track
✓ Updated summary counts
  - Complete: 2
  - In Progress: 2
  - Blocked: 0
  - Not Started: 1
✓ Tracking document synchronized
```

### Manual Status Update

```bash
# Update specific module status
update-tracking -module=M2 -status=complete

# Output:
Updating M2 (API Layer) status to complete...
✓ Updated in module-tracking.md
✓ Recalculated progress: 3/5 modules complete
```

### After L1 Pass

```bash
# Mark module as L1 passed
update-tracking -module=M3 -status=l1_pass

# Output:
Updating M3 (Core UI) status to l1_pass...
✓ Updated in module-tracking.md
✓ Summary: 2 complete, 2 in progress, 1 not started
```

## Integration with generate-code

The `generate-code` command automatically invokes tracking-update-agent:

```python
# Within generate-code command
def mark_module_l1_pass(module_id, coverage):
    """Mark module L1 tests passed."""
    INVOKE tracking-update-agent with:
      - Event: module_l1_pass
      - Module: {module_id}
      - Coverage: {coverage}

def mark_module_complete(module_id):
    """Mark module as complete after L2 pass."""
    INVOKE tracking-update-agent with:
      - Event: implementation_complete
      - Modules: all
```

## Module Tracking Document Format

```markdown
# Module Tracking

## Summary
- Total Modules: 5
- Complete: 2
- In Progress: 2
- Blocked: 0
- Not Started: 1

## Module Status

| Module | Name | Status | L1 Coverage | L2 Status | Dependencies | Notes |
|--------|------|--------|-------------|-----------|--------------|-------|
| M1 | Data Layer | Complete | 85% | Pass | None | All tests pass |
| M2 | API Layer | Complete | 82% | Pass | M1 | All tests pass |
| M3 | Core UI | L1 Pass | 75% | Pending | M1, M2 | Awaiting L2 |
| M4 | Content Mgmt | In Progress | - | - | M2 | Implementation ongoing |
| M5 | Editors | Not Started | - | - | M3, M4 | |

## Blockers

(List any blocked modules with reasons)

## History

### 2025-01-20
- M1: Complete (85% coverage, L2 pass)
- M2: Complete (82% coverage, L2 pass)
- M3: L1 Pass (75% coverage)
```

## Status Definitions

| Status | Description |
|--------|-------------|
| Not Started | Module implementation not yet begun |
| In Progress | Module currently being implemented |
| L1 Pass | Module passed L1 unit tests (meets coverage target) |
| Blocked | Module blocked by test failure or dependency |
| Complete | Module passed both L1 and L2 tests |
| Deployed | Module deployed to target environment |

## Best Practices

1. **Run after each module completion** to keep tracking current
2. **Use via tracking-update-agent** for automatic updates from generate-code
3. **Always sync before reviews** for accurate reporting

## Error Handling

Common issues and resolutions:

1. **Missing module-tracking.md**: Create initial file from architecture
2. **Parse errors**: Check markdown table formatting
3. **Invalid status**: Use only defined status values

---

**Command Type**: Tracking Synchronization
**Invocation**: Automatic (via generate-code) or Manual
**Output**: Updated module-tracking.md
**DCF Step**: Supports Step 4 (Implementation)
