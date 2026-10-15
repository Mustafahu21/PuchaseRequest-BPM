# 🛒 Purchase Request Workflow — IBM BAW
A fully automated **Purchase Request (PR) approval workflow** built with **IBM Business Automation Workflow (BAW)**. This project implements a multi-level approval process with role-based access control, dynamic UI visibility, budget validation, and reusable toolkit architecture — developed as the capstone project for the **NTI 3-Month ECM & BPM Internship**.
---
## 📋 Table of Contents
- [Overview](#overview)
- [Process Flow](#process-flow)
- [Architecture](#architecture)
- [Data Model](#data-model)
- [Features](#features)
- [Toolkit — Reusable Components](#toolkit--reusable-components)
- [Exposed Process Values (EPVs)](#exposed-process-values-epvs)
- [Technologies Used](#technologies-used)
- [Screenshots](#screenshots)
- [How to Import](#how-to-import)
---
## Overview
This project solves a common business challenge: **streamlining the purchase request lifecycle** from employee submission through multi-level management approval, budget validation, procurement, and final order placement.
---
## Process Flow
The purchase request follows a **5-lane approval workflow** with conditional routing and save-for-later capability at every step:
```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          PURCHASE REQUEST WORKFLOW                          │
│                                                                              │
│  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌────────────┐    ┌───────┐ │
│  │ EMPLOYEE │───►│ MANAGER  │───►│ FINANCE  │───►│PROCUREMENT │───►│  END  │ │
│  └─────────┘    └──────────┘    └──────────┘    └────────────┘    └───────┘ │
│       │              │               │ ▲              │                      │
│       │         Reject/Return        │ │         Over Budget?               │
│       │              │               │ │              │                      │
│       │              ▼               │ └──────────────┘                      │
│       │           [END]              │                                       │
│       │                              │ Over Budget                           │
│       │                              ▼                                       │
│  Save For Later              ┌──────────┐                                   │
│  (at any step)               │   CEO    │                                   │
│                              └──────────┘                                   │
│                               Approve / Reject                              │
└──────────────────────────────────────────────────────────────────────────────┘
```
### Step-by-Step Breakdown
---
|
|
 1 
|
**
Submit PR
**
|
 Employee 
|
 Employee fills in their info, item details, supplier, category, justification, and required date 
|
|
 2 
|
**
Manager Review
**
|
 Manager 
|
 Manager reviews the request and can 
**
Approve
**
, 
**
Reject
**
, or 
**
Return
**
 with comments 
|
|
 3 
|
**
Budget Allocation
**
|
 Finance 
|
 Finance team sets the initial budget for the request 
|
|
 4 
|
**
Market Research
**
|
 Procurement 
|
 Procurement performs market research and calculates the actual total price 
|
|
 5 
|
**
Budget Check
**
|
 Finance 
|
 If total price exceeds budget → route to CEO; otherwise proceed 
|
|
 6 
|
**
CEO Approval
**
|
 CEO 
*
(conditional)
*
|
 CEO reviews over-budget requests and can approve a higher budget or reject 
|
|
 7 
|
**
Budget Adjustment
**
|
 Finance 
|
 Finance adjusts the budget based on CEO feedback, or rejects the request 
|
|
 8 
|
**
Order Placement
**
|
 Procurement 
|
 Final system task to place the order 
|
> 💡 **Save for Later**: At any point in the process, any user can save the current state and resume later.
---
## Architecture
The project follows IBM BAW best practices by separating concerns into a **Process App** and a **Toolkit**:
```
HDMYA (Process App)
├── Purchase Process (BPD)          ← Main workflow with 5 swim lanes
├── UndercoverAgent                 ← Event/schedule-driven agent
├── Deployment Service Flow
├── Environment Variables
└── Process App Settings
    ▼ depends on ▼
OfficeSuppliers (Toolkit)
├── Business Objects (Data Model)
│   ├── pr (root object)
│   ├── EmployeeInfo
│   ├── RequestInfo
│   ├── supplierInfo
│   └── actionNcomments
├── Services
│   ├── Purchas_Request (Human Service — main UI)
│   ├── employeeInfo_service
│   ├── actionService
│   ├── Supplier Name
│   └── Request Info
├── Coach Views (UI Components)
│   ├── Employee Info
│   ├── Request Info
│   └── Supplier Info
├── Exposed Process Values (EPVs)
│   ├── Actions
│   ├── SupplierName
│   ├── ItemCategory
│   ├── departmentName
│   └── status
└── Localization Resources
```
### Why a Toolkit?
- **Reusability**: Data types, services, and UI components can be shared across multiple process apps
- **Maintainability**: Update the toolkit once and all consuming apps inherit the changes
- **Separation of concerns**: Business logic and UI are decoupled from process orchestration
- **Versioning**: Toolkit snapshots enable controlled deployments
---
## Data Model
The master business object `pr` encapsulates all data flowing through the process:
```
pr
├── employeeInfo : EmployeeInfo
│   ├── employeeId        (String)
│   ├── employeeName      (String)
│   ├── departmentID      (String)
│   └── departmentName    (String)
│
├── requestInfo : RequestInfo
│   ├── itemCategory      (String)
│   ├── itemName          (String)
│   ├── unitPrice         (Decimal)
│   ├── quantity          (Integer)
│   ├── totalAmount       (Decimal)
│   ├── requestDate       (Date)
│   ├── RequiredDate      (Date)
│   ├── justification     (String)
│   ├── budget            (Decimal, default: 1000.0)
│   └── otherCategory     (String)
│
├── supplierInfo : supplierInfo
│   ├── itemCategory      (String)
│   ├── itemName          (String)
│   └── supplierName      (String)
│
└── actionsNcomments : actionNcomments
    ├── action            (String)
    ├── mngComment        (String)   — Manager's comment
    ├── finComment        (String)   — Finance's comment
    └── ceoComment        (String)   — CEO's comment
```
---
## Features
### ✅ Multi-Level Approval Chain
Five distinct roles — Employee, Manager, Finance, Procurement, CEO — each with their own responsibilities and decision authority.
### ✅ Role-Based UI Visibility
The Coach UI dynamically adjusts field visibility and editability based on the current user's role:
```javascript
// Example: Finance can edit employee fields but budget is read-only
if (role == "Finance") {
    visibilityEmployee = "EDITABLE";
    budgetInfo = "READ ONLY";
}
// CEO sees everything as read-only
else if (role == "Ceo") {
    visibilityEmployee = "READ ONLY";
    budgetInfo = "READ ONLY";
}
```
### ✅ Conditional Routing (Budget Escalation)
If procurement's total price exceeds the allocated budget, the process automatically escalates to the CEO for approval before continuing.
### ✅ Save for Later
Any participant can pause their task at any step and resume later — the process instance persists in BAW.
### ✅ Comments & Audit Trail
Every approver (Manager, Finance, CEO) can attach comments that travel with the request, providing a full audit trail.
### ✅ Exposed Process Values (EPVs)
Dropdown options and action labels are configured via EPVs, allowing administrators to update values at runtime **without redeploying** the application:
|
 EPV 
|
 Values 
|
---
|
|
**
Actions
**
|
 Submit, Approve, Reject, Return, Send to Procurement, Send to CEO, Save For Later, Cancel 
|
|
**
Suppliers
**
|
 ifix, mall el computer, mall el bostan, computerFix, ikea, el kabany 
|
|
**
Item Categories
**
|
 maintenance, other, ... 
|
|
**
Departments
**
|
 HR, Finance, IT 
|
|
**
Status
**
|
 Pending, Approved, Rejected 
|
### ✅ Reusable Toolkit Architecture
All business objects, services, coach views, and EPVs are packaged in the **OfficeSuppliers** toolkit — promoting reusability and clean separation of concerns.
### ✅ Reusable Coach Views
Three modular coach views — **Employee Info**, **Request Info**, and **Supplier Info** — used across multiple human tasks in the process.
### ✅ Integration Services
Dedicated service flows for each data domain (employee info, supplier name, request info, actions) enable clean data fetching and processing.
---
## Exposed Process Values (EPVs)
EPVs allow runtime configuration without redeployment:
|
 EPV Name 
|
 Description 
|
 Sample Values 
|
---
|
|
`Actions`
|
 Available workflow actions 
|
 Submit, Approve, Reject, Return, Cancel, Save For Later, Send to Procurement, Send to CEO 
|
|
`SupplierName`
|
 Available suppliers 
|
 ifix, ikea, mall el computer, mall el bostan, computerFix, el kabany 
|
|
`ItemCategory`
|
 Purchase categories 
|
 maintenance, other 
|
|
`departmentName`
|
 Company departments 
|
 HR, Finance, IT 
|
|
`status`
|
 Request lifecycle states 
|
 Pending, Approved, Rejected 
|
---
## Technologies Used
|
 Technology 
|
 Usage 
|
---
|
|
**
IBM BAW 8.6.3
**
|
 Process automation platform 
|
|
**
BPMN 2.0
**
|
 Process modeling standard 
|
|
**
Coach Framework
**
|
 UI / form design 
|
|
**
JavaScript
**
|
 Client-side scripting in coaches and server scripts 
|
|
**
Toolkit Architecture
**
|
 Reusable component packaging 
|
|
**
EPVs
**
|
 Runtime-configurable values 
|
|
**
Undercover Agents
**
|
 Event/schedule-driven automation 
|
---
## How to Import
### Prerequisites
- IBM BAW v8.6.3 or later (or IBM BPM v8.6+)
- Admin access to Process Center
### Steps
1. **Import the Toolkit first:**
   - Open Process Center → Import → select `OfficeSuppliers - purchaseRequestV30.twx`
   
2. **Import the Process App:**
   - Open Process Center → Import → select `HDMYA - PurchaseRequest_5.twx`
   - BAW will automatically resolve the dependency on the OfficeSuppliers toolkit
3. **Configure Teams/Roles:**
   - Assign users to the lane participant groups: Employee, Manager, Finance, Procurement, CEO
4. **Deploy & Run:**
   - Create a snapshot and deploy to your Process Server
   - Start a new instance from Process Portal
---
## 👤 Author
Developed by **Mustafa Hussien** — NTI ECM & BPM Intern (2026)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?style=flat&logo=linkedin)](www.linkedin.com/in/mustafa-hussien-a696a327a)
