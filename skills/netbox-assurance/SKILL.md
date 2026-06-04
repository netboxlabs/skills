---
name: netbox-assurance
description: >
  NetBox Assurance drift detection and deviation management. Use when working with
  network state comparison, deviation review/remediation, data source configuration,
  or understanding how intended vs actual network state is reconciled in NetBox Cloud and Enterprise.
license: Apache-2.0
---

# NetBox Assurance

> **Your knowledge of NetBox Assurance may be outdated.** Deviation types, data source configuration, and API behavior evolve between releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Assurance docs | `https://netboxlabs.com/docs/assurance/` | Overview, getting started, UI workflows |
| NetBox Platform MCP | If configured — query deviations, data sources | Live drift status |

NetBox Assurance detects **drift** between your intended network state (what NetBox says should exist) and the actual network state (what's really out there). It surfaces differences as **deviations** that you review and resolve through the NetBox UI.

## When to Use This Skill

- Configuring or troubleshooting Assurance deviation workflows
- Understanding drift types and how to resolve them
- Setting up data sources that feed actual network state
- Integrating Assurance with Discovery and Diode
- Building operational workflows around deviation triage

## Quick Reference

### Deviation States

| State | Meaning |
|-------|---------|
| **Queued** | Ingested data awaiting analysis |
| **Open** | Drift detected, ready for review |
| **Applied** | Change accepted and pushed to NetBox |
| **Ignored** | Dismissed by operator |
| **Failed** | Apply attempted but errored |
| **No Changes** | Analysis found no differences |
| **Errored** | Processing error during analysis |

### Actions

| Action | What It Does | Available From |
|--------|-------------|----------------|
| **Apply** | Accept deviation, update NetBox to match actual state | Open, Failed |
| **Ignore** | Dismiss deviation permanently | Open |
| **Rediff** | Re-analyze against current NetBox state | Open, Applied, Failed |
| **Reopen** | Bring back an ignored deviation | Ignored |

All actions support **bulk operations** on multiple deviations.

### Drift Types

| Type | Description | Example |
|------|------------|---------|
| **Attribute drift** | Object exists in both places but properties differ | Device serial number changed |
| **Inventory drift (additive)** | Object discovered on network but missing from NetBox | New interface found on device |
| **Inventory drift (stale)** | Object in NetBox but not recently observed on network | Decommissioned device still in NetBox |

> **Topology drift** (relationship mismatches like device→wrong site) and **configuration drift** (running config vs intended) are future capabilities.

## Architecture

Assurance follows a three-stage pipeline:

```
1. INGESTION          2. ANALYSIS              3. RESOLUTION
   ─────────            ────────                 ──────────
   Actual state    →    Compare actual     →     Review deviations
   collected from       vs intended              and take action
   network              (NetBox) state           (apply/ignore)
```

**Data sources** feed actual network state into the system:
- **Discovery agents** — automated SNMP/API-based network scanning
- **Controller integrations** — VMware vCenter, Cisco Catalyst Center, HPE Mist, etc.
- **Diode SDK** — custom Python/Go integrations for any data source

The analysis engine compares ingested data against NetBox and produces deviations with field-level change details (before/after values for each differing attribute).

## Data Sources

Assurance requires at least one data source providing actual network state. See [references/data-sources.md](references/data-sources.md) for detailed setup.

### Discovery Agents (Recommended Starting Point)

The Orb Agent from NetBox Discovery actively scans your network and feeds results directly into Assurance. This is the most common path for getting started.

See the [netbox-discovery](../netbox-discovery/SKILL.md) skill for agent configuration.

### Controller Integrations

Pre-built integrations with infrastructure controllers (VMware vCenter, Cisco Catalyst Center, HPE Mist, etc.) pull state from management platforms.

### Diode SDK (Custom Integrations)

For data sources without built-in support, use the Diode SDK (Python or Go) to push entity data programmatically.

See the [netbox-diode](../netbox-diode/SKILL.md) skill for SDK usage patterns.

## Deviation Lifecycle

See [references/deviation-lifecycle.md](references/deviation-lifecycle.md) for the complete state machine.

### Typical Workflow

1. **Data ingested** — network data arrives from a data source → deviation enters **Queued**
2. **Analysis runs** — system compares against NetBox → deviation moves to **Open** (or No Changes)
3. **Operator reviews** — examines field-level changes in the UI
4. **Action taken**:
   - **Apply** → NetBox updated, deviation marked **Applied**
   - **Ignore** → deviation dismissed, marked **Ignored**
   - **Rediff** → re-analyzed against current NetBox state (useful after manual NetBox edits)

### Resolution Patterns

| Scenario | Recommended Action |
|----------|-------------------|
| Legitimate change detected (e.g., new interface added) | **Apply** — update NetBox |
| Known discrepancy you don't want to track | **Ignore** — dismiss it |
| You manually updated NetBox already | **Rediff** — re-analyze to confirm |
| Previously ignored but situation changed | **Reopen** → then Apply or Rediff |
| Apply failed due to transient error | Fix the issue, then **Rediff** or retry **Apply** |

## UI Workflow

### Navigation

Assurance adds an **Assurance** section to the NetBox navigation with these views:

| View | Purpose |
|------|---------|
| **Deviation Types** | Browse deviation categories with counts |
| **Active Deviations** | Unresolved deviations needing attention |
| **All Deviations** | Complete list including resolved |
| **Archived Deviations** | Historical resolved deviations |

### Reviewing a Deviation

1. Navigate to **Assurance → Active Deviations**
2. Click a deviation to open the detail view
3. **Overview tab** — source, object type, state, timestamps
4. **Changes tab** — field-level before/after comparison
5. Take action: Apply, Ignore, or Rediff

### Bulk Operations

Select multiple deviations from any list view to perform bulk actions:
- **Bulk Rediff** — re-analyze all selected against current NetBox state
- **Bulk Ignore** — dismiss all selected
- **Bulk Apply** — accept all selected changes into NetBox

### Filtering

Deviation lists support filtering by:
- **State** — Open, Applied, Failed, Ignored, etc.
- **Object type** — e.g., devices, interfaces, IP addresses
- **Deviation type** — classification of the drift
- **Data source** — which integration produced it
- **Time range** — when the data was ingested

## Deviation Types

Deviations are classified by type, combining the action needed with the object type:

- **Create** types — object needs to be added to NetBox (inventory drift)
- **Update** types — object exists but attributes differ (attribute drift)

Each deviation type aggregates a count of matching deviations, making the Deviation Types view useful for understanding drift patterns at a glance.

### What Gets Compared

Any entity type flowing through the ingestion pipeline can be compared. Common types:
- Devices (`dcim.device`)
- Interfaces (`dcim.interface`)
- Sites (`dcim.site`)
- Device types (`dcim.devicetype`)
- IP addresses (`ipam.ipaddress`)
- Prefixes (`ipam.prefix`)

The system is extensible — it compares whatever entity types the data sources provide.

## Integration Patterns

See [references/integration-patterns.md](references/integration-patterns.md) for detailed integration guidance.

### With NetBox Discovery

Discovery agents provide the "actual state" side of the comparison:

```
Network Devices → Orb Agent (Discovery) → Assurance → Deviations → NetBox
```

Discovery and Assurance are tightly coupled — Discovery collects, Assurance compares and remediates.

### With Diode SDK

The Diode SDK is the programmatic ingestion path for custom integrations:

```
Custom Script → Diode SDK (Python/Go) → Assurance → Deviations → NetBox
```

### With NetBox Branching

Deviations can be scoped to **NetBox branches**, allowing you to:
- Review drift against a specific branch's state
- Apply changes to a branch rather than the main database
- Use branches as a staging area for deviation resolution

### Day 1 / Day 1.5 / Day 2 Patterns

| Pattern | Use Case |
|---------|----------|
| **Day 1** — Initial population | Run Discovery against a new network, bulk-apply deviations to populate NetBox |
| **Day 1.5** — Reconciliation | Compare existing NetBox data against actual network, fix discrepancies |
| **Day 2** — Ongoing assurance | Continuous monitoring for drift, triage deviations as they appear |

## Permissions

Assurance defines two permission levels:

| Permission | Grants |
|-----------|--------|
| **View Assurance** | Read-only access to deviation views |
| **Add Assurance** | Ability to perform actions (apply, ignore, rediff, reopen) |

Assign these through NetBox's standard user/group permission system.

## Prerequisites

- **NetBox Cloud or NetBox Enterprise** with Assurance enabled (on Enterprise, your license file determines whether Assurance services are installed; on Cloud it's a licensed add-on). The Assurance plugin supports NetBox **4.4.10 through 4.6.x** (current plugin line v1.5.x).
- At least one configured data source
- Network connectivity between data sources and the Assurance service

> Assurance is an optional, licensed add-on for **NetBox Cloud and NetBox Enterprise** — it is not part of open-source NetBox Community.

## Troubleshooting

### No Deviations Appearing

1. Verify data sources are active and sending data
2. Check that Discovery agents are running and collecting (see [netbox-discovery](../netbox-discovery/SKILL.md))
3. Confirm ingestion is working (data should flow through to analysis)
4. Allow time — analysis is asynchronous and may take minutes

### Too Many False Positives

- Review which object types and fields are generating noise
- Use **Ignore** for known acceptable discrepancies
- Consider whether data source configuration needs tuning (e.g., SNMP community strings, API credentials)

### Failed Deviations

- Check the error details in the deviation detail view
- Common causes: permission issues writing to NetBox, validation errors, dependency ordering
- After fixing the root cause, use **Rediff** to re-analyze

### Performance with Large Networks

- Use filtering to narrow deviation views (by state, object type, time range)
- Address deviations incrementally — start with one object type or site
- Bulk operations help process large batches efficiently

## Anti-Patterns and Limitations

1. **Assurance compares what data sources provide** — if a data source doesn't collect a field, Assurance can't detect drift on it
2. **Ordering matters for applies** — some deviations depend on others (e.g., you must create a site before assigning a device to it). The system handles dependency ordering, but complex chains may need manual intervention
3. **Stale inventory detection is limited** — detecting objects that should be removed from NetBox (because they no longer exist on the network) depends on observation timestamps and is an evolving capability
4. **Not a real-time system** — there is inherent latency between network changes, discovery, analysis, and deviation availability
5. **Branch scoping** — when using branches, ensure you're reviewing deviations against the correct branch context

## References

| Document | When to Read |
|----------|-------------|
| [references/deviation-lifecycle.md](references/deviation-lifecycle.md) | Understanding state transitions, resolution patterns |
| [references/data-sources.md](references/data-sources.md) | Setting up what gets compared |
| [references/integration-patterns.md](references/integration-patterns.md) | Connecting Assurance with Discovery, Diode, and the NetBox Labs stack |

## Related Skills

- [netbox-discovery](../netbox-discovery/SKILL.md) — Orb Agent configuration for network scanning
- [netbox-diode](../netbox-diode/SKILL.md) — Diode SDK for programmatic data ingestion
- [netbox-branching](../netbox-branching/SKILL.md) — Branch-aware workflows
