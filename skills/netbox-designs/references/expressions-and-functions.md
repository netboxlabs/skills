# Expressions, Loops, Functions and Composition

Source: [Expressions and loops](https://netboxlabs.com/docs/designs/reference/expressions.md), [Functions](https://netboxlabs.com/docs/designs/reference/functions.md), [Variables](https://netboxlabs.com/docs/designs/reference/variables.md), [Composition](https://netboxlabs.com/docs/designs/reference/composition.md).

## Expressions

An expression is `{{ }}` inside a YAML string and can be a whole value or part of one. **Always quote the string.**

| Pattern | Example | Gives |
|---------|---------|-------|
| A variable | `'{{ variables.site_name }}'` | The variable's value |
| The containing object | `'{{ parent }}'` | The object that holds this one |
| A field of it | `'{{ parent.site }}'` | That object's `site` |
| The loop's variable | `'{{ loop.i }}'` | The current pass |
| A filter | `'{{ loop.i \| zeropad }}'` | `01`, `02`, … |
| Arithmetic | `'{{ loop.i * 2 - 1 }}'` | A number per pass |
| A function | `"{{ nbd.lookup.endpoint('SW-01', 'Interface', 'eth0') }}"` | The object found |
| Part of a value | `'{{ variables.rack_name }}-TOR-01'` | Text with the value in it |

A function's arguments can hold expressions of their own, worked out first. Write the outer string in **double quotes** so the arguments can use single quotes.

Braces that aren't one of these expressions are plain text and reach NetBox as written — a config template's own `{{ device.name }}` is text the design means.

### Filters

One filter per expression; a second filter or an unknown one refuses the render.

| Filter | Does | Example |
|--------|------|---------|
| `zeropad` | Pads a number to two digits | `7` → `07` |
| `zeropad(N)` | Pads to N digits | `zeropad(3)`: `7` → `007` |
| `lower` / `upper` | Changes case | `DC1` → `dc1` |
| `slugify` | Makes a NetBox slug | `Store 001 (US-West)` → `store-001-us-west` |

To apply two transformations, compute an intermediate variable: `site_slug: '{{ variables.site_name | slugify }}'`.

### What refuses the render

A loop variable used outside its loop, a variable the design doesn't declare, two chained filters, or a function that finds nothing. The message names the object and field.

## Loops

`nbd_loop` repeats an object, an `items:` group, or an included design once per pass:

```yaml
devices:
- nbd_loop:
    over: {start: 1, end: 10}
    as: i
  infra_type: Device
  name: 'SERVER-{{ loop.i | zeropad }}'
```

| `over` form | Passes |
|-------------|--------|
| `{start: 1, end: 10}` | 1 to 10, both included |
| `3` | 1 to 3 |
| `[red, green, blue]` | Each item, in order |

Inside the loop, with `as: i`: `loop.i` (pass number, or the list item), `loop.index` (from 1), `loop.index0` (from 0), `loop.first`, `loop.last`, `loop.length`.

Repeat several objects together with `items:`:

```yaml
cables:
- nbd_loop:
    over: {start: 1, end: 3}
    as: i
  items:
  - infra_type: Cable
    label: 'CABLE-A-{{ loop.i }}'
  - infra_type: Cable
    label: 'CABLE-B-{{ loop.i }}'
```

Repeat an included design by putting `design:` and `template_vars:` beside `nbd_loop` (see the compute rack in SKILL.md). **Loop bounds cannot come from a variable** — repeat the count wherever it is needed.

## Functions

Called inside an expression while the design renders. Each looks in the design's own objects first, then in what Designs read from the target NetBox. **None allocate.** A function that finds nothing refuses the render, saying what it looked for.

### `nbd.lookup.endpoint(device, kind, name)`

Finds a port on a device. Cables use it for their ends.

| Parameter | Is |
|-----------|----|
| Device name | The device the port belongs to |
| Kind | `Interface`, `ConsolePort`, `ConsoleServerPort`, `PowerPort`, `PowerOutlet`, `FrontPort` or `RearPort` |
| Port name | The port's name on that device |

```yaml
termination_a: "{{ nbd.lookup.endpoint('SW-01', 'Interface', 'GigabitEthernet1/0/1') }}"
```

The port may not exist in NetBox yet — it can be created in the same plan by a device type the design brings in.

### `nbd.lookup.mgmt_interface(device)`

The interface marked management-only, preferring one named `mgmt*` or `management*` when there are several; more than one candidate after that refuses the render.

```yaml
assigned_object: '{{ nbd.lookup.mgmt_interface(variables.name) }}'
```

### `nbd.lookup.object(type, field=value, …)`

Any object by its type (as `infra_type` names it) and field values.

```yaml
cable: "{{ nbd.lookup.object('Cable', label='Router to Switch') }}"
```

### `nbd.nbt.get_module_bay(device, position)`

A device's module bay by position (`1`, `PCIE1`). Device types from the library create their bays; a module goes in one.

```yaml
module_bay: "{{ nbd.nbt.get_module_bay(parent, 'PCIE1') }}"
```

### `nbd.dtl.device_type(path[, device])` and `nbd.dtl.module_type(path[, module_position, device])`

Bring in a type from the Device Type Library with its manufacturer and components. A `dtl:` reference in a `device_type` / `module_type` / `rack_type` field does the same and is the usual way to write it. `module_position` fills the `{module}` placeholder in a module type's interface names.

### `nbd.design.lookup(reference)`

Loads a design by its `nbd:` reference. Including with `design: nbd:<name>` is the usual way.

## Variables and `template_vars`

```yaml
variables:
  rack_name: $USER                                   # required; deploy page asks
  serial: $USER_OPTIONAL                             # optional; fields using it are left out if not given
  status: active                                     # default
  site_slug: '{{ variables.site_name | slugify }}'   # computed at render
```

An item with `design: nbd:<catalogue name>` includes another design; `template_vars` sets its variables:

```yaml
devices:
- design: nbd:devices/Dell_R640_w_module
  template_vars:
    name: '{{ variables.rack_name }}-COMPUTE-01'
    position: 1
    site: '{{ parent.site }}'
    rack: '{{ parent }}'
```

- Each value is worked out in the **including** design — `variables`, `parent` and `loop` are the including design's.
- The included design's required (`$USER`) variables must all be given; optional ones may be.
- Its defaults apply to whatever `template_vars` leaves out; computed variables are worked out per inclusion.
- Values pass **one level down**. A design two levels down gets only what the design that includes it passes on.
- Variables have no types during preview: a mistyped number or address is caught at plan time by the field.

## Composition rules

| Scheme | Includes | Example |
|--------|----------|---------|
| `nbd:` | A catalogue design, current version | `nbd:devices/Dell_R640_w_module` |
| `nbd:<name>@<version>` | A pinned version label or content hash (7+ chars) | `nbd:devices/Dell_R640_w_module@1.0.0` |
| `dtl:` | A Device Type Library file, pinned to a commit or tag (**required**) | `dtl:device-types/Dell/PowerEdge-R640.yaml@fdf12828` |

- The `nbd:` name must match a catalogue name **exactly**. An unpinned include follows the catalogue; each plan records which version it used.
- Every included design must be in the catalogue **before** the design that includes it. A design others include can't be deleted while they do; a pinned version can't be removed or relabelled while pinned.
- `device_type`, `module_type` and `rack_type` accept `dtl:` paths. The type brings in its manufacturer and the components the library defines: a device type's interfaces, console and power ports, module bays; a module type's interfaces. Not yet brought in: power outlets, front/rear ports, module components other than interfaces — declare those ports under the device if a cable needs them.

## Complete device design

A reusable device design the compute rack includes. Catalogue name: `devices/Dell_R640_w_module`.

```yaml
name: Dell_R640_w_module
netbox_versions: ["4.6.*", "4.7.*"]
description: Dell PowerEdge R640 with Intel X710-DA4 10GbE NIC
variables:
  name: $USER
  slug: $USER
  position: $USER_OPTIONAL
  face: $USER_OPTIONAL
  status: $USER
  serial: $USER_OPTIONAL
  site: $USER
  rack: $USER_OPTIONAL
infra:
- infra_type: Device
  name: '{{ variables.name }}'
  slug: '{{ variables.slug }}'
  position: '{{ variables.position }}'
  face: '{{ variables.face }}'
  status: '{{ variables.status }}'
  serial: '{{ variables.serial }}'
  site: '{{ variables.site }}'
  role:
    infra_type: DeviceRole
    name: Compute Server
    slug: compute-server
    color: 4caf50
  rack: '{{ variables.rack }}'
  device_type: 'dtl:device-types/Dell/PowerEdge-R640.yaml@fdf12828'
  modules:
  - infra_type: Module
    device: '{{ parent }}'
    module_bay: "{{ nbd.nbt.get_module_bay(parent, 'PCIE1') }}"
    module_type: 'dtl:module-types/Intel/X710-DA4.yaml@fdf12828'
    module_position: PCIE1
```

Points to copy: optional placement variables so the device works in or out of a rack; the role declared inline as an object (created or adopted by key `slug`); the device type and module type pinned to the same library commit; the module placed by bay position with a function.

## Connecting existing objects

A design can operate purely on objects that already exist (created by another design or by hand), referring to them by mapping:

```yaml
- infra_type: Cable
  label: '{{ variables.cable_label }}'
  type: cat6a
  status: connected
  termination_a: {device: {name: '{{ variables.host_device }}'}, name: '{{ variables.host_interface }}'}
  termination_b: {device: {name: '{{ variables.switch_device }}'}, name: '{{ variables.switch_interface }}'}
- infra_type: Interface
  device: {name: '{{ variables.switch_device }}'}
  name: '{{ variables.switch_interface }}'
  mode: access
  untagged_vlan: {vid: '{{ variables.vlan_id }}'}
```

The interface is **adopted** (it exists), and only `mode` and `untagged_vlan` become the design's to manage. Remember `untagged_vlan` requires `mode` to be set.
