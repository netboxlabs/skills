---
name: netbox-designs
description: >
  Write and review NetBox Designs — YAML files that NetBox Designs renders, plans
  against a NetBox and applies to build infrastructure (regions, sites, racks,
  devices, modules, cabling, IPAM). Use when asked to write a design, to model
  infrastructure as a reusable design, to compose designs from the catalogue or the
  Device Type Library, to fix a design a plan refused, or to read a Designs plan.
license: Apache-2.0
---

# NetBox Designs

> **Your knowledge of NetBox Designs may be outdated.** Designs is in **preview on NetBox Cloud** and its format, functions and limitations change between releases. Every docs page is also served as markdown — add `.md` to the page address — so read the reference before writing a design. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Design structure | `https://netboxlabs.com/docs/designs/reference/design-structure.md` | Top-level keys, values and empty fields, references, matching |
| Object types & keys | `https://netboxlabs.com/docs/designs/reference/object-types.md` | `infra_type` names and the key each type is matched by |
| Variables | `https://netboxlabs.com/docs/designs/reference/variables.md` | `$USER`, `$USER_OPTIONAL`, defaults, computed values, `template_vars` |
| Expressions & loops | `https://netboxlabs.com/docs/designs/reference/expressions.md` | `{{ }}` syntax, filters, `nbd_loop` |
| Functions | `https://netboxlabs.com/docs/designs/reference/functions.md` | `nbd.lookup.*`, `nbd.nbt.*`, `nbd.dtl.*` |
| Composition | `https://netboxlabs.com/docs/designs/reference/composition.md` | `nbd:` and `dtl:` references, pinning |
| Limitations | `https://netboxlabs.com/docs/designs/reference/limitations.md` | What Designs doesn't do yet |
| Design JSON Schema | `https://netboxlabs.com/docs/designs/reference/design.schema.json` | Structural validation in editors and tools |
| Read a plan | `https://netboxlabs.com/docs/designs/how-to/read-a-plan.md` | Every warning and refusal a plan can produce |
| Device Type Library | `https://github.com/netbox-community/devicetype-library` | Paths and commits for `dtl:` references |
| Example designs | `https://netboxlabs.com/docs/designs/tutorial/the-compute-rack.md` | Canonical rack, device and site designs |

## FIRST: Know the Target

A design is planned against a specific NetBox by NetBox Designs (a NetBox Cloud feature). Before writing one, establish:

1. **The target's NetBox version** — it sets `netbox_versions` and which `infra_type` names and fields exist. Designs requires **NetBox 4.6.0+**.
   ```bash
   curl -s -H "Authorization: Bearer $NETBOX_TOKEN" "$NETBOX_URL/api/status/" | python -c "import sys,json; print(json.load(sys.stdin)['netbox-version'])"
   ```
2. **What the catalogue already has** — reuse device, rack and site designs with `nbd:` rather than redefining them. Ask the user for the catalogue names, or list them in the Designs UI (**Design Catalogue**) or with `nbd catalogue versions <name>`.
3. **Which Device Type Library files to pin** — find the path in the library and a commit or tag to pin with `@`.
4. **Whether to plan into a branch** — the safe default against a NetBox that matters; needs the NetBox Branching plugin. See [netbox-branching](../netbox-branching/SKILL.md).

If the `nbd` CLI is available, check a file offline before adding it: `nbd validate <file> --netbox-version 4.7.0`.

---

## Scope

This skill covers **writing, composing and reviewing designs**, and reading the plans they produce. It does NOT cover:
- Deciding how to model your network (hierarchy, IPAM, naming) → [netbox-data-modeling](../netbox-data-modeling/SKILL.md)
- Branch lifecycle and merging → [netbox-branching](../netbox-branching/SKILL.md)
- Device type metadata and lifecycle data → [netbox-ndx](../netbox-ndx/SKILL.md)
- Writing to NetBox directly via REST/pynetbox → [netbox-api-integration](../netbox-api-integration/SKILL.md)

## How Designs Works

Every change goes through **render → plan → review → apply**. Designs fills in variables and includes, computes every object, reads the target NetBox, and produces a plan of creates, updates and deletes. Nothing is written until the plan is applied, and an applied design plans to nothing until the design or NetBox changes.

A **deployment** is one named instance of a design on one NetBox (like a Helm release). It records which objects it **created** and which it **adopted**. It deletes only what it created, never what it adopted or what another deployment still describes. One design can be deployed many times with different variables.

## Quick Reference

### Minimal design

```yaml
# yaml-language-server: $schema=https://netboxlabs.com/docs/designs/reference/design.schema.json
name: Site Foundation
description: A site in a country, with a /16 supernet for its subnets.
netbox_versions: ["4.6.*", "4.7.*"]
variables:
  site_name: $USER
  site_slug: '{{ variables.site_name | slugify }}'
  region: $USER
  supernet: $USER
  status: active
infra:
- infra_type: Site
  name: '{{ variables.site_name }}'
  slug: '{{ variables.site_slug }}'
  status: '{{ variables.status }}'
  region: '{{ variables.region }}'
- infra_type: Prefix
  prefix: '{{ variables.supernet }}'
  status: container
  site: '{{ variables.site_slug }}'
  description: 'Site supernet for {{ variables.site_name }}'
```

### Top-level keys

| Key | Required | Holds |
|-----|----------|-------|
| `name` | Yes | The design's title. Other designs refer to it by its **catalogue name**, not this |
| `description` | No | What the design builds |
| `version` | No | A label for this content, such as `1.2.0` |
| `netbox_versions` | **Yes** | `"*"` or a list of specifiers: `["4.7.*"]`, `[">=4.6,<4.8"]` |
| `variables` | Yes | Inputs and computed values; `{}` for none |
| `infra` | Yes | The objects the design builds |

### Variables

| Value | Means |
|-------|-------|
| `$USER` | Required; the deploy page asks for it |
| `$USER_OPTIONAL` | Optional; every field that uses it is left out when not given |
| `active`, `42` | A default |
| `'{{ variables.site_name \| slugify }}'` | Computed at render from other variables |

### Expressions, filters, loops

| Pattern | Example |
|---------|---------|
| Variable | `'{{ variables.rack_name }}'` |
| Containing object / its field | `'{{ parent }}'`, `'{{ parent.site }}'` |
| Loop variable, arithmetic | `'{{ loop.i }}'`, `'{{ loop.i * 2 - 1 }}'` |
| Filter (one per expression) | `zeropad`, `zeropad(N)`, `lower`, `upper`, `slugify` |
| Function (double-quote the outer string) | `"{{ nbd.lookup.endpoint('SW-01', 'Interface', 'eth0') }}"` |
| Loop | `nbd_loop: {over: {start: 1, end: 10}, as: i}` beside the object, `design:`, or an `items:` list |

### Functions

| Function | Finds |
|----------|-------|
| `nbd.lookup.endpoint(device, kind, name)` | A port on a device — `Interface`, `ConsolePort`, `ConsoleServerPort`, `PowerPort`, `PowerOutlet`, `FrontPort`, `RearPort` |
| `nbd.lookup.mgmt_interface(device)` | The device's management-only interface |
| `nbd.lookup.object(type, field=value, …)` | Any object by type and field values |
| `nbd.nbt.get_module_bay(device, position)` | A module bay created by the device type |
| `nbd.dtl.device_type(path)` / `nbd.dtl.module_type(path)` | A Device Type Library type — usually written as a `dtl:` reference instead |

Functions look in the design first, then in the target NetBox. **None of them allocate.**

### Composition

| Scheme | Includes | Example |
|--------|----------|---------|
| `nbd:` | A catalogue design, in place of an object | `design: nbd:devices/Dell_R640_w_module` + `template_vars:` |
| `nbd:<name>@<version>` | A pinned version (label or content hash) | `nbd:devices/Dell_R640_w_module@1.0.0` |
| `dtl:` | A Device Type Library file, **pin required** | `device_type: 'dtl:device-types/Dell/PowerEdge-R640.yaml@fdf12828'` |

See [references/object-types-and-keys.md](references/object-types-and-keys.md) for every common type's key and the rules for references, empty values and matching.

## Authoring Rules

These are the rules that are easy to get wrong. Follow them every time.

| ID | Rule |
|----|------|
| DSN-1 | Every design has `name`, `netbox_versions`, `variables` and `infra`. **`netbox_versions` is required** — the catalogue refuses a design without it, and a plan is refused against a version it doesn't list |
| DSN-2 | `infra_type` is the type's name in NetBox's data model: `DeviceType`, `IPAddress`, `VRF` — not `device_type` or `NetBoxDeviceType`. `nbd.lookup.endpoint` and `nbd.lookup.object` take the same names. Fields keep NetBox's API names: `device_type`, `primary_ip4` |
| DSN-3 | Write only the fields the user asks for. A field left out is left alone in NetBox (never drift, never written); a field the design writes is the design's to keep in step |
| DSN-4 | Each object is matched with NetBox's by its **key** (see object types). Changing a key field describes a *different* object — a plan shows it as delete + create |
| DSN-5 | Write values as NetBox stores them: no leading/trailing spaces, MAC addresses as upper-case pairs joined by colons, `\|-` rather than `\|` for multi-line text, lower-case `dns_name`, slugs without spaces, `vid` ≤ 4094 |
| DSN-6 | Refer to another object by its key, name or slug as text (`region: Latvia`), by `'{{ parent }}'` when it holds this one, by a mapping of fields (`device: {name: SW-01}`), or with a function. Add `_type: Site` when a mapping could match two types |
| DSN-7 | Prefer small designs composed with `nbd:`, and device, module and rack types from the Device Type Library with `dtl:` — **always pinned** with `@` and a commit or tag |
| DSN-8 | Designs don't allocate. Write prefixes, addresses and ASNs explicitly; no function picks the next free one |
| DSN-9 | A loop's bounds can't be variables. A design that loops over the same count in several places repeats the count in each |
| DSN-10 | Quote strings that hold expressions. When a function's arguments need single quotes, put the whole string in double quotes |
| DSN-11 | A device needs a `name`. An object's text key field (`name`, `slug`) must be present; `name: ""` is refused |
| DSN-12 | To clear a text field write `""`; an empty YAML value (`region:`) is null and only clears fields NetBox allows to be null |
| DSN-13 | Nest children under a non-field key (`regions`, `racks`, `devices`, `modules`, `cables`) and point them back with `parent: '{{ parent }}'` or `site: '{{ parent.site }}'` |

## Workflow

1. **Start from a working example** — the tutorial's region hierarchy (nesting), site foundation (variables + references) and compute rack (composition, loops, functions).
2. **Write the smallest design that builds something**, such as one device. Validate it (`nbd validate`, or the JSON Schema in your editor).
3. **Add it to the catalogue.** Designs that others include must be added **before** the designs that include them; the catalogue refuses a design whose `nbd:` parts are missing and names them. The catalogue name (e.g. `devices/Dell_R640_w_module`) is what `nbd:` matches — it must be exact.
4. **Plan against a non-production NetBox or into a branch.** Read the plan: counts, grouping, what each delete takes with it. A refusal says what to change — see [references/plan-review.md](references/plan-review.md).
5. **Grow it a piece at a time**, planning after each step. When a part repeats, make it a design of its own and include it with `nbd:`.
6. **Apply, then plan again.** A correct design plans to nothing once applied. Use **Check drift** on the deployment later to see whether NetBox still matches.

### When you finish a design, report

- The **catalogue names** the design includes with `nbd:`, in the order they must be added to the catalogue.
- The **variables** the user must give when they plan (`$USER`), and the optional ones.
- The **NetBox versions** it declares, and any Device Type Library pins it relies on.

## Canonical Example: a rack composed from device designs

```yaml
name: Compute Rack
netbox_versions: ["4.6.*", "4.7.*"]
description: A 42U compute rack in an existing site, with a top-of-rack switch, ten servers and their cabling.
variables:
  rack_name: $USER
  site_slug: $USER
  status: active
infra:
- infra_type: Rack
  name: '{{ variables.rack_name }}'
  site: '{{ variables.site_slug }}'
  rack_type: dtl:rack-types/APC/AR3350.yaml@fdf12828
  status: '{{ variables.status }}'
  devices:
  - design: nbd:devices/Cisco_3850_w_module          # added to the catalogue first
    template_vars:
      name: '{{ variables.rack_name }}-TOR-01'
      slug: '{{ variables.rack_name }}-tor-01'
      position: 42
      face: front
      status: active
      site: '{{ parent.site }}'
      rack: '{{ parent }}'
  - nbd_loop:
      over: {start: 1, end: 10}
      as: i
    design: nbd:devices/Dell_R640_w_module
    template_vars:
      name: '{{ variables.rack_name }}-COMPUTE-{{ loop.i | zeropad }}'
      slug: '{{ variables.rack_name }}-compute-{{ loop.i | zeropad }}'
      position: '{{ loop.i }}'
      face: front
      status: active
      site: '{{ parent.site }}'
      rack: '{{ parent }}'
  cables:
  - nbd_loop:
      over: {start: 1, end: 10}
      as: i
    items:
    - infra_type: Cable
      label: '{{ variables.rack_name }}-COMPUTE-{{ loop.i | zeropad }}-DATA-A'
      type: cat6a
      status: connected
      termination_a: "{{ nbd.lookup.endpoint('{{ variables.rack_name }}-COMPUTE-{{ loop.i | zeropad }}', 'Interface', 'Ethernet/PCIE1/1') }}"
      termination_b: "{{ nbd.lookup.endpoint('{{ variables.rack_name }}-TOR-01', 'Interface', 'GigabitEthernet1/0/{{ loop.i * 2 - 1 }}') }}"
```

Report for this design: add `devices/Cisco_3850_w_module` and `devices/Dell_R640_w_module` to the catalogue first; the user supplies `rack_name` and `site_slug`; the site must already exist (created by another design or in NetBox). The device designs each declare `name`, `slug`, `status`, `site` as `$USER` and `position`, `face`, `serial`, `rack` as `$USER_OPTIONAL`, and bring their device type and module type in from the Device Type Library — see [references/expressions-and-functions.md](references/expressions-and-functions.md) for the full device design.

## Anti-Patterns

| Anti-Pattern | Why It Matters | Fix |
|-------------|---------------|-----|
| `infra_type: device_type` or `NetBoxDeviceType` | Refused at render | Use the data-model name: `DeviceType` |
| Omitting `netbox_versions` | Catalogue refuses the design | Declare the versions the design is tested against |
| `dtl:` reference without `@<commit>` | Refused; the design would change as the library changes | Pin every `dtl:` path to a commit or tag |
| Writing every field "to be safe" | Each written field becomes the design's to own; hand edits in NetBox then show as drift and are overwritten | Write only what the user asked for |
| Renaming a device or moving an interface in a design | The key changed, so the plan deletes and recreates it, taking cables and IPs with it | Keep key fields stable; read the move line in the plan before applying |
| Expecting a function to allocate the next free prefix or IP | No function allocates | Write the address explicitly, or compute it from variables |
| `over: {start: 1, end: '{{ variables.count }}'}` | Loop bounds can't be variables | Hard-code the count, repeated where needed |
| Two filters in one expression (`\| lower \| slugify`) | Refused at render | One filter per expression; chain through a computed variable |
| Setting a custom field value | Silently not written during preview | Leave custom fields out; set them outside Designs |
| Describing plugin or custom-object models | Not in NetBox's own data model; refused | Design core objects only |
| Depending on a Device Type Library power outlet, front/rear port, or module component other than interfaces | Not brought in yet | Declare the port under the device so a cable has an end |
| Adding the rack design before the device designs it includes | Catalogue refuses, naming the missing parts | Add included designs first |
| `comments: \|` | Trailing newline NetBox strips → refused | Use `\|-` |
| Two deployments writing the same field with different values | Each apply flips the value back | Keep each field in one deployment |

## Version Notes

### Designs preview (NetBox Cloud, 2026)
- Requires **NetBox 4.6.0+** on the target, a **v2 `nbt_` token with write enabled** whose user can view the changelog and manage the object types the designs describe; v1 tokens are refused.
- Planning into a branch needs the NetBox Branching plugin and its permissions. On **NetBox 4.7.0–4.7.2** a write NetBox rejects inside a branch surfaces as a server error (netbox-community/netbox#23366, fixed in **4.7.3**); Designs retries one object at a time so the apply reports the real field error.
- Not yet supported: custom fields, custom objects, plugin models, typed variables, variable loop bounds, allocation, renaming designs or deployments, NDX device types. Planned for general availability: typed variables, NDX types, keeping an object through a move.
- Every design declares `netbox_versions`; the tutorial designs declare `["4.6.*", "4.7.*"]`.

## References

- [references/object-types-and-keys.md](references/object-types-and-keys.md) — Every common `infra_type` and its key; reference forms; empty values; how objects are matched. Load when writing any object or when a plan says a reference matches nothing or more than one object.
- [references/expressions-and-functions.md](references/expressions-and-functions.md) — Filters, loop variables, every function's parameters, `template_vars` rules, and a complete device design. Load when writing loops, cables or includes.
- [references/plan-review.md](references/plan-review.md) — What a plan shows, every warning and refusal, deployment semantics (created vs adopted, release, tear down), drift, branches, and a busy NetBox. Load when a plan is refused or before applying deletes.
