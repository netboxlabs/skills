# Deviation Lifecycle

## State Machine

```
                    ┌──────────┐
                    │  QUEUED   │  ← Data ingested
                    └────┬─────┘
                         │ Analysis runs
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         ┌────────┐ ┌──────────┐ ┌─────────┐
         │  OPEN  │ │NO_CHANGES│ │ ERRORED │
         └───┬────┘ └──────────┘ └─────────┘
             │
     ┌───────┼────────┐
     ▼       ▼        ▼
┌─────────┐┌───────┐┌─────────┐
│ APPLIED ││IGNORED││ FAILED  │
└────┬────┘└───┬───┘└────┬────┘
     │         │         │
     │    Reopen ──► OPEN │
     │                    │
     └──── Rediff ────────┘──► OPEN (if new changes found)
```

## States

| State | Description | Terminal? |
|-------|-------------|-----------|
| **Queued** | Data received, awaiting comparison against NetBox | No |
| **Open** | Drift detected with field-level changes, ready for review | No |
| **Applied** | Operator accepted changes; NetBox updated | Soft (can rediff) |
| **Ignored** | Operator dismissed the deviation | Soft (can reopen) |
| **Failed** | Apply was attempted but encountered an error | No |
| **No Changes** | Analysis found no differences | Yes |
| **Errored** | System error during analysis | Terminal unless re-ingested |

## Transitions

| From | To | Trigger |
|------|-----|---------|
| Queued | Open | Analysis finds differences |
| Queued | No Changes | Analysis finds no differences |
| Queued | Errored | Processing error |
| Open | Applied | User clicks **Apply** |
| Open | Ignored | User clicks **Ignore** |
| Open | Failed | Apply attempted but errored |
| Ignored | Open | User clicks **Reopen** |
| Applied | Open | **Rediff** finds new differences |
| Failed | Open | **Rediff** after fixing root cause |

## Actions in Detail

### Apply

Accepts the deviation and writes the change to NetBox. For **create** deviations, the object is added to NetBox. For **update** deviations, the differing fields are updated.

**When to use:** The actual network state is correct and NetBox should be updated to match.

**Watch out for:** Dependency ordering — creating a device requires its site, device type, and device role to exist first. The system handles most dependencies automatically, but complex chains may need attention.

### Ignore

Permanently dismisses the deviation. The drift still exists, but you've decided it's acceptable.

**When to use:** Known discrepancy that doesn't need correction (e.g., test equipment, expected differences between intended and actual state).

**Reversible:** Yes — use **Reopen** to bring it back.

### Rediff

Re-runs the comparison against the current NetBox state. Useful when:
- You manually updated NetBox and want to confirm the deviation resolves
- NetBox state changed since the deviation was created
- An apply failed and you've fixed the underlying issue

Rediff may result in the deviation moving to **No Changes** (if the drift was resolved externally) or remaining **Open** with updated change details.

### Reopen

Brings an **Ignored** deviation back to **Open** state for re-evaluation.

**When to use:** Circumstances changed and a previously ignored deviation now needs attention.

## Bulk Operations

All actions (Apply, Ignore, Rediff) support bulk execution from list views. Select multiple deviations and choose the bulk action.

**Tips for bulk operations:**
- Start with **Bulk Rediff** to refresh stale deviations before acting
- Use filters to narrow to a specific object type or source before bulk applying
- Review a sample of deviations before bulk-applying a large batch

## Resolution Patterns

### Clean Sweep (Day 1)
1. Run Discovery against the network
2. Filter deviations by object type (start with sites, then devices, then interfaces)
3. Review a sample for accuracy
4. Bulk apply by object type, respecting dependency order

### Ongoing Triage (Day 2)
1. Check Active Deviations daily (or on schedule)
2. Filter by newest first
3. Review and resolve individually or in small batches
4. Ignore known acceptable drift

### Post-Change Verification
1. Make a planned change to the network
2. Wait for Discovery to re-scan
3. Check that expected deviations appear
4. Apply to update NetBox, confirming the change is reflected
