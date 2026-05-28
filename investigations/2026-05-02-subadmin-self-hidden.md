# Sub-Admin Self Hidden From Role Access List

- Date: 2026-05-02
- Scope: Role and Access Management Sub-Admin listing behavior for the `SENSOR_USER` account
- Related systems: Angular Role Access UI, `GET /api/v1/admins/roleaccess/subadmin-list`, `GET /api/v1/admins/roleaccess/subadmin-details`

## Objective

Determine why an account created as a sub-admin is usable for login but does not appear in the Role and Access Management sub-admin list when logged in as that account.

## Sources Used

- Repository files:
  - `code/sensor-angular/src/app/modules/layout/modules/role-access/role-listing/service/role-listing.service.ts`
  - `code/sensor-angular/src/app/modules/layout/modules/role-access/role-listing/subadmin-listing.modal.ts`
  - `code/sensor-alarm-backend/src/routes/v1/roleAccess.routes.ts`
  - `code/sensor-alarm-backend/src/entities/roleAccess.entity.ts`
  - `code/sensor-alarm-backend/src/constants/app.ts`
- Live API checks:
  - `GET /api/v1/admins/profile?validateSSO=true`
  - `GET /api/v1/admins/roleaccess/subadmin-list?page=1&limit=10&search=<account-email>`
  - `GET /api/v1/admins/roleaccess/subadmin-details?id=59117`
  - `GET /api/v1/admins/roleaccess/roles-list-dropdown`

## Findings

The account exists and is active. The live profile/detail checks returned:

| Field | Observed value |
| --- | --- |
| `id` | `59117` |
| `userType` | `8` |
| `roleId` | `41` |
| `status` | `1` |
| `customPermission` | `1` |
| role name from dropdown | `Sensor Admin` |

In backend constants, `userType: 8` maps to `SUB_ADMIN`.

The Sub-Admin list endpoint deliberately excludes the current session user. In `RoleAccessEntity.getSubadminList()`, the Sequelize query adds:

```ts
query["where"]["userType"] = USER_TYPES.SUB_ADMIN.id;
query["where"]["id"] = { [Op.ne]: sessionData.userId };
```

The older raw-SQL helper has the same behavior:

```ts
admin.userType='${USER_TYPES.SUB_ADMIN.id}' AND admin.id!=${sessionData.userId}
```

The Angular list UI uses `admin.GET_ALL_SUBADMIN`, which resolves to the backend Sub-Admin list API. It does not have a separate "include myself" mode.

Live API confirmation:

| Call | Result |
| --- | --- |
| `subadmin-list` searched by the current account email | `200`, `count: 0` |
| `subadmin-details?id=59117` | `200`, returned the account |

Therefore, this is not evidence that the account failed to create. It is the current product/API behavior: a sub-admin cannot see itself in the Sub-Admin listing while logged in as that same sub-admin.

## Risks Or Constraints

- This behavior may be intentional to prevent users from editing, disabling, or deleting their own admin account.
- Because the list omits self server-side, changing only the Angular UI will not make the account appear.
- Any code change to include self in the list would touch production-impacting application behavior and should be explicitly approved before editing.

## Cost Optimization Opportunities

None.

## Recommended Next Steps

- To verify the account appears in the list, log in as a different super-admin or sub-admin with Role Access view permission and search for the account email.
- If the desired behavior is "show myself but disable self-destructive actions", propose a backend/UI change that includes self in the list and hides or disables edit/block/delete actions for the current user.
- If the current behavior is intentional, add a small UI hint or support note so admins understand that their own sub-admin account is hidden from the listing.

## Open Questions

- Should self-hiding remain the product behavior for sub-admin users?
- Should super-admin users be the only role allowed to see all sub-admin accounts, including the currently authenticated account?
