# ACL Expansion System for Looker Studio Integration
## Comprehensive Documentation and Implementation Guide

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Why This System is Necessary](#why-this-system-is-necessary)
3. [System Architecture Overview](#system-architecture-overview)
4. [How the Expansion Logic Works](#how-the-expansion-logic-works)
5. [Looker Studio Integration](#looker-studio-integration)
6. [Data Blending Considerations](#data-blending-considerations)
7. [End User Control Points](#end-user-control-points)
8. [Implementation Workflow](#implementation-workflow)
9. [Security and Compliance](#security-and-compliance)
10. [Troubleshooting and Maintenance](#troubleshooting-and-maintenance)
11. [Best Practices](#best-practices)

---

## Executive Summary

The ACL (Access Control List) Expansion System is a critical data governance tool that transforms hierarchical organizational access rules into granular, row-level permissions suitable for use in Looker Studio dashboards. This system ensures that users only see data relevant to their role, region, and specific areas of responsibility while maintaining a single source of truth for access control.

**Key Benefits:**
- **Centralized Access Control**: Manage all user permissions from a single Google Sheet
- **Automatic Expansion**: Complex hierarchical rules automatically expand into thousands of individual permission records
- **Looker Studio Compatible**: Generates flat, relational data perfect for row-level security filtering
- **Audit Trail**: Every permission includes its source and reasoning
- **Scalability**: Handles complex organizational structures with multiple regions, departments, and sites

---

## Why This System is Necessary

### The Problem: Complex Organizational Hierarchies

Organizations like Sense Scotland operate with complex, multi-dimensional access requirements:

1. **Regional Structure**: Multiple operational regions (Ops Central, Ops East, Ops North East, Ops West, Ops Glasgow & East Dunbartonshire)
2. **Departmental Functions**: Various service types (Supported Living, TouchBase services, Day Support, etc.)
3. **Local Sites**: Dozens of individual physical locations
4. **Role-Based Access**: Different access levels based on position (CEO, Directors, Heads, Registered Managers)

### Why Manual Permission Management Fails

**Without this system, you would need to:**
- Manually create thousands of individual permission records
- Update permissions every time someone changes role or region
- Maintain consistency across multiple dashboards
- Risk human error in permission assignment
- Spend countless hours on administrative overhead

**Example:** A Head of Operations for North East region with access to "ALL" departments would require:
- 32 local sites in Ops North East × 9 departments = **288 individual permission records**
- All must be created, maintained, and updated manually

### The Solution: Rule-Based Expansion

This system uses **filter-based rules** that automatically expand into granular permissions:

| Filter Type | Rule Description | Expansion Result |
|-------------|-----------------|------------------|
| **Own Region** | User sees their assigned functions across all sites in their region(s) | Function × All Regional Sites |
| **Own Areas** | User sees their assigned functions only at their specific local sites | Function × Specific Sites Only |

---

## System Architecture Overview

### Data Flow Diagram

```
┌─────────────────┐
│   ACL Sheet     │  ← Master source of truth
│  (User Input)   │     - Employee details
└────────┬────────┘     - Filter rules
         │              - Region/Function assignments
         │
         ▼
┌─────────────────┐
│  Lookup Tables  │  ← Reference data
│   - Regions     │     - All regions
│   - Departments │     - All departments/functions
│   - Sites       │     - Region-to-site mappings
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Expansion       │  ← Google Apps Script
│   Script        │     - Parses filter rules
└────────┬────────┘     - Applies expansion logic
         │              - Generates flat records
         │
         ▼
┌─────────────────┐
│ ACL Expanded    │  ← Output sheet
│    Sheet        │     - Flat, normalized data
└────────┬────────┘     - One row per permission
         │              - Ready for Looker Studio
         │
         ▼
┌─────────────────┐
│ Looker Studio   │  ← Reporting layer
│   Dashboard     │     - Row-level security
└─────────────────┘     - Data blending
```

### Component Descriptions

#### 1. ACL Sheet (Input)
The master control sheet containing:
- **Full Name**: Employee identifier
- **Position**: Job title
- **Directorate**: Organizational division
- **Filter**: Access rule type (Own Region / Own Areas)
- **Department/Function**: Can be specific (e.g., "Form 1"), multiple (e.g., "Form 1, Form 2, Form 3"), or "ALL"
- **Region**: Geographic area (specific region or "ALL")
- **Service/Reporting Unit**: Service line
- **Local Sites**: Comma-separated list of specific locations
- **Email**: User identifier for Looker Studio filtering

#### 2. Lookup Tables
Reference tables that define the organizational structure:
- **Region_Lookup**: Maps regions to all their local sites
- **Departmen_Lookup**: Lists all valid departments/functions
- **Filter_Type_Lookup**: Defines valid filter types

#### 3. Expansion Script
Google Apps Script that:
- Reads ACL and lookup tables
- Parses multi-value fields (comma-separated)
- Applies expansion logic based on filter type
- Handles "ALL" keyword expansion
- Generates normalized output records

#### 4. ACL Expanded Sheet (Output)
Flat, denormalized table with one row per permission:
- Email
- Full Name
- Region
- Local Site
- Department/Function
- Access Level
- Source Filter (audit trail)

---

## How the Expansion Logic Works

### Filter Type: "Own Region"

**Business Rule**: User can access their assigned functions across **all sites** in their specified region(s).

**Expansion Logic:**
1. Identify user's region(s)
   - If Region = "ALL" → Use all regions from Region_Lookup
   - If Region = specific (e.g., "Ops Central") → Use only that region
2. Get all sites for those regions from Region_Lookup
3. Identify user's functions
   - If Department = "ALL" → Use all departments from Departmen_Lookup
   - If Department = "Form 1, Form 2" → Parse comma-separated values
   - If Department = specific → Use that one function
4. Create cartesian product: Each function × Each site in region

**Example 1: Regional Head with ALL departments**
```
Input:
- Name: Eileen Keddie
- Filter: Own Region
- Department: ALL
- Region: Ops North East

Output: 32 sites × 9 departments = 288 records
- ekeddie@sensescotland.org.uk, Ops North East, Tullideph, Shops
- ekeddie@sensescotland.org.uk, Ops North East, Tullideph, Fundraising
- ekeddie@sensescotland.org.uk, Ops North East, Tullideph, HSE
- ... (285 more records)
```

**Example 2: Regional Head with specific department, ALL regions**
```
Input:
- Name: Jennifer Niven
- Filter: Own Region
- Department: Fundraising
- Region: ALL

Output: 5 regions × ~15 avg sites × 1 department = ~75 records
- jenniven@sensescotland.org.uk, Ops Central, TouchBase Dunbartonshire, Fundraising
- jenniven@sensescotland.org.uk, Ops Central, Pollok, Fundraising
- jenniven@sensescotland.org.uk, Ops East, TouchBase Fife, Fundraising
- ... (72 more records)
```

### Filter Type: "Own Areas"

**Business Rule**: User can access their assigned functions **only at specific local sites** listed in their profile.

**Expansion Logic:**
1. Parse Local Sites field (comma-separated)
2. Identify user's functions
   - If Department = "ALL" → Use all departments from Departmen_Lookup
   - If Department = "Form 1, Form 2, Form 3" → Parse comma-separated values
   - If Department = specific → Use that one function
3. Create product: Each function × Each specific site

**Example 1: Manager with multiple functions**
```
Input:
- Name: Lynette Keith
- Filter: Own Areas
- Department: Form 1, Form 2, Form 3
- Local Sites: Dundee Respite

Output: 3 functions × 1 site = 3 records
- lynettekeith@sensescotland.org.uk, Ops North East, Dundee Respite, Form 1
- lynettekeith@sensescotland.org.uk, Ops North East, Dundee Respite, Form 2
- lynettekeith@sensescotland.org.uk, Ops North East, Dundee Respite, Form 3
```

**Example 2: Manager with ALL functions, multiple sites**
```
Input:
- Name: Rebekha Gray
- Filter: Own Areas
- Department: (empty - defaults to Operations)
- Local Sites: Cruden Bay 1, Cruden Bay 2, Ellon 1, Ellon 2, Eilean Rise

Output: 1 function × 5 sites = 5 records
- RebekhaGray@sensescotland.org.uk, Ops North East, Cruden Bay 1, Operations
- RebekhaGray@sensescotland.org.uk, Ops North East, Cruden Bay 2, Operations
- RebekhaGray@sensescotland.org.uk, Ops North East, Ellon 1, Operations
- ... (2 more records)
```

### Special Cases and Edge Handling

#### Case 1: Empty Department Field
- **Default Behavior**: Use "Operations" as default department
- **Rationale**: Most users without specified departments are operational staff

#### Case 2: Empty Local Sites with "Own Areas" Filter
- **Fallback**: Expand to all sites in user's specified region
- **Warning**: Logs as "Own Areas (Regional)" to indicate fallback occurred

#### Case 3: Region = "ALL" with "Own Region" Filter
- **Behavior**: Expands to all regions in Region_Lookup (excluding the "ALL" entry)
- **Use Case**: Senior leadership with organization-wide visibility

#### Case 4: Department = "ALL"
- **Behavior**: Expands to all departments in Departmen_Lookup
- **Warning**: Can generate very large numbers of records (hundreds to thousands)

---

## Looker Studio Integration

### Why Row-Level Security Matters

Looker Studio dashboards typically connect to data sources containing information for the entire organization. Without row-level security, every user would see all data. The ACL Expanded sheet provides the granular permissions needed to filter data appropriately.

### Integration Architecture

```
┌──────────────────┐
│  Operational     │  ← Main data source (NO email field needed)
│  Data Source     │     Contains: Region, Site, Department, Date, Metrics
│  (Google Sheets, │     Example: Service records, incident reports
│   BigQuery, etc.)│
└────────┬─────────┘
         │
         │  Left Join (Data Blending)
         │  Join Keys: Region + Site + Department
         │
         ▼
┌──────────────────┐
│  ACL Expanded    │  ← Access Control List (ACL)
│     Sheet        │     Contains: Email, Region, Site, Department
│  [Email Filter   │     **Email filter applied at data source level**
│   Applied Here]  │     Only current viewer's rows loaded
└────────┬─────────┘
         │
         │  Blend filters operational data
         │  based on viewer's permissions
         │
         ▼
┌──────────────────┐
│   Blended Data   │  ← Filtered dataset
│     Source       │     Contains only data viewer has permission to see
└────────┬─────────┘
         │
         │  Automatic filtering by logged-in user
         │  No additional report filters needed
         │
         ▼
┌──────────────────┐
│  Looker Studio   │  ← Dashboard
│    Dashboard     │     Each user sees only their permitted data
└──────────────────┘
```

**Key Difference from Simple Approach:**
- Email filter is applied at the **data source level**, not in the report
- Operational data doesn't need email addresses
- Blending creates the many-to-many relationship (multiple users can see same data)
- More efficient: Only relevant ACL records are loaded per user

### Step-by-Step Looker Studio Setup

#### Step 1: Connect Data Sources

1. **Add Main Data Source (Your Operational Data)**
   - Connect to your operational data (Google Sheets, BigQuery, etc.)
   - Ensure it contains: Region, Local Site, Department/Function fields
   - Example fields: Date, Service Type, Incident Count, Staff Hours, etc.
   - **Important**: This table does NOT need email addresses

2. **Add ACL Expanded Data Source**
   - Connect to the "ACL Expanded" sheet
   - Fields available: Email, Full Name, Region, Local Site, Department/Function, Access Level, Source Filter
   - This serves as your Access Control List (ACL)

#### Step 2: Apply Email Filter to ACL Data Source

**This is the critical security step** - Apply the email filter BEFORE blending:

1. **Edit the ACL Expanded data source** (not in the report, but in the data source settings)
2. Click **"Add a filter"** 
3. Select **"Filter by email"** or create custom filter
4. Configure filter:
   - Field: `Email`
   - Operator: `Equals`
   - Value: Check **"Filter by email"** option
   
**Alternative method:**
1. In data source settings, find **"Access Control"** section
2. Enable **"Email owner's credential to access data"**
3. Set filter field to `Email`

**What this does:**
- When any user views the report, Looker Studio automatically filters the ACL data source
- Only rows where `Email` matches the viewer's logged-in email are loaded
- This happens BEFORE data blending, making it more efficient and secure

#### Step 3: Create Data Blend

**Blending creates the many-to-many relationship between users and data.**

1. In Looker Studio, create a new Data Blend
2. **Left Table (Data Table)**: Your main operational data source
   - This is the data you want to show
3. **Right Table (ACL Table)**: ACL Expanded sheet (with email filter applied)
   - This controls who can see what
4. **Join Configuration**:
   - Join Type: **Left Join** (we want all data from main source, filtered by ACL)
   - Join Keys (these are your common fields):
     ```
     Main Data.Region = ACL Expanded.Region
     Main Data.Local Site = ACL Expanded.Local Site
     Main Data.Department = ACL Expanded.Department/Function
     ```

**Why Left Join (not Inner Join)?**
- We start from the data table (left side)
- The ACL table is already filtered to current user's permissions
- Left join effectively "adds" the user's email as a column to their permitted data
- Since ACL is pre-filtered by email, only matching records join
- Result: User sees only data they have permission for

**Visual representation:**
```
Operational Data (Left)     ACL Expanded (Right - filtered by viewer email)
├─ Region: Ops Central      ├─ Email: user@example.com (only this user's rows)
├─ Site: Pollok            ├─ Region: Ops Central
├─ Department: Form 1       ├─ Site: Pollok
├─ Metrics: [data]          └─ Department: Form 1
└─ [Joins with ACL] ────────┘
   Result: User sees this data row
```

#### Step 4: Test Access Control

**Testing Checklist:**
1. ✅ View dashboard as yourself - verify you see expected data
2. ✅ Share dashboard with a limited-access user - verify they see only their permitted sites
3. ✅ Share with a regional head - verify they see all sites in their region
4. ✅ Check edge cases (new users, users without ACL entries)
5. ✅ Verify cross-region users don't see other regions' data

**Important Testing Note:**
- The email filter on ACL data source happens automatically based on viewer
- You don't need to manually apply filters in the report
- Each viewer sees different results based on their ACL permissions

### Field Mapping Best Practices

**Critical**: Field names and values must match **exactly** between your main data and ACL Expanded sheet.

| Main Data Field | ACL Expanded Field | Notes |
|-----------------|-------------------|-------|
| Region | Region | Must be identical (case-sensitive) |
| Site / Location | Local Site | Normalize naming conventions |
| Department / Service Type | Department/Function | May require data transformation |

**Common Mismatch Issues:**
- ❌ "Ops Central" vs "Ops-Central" vs "Operations Central"
- ❌ "TouchBase Fife" vs "Touchbase Fife" (capitalization)
- ❌ "Form 1" vs "Form1" vs "Form One"

**Solutions:**
1. **Standardize at source**: Update main data to match ACL conventions
2. **Use calculated fields**: Create normalized fields in Looker Studio
3. **Update lookup tables**: Ensure Region_Lookup uses exact naming from operational data

---

## Data Blending Considerations

### Understanding Data Blending in Looker Studio

Data blending combines multiple data sources into a single dataset. For access control, this is essential but has important implications.

### Join Types and Their Impact

#### Left Join (RECOMMENDED for this ACL approach)
**Behavior**: Returns all records from operational data (left), joined with matching ACL records (right)
**Use Case**: When ACL is pre-filtered by email at data source level
**Result**: Users see operational data they have permission for

**How it works:**
1. ACL data source is filtered to current viewer's email BEFORE the blend
2. Left join starts with all operational data
3. Only rows that match viewer's ACL records are kept
4. Effect is the same as inner join, but more efficient

**Example:**
```
Operational Data (Left - 3 records):
- Ops Central, Pollok, Form 1 → Sales: 100
- Ops East, TouchBase Fife, Form 2 → Sales: 200
- Ops West, Fort William, Form 3 → Sales: 300

ACL for current viewer user@example.com (Right - already filtered, 2 records):
- user@example.com, Ops Central, Pollok, Form 1
- user@example.com, Ops East, TouchBase Fife, Form 2

Blended Result (2 records - only matching):
- Ops Central, Pollok, Form 1 → Sales: 100 ✓
- Ops East, TouchBase Fife, Form 2 → Sales: 200 ✓
(Fort William record excluded - no ACL permission)
```

**Why Left Join (not Inner Join) with Email-Filtered ACL:**
- ACL table is already filtered to current user (only their permissions loaded)
- Starting from operational data (left) and filtering by user's ACL (right)
- More efficient than loading all ACL records and filtering later
- Follows Google's recommended pattern for row-level security

#### Inner Join (Alternative approach, but less efficient)
**Behavior**: Only returns records that exist in BOTH data sources
**Use Case**: When NOT using email filter at data source level
**Problem**: Loads entire ACL table (all users), then filters in report
**Result**: Same outcome but slower performance

**When to use:** Only if you cannot apply email filter at data source level

### Cardinality Considerations

**Problem**: One-to-Many relationships can cause data inflation

**Scenario:**
```
Main Data (1 record):
- Region: Ops Central, Site: Pollok, Incidents: 5

ACL Expanded (3 records for same user):
- Ops Central, Pollok, Form 1
- Ops Central, Pollok, Form 2
- Ops Central, Pollok, Form 3

Blended Result (3 records):
- Ops Central, Pollok, Form 1, Incidents: 5
- Ops Central, Pollok, Form 2, Incidents: 5
- Ops Central, Pollok, Form 3, Incidents: 5

Dashboard shows: 15 total incidents (5×3) ← WRONG!
```

**Solutions:**

1. **Include Department/Function in Join**
   - Join on: Region + Site + Department
   - Ensures 1:1 relationship
   - Requires main data to have department field

2. **Use DISTINCT COUNT or COUNT_DISTINCT**
   - For metrics, use: `COUNT_DISTINCT(Incident_ID)`
   - Avoids double-counting from multiple permission records

3. **Aggregate Before Blending**
   - Pre-aggregate main data at same granularity as ACL
   - Create summary tables: Region + Site + Department + Aggregated Metrics

4. **Use Calculated Fields with CASE**
   - Create conditional metrics that handle duplicates
   - Example: `IF(ROW_NUMBER() = 1, Incident_Count, 0)`

### Performance Optimization

**Challenge**: Large ACL Expanded sheets (10,000+ records) can slow dashboard loading

**Optimization Strategies:**

1. **Apply Email Filter at Data Source Level (MOST IMPORTANT)**
   - In ACL Expanded data source settings, enable "Filter by email"
   - This filters ACL records BEFORE blending
   - Instead of loading 10,000 ACL records, only loads 10-100 for current user
   - **Massive performance improvement** - this is the key optimization

2. **Use Left Join (as recommended by Google)**
   - More efficient when ACL is pre-filtered by email
   - Reduces join complexity

3. **Use BigQuery Instead of Sheets (for very large organizations)**
   - Export ACL Expanded to BigQuery table
   - BigQuery handles large datasets more efficiently
   - Enables SQL-based filtering before Looker Studio
   - Consider when ACL has 50,000+ records or 500+ users

4. **Create Regional ACL Sheets (if needed)**
   - Split ACL Expanded into regional sheets
   - Dashboards connect to relevant regional ACL only
   - Reduces join complexity
   - Only needed for extremely large organizations

5. **Schedule Regular Expansions**
   - Run expansion script on schedule (daily/weekly)
   - Don't expand on every dashboard load
   - Balance freshness vs performance

### Handling Multiple Dashboards

**Best Practice**: Use ACL Expanded as single source of truth for all dashboards

**Architecture:**
```
┌──────────────────┐
│  ACL Expanded    │  ← Single permission source
│     (Master)     │
└────────┬─────────┘
         │
         ├──────────────┬──────────────┬──────────────┐
         │              │              │              │
         ▼              ▼              ▼              ▼
┌──────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Service     │ │  Financial  │ │  Incidents  │ │  HR         │
│  Dashboard   │ │  Dashboard  │ │  Dashboard  │ │  Dashboard  │
└──────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
```

**Benefits:**
- Consistent permissions across all reports
- Single point of maintenance
- Easier auditing and compliance
- Reduced risk of permission mismatches

---

## End User Control Points

### What End Users Need to Manage

The system is designed so **non-technical administrators** can manage all access control through Google Sheets without touching code.

### ACL Sheet Management

#### Adding a New User

**Steps:**
1. Open the ACL sheet
2. Add new row with user details:
   - Full Name
   - Position
   - Directorate (Development / Finance / Operations)
   - **Filter**: Choose "Own Region" or "Own Areas"
   - **Department/Function**: Enter specific function(s) or "ALL"
   - **Region**: Enter specific region or "ALL"
   - Service/Reporting Unit (optional)
   - **Local Sites**: Enter comma-separated sites (required for "Own Areas")
   - **Email**: User's email (must match Looker Studio login)
3. Run: ACL Expansion > Expand ACL by Filters
4. Verify expansion in "ACL Expanded" sheet
5. Permissions are now active in Looker Studio

**Decision Matrix for New Users:**

| User Type | Filter | Department | Region | Local Sites |
|-----------|--------|------------|--------|-------------|
| CEO / Director | Own Region | ALL | ALL | (leave empty) |
| Regional Head | Own Region | ALL | Specific region | (leave empty) |
| Department Head | Own Region | Specific dept | ALL | (leave empty) |
| Registered Manager | Own Areas | Specific or ALL | Specific region | List specific sites |
| Site Supervisor | Own Areas | Specific | Specific region | Single site |

#### Modifying Existing User Access

**Scenario 1: User Promoted to Regional Head**
- Change Filter from "Own Areas" to "Own Region"
- Set Department to "ALL" (if they should see all departments)
- Update Region to their new region
- Clear Local Sites field (no longer needed)
- Re-run expansion script

**Scenario 2: User Transfers to Different Region**
- Update Region field
- Update Local Sites if using "Own Areas" filter
- Re-run expansion script

**Scenario 3: User Gains Additional Functions**
- Update Department field: "Form 1" → "Form 1, Form 2, Form 3"
- Re-run expansion script

**Scenario 4: Temporary Access Grant**
- Add temporary row with same email but different permissions
- Both rows will be expanded (user gets union of permissions)
- Remove temporary row when access should be revoked
- Re-run expansion script

#### Removing User Access

**Steps:**
1. Delete user's row from ACL sheet
2. Re-run expansion script
3. User's records removed from ACL Expanded
4. Looker Studio will show "No data available" for that user

**Important**: Don't just hide the row - delete it completely to ensure clean removal

### Lookup Table Management

#### Adding New Regions/Sites

**When needed:**
- Opening new service location
- Restructuring operational regions
- Adding new service delivery models

**Steps:**
1. Open Region_Lookup sheet
2. Add new rows:
   ```
   Region: Ops Highlands
   Local Site: Inverness Hub
   
   Region: Ops Highlands
   Local Site: Fort Augustus
   ```
3. Re-run ACL expansion script
4. Update existing users' permissions if they should access new sites

**Important**: Any user with "Own Region" filter and matching region will **automatically** get access to new sites when expansion runs.

#### Adding New Departments/Functions

**When needed:**
- Introducing new service types
- Restructuring organizational departments
- Adding new reporting categories

**Steps:**
1. Open Departmen_Lookup sheet
2. Add new department name (exactly as it appears in operational data)
3. Re-run ACL expansion script
4. Any user with Department = "ALL" will **automatically** get access to new department

**Critical**: Ensure naming matches exactly between:
- Departmen_Lookup sheet
- Operational data sources
- Looker Studio data sources

### Running the Expansion Script

**When to run:**
- After any ACL sheet changes
- After lookup table updates
- Before users need updated permissions
- As part of regular maintenance (weekly/monthly)

**How to run:**
1. Open your Google Sheet
2. Menu: ACL Expansion > Expand ACL by Filters
3. Wait for completion message
4. Review ACL Expanded sheet for accuracy
5. Permissions active immediately in Looker Studio

**Frequency Recommendations:**
- **Real-time changes needed**: Run immediately after changes
- **Regular operations**: Schedule weekly expansion
- **Stable environment**: Run monthly with change management process

### Validation and Quality Control

**Pre-Expansion Checklist:**
- ☑ All emails valid and match Looker Studio user emails
- ☑ All regions exist in Region_Lookup
- ☑ All departments exist in Departmen_Lookup (if specific)
- ☑ Local Sites properly formatted (comma-separated, no extra spaces)
- ☑ Filter values are "Own Region" or "Own Areas" (case-sensitive)

**Post-Expansion Verification:**
1. Check record count - does it make sense?
   - Own Region with ALL: 100s-1000s records per user
   - Own Areas: 1-50 records per user typically
2. Spot-check specific users
   - Review their expanded records
   - Verify regions and sites are correct
3. Check for duplicates
   - Same Email + Region + Site + Department shouldn't repeat
4. Verify "ALL" expansions
   - Users with ALL should have records for every department

---

## Implementation Workflow

### Phase 1: Initial Setup (First Time)

**Week 1: Data Preparation**
1. Inventory all organizational data:
   - List all regions
   - List all local sites
   - List all departments/functions
   - Map sites to regions
2. Create lookup tables in Google Sheets
3. Validate naming conventions across all systems

**Week 2: ACL Population**
1. Create ACL sheet structure
2. Populate with all current users
3. Assign filters based on roles
4. Assign regions and departments
5. Document local sites for each manager

**Week 3: Script Implementation**
1. Install Google Apps Script
2. Test with small user subset
3. Validate expansion logic
4. Fix any data mismatches
5. Run full expansion

**Week 4: Looker Studio Integration**
1. Connect ACL Expanded to Looker Studio
2. Create data blends for each dashboard
3. Apply VIEWER_EMAIL() filters
4. Test with multiple user accounts
5. Document any issues

### Phase 2: Testing and Validation

**User Acceptance Testing:**
1. **Executive Testing** (CEO, Directors)
   - Should see organization-wide data
   - Verify all regions visible
   - Check all departments accessible

2. **Regional Head Testing**
   - Should see only their region
   - Verify all sites in region visible
   - Check department filtering

3. **Manager Testing**
   - Should see only their specific sites
   - Verify no access to other sites
   - Check function-level filtering

4. **Edge Case Testing**
   - New user with no ACL entry
   - User with multiple roles
   - Temporary access scenarios

**Test Scenarios Document:**
```
Test ID: ACL-001
User: jennifer.niven@sensescotland.org.uk
Expected: See Fundraising data for all regions
Steps:
1. Log into Dashboard A
2. Check regions visible
3. Check departments visible
4. Verify metrics match test dataset
Result: PASS / FAIL
Notes: [Any observations]
```

### Phase 3: Rollout

**Soft Launch:**
- Deploy to power users and early adopters
- Collect feedback for 2 weeks
- Make iterative improvements
- Document common questions

**Full Launch:**
- Email all users with instructions
- Provide training materials
- Schedule training sessions
- Establish support process

### Phase 4: Ongoing Maintenance

**Daily:**
- Monitor Looker Studio for access issues
- Respond to user access requests

**Weekly:**
- Review pending access changes
- Run expansion script (if changes made)
- Check for data quality issues

**Monthly:**
- Audit ACL entries for accuracy
- Review and remove inactive users
- Validate lookup tables against operational reality
- Document any organizational changes

**Quarterly:**
- Full permission review with department heads
- Update documentation
- Analyze expansion output for optimization opportunities
- Review security and compliance

---

## Security and Compliance

### Data Protection Principles

**Principle 1: Least Privilege**
- Users should have minimum access necessary for their role
- Default to restrictive (Own Areas), expand as needed (Own Region)
- Never give "ALL" access unless truly required

**Principle 2: Explicit Permission**
- No access is granted unless explicitly defined in ACL
- Inner join ensures no data leakage
- Missing ACL entry = no access (fail-secure)

**Principle 3: Auditability**
- Every permission includes source filter
- ACL changes tracked in Google Sheets revision history
- Expansion output serves as permission audit log

**Principle 4: Separation of Duties**
- ACL administrators ≠ Dashboard developers
- Data access ≠ Data modification
- Permission changes require re-expansion (not automatic)

### Audit Trail Maintenance

**What to Log:**
- Date/time of each expansion run
- User who ran expansion
- Number of records generated
- Any errors or warnings

**How to Log:**
1. Add "Expansion Log" sheet to workbook
2. Modify script to write log entries:
   ```javascript
   function logExpansion(recordCount, errors) {
     const logSheet = ss.getSheetByName('Expansion Log');
     logSheet.appendRow([
       new Date(),
       Session.getActiveUser().getEmail(),
       recordCount,
       errors || 'None'
     ]);
   }
   ```

**Audit Reports to Generate:**
- Monthly: "Who has access to what"
- Quarterly: "Access changes over time"
- On-demand: "Who can access Site X"

### Compliance Considerations

**GDPR / Data Privacy:**
- ACL system itself processes employee data (names, emails)
- Ensure users consent to data processing
- Implement data retention policy for ACL Expanded sheet
- Document processing in data protection impact assessment

**Industry-Specific Regulations:**
- Care sector regulations may require access logging
- Some data may require additional access restrictions
- Consult with compliance team on specific requirements

**Access Review Requirements:**
- Many compliance frameworks require periodic access reviews
- Schedule quarterly reviews with department heads
- Document review outcomes and changes made

### Security Best Practices

**Google Sheets Access Control:**
- ACL sheet: Restrict to HR and ACL administrators only
- ACL Expanded sheet: Read-only for most users
- Lookup tables: Protected ranges to prevent accidental changes
- Script editor: Restricted to administrators only

**Looker Studio Access:**
- Don't share dashboard links publicly
- Use "Viewer" or "Explorer" roles (not "Editor")
- Regularly audit who has dashboard access
- Remove access for departing employees promptly

**Email Verification:**
- Verify user emails match their actual login emails
- Watch for typos (jenniferniven vs jennifer.niven)
- Implement email validation in script if possible

---

## Troubleshooting and Maintenance

### Common Issues and Solutions

#### Issue 1: User Can't See Any Data

**Symptoms:**
- Looker Studio shows "No data available"
- Dashboard appears blank

**Causes & Solutions:**
1. **No ACL entry exists**
   - Solution: Add user to ACL sheet, run expansion
2. **Email mismatch**
   - Check: ACL email matches Looker Studio login email exactly
   - Common issue: firstname.lastname vs firstnamelastname
   - Solution: Update email in ACL to match Google account
3. **Email filter not applied to ACL data source**
   - Check: ACL data source has "Filter by email" enabled
   - This is the MOST COMMON issue
   - Solution: Edit ACL data source, enable email filtering on Email field
4. **Blend not configured**
   - Check: Data blend includes ACL Expanded
   - Check: Left join configured with proper join keys
   - Solution: Recreate data blend following documentation
5. **No matching records after join**
   - Check: Join keys match between data sources
   - Check: Region/Site/Department names match exactly
   - Solution: Standardize naming conventions

#### Issue 2: User Sees Too Much Data

**Symptoms:**
- User sees data for sites they shouldn't access
- User sees other regions' data

**Causes & Solutions:**
1. **Email filter not applied to ACL data source**
   - **Most likely cause** - ACL shows all users' permissions
   - Check: ACL data source settings
   - Solution: Enable "Filter by email" on ACL data source Email field
2. **Multiple ACL entries**
   - Check: Search for duplicate email entries in ACL
   - User gets union of all permissions
   - Solution: Remove unintended duplicate entries (or keep if intentional)
3. **Filter = "Own Region" instead of "Own Areas"**
   - User may have been incorrectly assigned broader access
   - Solution: Change filter to "Own Areas", specify local sites
4. **Wrong join type (Full Outer Join)**
   - Check: Blend configuration should be Left Join
   - Solution: Change to Left Join with ACL on right side

#### Issue 3: Duplicate Records in Dashboard

**Symptoms:**
- Metrics are inflated (e.g., 15 incidents instead of 5)
- Same record appears multiple times

**Causes & Solutions:**
1. **Missing Department in join**
   - Join on Region + Site only = one-to-many relationship
   - Solution: Add Department to join keys
2. **User has multiple function permissions for same site**
   - Expected if user should access multiple functions
   - Solution: Use COUNT_DISTINCT for metrics
3. **Data source itself has duplicates**
   - Check main data source for duplicate rows
   - Solution: Clean source data before blending

#### Issue 4: Slow Dashboard Performance

**Symptoms:**
- Dashboard takes 10+ seconds to load
- Timeout errors

**Causes & Solutions:**
1. **Large ACL Expanded sheet**
   - 10,000+ records can slow joins
   - Solution: Export to BigQuery, use SQL filters
2. **Complex blends**
   - Multiple data sources blended together
   - Solution: Pre-aggregate data before blending
3. **Inefficient metrics**
   - Calculated fields with complex logic
   - Solution: Simplify calculations, use aggregated sources

#### Issue 5: Expansion Script Fails

**Symptoms:**
- Error message when running script
- ACL Expanded sheet not updated

**Common Errors & Solutions:**

**Error: "Sheet not found"**
- Cause: ACL, Region_Lookup, or Departmen_Lookup sheet renamed or deleted
- Solution: Restore sheet with exact name, or update script

**Error: "Cannot read property"**
- Cause: Empty cells in critical fields
- Solution: Check for blank rows, remove or fill required fields

**Error: "Timeout"**
- Cause: Script runs too long (6-minute limit)
- Solution: Break ACL into smaller batches, run separately

**Error: "Authorization required"**
- Cause: Script not authorized to access sheets
- Solution: Run script manually first time to grant permissions

### Maintenance Tasks

**Weekly:**
- [ ] Check for pending access requests
- [ ] Run expansion if ACL modified
- [ ] Spot-check dashboard access for sample users

**Monthly:**
- [ ] Review expansion log for errors
- [ ] Verify lookup tables still match operational reality
- [ ] Clean up ACL entries for departed employees
- [ ] Generate access report for department heads

**Quarterly:**
- [ ] Full access audit with stakeholders
- [ ] Update documentation for any process changes
- [ ] Review and optimize script performance
- [ ] Train new administrators if needed

**Annually:**
- [ ] Complete security review
- [ ] Update compliance documentation
- [ ] Archive old expansion logs
- [ ] Strategic review: Is the access model still appropriate?

### Getting Help

**Escalation Path:**
1. **First Line**: Check this documentation
2. **Second Line**: Check Google Sheets revision history (was something changed?)
3. **Third Line**: Contact ACL administrator
4. **Fourth Line**: Contact Looker Studio administrator
5. **Final**: Contact technical support / system developer

**Information to Provide When Reporting Issues:**
- User email address
- Expected behavior vs actual behavior
- Screenshot of error or unexpected result
- Steps to reproduce
- Recent changes to ACL or dashboards

---

## Best Practices

### Organizational Best Practices

1. **Single Source of Truth**
   - Maintain ACL as THE authoritative permission source
   - Don't create manual permission lists elsewhere
   - All dashboards use same ACL Expanded sheet

2. **Change Management**
   - Document why access changes were made
   - Get approval for "Own Region" or "ALL" access grants
   - Implement access request form/process

3. **Regular Reviews**
   - Schedule quarterly access reviews
   - Remove access for departed employees promptly
   - Challenge "ALL" access grants periodically

4. **Documentation**
   - Keep this documentation updated
   - Document organizational structure changes
   - Maintain training materials for new administrators

### Technical Best Practices

1. **Naming Conventions**
   - Use consistent naming across all systems
   - Document standard names for regions/sites/departments
   - Create data dictionary if needed

2. **Version Control**
   - Make copy of ACL before major changes
   - Name copies with date: "ACL Backup 2025-10-24"
   - Keep expansion script in version control if possible

3. **Testing Before Production**
   - Test expansion with sample users first
   - Verify dashboard access before rolling out widely
   - Have rollback plan for failed changes

4. **Performance Optimization**
   - Keep ACL Expanded sheet clean (no old data)
   - Consider BigQuery for very large organizations
   - Optimize lookup tables (remove unused entries)

5. **Error Handling**
   - Script should log errors, not fail silently
   - Alert administrators when expansion fails
   - Implement validation checks before expansion

### User Experience Best Practices

1. **Clear Communication**
   - Explain to users why they see certain data
   - Set expectations for what they can access
   - Provide self-service access request process

2. **Helpful Error Messages**
   - If no data available, explain why
   - Provide contact info for access requests
   - Use dashboard filters to show "No permission" vs "No data"

3. **Training**
   - Train users on what they should see
   - Explain how to interpret filtered data
   - Provide FAQ for common questions

4. **Feedback Loop**
   - Collect user feedback on access issues
   - Iterate on access model based on real needs
   - Adjust filters/regions as organization evolves

---

## Appendix

### Glossary

- **ACL (Access Control List)**: Master list defining who can access what
- **Filter**: Rule type determining scope of access (Own Region / Own Areas)
- **Own Region**: User can access all sites in their region(s)
- **Own Areas**: User can access only specified local sites
- **Department/Function**: Service type or organizational function
- **Local Site**: Physical location where services are delivered
- **Expansion**: Process of converting ACL rules into granular permission records
- **Data Blending**: Combining multiple data sources in Looker Studio
- **Inner Join**: Join type that only returns matching records from both sources
- **VIEWER_EMAIL()**: Looker Studio function returning current user's email
- **Row-Level Security**: Filtering data so users see only permitted rows

### Quick Reference Card

**Adding New User:**
1. Add row to ACL sheet
2. Fill: Name, Filter, Department, Region, Sites, Email
3. Run: ACL Expansion > Expand ACL by Filters
4. Done!

**Filter Selection Guide:**
- **Senior Leadership** (see everything): Own Region + ALL departments + ALL regions
- **Regional Head** (see their region): Own Region + ALL departments + Specific region
- **Department Head** (see their dept everywhere): Own Region + Specific dept + ALL regions
- **Manager** (see their sites only): Own Areas + Specific dept + List sites

**Common Formulas:**
- Filter all sites in a region: `=FILTER(Region_Lookup!B:B, Region_Lookup!A:A="Ops Central")`
- Count user's permissions: `=COUNTIF('ACL Expanded'!A:A, "email@example.com")`
- List user's sites: `=UNIQUE(FILTER('ACL Expanded'!C:C, 'ACL Expanded'!A:A="email@example.com"))`

---

## Conclusion

The ACL Expansion System is a powerful tool for managing complex, hierarchical access control in a scalable and maintainable way. By centralizing permission management in Google Sheets and automatically expanding rules into granular permissions, this system:

- **Reduces administrative overhead** from hours to minutes
- **Improves security** through consistent, auditable access control
- **Enhances user experience** by showing relevant data automatically
- **Scales with your organization** as you add regions, sites, and services
- **Integrates seamlessly** with Looker Studio for row-level security

Success with this system requires:
1. Clean, consistent master data
2. Regular maintenance and auditing
3. Clear communication with end users
4. Proper Looker Studio configuration
5. Ongoing optimization as needs evolve

By following this documentation and best practices, administrators can confidently manage access control for hundreds of users across complex organizational structures without technical expertise beyond basic spreadsheet skills.

---

**Document Version**: 1.0  
**Last Updated**: October 2025  
**Next Review Date**: January 2026  
**Owner**: ICT / Data Governance Team  
**Approver**: None
