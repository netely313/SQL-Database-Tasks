# SQL Database Tasks

## Project Overview

This document contains SQL queries developed for the database analysis project. The tasks were designed to support various business stakeholders with critical data insights and operational reporting needs.

### Business Context
This project involves a fictional manufacturing company that requires comprehensive data analysis across its product catalogue, manufacturing operations, and sales processes. The following tasks address specific business requirements from different departments:

**Product Management Team** - Needs detailed product information with proper categorisation to support inventory management and product strategy decisions.

**Manufacturing Operations** - Requires work order analysis to optimise production scheduling, cost management, and facility utilisation.

**Data Quality Assurance** - Identified several problematic queries in the existing codebase that need debugging and optimisation to ensure accurate business reporting.

### Stakeholder Requirements Summary

- **Task 1:** Product catalog analysis with hierarchical categorization (Product → Subcategory → Category)
- **Task 2:** Manufacturing cost and efficiency analysis for a specific month and year of operations
- **Task 3:** Code review and debugging of existing queries used in production reports
---

## Task 1: Product Data Extraction

### 1.1 Extract Products with Subcategories

```sql
-- Query to retrieve all products that have subcategories
-- Uses INNER JOIN to ensure only products with subcategories are included
SELECT 
    p.ProductID,                    -- Unique product identifier
    p.Name,                         -- Product name
    p.ProductNumber,                -- Product SKU/model number
    p.Size,                         -- Product size specification
    p.Color,                        -- Product color
    p.ProductSubcategoryId,         -- Foreign key to subcategory
    ps.Name AS SubCategoryName      -- Subcategory name for better readability
FROM `product` AS p
JOIN `productsubcategory` AS ps     -- INNER JOIN excludes products without subcategories
    ON p.ProductSubcategoryID = ps.ProductSubcategoryID 
ORDER BY SubCategoryName;           -- Sort alphabetically by subcategory name
```

### 1.2 Add Product Category Name

```sql
-- Enhanced query to include full product hierarchy: Product → Subcategory → Category
-- Provides complete categorization for better product management
SELECT 
    p.ProductID,                    -- Unique product identifier
    p.Name,                         -- Product name
    p.ProductNumber,                -- Product SKU/model number
    p.Size,                         -- Product size specification
    p.Color,                        -- Product color
    p.ProductSubcategoryId,         -- Foreign key to subcategory
    ps.Name AS SubCategoryName,     -- Subcategory name (e.g., "Mountain Bikes")
    pc.Name AS CategoryName         -- Main category name (e.g., "Bikes")
FROM `product` AS p
JOIN `productsubcategory` AS ps     -- First join to get subcategory details
    ON p.ProductSubcategoryID = ps.ProductSubcategoryID 
JOIN `productcategory` AS pc        -- Second join to get parent category
    ON ps.ProductCategoryID = pc.ProductCategoryID
ORDER BY CategoryName;              -- Sort by main category for better organization
```

### 1.3 Select Most Expensive Bikes

```sql
-- Query to find premium bikes currently for sale
-- Business case: Identify high-value inventory for sales team focus
SELECT
    p.ProductID,                    -- Unique product identifier
    p.Name,                         -- Product name
    p.ListPrice,                    -- Retail price (added for price analysis)
    p.ProductNumber,                -- Product SKU/model number
    p.Size,                         -- Product size specification
    p.Color,                        -- Product color
    p.ProductSubcategoryId,         -- Foreign key to subcategory
    ps.Name AS SubCategoryName,     -- Subcategory name
    pc.Name AS CategoryName         -- Main category name
FROM `product` AS p
JOIN `productsubcategory` AS ps     -- Join to get subcategory details
    ON p.ProductSubcategoryID = ps.ProductSubcategoryID
JOIN `productcategory` AS pc        -- Join to get category details
    ON ps.ProductCategoryID = pc.ProductCategoryID
WHERE p.ListPrice > 2000            -- Filter: Only premium products (>$2000)
    AND p.SellEndDate IS NULL       -- Filter: Only currently active products
    AND pc.Name = 'Bikes'           -- Filter: Only bikes category
ORDER BY p.ListPrice DESC;          -- Sort: Most expensive first
```

## Task 2: Work Order Analysis

### 2.1 Aggregated Work Order Query

```sql
-- Manufacturing operations summary for January 2004
-- Business case: Monthly performance analysis by facility location
SELECT 
    LocationID,                                 -- Manufacturing facility identifier
    COUNT(DISTINCT ProductID) AS No_Unique_Products,    -- Product variety per location
    COUNT(DISTINCT WorkOrderID) AS No_Unique_WorkOrders, -- Work order volume per location
    SUM(ActualCost) AS TotalCost               -- Total manufacturing cost per location
FROM `workorderrouting`
WHERE ActualStartDate >= '2004-01-01'          -- Filter: January 2004 start
    AND ActualStartDate < '2004-02-01'         -- Filter: Before February (exclusive)
GROUP BY LocationID                             -- Aggregate: Group by facility
ORDER BY LocationID DESC;                      -- Sort: Descending location ID
```

### 2.2 Add Location Names and Average Duration

```sql
-- Enhanced manufacturing operations report with location details and efficiency metrics
-- Business case: Facility performance comparison including processing time analysis
SELECT
    w.LocationID,                               -- Manufacturing facility identifier
    l.Name AS LocationName,                     -- Human-readable location name
    COUNT(DISTINCT w.ProductID) AS No_Unique_Products,      -- Product variety per location
    COUNT(DISTINCT w.WorkOrderID) AS No_Unique_WorkOrders,  -- Work order volume per location
    SUM(w.ActualCost) AS TotalCost,            -- Total manufacturing cost per location
    -- Calculate average processing time in days for efficiency analysis
    ROUND(AVG(DATE_DIFF(DATE(w.ActualEndDate), DATE(w.ActualStartDate), DAY)), 2) AS AverageDayDifference
FROM `workorderrouting` AS w
JOIN `location` AS l                            -- Join to get location names
    ON w.LocationID = l.LocationID
WHERE w.ActualStartDate >= '2004-01-01'        -- Filter: January 2004 start
    AND w.ActualStartDate < '2004-02-01'       -- Filter: Before February (exclusive)
GROUP BY w.LocationID, l.Name                   -- Aggregate: Group by location and name
ORDER BY w.LocationID DESC;                     -- Sort: Descending location ID
```

### 2.3 Expensive Work Orders

```sql
-- Identify high-cost work orders for cost management analysis
-- Business case: Flag work orders exceeding budget thresholds for review
SELECT
    w.WorkOrderID,                             -- Work order identifier
    SUM(w.ActualCost) AS TotalActualCost       -- Total cost per work order
FROM `workorderrouting` AS w
WHERE w.ActualStartDate >= '2004-01-01'       -- Filter: January 2004 start
    AND w.ActualStartDate < '2004-02-01'      -- Filter: Before February (exclusive)
GROUP BY w.WorkOrderID                         -- Aggregate: Sum costs per work order
HAVING SUM(w.ActualCost) > 300;               -- Filter: Only expensive work orders (>$300)
```

## Task 3: Query Debugging and Optimization

### 3.1 Special Offers Query Investigation

```sql
-- Query to analyze sales orders with special offer details
-- POTENTIAL ISSUE: LEFT JOIN may include orders without special offers
SELECT 
    sales_detail.SalesOrderId,      -- Order identifier
    sales_detail.OrderQty,          -- Quantity ordered
    sales_detail.UnitPrice,         -- Price per unit
    sales_detail.LineTotal,         -- Total line amount (qty * unit price)
    sales_detail.ProductId,         -- Product identifier
    sales_detail.SpecialOfferID,    -- Special offer identifier (may be NULL)
    spec_offer.Category,            -- Special offer category (may be NULL)
    spec_offer.Description          -- Special offer description (may be NULL)
FROM `salesorderdetail` AS sales_detail
-- LEFT JOIN includes ALL sales orders, even those without special offers
-- This may cause "numbers to be off" if business only wants orders WITH special offers
LEFT JOIN `specialoffer` AS spec_offer
    ON sales_detail.SpecialOfferID = spec_offer.SpecialOfferID
ORDER BY LineTotal DESC;            -- Sort by highest value orders first
```

**Potential Issues:**
- The LEFT JOIN may be including records without special offers, which could skew numerical calculations
- Consider using INNER JOIN if you only want orders that actually have special offers
- Check for duplicate records if there are multiple special offers per order

### 3.2 Vendor Information Query Fix

**Original (Broken) Query:**
```sql
-- BROKEN QUERY - Multiple syntax errors identified below
SELECT a.VendorId AS Id,
       vendor_contact.ContactId, 
       vendor_contact.ContactTypeId, 
       a.Name, 
       a.CreditRating, 
       a.ActiveFlag, 
       c.AddressId,
       adr.City
FROM 'vendor' AS a                          -- ERROR: Single quotes instead of backticks
LEFT join 'vendorcontact' AS vendor_contact -- ERROR: Single quotes instead of backticks
ON a.VendorId = vendor_contact.VendorId 
LEFT join 'vendoraddress' AS c              -- ERROR: Single quotes instead of backticks
ON vendor_contact.VendorId = c.VendorId     -- ERROR: Wrong join logic
LEFT 'address' AS adr                       -- ERROR: Missing "JOIN" keyword, wrong quotes
ON c.AddressId = adr.AddressId
```

**Corrected Query:**
```sql
-- CORRECTED QUERY - Comprehensive vendor information with proper joins
-- Business case: Complete vendor contact and address information for procurement team
SELECT 
    v.VendorId AS Id,               -- Vendor unique identifier
    vc.ContactId,                   -- Contact person identifier
    vc.ContactTypeId,               -- Type of contact (primary, billing, etc.)
    v.Name AS VendorName,           -- Vendor company name
    v.CreditRating,                 -- Vendor credit worthiness score
    v.ActiveFlag,                   -- Whether vendor is currently active
    va.AddressId,                   -- Address record identifier
    a.City                          -- City where vendor is located
FROM `vendor` AS v                  -- Main vendor table (corrected quotes)
LEFT JOIN `vendorcontact` AS vc     -- Contact information (optional)
    ON v.VendorId = vc.VendorId     -- Join on vendor ID
LEFT JOIN `vendoraddress` AS va     -- Address linking table (optional)
    ON v.VendorId = va.VendorId     -- Fixed: Join on vendor, not contact
LEFT JOIN `address` AS a            -- Address details (optional)
    ON va.AddressId = a.AddressId;  -- Join on address ID
```

**Improvements for Better Readability:**
1. **Consistent alias naming:** Use meaningful, consistent aliases (v for vendor, vc for vendorcontact, etc.)
2. **Proper indentation:** Align JOIN conditions for better readability
3. **Clear column aliases:** Use descriptive aliases like `VendorName` instead of just `Name`
4. **Logical join order:** Join tables in a logical sequence
5. **Fix join logic:** The vendoraddress should join on VendorId from vendor table, not contact table
6. **Add semicolons:** End statements with semicolons for consistency
