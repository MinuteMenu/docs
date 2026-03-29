# CX SFSP/ARAS Overview

SFSP (Summer Food Service Program) and ARAS (At-Risk Afterschool Meals) are USDA meal programs for center-based child care providers. In KidKare, SFSP/ARAS centers use a separate set of pages and features from regular centers.

This section documents how SFSP/ARAS works in the CX (Centers) product.

---

## Programs

| Program | Full Name | Purpose |
|---------|-----------|---------|
| **SFSP** | Summer Food Service Program | Provides free meals to children during summer months when school is not in session |
| **ARAS** | At-Risk Afterschool Meals | Provides meals and snacks to children in afterschool care programs in low-income areas |

Both programs share the same features in KidKare. The system treats them identically — the only difference is the license type assigned to the center.

---

## Roles

| Role | Description |
|------|-------------|
| **Sponsor Admin** | Manages multiple SFSP/ARAS centers. Can view and enter data for all centers under their sponsorship. |
| **Sponsor Staff** | Works for a sponsor. Access depends on assigned permissions. |
| **Center Admin** | Manages a single SFSP/ARAS center. Records attendance and meals for that center. |
| **Center Staff** | Works at a center. Access depends on assigned permissions. |
| **IC Admin** | Independent Center administrator. Manages their own center without a sponsor. |
| **IC Staff** | Works at an independent center. Access depends on assigned permissions. |
| **State User** | State agency user. Can view reports across all sponsors in their state. |

---

## Center/IC Navigation

Open Enrolled (OE) SFSP/ARAS Centers and ICs can access the pages listed below. Some pages are shared with regular centers, but several are SFSP/ARAS-specific.

### Menus/Attendance

| Page | URL | Notes |
|------|-----|-------|
| Attendance & Meal Counts | `/#/cx-aras-sfsp-meal-attendance` | [See Attendance & Meal Counts](attendance.md) |
| Non-Congregate Meal Delivery | `/#/cx-aras-sfsp-non-congregate-meals` | Only visible when Non-Congregate Meal Settings = Yes. [See Non-Congregate Meals](attendance.md#non-congregate-meal-delivery) |
| Meal Delivery/Pickup | `/#/delivery-pickup-list` | [See Meal Delivery & Pickup](meal-delivery-pickup.md) |
| Daily Menu | `/#/cx-daily-menus` | |
| Menu Templates | `/#/cx-food-program/cx-manage-menus` | |
| Add Menu | `/#/cx-food-program/add-menu` | |

### Claims

| Page | URL |
|------|-----|
| List Claims | `/#/sponsor-claims-list` |
| Claim Information | `/#/sponsor-claim-detail-arassfsp` |

### Reports

| Page | URL | Notes |
|------|-----|-------|
| SFSP/ARAS Reports | `/#/sfsparas-reports` | [See Reports](reports.md) |

### Expenses

| Page | URL | Notes |
|------|-----|-------|
| Receipts | `/#/cx-view-expenses` | |
| Add Receipts | `/#/cx-add-expense` | |
| Manage Vendors | `/#/vendors` | IC only |

### Administration

| Page | URL |
|------|-----|
| Site Details | `/#/cx-site-details` |
| User Permissions | `/#/sponsor-center-user-permissions` |
| User Details | `/#/sponsor-center-user-details` |
| Manage Formula Types | `/#/cx-manage-formula-type` |

### Other

| Page | URL | Notes |
|------|-----|-------|
| Messages | `/#/messages` | |
| My Account | `/#/my-account` | |
| Add Signature | | IC only |
| Settings | `/#/global-settings` | [See Custom Fields Settings](attendance.md#custom-fields-settings) |

---

## LA vs Non-LA

Louisiana (LA) and Oklahoma (OK) have state-specific UI layouts for some SFSP/ARAS pages. Throughout this documentation:

- **Non-LA** refers to the standard layout used by all states except Louisiana
- **LA** refers to the Louisiana-specific layout

The data model and business logic are the same. Only the UI differs.

---

## Multi-License Centers

Some centers hold both a Regular license and an SFSP/ARAS license. These centers see a **Program Type** dropdown (Regular or SFSP/ARAS) that switches between the standard CX pages and the SFSP/ARAS pages.

---

## Feature Pages

| Feature | Description |
|---------|-------------|
| [Attendance & Meal Counts](attendance.md) | Recording attendance and meal data. Includes Bulk Entry, Non-Congregate Meals, and Custom Fields. |
| [Food Transfer](food-transfer.md) | Settings and page changes when Food Transfer is enabled for a center. |
| [Meal Delivery & Pickup](meal-delivery-pickup.md) | Creating and managing satellite delivery/pickup forms. |
| [Reports](reports.md) | Served Meals Report, Bulk Attendance Last Modified, and Satellite Report. |
