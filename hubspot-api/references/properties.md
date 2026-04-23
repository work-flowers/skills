# Properties

Creating, editing, and archiving CRM properties (custom fields) on contacts, companies, deals, tickets, and custom objects.

## Table of contents

- [Listing properties](#listing-properties)
- [Creating a property](#creating-a-property)
- [Field type reference](#field-type-reference)
- [Updating a property](#updating-a-property)
- [Managing enumeration options](#managing-enumeration-options)
- [Property groups](#property-groups)
- [Archiving a property](#archiving-a-property)

Base endpoint: `/crm/v3/properties/{objectType}` where `objectType` is `contacts`, `companies`, `deals`, `tickets`, a custom object name, or an objectTypeId.

## Listing properties

### All properties for an object type

```python
props = client.crm.properties.core_api.get_all(object_type="contacts")
for p in props.results:
    print(p.name, p.label, p.type, p.field_type)
```

Filter to custom (non-HubSpot-defined) properties only:

```python
custom = [p for p in props.results if not p.hubspot_defined]
```

### Single property

```python
p = client.crm.properties.core_api.get_by_name(
    object_type="contacts",
    property_name="preferred_contact_method",
)
```

## Creating a property

```python
from hubspot.crm.properties import PropertyCreate

new = PropertyCreate(
    name="preferred_contact_method",       # internal name — immutable once created
    label="Preferred Contact Method",      # display label — can be changed later
    type="enumeration",
    field_type="select",
    group_name="contactinformation",       # must be an existing group
    description="How the contact prefers to be reached",
    options=[
        {"label": "Email",  "value": "email",  "displayOrder": 1, "hidden": False},
        {"label": "Phone",  "value": "phone",  "displayOrder": 2, "hidden": False},
        {"label": "Slack",  "value": "slack",  "displayOrder": 3, "hidden": False},
    ],
    has_unique_value=False,
    form_field=True,
)

created = client.crm.properties.core_api.create(
    object_type="contacts",
    property_create=new,
)
print(f"Created {created.name}")
```

### Required fields

- `name` — internal name, snake_case, lowercase. **Cannot be changed later.**
- `label` — human-readable label
- `type` — the data type (see below)
- `field_type` — the UI rendering (see below)
- `group_name` — must be an existing property group. List groups first if unsure.

### Optional but useful

- `description` — shows in the HubSpot UI as help text
- `display_order` — where it sorts within the group
- `form_field` — whether it can appear in forms
- `has_unique_value` — enforce uniqueness (text properties only)
- `options` — required for `enumeration` types

## Field type reference

The `type` / `field_type` combinations are not freely mixable. Use the right pair:

| Use case | `type` | `field_type` |
|---|---|---|
| Single-line text | `string` | `text` |
| Multi-line text | `string` | `textarea` |
| Rich text | `string` | `html` |
| Phone number | `string` | `phonenumber` |
| Email | `string` | `text` (with validation via rules) |
| URL | `string` | `text` |
| Number | `number` | `number` |
| Calculated number | `number` | `calculation_equation` |
| Date (date only) | `date` | `date` |
| Date + time | `datetime` | `date` |
| Single checkbox (boolean) | `enumeration` | `booleancheckbox` |
| Multiple checkboxes | `enumeration` | `checkbox` |
| Dropdown select | `enumeration` | `select` |
| Radio select | `enumeration` | `radio` |
| HubSpot user | `enumeration` | `select` (with `externalOptions: true`, `referencedObjectType: "OWNER"`) |
| File | `string` | `file` |

### Boolean (single checkbox) example

```python
PropertyCreate(
    name="is_strategic_account",
    label="Is Strategic Account",
    type="enumeration",
    field_type="booleancheckbox",
    group_name="companyinformation",
    options=[
        {"label": "Yes", "value": "true",  "displayOrder": 1, "hidden": False},
        {"label": "No",  "value": "false", "displayOrder": 2, "hidden": False},
    ],
)
```

### HubSpot user picker

```python
PropertyCreate(
    name="secondary_owner",
    label="Secondary Owner",
    type="enumeration",
    field_type="select",
    group_name="contactinformation",
    form_field=False,
    external_options=True,
    referenced_object_type="OWNER",
)
```

Note: this requires using `client.api_request()` directly since `external_options` and `referenced_object_type` may not be exposed on all versions of the `PropertyCreate` model:

```python
resp = client.api_request({
    "path": "/crm/v3/properties/contacts",
    "method": "POST",
    "body": {
        "name": "secondary_owner",
        "label": "Secondary Owner",
        "type": "enumeration",
        "fieldType": "select",
        "groupName": "contactinformation",
        "formField": False,
        "externalOptions": True,
        "referencedObjectType": "OWNER",
    },
})
```

### Calculation property

```python
PropertyCreate(
    name="deal_value_sgd",
    label="Deal Value (SGD)",
    type="number",
    field_type="calculation_equation",
    group_name="dealinformation",
    calculation_formula="number(amount) * 1.34",
)
```

## Updating a property

Use PATCH. You can update most fields except `name` and `type`/`field_type` (structural changes are blocked).

```python
from hubspot.crm.properties import PropertyUpdate

update = PropertyUpdate(
    label="Preferred Contact Method (updated)",
    description="Updated description",
)
client.crm.properties.core_api.update(
    object_type="contacts",
    property_name="preferred_contact_method",
    property_update=update,
)
```

## Managing enumeration options

To add, remove, or reorder options on a dropdown/radio/multi-checkbox property, PATCH the full `options` array. The API replaces the whole array — omitted options are removed.

```python
# Fetch current options first
current = client.crm.properties.core_api.get_by_name(
    object_type="contacts",
    property_name="preferred_contact_method",
)
current_options = [
    {
        "label": o.label,
        "value": o.value,
        "displayOrder": o.display_order,
        "hidden": o.hidden,
        "description": o.description or "",
    }
    for o in current.options
]

# Add a new option
current_options.append({
    "label": "WhatsApp",
    "value": "whatsapp",
    "displayOrder": len(current_options) + 1,
    "hidden": False,
})

# PATCH back
update = PropertyUpdate(options=current_options)
client.crm.properties.core_api.update(
    object_type="contacts",
    property_name="preferred_contact_method",
    property_update=update,
)
```

**Important**: if a record has a value for an option you remove, that record keeps the now-orphaned value. Audit with a search for the old value before removing.

## Property groups

Properties must belong to a group. List, create, and manage groups via:

```python
# List all groups
groups = client.crm.properties.groups_api.get_all(object_type="contacts")
for g in groups.results:
    print(g.name, g.label)

# Create a group
from hubspot.crm.properties import PropertyGroupCreate

new_group = PropertyGroupCreate(
    name="terrascope_custom",
    label="Terrascope Custom",
    display_order=10,
)
client.crm.properties.groups_api.create(
    object_type="contacts",
    property_group_create=new_group,
)
```

## Archiving a property

Archival is soft; the property and its values are retained for 90 days.

```python
client.crm.properties.core_api.archive(
    object_type="contacts",
    property_name="legacy_field_to_remove",
)
```

To restore a recently archived property, contact HubSpot support — there's no direct API endpoint.

## Cloning properties across portals

Common pattern: export property definitions from one portal (e.g. staging) and import to another (production).

```python
def export_custom_properties(client, object_type):
    """
    Export all non-HubSpot-defined properties as a list of dicts
    suitable for re-creating in another portal.
    """
    props = client.crm.properties.core_api.get_all(object_type=object_type)
    exports = []
    for p in props.results:
        if p.hubspot_defined:
            continue
        exports.append({
            "name": p.name,
            "label": p.label,
            "type": p.type,
            "fieldType": p.field_type,
            "groupName": p.group_name,
            "description": p.description,
            "displayOrder": p.display_order,
            "formField": p.form_field,
            "options": [
                {
                    "label": o.label,
                    "value": o.value,
                    "displayOrder": o.display_order,
                    "hidden": o.hidden,
                }
                for o in (p.options or [])
            ],
        })
    return exports
```

Save to JSON, swap portal tokens, and POST back via `client.api_request()` to recreate.
