# Role Hierarchy And Access Control

This document explains how the backend-v2 admin permission system works.

It is meant to be the source of truth for debugging:

- why a user can or cannot see data
- how Supabase Auth maps to app users
- how company/salon scope is calculated
- how staff and user salon moves are recorded
- how to seed and test roles safely

## 1. High-Level Model

The system has five roles:

```text
super_admin > company_admin > salon_admin > user > staff
```

The hierarchy means:

- `super_admin` can see and manage everything.
- `company_admin` can see and manage data inside one company.
- `salon_admin` can see and manage users/staff inside one salon.
- `user` and `staff` are valid account roles, but they do not get admin CRUD access.

Backend security does not trust the browser UI.

Every protected admin request must send:

```http
Authorization: Bearer <Supabase Auth access_token>
```

The backend then:

1. Verifies the JWT with Supabase Auth.
2. Reads the authenticated email from the Supabase Auth user.
3. Finds the matching row in the app `users` table by email.
4. Loads active rows from `user_memberships`.
5. Builds `req.access`, which describes what the actor can see and do.
6. Controllers use `req.access` to filter reads and block writes.

## 2. Tables

### `users`

This is the app-level user table.

Important fields:

- `id`: app user id used by internal tables.
- `email`: must match the email in Supabase Auth for login mapping.
- `full_name`
- `phone`
- `status`

Important rule:

```text
Supabase Auth user email must match users.email.
```

If a Supabase Auth account exists but no matching app `users` row exists, `/api/me` returns 403.

### Supabase Auth Users

Supabase Auth stores login credentials and produces JWTs.

The app does not currently store Supabase auth user id in `users`.
Instead, backend maps:

```text
Supabase Auth email -> public.users.email
```

This is why test accounts need both:

- a Supabase Auth user
- a row in public `users`

### `user_memberships`

This table gives an app user a role and scope.

Important fields:

- `user_id`: references `users.id`
- `role`: one of:
  - `super_admin`
  - `company_admin`
  - `salon_admin`
  - `user`
  - `staff`
- `company_id`: only used for `company_admin`
- `salon_id`: used for `salon_admin`, `user`, and `staff`
- `status`: only `active` memberships count for permissions

Scope rules:

| Role | `company_id` | `salon_id` | Meaning |
| --- | --- | --- | --- |
| `super_admin` | null | null | global access |
| `company_admin` | required | null | access one company and its salons |
| `salon_admin` | null | required | access one salon |
| `user` | null | required | regular salon-scoped account |
| `staff` | null | required | staff salon-scoped account |

For salon-scoped roles, company is inferred from:

```sql
salons.company_id
```

Do not set `company_id` manually for `salon_admin`, `user`, or `staff`.

### `companies`

Company owner/tenant entity.

Company admins point directly to this table through:

```text
user_memberships.company_id
```

### `salons`

Salon/location entity.

Each salon belongs to one company:

```text
salons.company_id
```

Salon-scoped roles point to this table through:

```text
user_memberships.salon_id
```

### `staff`

Staff profile table.

Important fields:

- `salon_id`: current primary salon
- `user_id`: optional linked app user
- `display_name`
- `email`
- `phone`
- `is_active`
- `bookable`

### `staff_salon_assignments`

This table stores salon assignment history for staff.

Important fields:

- `staff_id`
- `salon_id`
- `active`
- `from_date`
- `to_date`

When a staff member moves salons:

1. Current active assignment is closed.
2. New assignment is created.
3. `staff.salon_id` is updated to the new salon.
4. If staff has a linked `user_id`, that user's salon-scoped membership is synced.

### `user_membership_history`

This table stores history for user membership scope changes.

It records:

- old role/company/salon
- new role/company/salon
- who changed it
- timestamp

This is used when a user account changes role or salon.

## 3. Auth And Request Flow

### Middleware

Main file:

```text
backend-v2/src/middleware/authContext.js
```

Routes use:

```js
requireAuthContext
requireAdminContext
```

`requireAuthContext` does:

1. Reads `Authorization: Bearer <token>`.
2. Verifies the token using `supabase.auth.getUser(token)`.
3. Loads app user by email.
4. Loads active memberships.
5. Builds `req.access`.

`requireAdminContext` blocks accounts whose highest role is only:

- `user`
- `staff`

### `req.access`

Controllers rely on this object.

Shape:

```js
{
  effectiveRole,
  isSuperAdmin,
  companyIds,
  salonIds,
  canUseAdmin,
  canMoveSalon
}
```

Meaning:

- `effectiveRole`: highest role from active memberships.
- `isSuperAdmin`: true if actor has a `super_admin` membership.
- `companyIds`: null for super admin, otherwise allowed company ids.
- `salonIds`: null for super admin, otherwise allowed salon ids.
- `canUseAdmin`: true for `super_admin`, `company_admin`, `salon_admin`.
- `canMoveSalon`: true only for `super_admin`, `company_admin`.

`null` means no limit.

Example:

```text
super_admin:
companyIds = null
salonIds = null
```

Example:

```text
company_admin:
companyIds = [company A]
salonIds = [all salons under company A]
```

Example:

```text
salon_admin:
companyIds = []
salonIds = [salon A]
```

## 4. Permission Rules

### Super Admin

Can:

- view all companies
- view all salons
- view all users
- view all staff
- assign users/staff to any salon
- manage company and salon records

Membership:

```sql
insert into user_memberships (user_id, role, status)
values ('USER_ID', 'super_admin', 'active');
```

### Company Admin

Can:

- view their company
- view salons under their company
- view users/staff under salons in their company
- move users/staff between salons in their company
- create/update salons under their company

Cannot:

- view or edit another company
- assign staff/user to salon outside their company

Membership:

```sql
insert into user_memberships (user_id, company_id, role, status)
values ('USER_ID', 'COMPANY_ID', 'company_admin', 'active');
```

### Salon Admin

Can:

- view staff in their salon
- view `user` and `staff` memberships in their salon
- create/update users/staff inside their salon, depending on endpoint behavior

Cannot:

- move staff/user to another salon
- view other salon admins
- manage company records
- manage salon records

Membership:

```sql
insert into user_memberships (user_id, salon_id, role, status)
values ('USER_ID', 'SALON_ID', 'salon_admin', 'active');
```

Important:

```text
salon_admin does not store company_id.
The company is inferred from salons.company_id.
```

### User

Can:

- no admin CRUD access currently

Membership:

```sql
insert into user_memberships (user_id, salon_id, role, status)
values ('USER_ID', 'SALON_ID', 'user', 'active');
```

### Staff

Can:

- no admin CRUD access currently

Membership:

```sql
insert into user_memberships (user_id, salon_id, role, status)
values ('USER_ID', 'SALON_ID', 'staff', 'active');
```

## 5. API Behavior

### `GET /api/me`

Purpose:

Frontend bootstrap endpoint.

Returns:

- current app user
- highest role
- permissions
- memberships
- companies/salons actor can select

Example response:

```js
{
  user: {...},
  effectiveRole: 'company_admin',
  permissions: {
    canUseAdmin: true,
    canMoveSalon: true,
    canManageCompanies: false,
    canManageSalons: true,
    canManageUsers: true,
    canManageStaff: true
  },
  memberships: [...],
  scope: {
    companyIds: [...],
    salonIds: [...],
    companies: [...],
    salons: [...]
  }
}
```

### `GET /api/companies`

Returns:

- super admin: all companies
- company admin: only their company
- salon admin: currently no company access unless explicitly allowed elsewhere

### `GET /api/company/:companyId/salons`

Returns salons under a company only if actor can access that company.

Important:

If `companyId` is invalid or multiple ids are accidentally joined into one string, Supabase may return a UUID error.
This should ideally be hardened into a 400 response in a future patch.

### `GET /api/users`

Returns users by visible memberships.

Rules:

- super admin: all users
- company admin: users with memberships in company or salons under company
- salon admin: only `user` and `staff` memberships in their salon

Response includes both:

```js
membership
memberships
```

Why both?

- `membership`: shortcut to first/current membership for simple UI.
- `memberships`: full list for future multi-role/multi-scope support.

### `POST /api/users`

Creates:

- row in `users`
- initial row in `user_memberships`

If membership insert fails, backend deletes the newly created user to avoid orphan users.

### `PUT /api/users/:id`

Can update:

- user fields
- membership role/scope

If role/scope changes, backend writes `user_membership_history`.

### `GET /api/staff`

Returns staff through `staff_salon_assignments` in actor scope.

Rules:

- super admin: all staff
- company admin: staff in salons under their company
- salon admin: staff in their salon

### `PUT /api/staff/:id`

Can update staff fields.

If `salonId` changes:

1. actor must be `super_admin` or `company_admin`
2. target salon must be assignable by actor
3. old active assignment is closed
4. new assignment is created
5. `staff.salon_id` is synced
6. linked user membership is synced if `staff.user_id` exists

Known issue found during testing:

```text
PUT /api/staff/:id currently treats body.salonId as a read filter before moving.
```

Symptom:

```json
{"error":"Staff member not found in this scope"}
```

Why:

- request body contains destination `salonId`
- lookup tries to find current staff assignment in destination salon
- staff is still in old salon, so lookup fails

Expected future fix:

- when loading current staff assignment for update/delete, ignore `body.salonId`
- use `body.salonId` only as destination after current assignment is found

## 6. Direct Supabase Editing Rules

When editing directly in Supabase, backend/UI validation does not run.

Keep these rules manually:

### Company Admin

Correct:

```text
role = company_admin
company_id = some company id
salon_id = null
```

### Salon Admin

Correct:

```text
role = salon_admin
company_id = null
salon_id = salon id
```

The salon's company is:

```sql
select company_id
from salons
where id = 'SALON_ID';
```

If you want salon admin to be under the same company as a company admin, choose a salon whose `salons.company_id` matches the company admin's `company_id`.

### User / Staff

Correct:

```text
role = user or staff
company_id = null
salon_id = salon id
```

### Do Not Do This

Do not set:

```text
role = salon_admin
company_id = some company id
salon_id = salon from another company
```

The DB constraint does not allow `company_id` for salon-scoped roles after migration, and conceptually this is wrong.

## 7. Testing Guide

### Step 1: Seed Test Account

For every login test user, create:

1. Supabase Auth user
2. app `users` row with same email
3. `user_memberships` row

Example:

```sql
insert into users (full_name, email, phone, status)
values ('Company Admin Test', 'companyadmin@test.com', '604-555-9001', 'active')
returning id;
```

```sql
insert into user_memberships (user_id, company_id, role, status)
values ('USER_ID', 'COMPANY_ID', 'company_admin', 'active');
```

### Step 2: Login And Get Token

PowerShell:

```powershell
$body = @{
  email = "companyadmin@test.com"
  password = "PASSWORD"
} | ConvertTo-Json

$companyAdmin = Invoke-RestMethod `
  -Uri "https://YOUR_PROJECT.supabase.co/auth/v1/token?grant_type=password" `
  -Method Post `
  -Headers @{
    apikey = "YOUR_SUPABASE_ANON_OR_PUBLISHABLE_KEY"
    "Content-Type" = "application/json"
  } `
  -Body $body
```

Token field:

```powershell
$companyAdmin.access_token
```

Do not use:

```powershell
$companyAdmin.access\_token
```

### Step 3: Test `/api/me`

```powershell
Invoke-RestMethod `
  -Uri "https://bookings-3mtq.onrender.com/api/me" `
  -Headers @{ Authorization = "Bearer $($companyAdmin.access_token)" }
```

### Step 4: Test Company Admin Scope

```powershell
$caCompanies = @(Invoke-RestMethod `
  -Uri "https://bookings-3mtq.onrender.com/api/companies" `
  -Headers @{ Authorization = "Bearer $($companyAdmin.access_token)" })

$caCompanies | Format-Table id,name
```

Expected:

```text
Only company admin's company.
```

### Step 5: Test Salon Admin Scope

```powershell
$saStaff = @(Invoke-RestMethod `
  -Uri "https://bookings-3mtq.onrender.com/api/staff?includeDeleted=true&includeInactiveAssignments=true" `
  -Headers @{ Authorization = "Bearer $($salonAdmin.access_token)" })

$saStaff.Count
$saStaff | Format-Table displayName,email,salonId
```

Expected:

```text
Only staff from salon admin's salon.
```

### Step 6: Test 403 Outside Scope

Try to access a company outside company admin scope:

```powershell
try {
  Invoke-RestMethod `
    -Uri "https://bookings-3mtq.onrender.com/api/companies/$outsideCompanyId" `
    -Headers @{ Authorization = "Bearer $($companyAdmin.access_token)" }
} catch {
  $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream())
  $reader.ReadToEnd()
}
```

Expected:

```json
{"error":"You do not have permission to view this company"}
```

## 8. PowerShell Gotchas

### `access_token`, not `access\_token`

Correct:

```powershell
$response.access_token
```

Wrong:

```powershell
$response.access\_token
```

### Backtick Must Be At End Of Line

Correct:

```powershell
Invoke-RestMethod `
  -Uri "https://example.com" `
  -Headers @{ Authorization = "Bearer $token" }
```

Wrong:

```powershell
Invoke-RestMethod `  -Uri "https://example.com"`
```

### Force Arrays

PowerShell can unwrap JSON arrays in surprising ways.

Use:

```powershell
$items = @(Invoke-RestMethod ...)
```

If the first item itself appears to contain multiple objects, flatten carefully:

```powershell
$flat = @($items[0])
$first = $flat[0]
```

## 9. Common Error Messages

### `Authenticated user is not registered in this admin system`

Meaning:

Supabase Auth user exists, but no app `users` row has the same email.

Fix:

```sql
insert into users (full_name, email, status)
values ('Dev Admin', 'same-email-as-auth@example.com', 'active');
```

### `Authenticated user has no active memberships`

Meaning:

App user exists, but `user_memberships` has no active row.

Fix:

```sql
insert into user_memberships (user_id, role, status)
values ('USER_ID', 'super_admin', 'active');
```

### `Staff member not found in this scope`

Meaning:

Backend could not find a visible `staff_salon_assignments` row for that staff id.

Possible causes:

- actor really does not have access to that staff
- request URL accidentally contains multiple ids
- known staff move bug where body `salonId` is used as a read filter

### `invalid input syntax for type uuid`

Meaning:

The route param contains something that is not a single UUID.

Example bad route:

```text
/api/company/id1 id2 id3/salons
```

Usually caused by PowerShell variable holding multiple objects.

## 10. Future Improvements

Recommended patches:

1. Add UUID validation middleware so malformed ids return 400 instead of 500.
2. Fix `PUT /api/staff/:id` so destination `body.salonId` is not used as read filter.
3. Add UI rule:
   - `company_admin`: show company selector only.
   - `salon_admin/user/staff`: show salon selector only.
   - clear invalid selector when role changes.
4. Consider adding `auth_user_id` to `users` to avoid email-based mapping.
5. Add automated API tests for:
   - super admin global access
   - company admin company scope
   - salon admin salon scope
   - staff move permissions
   - user membership history

