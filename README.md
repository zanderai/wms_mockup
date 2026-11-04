# wms_mockup

## Picklist start flow

The Start actions on Picklist and Group Picklist use the themed offline-device confirmation and preserve the existing outbound role/permission checks. Choosing **No** leaves the record unchanged. Choosing **YES** simulates the on-prem start response (`code: "0000"`, `message: "Scan Started"`), updates the mock record, and opens the corresponding checking view; barcode submission remains available from that view. In on-prem, the success routes are `/en/home/invoice-checking/{picklistNumber}` for a picklist and `/en/home/invoice-details/{groupPicklistNumber}` for a group picklist.

This is a standalone prototype and does not call the live API or device. On-prem proceeds directly when `device-handshake` returns `"0000"`; any other response opens the manual-processing confirmation. After confirmation, it starts a picklist with `pick-list-start-scan/{picklistId}` or a group picklist with `group-pick-list-start-scan/{groupPicklistItemId}`. Those endpoints return the common `{ code, message, data }` result; the services report `"0000"` with `"Scan Started"` on success and `"1111"` with `"Error While Started Scan"` when start fails. The web client also handles `"2222"` as an already-started response.
