                     ## Bynry Backend Engineering Case Study

           ## Introduction

I first tried to understand the problem and then worked on each part step by step. I didn’t try to over-optimize things, just focused on writing simple and correct logic based on the requirements.


                 ## Part 1: Code Review & Debugging

While reading the code, I tried to follow the flow first and then I found some issues which can cause problems in real use.

     ### Issues I found

1. Product and inventory are saved separately.  
   If inventory fails, product will still be saved which is not correct.

2. No validation on input data.  
   If any field is missing, the API can crash.

3. SKU is not unique.  
   Same SKU can be inserted multiple times.

4. Product is directly linked to warehouse.  
   But a product can be in multiple warehouses.

5. No error handling.  
   If something goes wrong, user won’t get proper response.

6. No check if warehouse exists.  
   Invalid warehouse_id can be passed.

7. No validation for price and quantity.  
   Negative values can be stored.

    ### Impact

These issues can cause wrong data, duplicate entries, and API failures. It will also be difficult to scale the system later.

                    ### Fix (Spring Boot style)  

```java  

@PostMapping("/api/products")
@Transactional
public ResponseEntity<?> createProduct(@RequestBody ProductRequest request) {

    if (request.getName() == null || request.getSku() == null) {
        return ResponseEntity.badRequest().body("Missing required fields");
    }

    if (request.getPrice() <= 0) {
        return ResponseEntity.badRequest().body("Invalid price");
    }

    if (productRepository.existsBySku(request.getSku())) {
        return ResponseEntity.badRequest().body("SKU already exists");
    }

    Optional<Warehouse> warehouseOpt = warehouseRepository.findById(request.getWarehouseId());
    if (warehouseOpt.isEmpty()) {
        return ResponseEntity.badRequest().body("Invalid warehouse");
    }

    try {
        Product product = new Product();
        product.setName(request.getName());
        product.setSku(request.getSku());
        product.setPrice(request.getPrice());

        productRepository.save(product);

        Inventory inv = new Inventory();
        inv.setProductId(product.getId());
        inv.setWarehouseId(request.getWarehouseId());
        inv.setQuantity(request.getInitialQuantity());

        inventoryRepository.save(inv);

        return ResponseEntity.ok(Map.of(
                "message", "Product created",
                "product_id", product.getId()
        ));

    } catch (Exception e) {
        return ResponseEntity.status(500).body("Something went wrong");
    }
}

```

               ## Part 2: Database Design

I tried to keep the database simple but flexible so it can support multiple warehouses and suppliers.

          ## Schema

companies(id, name)

warehouses(id, company_id, name, location)

products(id, name, sku, price)

inventory(id, product_id, warehouse_id, quantity)

suppliers(id, name, email)

product_suppliers(product_id, supplier_id)

inventory_history(id, product_id, warehouse_id, change_qty, created_at)

product_bundles(parent_product_id, child_product_id, quantity)

          Questions I would ask - 

Is SKU unique globally or per company?
Can a product have multiple suppliers?
What is considered as recent sales?
How will bundle pricing work?
How much history do we need to store?


           Design Decisions
Inventory table is used to connect product and warehouse.
Separate table for inventory history to track changes.
Product_suppliers table is used for flexibility.
Bundle table is used for product combinations. 


              ## Part 3: API Implementation

 Here I tried to build a simple API to get low stock products.

Approach

First I check if the company exists. Then I get all warehouses of that company. After that I get inventory and check which products have stock less than threshold. I also check if the product has recent sales. Then I add supplier info and calculate days until stockout.

                      Code
```java

@GetMapping("/api/companies/{companyId}/alerts/low-stock")
public ResponseEntity<?> getLowStock(@PathVariable Long companyId) {

    Optional<Company> comp = companyRepository.findById(companyId);
    if (comp.isEmpty()) {
        return ResponseEntity.status(404).body("Company not found");
    }

    List<Map<String, Object>> alerts = new ArrayList<>();

    List<Warehouse> warehouses = warehouseRepository.findByCompanyId(companyId);

    if (warehouses.isEmpty()) {
        return ResponseEntity.ok(Map.of("alerts", alerts, "total_alerts", 0));
    }

    for (Warehouse w : warehouses) {

        List<Inventory> invList = inventoryRepository.findByWarehouseId(w.getId());

        for (Inventory inv : invList) {

            Optional<Product> prodOpt = productRepository.findById(inv.getProductId());
            if (prodOpt.isEmpty()) continue;

            Product p = prodOpt.get();

            int stock = inv.getQuantity();
            int threshold = p.getThreshold();

            if (stock < threshold) {

                boolean recent = salesRepository.existsRecentSales(p.getId());
                if (!recent) continue;

                Supplier sup = supplierRepository.findByProductId(p.getId());

                Integer days = null;
                int avg = salesRepository.getAvgDailySales(p.getId());

                if (avg > 0) {
                    days = stock / avg;
                }

                Map<String, Object> alert = new HashMap<>();
                alert.put("product_id", p.getId());
                alert.put("product_name", p.getName());
                alert.put("sku", p.getSku());
                alert.put("warehouse_id", w.getId());
                alert.put("warehouse_name", w.getName());
                alert.put("current_stock", stock);
                alert.put("threshold", threshold);
                alert.put("days_until_stockout", days);

                if (sup != null) {
                    Map<String, Object> s = new HashMap<>();
                    s.put("id", sup.getId());
                    s.put("name", sup.getName());
                    s.put("contact_email", sup.getEmail());
                    alert.put("supplier", s);
                }

                alerts.add(alert);
            }
        }
    }

    return ResponseEntity.ok(Map.of(
            "alerts", alerts,
            "total_alerts", alerts.size()
    ));
}         

```

                      Edge Cases

Company not found
No warehouses
No inventory
No recent sales
Division by zero handled
Supplier may be null

                      Assumptions

Each product has a threshold value
Sales data is available
Each product has a supplier
Average daily sales can be calculated

                      ## Final Note

I tried to keep the solution simple and focused on basic backend logic. There is scope for improvement, but this is my approach based on the given time and requirements.