# protect_main

## Overview

When enabled, `protect_main` blocks all direct writes to branch-aware models
unless a branch context is active. This forces all changes through the
branching + change request workflow.

## Configuration

**Disabled by default.** Enable in NetBox config:

```python
PLUGINS_CONFIG = {
    'netbox_changes': {
        'protect_main': True,
    }
}
```

## How It Works

When protect_main is enabled, every save or delete operation is checked:

1. If the model is not branch-aware, the write is allowed
2. If the user has the **bypass** action on the Policy object, the write is allowed
3. If an active branch context exists (`X-NetBox-Branch` header), the write is allowed
4. Otherwise, the write is **blocked** with: `"Changes directly to main are not permitted."`

## Bypass Permission

**v1.0+:** grant the **`bypass`** custom action on the **Policy** object (NetBox
Change Management → Policy in the ObjectPermission form). Superusers get it
implicitly. Internally the codename is `bypass` while the enforcement check still
references `bypass_policy` — describe it to users as "the bypass action on Policy."

Users/roles with this permission can make direct changes to main even when
protect_main is enabled. This is the **only custom permission** added by the
plugin (all other access uses standard NetBox model permissions).

## Merge and Revert Safety

During branch merge/revert operations, protect_main is temporarily suspended
to allow the merge writes to proceed normally.

## Combined with Merge Gating

When both protect_main and merge gating are active (merge gating is always on),
the full enforcement chain is:

1. **protect_main** → all changes must go through a branch
2. **Merge gating** → branch can only merge with an approved CR
3. **Policy system** → CR approval requires reviews per policy rules

This creates: changes → branch → CR → review → approve → merge.
