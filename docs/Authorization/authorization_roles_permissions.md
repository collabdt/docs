# Authorization: Roles, Permissions, and CASL

This document explains how **authorization** works in the cdt app, how **roles and permissions** are stored, how **CASL** is used to evaluate access, and how permissions are enforced on the **API and UI layers**.

The app uses a **role-based access control (RBAC)** system. We use:
- Prisma to store roles and permissions in a PostgreSQL database.
- CASL as the authorization engine
- Server-side enforcement in API routes
- UI-side enforcement

## Organization-scoped authorization:
Permissions are NOT global to a user. A user has permissions within an organization.
This allows users to have **different roles in different organizations**. 

## Prisma:
Roles are represented with the Role entity. The UserRole entity joins a user to a role within an organization.
Permissions are stored as JSON on each role
ex) 
```json
	[
	  { "action": "read", "subject": "Building" },
	  { "action": "update", "subject": "Building" }
	]
	```

### Fields:
- **action**: What the user can do (read, create, update, delete, etc.)
- **subject**: What resource the action applies to (Building, Organization, User, etc.)

CASL rules are derived from this data at runtime.


## CASL:

### Why CASL is used:
CASL provides:
- A simple method of defining rules (ability.can(action, subject))
- A shared authorization model for server and UI
- Potential to go from simple RBAC to more advanced conditional rules later.

### Building abilities from the database:
Step 1: Fetch user permissions for an organization from the database (getUserPermissions(userId, orgId)).
Step 2: The CASL AbilityBuilder converts database permissions into executable rules (buildAbilityFromPermissions(perms))
Step 3: Get rules for a request
- Fetches permissions for (userId, orgId)
- Builds and returns a CASL ability

## Enforcing permissions (server-side):
Sensitive operations are protected within the api routes as such: 
```javascript
if (!ability.can("read", "Building")) {
  return NextResponse.json({ error: "Forbidden" }, { status: 403 });
}
```

## Enforcing permissions (UI-side):
The same permissions can be used to hide UI elements:
```javascript
{ability.can("create", "Organization") && (
  <Button>Create Organization</Button>
)}
```

This improves UX but cannot replace API enforcement.
