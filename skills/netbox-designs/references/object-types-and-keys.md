# Object Types, Keys, References and Matching

Source: [Object types](https://netboxlabs.com/docs/designs/reference/object-types.md) and [Design structure](https://netboxlabs.com/docs/designs/reference/design-structure.md). A design can build **any object type in the data model of the target's NetBox version**; this page lists the common ones. The render refuses a type written in snake case (`device_type`) or with a `NetBox` prefix (`NetBoxDeviceType`) and says what to write instead. Fields use NetBox's REST API names.

## Common types and their keys

A plan matches each object in the design with the NetBox object that has the same **key**. Changing a key field describes another object.

| `infra_type` | Key |
|--------------|-----|
| `Region` | `name` |
| `SiteGroup` | `name`, `parent` |
| `Site` | `slug` |
| `Location` | `name`, `site`, `parent` |
| `Tenant` | `slug` |
| `Manufacturer` | `slug` |
| `RackType` | `manufacturer`, `model` |
| `Rack` | `name`, `site` |
| `DeviceRole` | `slug` |
| `Platform` | `name`, `manufacturer` |
| `DeviceType` | `manufacturer`, `model` |
| `Device` | `name`, `site` |
| `ModuleType` | `manufacturer`, `model` |
| `ModuleBay` | `name`, `device` |
| `Module` | `module_bay` |
| `Interface` | `name`, `device` |
| `ConsolePort` | `name`, `device` |
| `PowerPort` | `name`, `device` |
| `FrontPort` | `name`, `device` |
| `RearPort` | `name`, `device` |
| `Cable` | `termination_a`, `termination_b` |
| `VRF` | `rd` |
| `Prefix` | `prefix`, `vrf` |
| `IPAddress` | `address`, `vrf` |
| `VLANGroup` | `name` |
| `VLAN` | `vid`, `group` |
| `RIR` | `slug` |
| `ASN` | `asn` |
| `Tag` | `slug` |
| `Cluster` | `name`, `group` |
| `VirtualMachine` | `name` |

## Matching rules

- **A key field NetBox requires can be left out.** `device: {name: SW-01}` matches the one SW-01 whatever its site. If two sites each have an SW-01 the plan is refused, naming both — add the site.
- **A key field NetBox allows to be empty means empty when left out.** A prefix without `vrf` is the global prefix; a VLAN without `group` is the ungrouped one.
- **A device needs a name.** Only its id tells one unnamed device from another, so the render refuses a design that declares one.
- When a design describes an object NetBox already has, the plan **adopts** it and changes only the fields the design declares. Adopted objects are never deleted by the deployment.

## Nesting

An object can hold other objects in a list under a key that is **not one of its fields**: a region's `regions`, a site's `racks`, a rack's `devices` and `cables`, a device's `modules`. Each item is an object of its own; inside it `{{ parent }}` is the object that holds it.

```yaml
- infra_type: Site
  name: DC1
  slug: dc1
  status: active
  racks:
  - infra_type: Rack
    name: RACK-01
    site: '{{ parent }}'
    status: active
```

## Reference forms

A field that refers to another object accepts:

| Form | Example | Resolved |
|------|---------|----------|
| Text, matched against key → name → slug | `site: dc1`, `region: Latvia` | At plan time when the key is one field; at apply time when the key has several fields (e.g. device `name` + `site`) |
| The containing object or a field of it | `site: '{{ parent }}'`, `site: '{{ parent.site }}'` | At render |
| A mapping of fields (needn't be the key) | `device: {name: SW-01}`, `group: {slug: branch-offices}` | At plan time; must match exactly one object in NetBox or the design |
| A function | `"{{ nbd.lookup.endpoint('SW-01', 'Interface', 'eth0') }}"` | At render |

Text or a mapping that matches nothing, or more than one object, **refuses the plan**.

**List-of-reference fields** (`tags`, a VLAN group's `vid_ranges`) take a list of any of these and stay fields rather than becoming children. An item written with an `infra_type` is created by the design and referred to by the field:

```yaml
tags:
- acme-apj
- {slug: branch-office}
- {infra_type: Tag, name: Dunedin, slug: dunedin}
```

### Fields that hold one of several types

An address's `interface` (interface, VM interface, FHRP group), a circuit termination's `termination` (site, location, region, site group, provider network) and a cable's `termination_a`/`termination_b` (any port type) accept several types. The plan infers the type from the mapping's fields and from what exists. When a mapping could mean two types, the plan is refused, naming both — disambiguate with `_type`:

```yaml
- infra_type: CircuitTermination
  circuit: {cid: ACME-HKG-01}
  term_side: A
  termination: {slug: acme-apj-hong-kong, _type: Site}
```

A **cable** ends on an interface, front port, rear port, console port, console server port, power port, power outlet, power feed or circuit termination. Its ends are its identity; write each as a function or a mapping:

```yaml
- infra_type: Cable
  status: connected
  termination_a: "{{ nbd.lookup.endpoint('srv-01', 'PowerPort', 'psu1') }}"
  termination_b: {name: feed-a, power_panel: {name: panel-1, site: {slug: dc1}}}
```

## Fields and empty values

A design owns only the fields it writes.

| You write | It means |
|-----------|----------|
| `description: "Uplink"` | Set the field |
| `description: ""` | Clear a text field |
| `region:` | Null — clears a field NetBox allows to be null, such as a reference |
| The field left out | Leave the field alone (never drift, never written, even if NetBox has a default) |
| `serial: '{{ variables.serial }}'` with `serial: $USER_OPTIONAL` not given | Left out, as if the line weren't there |

### Values refused at render

The render refuses a value NetBox wouldn't keep as written, before a plan exists, and says what to write instead. Rules come from the target version's data model:

- **A null NetBox refuses** in most text fields (`description:` with nothing after it) — write `""` or remove the line.
- **A blank NetBox refuses**, such as `name: ""`.
- **A field NetBox drops without another set** — an interface's `untagged_vlan` without `mode`; a rack's `outer_unit` without an outer dimension.
- **A value NetBox stores in another form** — trimmed text, lower-case `dns_name`, empty VRF `rd` or asset tag stored as null, MAC addresses as upper-case colon pairs, `comments: |` (trailing newline) → use `|-`.
- **A value outside a bound** — names over 100 characters (most fields), `vid` > 4094, device `position` > 100.5, a slug with a space, power feed `voltage` of 0. The message gives the bound.

Types from the Device Type Library are never refused this way: their values arrive as NetBox would store them.

## `netbox_versions`

| Declaration | Matches |
|-------------|---------|
| `"*"` | Any NetBox version |
| `["4.7.*"]` | Any 4.7 release |
| `["4.6.*", "4.7.*"]` | Any 4.6 or 4.7 release |
| `[">=4.6,<4.8"]` | 4.6.0 up to, but not including, 4.8.0 |
| `["4.7.0"]` | Exactly 4.7.0 |

Designs refuses to plan a design against a NetBox whose version it doesn't list, and the catalogue refuses a design without the key.
