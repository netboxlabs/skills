# Integration Patterns

How Assurance fits within the NetBox Labs product stack.

## The Full Stack

```
┌─────────────────────────────────────────────────────┐
│                   DATA SOURCES                       │
│  Discovery Agents  │  Controller Integrations  │ SDK │
└────────┬───────────┴──────────┬────────────────┴──┬──┘
         │                      │                   │
         ▼                      ▼                   ▼
┌─────────────────────────────────────────────────────┐
│              INGESTION PIPELINE (Diode)              │
│         Receives, normalizes, and stores data        │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│               ASSURANCE (Analysis)                   │
│     Compares actual state vs NetBox intended state   │
│              Produces deviations                     │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              ASSURANCE UI (in NetBox)                 │
│       Review deviations, apply/ignore/rediff         │
└──────────────────────┬──────────────────────────────┘
                       │ Apply
                       ▼
┌─────────────────────────────────────────────────────┐
│                     NETBOX                           │
│              Source of truth updated                  │
└─────────────────────────────────────────────────────┘
```

## Discovery + Assurance

Discovery and Assurance are complementary:

| Component | Role |
|-----------|------|
| **Discovery** (Orb Agent) | Collects actual network state |
| **Assurance** | Compares collected state against NetBox, surfaces deviations |

**Typical setup:**
1. Deploy Discovery agents (see [netbox-discovery](../../netbox-discovery/SKILL.md))
2. Agents scan network on configured schedules
3. Collected data flows automatically into Assurance
4. Deviations appear in the Assurance UI for review

**Key point:** Discovery runs independently of Assurance. You can use Discovery without Assurance (data goes to NetBox via Diode), but Assurance without Discovery requires an alternative data source.

## Diode SDK + Assurance

The [Diode SDK](../../netbox-diode/SKILL.md) is the programmatic interface for pushing data into the ingestion pipeline. When Assurance is licensed, SDK-ingested data goes through the comparison engine rather than directly into NetBox.

**Without Assurance:** Diode SDK → ingestion → NetBox (auto-applied)
**With Assurance:** Diode SDK → ingestion → comparison → deviations → operator review → NetBox

This means enabling Assurance adds a human review step to what was previously an automated pipeline. Plan your workflows accordingly.

## NetBox Branching + Assurance

Assurance integrates with [NetBox Branching](../../netbox-branching/SKILL.md):

- **Deviations can target branches** — apply changes to a branch instead of main
- **Branch-scoped review** — filter deviations by branch context
- **Staged remediation** — use a branch as a staging area, review all changes, then merge

**Workflow:**
1. Create a branch for remediation work
2. Apply deviations to the branch
3. Review the branch diff in NetBox
4. Merge the branch when satisfied

## Operational Patterns

### Day 1: Initial Network Population

**Goal:** Populate an empty NetBox from actual network state.

1. Deploy Discovery agents across the network
2. Wait for initial scan to complete
3. Review deviations in Assurance (all will be "create" type)
4. Bulk apply by object type: sites → device types → devices → interfaces → IPs
5. Validate NetBox reflects the network accurately

### Day 1.5: Reconciliation

**Goal:** Align an existing NetBox instance with actual network state.

1. Enable Assurance against an already-populated NetBox
2. Review mix of create (missing from NetBox) and update (attribute differences) deviations
3. Triage: apply legitimate updates, ignore acceptable differences
4. Iterate until deviation count stabilizes

### Day 2: Continuous Assurance

**Goal:** Ongoing drift detection and remediation.

1. Discovery agents run on schedule (e.g., every 4-8 hours)
2. New deviations surface as network changes occur
3. Operations team reviews deviations daily or per-shift
4. Integrate deviation review into change management workflows

### Multi-Source Correlation

When running multiple data sources (e.g., Discovery + vCenter integration):
- Each source independently reports what it observes
- Deviations track their source for attribution
- Same object may have deviations from different sources
- Use source filtering in the UI to review by integration

## Capacity Considerations

| Factor | Impact |
|--------|--------|
| Number of devices | More devices = more deviations to review |
| Scan frequency | More frequent scans = faster drift detection, more analysis load |
| Number of data sources | Multiple sources increase coverage but also deviation volume |
| Object type scope | Broader collection = more comprehensive but noisier |

**Recommendation:** Start narrow (one site, one object type) and expand as your team builds comfort with the triage workflow.
