# Zander WMS — Setup, Data and API Requirements

## Purpose

This report explains how the WMS shown in [`index.html`](./index.html) should
work when connected to the existing Zander on-premises API and database. It
separates what is already present from what needs to be added.

**In this report:**

- **[REUSE]** — an existing table, API or capability found in the on-prem
  repositories.
- **[ADD]** — new API, table or data that the WMS needs.
- **[EXTEND]** — an existing API or table that needs a reviewed migration or
  change before it can support the WMS.

The current Zander web application is Angular. The intended next web
application is Next.js, with the existing Spring Boot API and PostgreSQL kept
as the backend and database. The mockup itself is a static HTML prototype: its
sample records and settings are not proof of data saved in PostgreSQL or served
by an API.

## 1. Complete installation and first-time setup flow

This is the most important part of the requirement.

1. **Deploy application.** Install a versioned application package at the
   client's premises. It contains:
   - Next.js frontend
   - Spring Boot backend
   - PostgreSQL setup/configuration
   - Database migrations

   No client-specific credentials should be baked into the application image.
2. **Initialize PostgreSQL.** PostgreSQL starts and creates or opens the local
   database. Spring Boot connects using deployment-provided configuration.
3. **Run database migrations.** Spring Boot checks the database schema version
   and applies pending migrations. These cover user/authentication, role and
   permission, site configuration, department/sub-department and WMS
   operational tables.
4. **Initialize site identity.** The installer supplies the six-digit site
   code and other required site information. Example: `482731`. This code
   identifies the client's installation; it is not a password.
5. **Check bootstrap status.** The backend checks whether the first Super
   Admin exists. If not, it enters initial setup mode; if already created, it
   disables initial setup and shows normal login.
6. **Create first Super Admin.** An authorized installer enters the Super
   Admin login ID and initial password through a secure setup prompt. The
   password is not hardcoded in source or the image. The backend hashes it and
   stores the account in PostgreSQL.
7. **Complete bootstrap.** The backend assigns the immutable `SUPER_ADMIN`
   role, requires a password change at first login, records the bootstrap
   event, and marks initial setup complete. The normal setup mechanism cannot
   be repeated.
8. **Super Admin logs in.** Next.js sends login credentials to Spring Boot.
   The backend validates them and returns an authenticated session/token.
9. **Super Admin configures the site.** They configure departments,
   sub-departments, users, managers, permissions, terminology and ERP
   integration.
10. **Warehouse starts operating.** Staff log in and use the enabled WMS
    workflows.

## 2. What the mockup needs to support

The prototype presents these fixed departments: **Inbound, Inventory and
Outbound** (the Returns workflow is outside this report's current scope). It
also demonstrates:

- A role hierarchy: Super Admin, Head, Manager and Staff.
- Sub-departments, with department-specific manager/staff display labels.
- Role-based page access and column visibility/order.
- Site terminology settings, such as role and picklist labels.
- Outbound picklist, group-picklist, checker, scan and submission views.
- Inventory views for stock, putaway, transfers and counts.
- Inbound work views.
- User administration and password-reset flows.

In the current mockup, these settings and sample records are in the browser,
not shared server data. Production must load and save them through the API.
Hiding a page in Next.js is not permission enforcement: Spring Boot must check
the user's role and scope for every protected operation.

## 3. Existing database tables and columns

The table/column inventory below is based on the checked-in
[`db-schema.md`](../onprem-zander-api/db-schema.md) snapshot and the existing
Spring Boot models. That file shows both tenant and `public` schema snapshots,
which do not have identical columns. Confirm the exact target site's schema
before writing migrations.

### User, authentication and permissions

| Tag | Existing table | Columns in the tenant-schema snapshot | WMS use / limitation |
|---|---|---|---|
| [REUSE] | `tbl_user` | `user_id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `checked`, `country_code`, `dispatched`, `email`, `first_name`, `hierarchy_id`, `image_url`, `isenable`, `is_user_onboarded`, `last_name`, `manufacturer_id`, `mobile_channel`, `new_user_validate_mobile`, `new_user_validate_web`, `non_saleable`, `online`, `otp_verified`, `packed`, `phone_no`, `picked`, `referrer_code`, `role_id`, `role_name`, `manager_id`, `saleable`, `status`, `user_app_type`, `web_channel`, `tenant_name`, `warehouse_location` | User identity, current role and manager link. It has no WMS department/sub-department assignment or explicit first-login password-change flag. |
| [REUSE] | `tbl_user_auth` | `id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `enabled`, `credentials_expired`, `email_id`, `passcode`, `password`, `password_expiry_date`, `user_id` | Existing authentication storage. Reuse only with the production password-hashing policy; never store the initial password as plain text. |
| [REUSE] | `tbl_user_session` | `session_id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `city_name`, `country_name`, `ip`, `last_updated_on`, `port`, `postal`, `state`, `status`, `user_agent`, `user_id` | Existing session tracking. |
| [REUSE] | `tbl_role` | `role_id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `description`, `role_name` | Existing roles. Keep authorization identifiers stable; do not authorize using editable display labels. |
| [REUSE] | `tbl_screen` | `screen_id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `count`, `createenable`, `deleteenable`, `icon`, `menu_id`, `path`, `screendescription`, `screenname`, `screen_order`, `type`, `updateenable`, `viewenable` | Existing screen catalogue and screen-level action flags. Verify how its route IDs map to the WMS pages before reuse. |
| [REUSE] | `tbl_role_services` | `id`, `role_id`, `screen_id` | Existing role-to-screen mapping. It does not by itself prove that the backend enforces every WMS operation. |
| [REUSE] | `tbl_user_role_services` | `id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `role_id`, `screen_id`, `screenname`, `user_id` | Existing user/screen mapping. Do not treat frontend visibility as authorization. |
| [REUSE] | `tbl_user_manager` | `id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `manager`, `user_id` | Existing manager/user relationship data. Confirm semantics before using it as the sole WMS assignment source. |
| [REUSE] | `tbl_user_preference` | `preference_id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `currency`, `date_format`, `language`, `time_zone`, `user_id` | User preferences, not role/page column preferences. |
| [REUSE] | `tbl_tenant_label` | `label_code`, `description`, `label_name` | Existing label storage may help with terminology; it does not express all of the mockup's role, department and page scopes. |

### Warehouse and outbound data

| Tag | Existing table | Columns in the tenant-schema snapshot | WMS use / limitation |
|---|---|---|---|
| [REUSE] | `tbl_product` | `product_code`, `tenant_id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `product_name`, `version_no`, `id`, `box_quantity`, `case_quantity`, `manufacture_id`, `status` | Product master. |
| [REUSE] | `tbl_batch` | `id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `batch_code`, `batch_number`, `box_quantity`, `expiry_date`, `manufacturer`, `mrp`, `product_code`, `stock`, `purchase_price`, `status`, `last_received_date`, `gst_percentage`, `sale_price`, `unit` | Existing batch/stock fields. Confirm whether they support the mockup's inventory location and stock movement needs. |
| [REUSE] | `tbl_godown` | `godown_id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `godown_name`, `floor`, `status` | Existing warehouse/location master. |
| [REUSE] | `tbl_invoice` | `invoice_number`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `due_date`, `group_picklist_number`, `invoice_date`, `invoice_value`, `line_item`, `salesman_code`, `start_time`, `end_time`, `scanned_by` | Existing invoice header data for outbound work. |
| [REUSE] | `tbl_pick_list` | `picklist_id`, `picklist_number`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `delivery_type`, `godown_location`, `gross_amount`, `group_picklist_number`, `location_id`, `manufacturer_id`, `picklist_date`, `picklist_time`, `related_invoices`, `salesman_code`, `stockist_id`, `total_amount`, `total_bills`, `total_items`, `total_quantity`, `total_tax`, `status`, `scan`, `remarks`, `completed`, `start_time`, `end_time`, `scanned_by`, `floor`, `api_status`, `checker_id`, `tray_id`, `batch_code`, `packaging_cases_count`, `packaging_box_count`, `packaging_bag_count`, `packaging_ice_pack_count`, `device_id` | Existing outbound picklist records and workflow state. |
| [REUSE] | `tbl_pick_list_group` | `group_picklist_number`, `picklist_time`, `salesman_code`, `related_invoices`, `completed`, `status`, `remarks`, `scan`, `start_time`, `end_time`, `scanned_by`, `api_status`, `checker_id`, `assigned_at`, `batch_code`, `packaging_cases_count`, `packaging_box_count`, `packaging_bag_count`, `packaging_ice_pack_count`, `idle_time_seconds`, `device_id`, `v_dt` | Existing group-picklist header/work state. |
| [REUSE] | `tbl_pick_list_item` | `picklist_item_id`, `created_by`, `created_on`, `modified_by`, `tenant_id`, `version_no`, `unit_description`, `batch_number`, `expiry_date`, `gross_amount`, `location_id`, `mrp`, `pack`, `picklist_number`, `picklist_id`, `product_code`, `product_name`, `purchase_price`, `sale_unit`, `sales_price`, `total_amount`, `total_tax`, `transdate`, `transno`, `status`, `scanned_quantity`, `quantity`, `free_quantity`, `manual_scanned`, `modified_on`, `scanned_time`, `manufacturer_id`, `split`, `start_time`, `end_time`, `scanned_by`, `increment_count`, `decrement_count`, `ok_count`, `godown_location`, `floor`, `stock_quantity`, `v_no`, `v_dt`, `v_type`, `rop_code`, `bill_modify_status`, `batch_code`, `stock_location` | Existing picklist lines, product/batch quantities and scan state. |
| [REUSE] | `tbl_checker_master` | `id`, `checker_id`, `user_id`, `checker_name`, `status`, `created_on`, `modified_on`, `created_by`, `modified_by`, `version_no`, `tenant_id`, `alter_code` | Existing checker master used by outbound work. |
| [REUSE] | `tbl_tray_master` | `id`, `tray_id`, `tray_name`, `status`, `created_on`, `modified_on`, `created_by`, `modified_by`, `version_no`, `tenant_id` | Existing tray master. |
| [REUSE] | `tbl_device` | `device_id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `ip`, `port`, `protocol`, `scanner_type`, `status`, `url`, `application`, `station`, `device_name` | Existing device master. |
| [REUSE] | `tbl_device_user_mapping` | `id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `device_id`, `user_id` | Existing device/user mapping. |
| [REUSE] | `tbl_integraration` | `integratation_id`, `created_by`, `created_on`, `modified_by`, `modified_on`, `tenant_id`, `version_no`, `password`, `status`, `url`, `user_name` | Existing integration settings. Protect credentials; never return them to the browser or include them in logs. |

## 4. Data and migrations needed

### [ADD] Configuration and bootstrap data

| Proposed table/data | Required contents |
|---|---|
| `site_configuration` | One row per installation: six-digit `site_code`, site display name, setup-complete state, and any approved local settings. The site code is an identifier, not a credential. |
| `bootstrap_event` | One audit row for successful initial setup: event time, installer/operator identity where available, created Super Admin ID and completion state. Never store the submitted password. |
| `tbl_user_auth` migration | Add a first-login password-change flag (for example `force_password_change`) if the current schema does not already provide the required behavior. Store a strong password hash. |
| `tbl_sub_department` | One row per client-configured sub-department: stable ID/key, fixed parent department key, display name, optional manager/staff labels, enabled state and display order. |
| `wms_user_assignment` | Link a user to the configured department/sub-department and, where applicable, the responsible manager. Enforce assignment scope in the API. |
| `wms_role_alias` | Optional per-site display names for stable role keys. A changed label must not change the underlying permission identity. |
| `wms_role_page_permission` | Role key, fixed page/sub-page key and allowed action(s). Backend checks these permissions on every protected request. |
| `wms_site_term` | Stable terminology key and the client-facing label to show. |
| `wms_page_column_preference` | Role, page/tab, stable column key, visibility and order. Validate keys against the server's allowed column list. |

The four top-level departments are fixed product structure in this mockup;
they should be application-defined keys, not freely created departments.
Create sub-department rows only for names supplied during site configuration.
Do not seed invented client users, passwords, site codes, sub-departments or
warehouse transactions.

### Rows to initialize

1. **Site row:** one site configuration row, created from installer-provided
   site information.
2. **Role rows:** ensure the stable roles `SUPER_ADMIN`, `HEAD`, `MANAGER`
   and `STAFF` exist. Preserve existing role IDs/records where equivalent
   roles already exist; do not create duplicate roles on every migration.
3. **Page and access rows:** establish the fixed WMS page keys and the
   approved default grants for each role. Confirm those grants with the
   product owner; do not infer them from the mockup's role-preview mode.
4. **Terminology rows:** seed standard default labels for the stable
   terminology keys. Client overrides are site-local rows.
5. **Sub-department and assignment rows:** create only after the Super Admin
   supplies the site's actual structure and user assignments.
6. **Super Admin rows:** bootstrap creates one `tbl_user` row and matching
   `tbl_user_auth` row, assigns the stable Super Admin role, marks password
   change required, records the bootstrap event, and closes setup in one
   transaction.
7. **Operational rows:** retain or import actual client product, batch,
   invoice and picklist data through approved integrations. The
   prototype's sample data is not production seed data.

## 5. Existing API endpoints to reuse

The API uses `/rest/api/v1` for controllers implementing
`BaseRestController`/`BaseCrudController`. `SecurityController` login is a
root-level route. Existing responses are not uniform: some endpoints return
plain arrays/models, while others return `Result<T>` with `code`, `message`
and `data`. Unless a controller sets an HTTP status explicitly, normal method
returns use Spring's default success status; business errors may be encoded
inside `Result`. Confirm exception-handler behavior and the deployed API
version before relying on exact HTTP error codes.

| Existing endpoint | Request | Response and status observed | WMS use / note |
|---|---|---|---|
| `GET /{tenantName}/login` | `Authorization: Basic …` (login ID and password); `channel: WEB` header. The path is current tenant/schema name, not the six-digit site code. | `Result<LoginResponse>`; success body has `code`, `message`, and `data` containing token, user, role and language/features. Controller returns the `Result`; it does not set an HTTP status code. | Reuse only with a local mapping from installed site identity to the API's tenant/schema. Current route does not accept a six-digit site code as a separate field. |
| `GET /rest/api/v1/users`, `GET /rest/api/v1/all-user-list` | Authenticated request; no body. | User list. Normal response status defaults to HTTP 200. | Existing user listing. Apply WMS scope/permission checks before exposing it. |
| `POST /rest/api/v1/user`, `PUT /rest/api/v1/user` | Existing `User` JSON body. | `Result<User>` (`code`, `message`, `data`); normal response defaults to HTTP 200. | Existing user create/update. Extend/migrate for WMS department/sub-department assignment and first-login password behavior; don't assume current request contains those fields. |
| `GET /rest/api/v1/roles` | Authenticated request; no body. | `List<Role>`; normal response defaults to HTTP 200. | Read existing roles. Do not allow this general CRUD API to rename the immutable Super Admin identity. |
| `GET /rest/api/v1/screens`, `GET /rest/api/v1/screens-by-roll?roleId={roleId}` | Authenticated request; role ID for the second route. | `List<Screen>`; normal response defaults to HTTP 200. | Existing screens and role-screen data can inform navigation. WMS page grants and server-side authorization still need verification/extension. |
| `GET /rest/api/v1/products`, `GET /rest/api/v1/product-list` | Authenticated request; endpoint-specific query parameters may apply. | Product data; normal response defaults to HTTP 200. | Product master reads for warehouse flows. |
| `GET /rest/api/v1/fetch-pick-list-all` | Query: `page`, `size`, `sortBy`, `sortDir`, optional `godownId`, `floorId`, `searchTerm`, required `dataSetType`. | `ItrazePickListScanDataDTO` via `ResponseEntity.ok` (HTTP 200). | Existing outbound picklist list. |
| `GET /rest/api/v1/fetch-pick-list-detail/{picklistId}`, `GET /rest/api/v1/fetch-pick-list-one/{picklistId}` | Picklist ID in path. | Picklist detail or list of picklist items; normal response defaults to HTTP 200. | Existing outbound detail/line reads. |
| `PUT /rest/api/v1/pick-list-start-scan/{picklistId}`, `PUT /rest/api/v1/pick-list-end-scan/{picklistId}` | Picklist ID in path; end scan may include `PickListPackagingDTO` JSON. | `Result<PickList>`; normal response defaults to HTTP 200. | Existing scan workflow. Business success/failure uses the `Result` body. |
| `GET /rest/api/v1/fetch-group-pick-list-all`, `GET /rest/api/v1/fetch-group-pick-list-one/{groupPicklistItemId}` | List query includes paging, `dataSetType`, search and sort; detail uses path ID. | Group DTO; normal response defaults to HTTP 200. | Existing group-picklist views. |
| `GET /rest/api/v1/godowns`, `GET /rest/api/v1/get-all-batch-numbers` | Authenticated request; endpoint-specific query parameters may apply. | Warehouse or batch data; normal response defaults to HTTP 200. | Existing location and batch data, subject to API contract checks. |

The existing generic `POST /register` is not a substitute for one-time
Super-Admin bootstrap. The inspected controllers do not provide the WMS
bootstrap-status/completion flow, WMS sub-department management, WMS
role/page/column/terminology persistence or a complete API contract for the
mockup's inbound and stock movement/count pages.

## 6. New API endpoints required

The paths and body shapes below are **proposed WMS contracts**, not endpoints
that exist today. They keep each request tied to the setup and mockup needs.
The response/status expectations here are recommendations for the new API;
they are not claims about the existing API.

| Method and proposed path | Request | Success response and HTTP status | Other expected statuses |
|---|---|---|---|
| `GET /rest/api/v1/wms/setup/status` | No body. May be called before login; return only whether setup is required. | `200` `{ "setupRequired": true }` or `{ "setupRequired": false }`. Do not reveal credentials or password data. | `503` if setup status cannot be checked; return an explicit error. |
| `POST /rest/api/v1/wms/setup/bootstrap` | `{ "siteCode": "six digits", "siteName": "...", "loginId": "...", "initialPassword": "..." }` over the protected installer setup path. | `201` `{ "setupComplete": true, "userId": "...", "forcePasswordChange": true }`. Never return the password/hash. | `400` invalid input; `409` setup already completed; `401/403` installer not authorized; `503` setup persistence failure. |
| `POST /rest/api/v1/wms/auth/change-initial-password` | Authenticated Super Admin's current credential/session and `{ "newPassword": "..." }`. | `200` `{ "passwordChanged": true }`; clear the first-login requirement and audit the change. | `400` invalid password; `401` invalid/expired session; `409` if the account is not in the required state. |
| `GET /rest/api/v1/wms/config` | Authenticated request. | `200` fixed department/page catalogue plus this site's sub-departments, terms and role permissions; no secrets. | `401/403` unauthenticated or unauthorized. |
| `PUT /rest/api/v1/wms/config/terminology` | Authenticated Super Admin; `{ "terms": [{ "key": "...", "label": "..." }] }`. | `200` saved terms, with stable keys and labels. | `400` unknown/invalid key; `401/403` unauthorized. |
| `GET/POST/PUT /rest/api/v1/wms/config/sub-departments` | Authenticated Super Admin. Create/update body includes the fixed `departmentKey`, sub-department name and optional manager/staff display labels. | `200` on list/update; `201` on create; response returns saved records and stable IDs. | `400` invalid parent/key; `401/403` unauthorized; `409` duplicate key/name under the agreed uniqueness rule. |
| `GET/PUT /rest/api/v1/wms/config/role-access` | Authenticated Super Admin; stable role/page keys and allowed actions. | `200` saved permission configuration. | `400` unknown page/action; `401/403` unauthorized. |
| `GET/PUT /rest/api/v1/wms/config/column-preferences` | Authenticated user or authorized admin; role/page/tab and stable column keys with visibility/order. | `200` saved preferences. | `400` unknown column/page; `401/403` unauthorized. |
| `POST /rest/api/v1/wms/password-reset-requests`, `POST /rest/api/v1/wms/password-reset-requests/{id}/approve` | Authenticated request/approver; target user and reason/request ID. The backend verifies role hierarchy and site/department scope. | `201` request created; `200` approval completed. Return request/status only, not a reusable shared password. | `400` invalid request; `401/403` unauthorized; `404` request not found; `409` already resolved. |

**Operational API gap:** the inspected API does not establish a complete,
matching contract for the mockup's receiving, putaway, stock transfer or stock
count workflows. Confirm the business fields, validation, approval steps, ERP
source and transaction rules before defining those request/response bodies or
status transitions. This avoids presenting unverified payloads as existing
requirements.

## 7. Benefits of the target system over the current flow

| Current Angular/prototype limitation | Target benefit |
|---|---|
| The production web application is Angular; the standalone mockup stores demo settings and records in browser storage. | Next.js provides the planned web application while Spring Boot remains the single source of business rules and PostgreSQL remains the source of truth. |
| Browser-local settings are not shared reliably between users/devices and can be lost when browser data changes. | Site settings and permissions are saved in the site's PostgreSQL database and used consistently by each authorized user. |
| Mockup role switching and hidden pages are demonstrations, not security controls. | The API authorizes each operation using stable roles and scope; the web UI reflects the same permissions. |
| Existing site setup and first-admin creation are not the one-time WMS bootstrap described here. | A controlled install flow creates the first Super Admin once, records setup completion and prevents normal re-bootstrap. |
| Configuration and operational integrations are not all represented by the mockup/API contract. | Versioned migrations and clearly defined APIs make supported configuration and data changes repeatable at each client site. |
| A local demo can include placeholder data and credentials. | Production uses actual site data and installer-provided secrets; no client credentials or default Super Admin password are shipped in images. |

Next.js does not replace Spring Boot or PostgreSQL. It replaces the Angular
frontend in the planned migration. This keeps the UI separate from the API,
database and authorization rules.

## 8. Suggested delivery order

1. Confirm the actual on-prem deployment topology, target PostgreSQL schema,
   site-code ownership and approved installer access method.
2. Add and test the one-time bootstrap and site configuration migrations and
   APIs. Confirm password hashing and first-login password change.
3. Reuse existing user, role, product, batch, warehouse and outbound API
   functions only after matching their request/response contracts to the WMS.
4. Add persistent sub-department assignments, page permissions, terminology
   and column preferences.
5. Wire the Next.js screens to the API; remove reliance on browser storage for
   shared configuration and live warehouse records.
6. Confirm the inbound and inventory operation rules and ERP field
   mapping before creating their API contracts or database tables.
7. Test migrations and bootstrap on a clean local installation, then test
   repeat installation/upgrade behavior without duplicating seed rows.
