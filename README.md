# bynry-backend-case-study

## Inventory Management System for B2B SaaS (StockFlow)

**Candidate:** Kabir Singh  
**Role Applied:** Backend Engineering Intern  
**Primary Backend Stack:** Java, Spring Boot 
Note: Java/Spring Boot is used to demonstrate my primary backend expertise; concepts are transferable to Flask/Python.

---

## Notes
- Original problem statement was provided in Flask / Python; implementation is adapted to Spring Boot while preserving business logic.
- Solution is implemented in Spring Boot (Java) as my primary backend stack
- Business logic remains framework-agnostic
- Explicit assumptions are documented due to intentionally incomplete requirements

---

## Overview

StockFlow is a B2B inventory management platform that enables small businesses to:
- Track products across multiple warehouses
- Maintain accurate inventory levels
- Manage supplier relationships
- Support bundled products

---

# Part 1: Code Review & Debugging

## 1. Issues Identified
1. No input validation  
2. SKU uniqueness not enforced  
3. Incorrect product–warehouse coupling  
4. No transaction management  
5. No exception handling  
6. Price precision issues  
7. Missing company / tenant context  

---

## 2. Impact in Production
- Invalid or missing data can crash the API or corrupt the database  
- Duplicate SKUs break inventory tracking and reporting  
- Single-warehouse coupling prevents scalability  
- Partial writes lead to inconsistent data  
- Poor error handling reduces debuggability  
- Floating-point prices cause billing inaccuracies  
- Lack of tenant isolation risks data leakage  

---

## 3. Corrected Implementation (Spring Boot – Java)

```java
@PostMapping("/api/products")
@Transactional // @Transactional ensures atomicity between Product and Inventory creation
public ResponseEntity<?> createProduct(@RequestBody CreateProductRequest request) {

    if (request.getName() == null || request.getSku() == null || request.getPrice() == null) {
        return ResponseEntity.badRequest()
                .body("Name, SKU, and price are required");
    }

    if (productRepository.existsBySku(request.getSku())) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body("SKU already exists");
    }

    try {
        Product product = new Product();
        product.setName(request.getName());
        product.setSku(request.getSku());
        product.setPrice(request.getPrice()); // BigDecimal
        product.setCompanyId(request.getCompanyId());

        productRepository.save(product);

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
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Failed to create product");
    }
}
```

---

# Part 2: Database Design

## 1. Schema Design

### Company
- id (BIGINT, PK)
- name (VARCHAR)
- created_at (TIMESTAMP)

Represents a tenant in the system.

### Warehouse
- id (BIGINT, PK)
- company_id (BIGINT, FK → Company.id)
- name (VARCHAR)
- location (VARCHAR)
- created_at (TIMESTAMP)

A company can have multiple warehouses.

### Product
- id (BIGINT, PK)
- company_id (BIGINT, FK → Company.id)
- name (VARCHAR)
- sku (VARCHAR)
- price (DECIMAL(10,2))
- is_bundle (BOOLEAN)
- created_at (TIMESTAMP)

Constraint: UNIQUE(company_id, sku)

### Inventory
- id (BIGINT, PK)
- product_id (BIGINT, FK → Product.id)
- warehouse_id (BIGINT, FK → Warehouse.id)
- quantity (INT)
- updated_at (TIMESTAMP)

Supports multiple warehouses per product.

### Inventory_History
- id (BIGINT, PK)
- inventory_id (BIGINT, FK → Inventory.id)
- change_quantity (INT)
- reason (VARCHAR)
- created_at (TIMESTAMP)

Tracks inventory changes.

### Supplier
- id (BIGINT, PK)
- name (VARCHAR)
- contact_email (VARCHAR)

### Product_Supplier
- product_id (BIGINT, FK → Product.id)
- supplier_id (BIGINT, FK → Supplier.id)

Many-to-many relationship.

### Bundle_Product
- bundle_id (BIGINT, FK → Product.id)
- child_product_id (BIGINT, FK → Product.id)
- quantity (INT)

Supports bundled products.

---

## 2. Missing Requirements / Questions
- Is SKU unique globally or per company?
- Can a product have multiple suppliers?
- How is bundle pricing calculated?
- Can inventory go negative?
- What events should be recorded in inventory history?
- Should soft deletes be used?
- What defines recent sales activity?
- Are multiple units of measurement supported?

---

## 3. Design Decisions
- Separate Inventory enables multi-warehouse support
- SKU uniqueness ensures data integrity
- Inventory history enables auditing and analytics
- Foreign keys enforce referential integrity
- Indexes improve query performance
- Company-level isolation supports SaaS multi-tenancy

---

# Part 3: API Implementation – Low Stock Alerts

## Assumptions
- Each product has a low_stock_threshold
- Recent sales activity means last 30 days
- Inventory is tracked per warehouse
- Each product has one primary supplier
- Inventory reflects real-time available stock
- API is scoped by company_id

---

## Endpoint
GET /api/companies/{company_id}/alerts/low-stock

---

## Implementation (Spring Boot – Java)

```java
@GetMapping("/api/companies/{companyId}/alerts/low-stock")
public ResponseEntity<?> getLowStockAlerts(@PathVariable Long companyId) {

    List<Map<String, Object>> alerts = new ArrayList<>();
    List<Inventory> inventories =
            inventoryRepository.findByCompanyId(companyId);

    for (Inventory inventory : inventories) {
        Product product = inventory.getProduct();

        if (!salesRepository.hasRecentSales(product.getId(), 30)) {
            continue;
        }

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
```

---

## Edge Cases Handled
- Products without recent sales are ignored
- Products without suppliers return null
- Alerts are generated per warehouse
- Zero or negative inventory triggers alerts
- Empty result returns an empty list

---

## Submission Notes
- All assumptions are explicitly documented
- Design prioritizes scalability, correctness, and SaaS isolation
- Prepared to discuss trade-offs and improvements in live session

---

## Evaluation Alignment
- Clean backend design
- Real-world business constraints
- Defensive coding practices
- Clear communication of decisions
- Comfort working with incomplete requirements

---

## Additional Information

• This case study is submitted as part of the Backend Engineering Intern application at Bynry Inc.
• The solution is intentionally designed to be framework-agnostic, with implementation demonstrated using Java & Spring Boot.
• All assumptions have been explicitly documented due to intentionally incomplete requirements.
• I am comfortable implementing the same solution in Flask / Python if required.
• I look forward to discussing design decisions, trade-offs, and improvements during the live discussion.

Case Study Document Link
[Doc_Link] (https://docs.google.com/document/d/1-t-3LmWOm4NQzMl0NTQSNz6WN4V-W8dO6skm8IxS054/edit?usp=sharing)

Access: Anyone with the link (Viewer)

---

## Final Notes

• This submission focuses on correctness, scalability, and real-world SaaS considerations.
• Design decisions were made with multi-tenancy, data integrity, and future extensibility in mind.
• Trade-offs and alternative approaches can be discussed during the live interview.
• Thank you for reviewing my submission.


