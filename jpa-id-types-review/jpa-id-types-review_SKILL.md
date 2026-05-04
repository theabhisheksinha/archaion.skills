# Skill: JPA ID Types Review

Analyze JPA entity identifier types in an application using CAST Imaging MCP. Identifies all @Id fields and classifies them by Java type (Long, Integer, String, UUID, composite keys).

## Steps

1. **Identify the application** — if the user hasn't specified one, call `mcp__imaging__applications` to list available applications and ask which one to review.

2. **Fetch all @Id fields** — call `mcp__imaging__objects` with:
   - `filters="annotation:contains:Id,type:contains:field"`
   - Read the full result to count total items

3. **For each unique entity file**, determine the ID type using `mcp__imaging__source_file_details` with `nature="inventory"`:
   - Look for `getId()` method signature in the result (e.g., `getId() return java.lang.Long`)
   - If the entity extends a generic base class (e.g., `SalesManagerEntity<Long, User>`), extract the first generic parameter from `Java Instantiated Class` element
   - Check for `@EmbeddedId` or `@IdClass` annotations for composite keys

4. **Identify composite keys** — call `mcp__imaging__objects` with:
   - `filters="annotation:contains:Embeddable,type:contains:class"`
   - Check which embeddable classes are used in `@EmbeddedId` fields

5. **Classify and report** each entity by ID type:
   - **Long** (wrapper) — most common
   - **Integer** (wrapper)
   - **String** — natural keys, UUID strings
   - **UUID** — java.util.UUID
   - **Composite** — @EmbeddedId or @IdClass
   - **Unresolved** — cannot determine from available data

6. **Report statistics**:
   - Total entities with @Id
   - Count by type (Long, Integer, String, UUID, Composite)
   - List each entity with its ID type

## Example Output

| Entity | ID Type | Details |
|--------|---------|---------|
| User | Long | SalesManagerEntity<Long, User> |
| Customer | Long | getId() return java.lang.Long |
| UserConnection | Composite | @EmbeddedId → UserConnectionPK (String, String, String) |
| ApiKey | String | Natural key from column name |

## Notes

- Uses case-insensitive substring matching for annotations
- `getId()` method signature is the most reliable source for ID type
- Generic base class parameter is used when getId() is not available (e.g., Lombok-generated)
- Composite keys with @EmbeddedId are reported with their embeddable class name
