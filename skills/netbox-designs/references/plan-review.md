# Reading Plans, Deployments, Drift and Branches

Source: [Read a plan](https://netboxlabs.com/docs/designs/how-to/read-a-plan.md), [Deployments](https://netboxlabs.com/docs/designs/concepts/deployments.md), [Check for drift](https://netboxlabs.com/docs/designs/how-to/check-for-drift.md), [Plan into a branch](https://netboxlabs.com/docs/designs/how-to/plan-into-a-branch.md), [When a NetBox is busy](https://netboxlabs.com/docs/designs/how-to/when-a-netbox-is-busy.md).

## The review

- **Deployment** — which deployment the plan is for and whether it is new. A new deployment is recorded when the plan is applied.
- **Create / Update / Delete** — counts.
- **Changes** — grouped by where each object sits (a rack's devices under the rack, cables by the devices they connect); **Flat** for a plain list. Open an update to see each changed field's current and new value. Reference fields show the target's name as the plan knows it.

A plan is computed against NetBox **as it was when you planned**. If NetBox changes before you apply, or a newer plan in the same session replaces it, applying is refused and you plan again.

## Warnings (don't block apply)

| Warning | Meaning | Do |
|---------|---------|----|
| This plan starts a new deployment of a design that already has deployments | You may have meant to change an existing one | Use **Redeploy** on that deployment instead |
| Released, not deleted | An object this deployment created and no longer describes, but another deployment still does | Nothing — it stays, owned by the other deployment |
| Not one of the known values | A choice field set outside its known choices | NetBox rejects at apply unless that value was added to the field's choices |
| Another deployment also declares a field with a different value | Each apply flips the value | Keep the field in one deployment, or make them agree |

## What a delete takes with it

NetBox cascades some deletes (a device's interfaces and modules, a site's locations and scoped prefixes) and clears others (a deleted VLAN is removed from every interface). A cable that loses an end stays, unterminated. The plan lists each effect under the delete that causes it, with its owner. When deletes reach objects the plan doesn't list as its own changes, applying requires you to **accept them** (the button becomes **Apply and also delete**); who accepted and when is recorded on the plan.

A cable the design wants on a port another cable still holds (often one left half-connected after a device was deleted outside Designs) is listed the same way; accepting deletes that cable first.

## Refused before review

| Refused because | What to do |
|-----------------|------------|
| The design doesn't support the NetBox's version | Add the version to `netbox_versions` once the design works with it |
| A value NetBox wouldn't keep as written (blank name, text it would trim, MAC in another form) | Write the value as the message says NetBox stores it |
| A value outside a bound (name too long, `vid` > 4094, slug with a space) | Change the value to fit the bound, or remove the line |
| A reference names no object, or more than one | Name one that exists (in NetBox or the design) by name, slug or key; add `_type` or more key fields |
| A delete NetBox would refuse (a site that still has racks) | Delete or move what's in the way first, or keep the object in the design |
| An expression the render can't work out | Fix the expression the message names |
| An object type in the old form (`device_role`, `NetBoxDeviceRole`) | Write the name the message gives (`DeviceRole`) |

## Moves are delete + create

Designs identifies an object by its key. Rename a device or move an interface to another device and the plan deletes the object and creates another, pairing them as a **move** and listing what the delete takes with it (cables, IP addresses the design doesn't declare again). Read that line before applying. Keeping an object through a move is planned.

## Deployments

A deployment is one named instance of a design on one NetBox. It records what it **created** and what it **adopted**.

- **It deletes only what it created.** When the design stops describing a created object, the next plan deletes it.
- **It never deletes what it adopted.**
- **It never deletes what another deployment still describes** — that object is **released** instead and stays.
- A deployment can hold several designs (a site and its racks) managed under one name; each design has its own variables, sessions and plans.

| Action | Does |
|--------|------|
| **Redeploy** | Opens the deploy page with the variables and design version it applied; change and plan. **Plan the newer version** moves it to the catalogue's current version — it never moves on its own |
| **Check drift** | Plans again in the background against the version it applied; **In sync** or **Drifted** with a reviewable plan to put NetBox back. **Check drift of all** for multi-design deployments |
| **Release** | Stops managing; objects stay in NetBox, owned by nothing |
| **Tear down** | Plans deletion of what it created; you review and apply like any plan. Adopted and shared objects stay. **Tear down all** orders a multi-design deployment's deletes so NetBox accepts them |
| **Remove** | Forgets a released or torn-down deployment; nothing in NetBox changes |

Only a session's newest plan can be applied. **Replan** re-plans an open session; **Discard plan** closes it without changes.

### When a design changes or is deleted

Each deployment records the design version it applied. After an edit, the Deployments page says the design changed; nothing happens until you Redeploy and choose the newer version. Deleting a design leaves its deployments and everything they own (the delete must be acknowledged: `--force` on the CLI, `acknowledge_deployments=true` on the API). A version an active deployment applied cannot be removed from the catalogue.

## Drift

Drift is NetBox changing under a deployment: a field the design sets edited by hand, or a created object deleted. Only declared fields count — a field the design doesn't set can change freely and is never reported. An edited design is **not** drift (the page says the design changed instead).

## Branches

A plan can target a NetBox branch (needs the NetBox Branching plugin and its permissions on the token's user). Enter a branch name on the deploy page; Designs creates it if NetBox lacks it. The plan is computed against the branch and the apply writes to it; review and merge in NetBox.

- A deployment planned into a branch belongs to that branch, separate from main's deployments, until the branch ends. Designs checks the branch's state when you open the Deployments page or use the branch.
- **Merged**: the deployment moves to main with everything it owns (NetBox keeps ids across a merge). If main already has an active deployment of the same name and design, the branch's is released instead.
- **Deleted**: its objects went with the branch; the deployment shows **Branch gone** and can be removed.
- On **NetBox 4.7.0–4.7.2** a write NetBox rejects inside a branch returns a server error instead of a field error (netbox-community/netbox#23366). Designs retries the objects one at a time so the apply names the object and field refused.

See [netbox-branching](../../netbox-branching/SKILL.md) for branch lifecycle and merging.

## A busy NetBox

Designs keeps its own copy of what it manages in each NetBox and plans against that copy. One operation at a time works on a NetBox: an apply, the read-back after it, or a refresh. Anything else gets a busy response saying what is running and how far along (`GET /api/v1/services/{name}/activity`). The first plan against a NetBox reads all of it and can take a minute.

If the read-back after an apply fails, the NetBox is marked possibly behind and plans say so in their notes. Refresh everything rather than replaying the changelog:

```bash
nbd service refresh netbox-prod --full      # or POST /api/v1/services/{name}/refresh?full=true
```

## CLI touchpoints seen in the docs

| Command | Does |
|---------|------|
| `nbd validate <file> [--netbox-version 4.7.0]` | Offline check of structure, variables, functions and (given a version) object type names |
| `nbd catalogue push --update` | Save a file as a new version of a design |
| `nbd catalogue versions <name>` | List a design's versions |
| `nbd catalogue remove <name> [--force]` | Delete a design; `--force` acknowledges deployments that stay |
| `nbd service refresh <name> --full` | Re-read everything from a NetBox |
