# bynry-backend-case-study

## Inventory Management System for B2B SaaS (StockFlow)

### Candidate
**Name:** Kabir Singh  
**Role Applied:** Backend Engineering Intern  
**Primary Stack:** Java, Spring Boot  

---

## Notes
- Original problem statement was provided in **Flask / Python**
- Solution is implemented in **Spring Boot (Java)** as my primary backend stack
- Business logic and architectural reasoning remain framework-agnostic
- Explicit assumptions are documented due to intentionally incomplete requirements

---

## Overview

You are joining a team building **StockFlow**, a B2B inventory management platform where small businesses:
- Track products across **multiple warehouses**
- Manage **inventory levels**
- Maintain **supplier relationships**
- Support **bundled products**

This submission covers:
1. Code Review & Debugging
2. Database Design
3. API Implementation for Low Stock Alerts

---

## Time Allocation (As Per Instructions)

- **Take-Home Portion:** 90 minutes
- **Live Discussion:** 30–45 minutes (scheduled separately)

---

# Part 1: Code Review & Debugging

### Given Problem
A Flask API endpoint for creating a product compiles but fails in production.

---

## 1. Issues Identified

### 1. No Input Validation
The API directly accesses request fields without checking for presence or correctness.

### 2. SKU Uniqueness Not Enforced
Business rule requires SKU uniqueness, but no check exists.

### 3. Incorrect Product–Warehouse Coupling
Product is directly tied to a single warehouse, violating the requirement that products can exist in multiple warehouses.

### 4. No Transaction Management
Product creation and inventory creation are committed separately, leading to partial data persistence.

### 5. No Exception Handling
Any runtime or database error causes an unhandled failure.

### 6. Price Precision Issue
Floating-point values can introduce rounding errors for monetary values.

### 7. No Company / Tenant Context
Critical for a multi-tenant B2B SaaS system.

---

## 2. Impact in Production

- Invalid or missing data can crash the API or corrupt the database
- Duplicate SKUs break inventory tracking, reporting, and integrations
- System cannot scale to multiple warehouses
- Partial writes lead to orphaned products without inventory
- Poor error handling reduces debuggability and user trust
- Rounding errors can cause billing discrepancies
- Lack of tenant isolation can lead to serious data leaks

---

### 3. Corrected Implementation (Spring Boot – Java)


@PostMapping("/api/products")
@Transactional
public ResponseEntity<?> createProduct(@RequestBody CreateProductRequest request) {

    // Input validation
    if (request.getName() == null || request.getSku() == null || request.getPrice() == null) {
        return ResponseEntity.badRequest()
                .body("Name, SKU, and price are required");
    }

    // SKU uniqueness check
    if (productRepository.existsBySku(request.getSku())) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body("SKU already exists");
    }

    try {
        // Create product (not tied directly to warehouse)
        Product product = new Product();
        product.setName(request.getName());
        product.setSku(request.getSku());
        product.setPrice(request.getPrice()); // BigDecimal
        product.setCompanyId(request.getCompanyId());

        productRepository.save(product);

        // Create inventory entry
        Inventory inventory = new Inventory();
        inventory.setProduct(product);
        inventory.setWarehouseId(request.getWarehouseId());
        inventory.setQuantity(
                request.getInitialQuantity() != null ? request.getInitialQuantity() : 0
        );

        inventoryRepository.save(inventory);

        return ResponseEntity.status(HttpStatus.CREATED)
                .body(Map.of(
                        "message", "Product created successfully",
                        "product_id", product.getId()
                ));

    } catch (Exception e) {
        // Rollback handled automatically by @Transactional
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Failed to create product");
    }
}
---

## Part 2: Database Design
1. Schema Design
Company

id (BIGINT, PK)

name (VARCHAR)

created_at (TIMESTAMP)

Represents a tenant in the system.

Warehouse

id (BIGINT, PK)

company_id (BIGINT, FK → Company.id)

name (VARCHAR)

location (VARCHAR)

created_at (TIMESTAMP)

A company can own multiple warehouses.

Product

id (BIGINT, PK)

company_id (BIGINT, FK → Company.id)

name (VARCHAR)

sku (VARCHAR)

price (DECIMAL(10,2))

is_bundle (BOOLEAN)

created_at (TIMESTAMP)

Constraint: UNIQUE(company_id, sku)

Inventory

id (BIGINT, PK)

product_id (BIGINT, FK → Product.id)

warehouse_id (BIGINT, FK → Warehouse.id)

quantity (INT)

updated_at (TIMESTAMP)

Supports multiple warehouses per product.

Inventory_History

id (BIGINT, PK)

inventory_id (BIGINT, FK → Inventory.id)

change_quantity (INT)

reason (VARCHAR)

created_at (TIMESTAMP)

Tracks all inventory changes for auditing.

Supplier

id (BIGINT, PK)

name (VARCHAR)

contact_email (VARCHAR)

Product_Supplier

product_id (BIGINT, FK → Product.id)

supplier_id (BIGINT, FK → Supplier.id)

Many-to-many relationship.

Bundle_Product

bundle_id (BIGINT, FK → Product.id)

child_product_id (BIGINT, FK → Product.id)

quantity (INT)

Supports bundled products.

2. Missing Requirements / Questions

Is SKU unique globally or per company?

Can a product have multiple suppliers?

How is bundle pricing calculated?

Can inventory go negative?

What events should be logged in inventory history?

Should soft deletes be used?

What defines “recent sales activity”?

Are multiple units of measurement supported?

3. Design Decisions

Separate Inventory enables multi-warehouse support

SKU uniqueness ensures data integrity

Inventory history enables audits and analytics

Foreign keys enforce referential integrity

Indexes on SKU, product_id, warehouse_id improve performance

Company-level isolation supports SaaS multi-tenancy

Part 3: API Implementation – Low Stock Alerts
Assumptions

Each product has a low_stock_threshold

Recent sales = sales within last 30 days

Inventory is tracked per warehouse

Each product has one primary supplier

Quantity reflects real-time available stock

Requests are scoped by company_id

Endpoint
GET /api/companies/{company_id}/alerts/low-stock

Implementation (Spring Boot – Java)
@GetMapping("/api/companies/{companyId}/alerts/low-stock")
public ResponseEntity<?> getLowStockAlerts(@PathVariable Long companyId) {

    List<Map<String, Object>> alerts = new ArrayList<>();
    List<Inventory> inventories =
            inventoryRepository.findByCompanyId(companyId);

    for (Inventory inventory : inventories) {
        Product product = inventory.getProduct();

        // Skip products without recent sales
        if (!salesRepository.hasRecentSales(product.getId(), 30)) {
            continue;
        }

        // Low stock check
        if (inventory.getQuantity() < product.getLowStockThreshold()) {
            Supplier supplier = product.getPrimarySupplier();

            Map<String, Object> alert = new HashMap<>();
            alert.put("product_id", product.getId());
            alert.put("product_name", product.getName());
            alert.put("sku", product.getSku());
            alert.put("warehouse_id", inventory.getWarehouse().getId());
            alert.put("warehouse_name", inventory.getWarehouse().getName());
            alert.put("current_stock", inventory.getQuantity());
            alert.put("threshold", product.getLowStockThreshold());
            alert.put("days_until_stockout",
                    product.getEstimatedDaysUntilStockout());

            alert.put("supplier", supplier != null ? Map.of(
                    "id", supplier.getId(),
                    "name", supplier.getName(),
                    "contact_email", supplier.getContactEmail()
            ) : null);

            alerts.add(alert);
        }
    }

    return ResponseEntity.ok(Map.of(
            "alerts", alerts,
            "total_alerts", alerts.size()
    ));
}

Edge Cases Handled

Products without recent sales are ignored

Products without suppliers return null

Alerts are warehouse-specific

Zero or negative stock triggers alerts

Empty results return an empty list

Submission Notes

All assumptions are explicitly documented

Design prioritizes scalability, correctness, and SaaS isolation

Prepared to discuss trade-offs, alternatives, and improvements during live session

Evaluation Focus Alignment

Clean backend design

Real-world business constraints

Defensive coding and scalability

Clear communication of decisions

Comfort working with incomplete requirements

Thank you for reviewing my submission.


If you want, next I can:
- Trim this to **exactly 90-minute expectation**
- Add **architecture diagram (text-based)**
- Review it like a **Bynry interviewer would** 🔍
