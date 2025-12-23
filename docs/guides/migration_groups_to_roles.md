# Migration Guide: From Groups to Roles

## Why Migrate?

**Groups** in Redshift are collections of users, but they don't support inheritance.  
**Roles** are proper permission containers that CAN inherit from other roles, enabling true role hierarchy.

## Key Differences

| Feature | Groups | Roles |
|---------|--------|-------|
| Can contain users | ✅ Yes | ❌ No (use GRANT ROLE instead) |
| Can contain other groups | ❌ No | ✅ Yes (via GRANT ROLE) |
| Supports inheritance | ❌ No | ✅ Yes |
| Grants to users/groups | ✅ Yes | ✅ Yes |
| Can be granted to roles | ❌ No | ✅ Yes |

## Migration Steps

### Before (using Groups - no inheritance)

```hcl
# Problem: Permissions are duplicated across groups
resource "redshift_group" "junior_analysts" {
  name = "junior_analysts"
}

resource "redshift_grant" "junior_select" {
  group       = "junior_analysts"
  schema      = "data"
  object_type = "table"
  privileges  = ["select"]
}

resource "redshift_group" "senior_analysts" {
  name = "senior_analysts"
}

resource "redshift_grant" "senior_grants" {
  group       = "senior_analysts"
  schema      = "data"
  object_type = "table"
  # 😞 Have to repeat SELECT here!
  privileges  = ["select", "insert", "update"]
}
```

### After (using Roles - with inheritance)

```hcl
# Solution: Roles with proper inheritance
resource "redshift_role" "junior_role" {
  name = "junior_analyst_role"
}

resource "redshift_grant" "junior_select" {
  role        = redshift_role.junior_role.name
  schema      = "data"
  object_type = "table"
  privileges  = ["select"]
}

resource "redshift_role" "senior_role" {
  name = "senior_analyst_role"
}

# Senior role inherits SELECT from junior role
resource "redshift_role_grant" "senior_inherits_junior" {
  role_name      = redshift_role.junior_role.name
  grant_to_type  = "role"
  grant_to_name  = redshift_role.senior_role.name
}

# Only grant additional privileges
resource "redshift_grant" "senior_extra" {
  role        = redshift_role.senior_role.name
  schema      = "data"
  object_type = "table"
  # 😊 No duplication! SELECT is inherited
  privileges  = ["insert", "update"]
}
```

## Complete Migration Example

### Step 1: Create Roles (parallel to existing groups)

```hcl
# Keep existing groups for now
resource "redshift_group" "analysts" {
  name = "analysts"
}

# Create new role equivalent
resource "redshift_role" "analyst_role" {
  name = "analyst_role"
}
```

### Step 2: Grant Same Permissions to Roles

```hcl
# Existing group grant
resource "redshift_grant" "group_grant" {
  group       = redshift_group.analysts.name
  schema      = "public"
  object_type = "table"
  privileges  = ["select"]
}

# New role grant (identical permissions)
resource "redshift_grant" "role_grant" {
  role        = redshift_role.analyst_role.name
  schema      = "public"
  object_type = "table"
  privileges  = ["select"]
}
```

### Step 3: Grant Role to Existing Users

```hcl
# Users that were in the group
resource "redshift_group_membership" "old_way" {
  name  = redshift_group.analysts.name
  users = ["john", "jane", "bob"]
}

# Grant role to same users (new way)
resource "redshift_role_grant" "john" {
  role_name      = redshift_role.analyst_role.name
  grant_to_type  = "user"
  grant_to_name  = "john"
}

resource "redshift_role_grant" "jane" {
  role_name      = redshift_role.analyst_role.name
  grant_to_type  = "user"
  grant_to_name  = "jane"
}

resource "redshift_role_grant" "bob" {
  role_name      = redshift_role.analyst_role.name
  grant_to_type  = "user"
  grant_to_name  = "bob"
}
```

### Step 4: Test & Verify

```bash
# Connect as one of the users
psql -U john -d mydb

# Verify they have the same permissions
SELECT * FROM public.my_table;  -- Should work
```

### Step 5: Remove Old Groups

```hcl
# Remove or comment out old group resources
# resource "redshift_group" "analysts" {
#   name = "analysts"
# }

# resource "redshift_grant" "group_grant" {
#   group = redshift_group.analysts.name
#   ...
# }
```

## Advanced: Multi-Level Hierarchy

### Before (Groups - flat structure)

```hcl
resource "redshift_group" "level1" {
  name = "level1"
}

resource "redshift_grant" "l1_grant" {
  group       = "level1"
  privileges  = ["select"]
  # ... config
}

resource "redshift_group" "level2" {
  name = "level2"
}

resource "redshift_grant" "l2_grant" {
  group       = "level2"
  privileges  = ["select", "insert"]  # Duplicated SELECT!
  # ... config
}

resource "redshift_group" "level3" {
  name = "level3"
}

resource "redshift_grant" "l3_grant" {
  group       = "level3"
  privileges  = ["select", "insert", "update"]  # Duplicated again!
  # ... config
}
```

### After (Roles - hierarchical)

```hcl
resource "redshift_role" "level1" {
  name = "level1_role"
}

resource "redshift_grant" "l1_grant" {
  role        = redshift_role.level1.name
  privileges  = ["select"]
  # ... config
}

resource "redshift_role" "level2" {
  name = "level2_role"
}

resource "redshift_role_grant" "l2_inherits_l1" {
  role_name      = redshift_role.level1.name
  grant_to_type  = "role"
  grant_to_name  = redshift_role.level2.name
}

resource "redshift_grant" "l2_extra" {
  role        = redshift_role.level2.name
  privileges  = ["insert"]  # Only additional privilege
  # ... config
}

resource "redshift_role" "level3" {
  name = "level3_role"
}

resource "redshift_role_grant" "l3_inherits_l2" {
  role_name      = redshift_role.level2.name
  grant_to_type  = "role"
  grant_to_name  = redshift_role.level3.name
}

resource "redshift_grant" "l3_extra" {
  role        = redshift_role.level3.name
  privileges  = ["update"]  # Only additional privilege
  # ... config
}
```

## Best Practices

### 1. Use Roles for Permissions, Not Users

❌ **Bad**: Grant directly to users
```hcl
resource "redshift_grant" "user_grant" {
  user = "john"
  # ...
}
```

✅ **Good**: Grant to role, then assign role to users
```hcl
resource "redshift_role" "analyst" {
  name = "analyst_role"
}

resource "redshift_grant" "role_grant" {
  role = redshift_role.analyst.name
  # ...
}

resource "redshift_role_grant" "user_gets_role" {
  role_name      = redshift_role.analyst.name
  grant_to_type  = "user"
  grant_to_name  = "john"
}
```

### 2. Create Clear Hierarchy

```
base_role (SELECT)
  ↓ inherits
intermediate_role (INSERT, UPDATE)
  ↓ inherits
advanced_role (DELETE, DROP)
```

### 3. Use Descriptive Names

- `readonly_role` instead of `r1`
- `data_analyst_role` instead of `role2`
- `admin_full_access_role` instead of `admin`

### 4. Document Dependencies

```hcl
resource "redshift_role_grant" "inherit" {
  role_name      = redshift_role.parent.name
  grant_to_type  = "role"
  grant_to_name  = redshift_role.child.name

  # Critical: ensure parent grants exist first
  depends_on = [
    redshift_grant.parent_permissions
  ]
}
```

## Troubleshooting

### Issue: "Role does not exist"

**Cause**: Trying to grant a role that hasn't been created yet.

**Solution**: Use `depends_on`
```hcl
resource "redshift_role_grant" "grant" {
  # ...
  depends_on = [redshift_role.my_role]
}
```

### Issue: "Cannot drop role, still has members"

**Cause**: Trying to destroy a role that's granted to users/groups/roles.

**Solution**: Remove all `redshift_role_grant` resources first.

### Issue: Users don't have expected permissions

**Cause**: Role hierarchy not set up correctly.

**Solution**: Verify with:
```sql
-- Check role membership
SELECT * FROM pg_auth_members;

-- Check role privileges
SELECT * FROM information_schema.role_table_grants 
WHERE grantee = 'your_role_name';
```

## Performance Considerations

- **Roles are more efficient**: Permissions are evaluated once per role, not per user
- **Easier auditing**: Check role permissions instead of individual user permissions
- **Better scalability**: Add 1000 users to a role instead of creating 1000 grants

## Summary

| Aspect | Groups | Roles |
|--------|--------|-------|
| Code duplication | High | Low |
| Maintenance effort | High | Low |
| Flexibility | Limited | High |
| Native inheritance | No | Yes |
| Recommended | ❌ | ✅ |
