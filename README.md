# neo4j-northwind
In this guide, we will be using the NorthWind dataset, an often-used SQL dataset. This data depicts a product sale system - storing and tracking customers, products, customer orders, warehouse stock, shipping, suppliers, and even employees and their sales territories. Although the NorthWind dataset is often used to demonstrate SQL and relational databases, the data also can be structured as a graph.


An entity-relationship diagram (ERD) of the Northwind dataset is shown below.

![Northwind_diagram_focus](https://github.com/jhomarolo/neo4j-northwind/blob/main/assets/Northwind_diagram_focus.jpg?raw=true)

If we draw our translation out on the whiteboard, we have this graph data model.

![northwind_graph_simple](https://github.com/jhomarolo/neo4j-northwind/blob/main/assets/northwind_graph_simple.jpg?raw=true)

In the graph.db.zip file you will find the database ( in 3.5.24 version) for local import and running, but if you want to create yourself, just follow the steps bellow.

The entire Cypher script is available on import_csv.cypher file for you to copy and run, but we will step through each section below to explain what each piece of the script is doing.

Now that we have our files where we can access them easily, we can use Cypher’s LOAD CSV command to read each file and add Cypher statements after it to take the row/column data and transform it to the graph.


```
// Create orders
LOAD CSV WITH HEADERS FROM 'file:///orders.csv' AS row
MERGE (order:Order {orderID: row.OrderID})
  ON CREATE SET order.shipName = row.ShipName;

// Create products
LOAD CSV WITH HEADERS FROM 'file:///products.csv' AS row
MERGE (product:Product {productID: row.ProductID})
  ON CREATE SET product.productName = row.ProductName, product.unitPrice = toFloat(row.UnitPrice);

// Create suppliers
LOAD CSV WITH HEADERS FROM 'file:///suppliers.csv' AS row
MERGE (supplier:Supplier {supplierID: row.SupplierID})
  ON CREATE SET supplier.companyName = row.CompanyName;

// Create employees
LOAD CSV WITH HEADERS FROM 'file:///employees.csv' AS row
MERGE (e:Employee {employeeID:row.EmployeeID})
  ON CREATE SET e.firstName = row.FirstName, e.lastName = row.LastName, e.title = row.Title;

// Create categories
LOAD CSV WITH HEADERS FROM 'file:///categories.csv' AS row
MERGE (c:Category {categoryID: row.CategoryID})
  ON CREATE SET c.categoryName = row.CategoryName, c.description = row.Description;

```


Now create the indexes

```
CREATE INDEX product_id FOR (p:Product) ON (p.productID);
CREATE INDEX product_name FOR (p:Product) ON (p.productName);
CREATE INDEX supplier_id FOR (s:Supplier) ON (s.supplierID);
CREATE INDEX employee_id FOR (e:Employee) ON (e.employeeID);
CREATE INDEX category_id FOR (c:Category) ON (c.categoryID);
CREATE CONSTRAINT order_id ON (o:Order) ASSERT o.orderID IS UNIQUE;
```

Initial nodes and indexes in place, we can now create the relationships for orders to products and orders to employees.

```
// Create relationships between orders and products
LOAD CSV WITH HEADERS FROM 'file:///orders.csv' AS row
MATCH (order:Order {orderID: row.OrderID})
MATCH (product:Product {productID: row.ProductID})
MERGE (order)-[op:CONTAINS]->(product)
  ON CREATE SET op.unitPrice = toFloat(row.UnitPrice), op.quantity = toFloat(row.Quantity);

// Create relationships between orders and employees
LOAD CSV WITH HEADERS FROM "file:///orders.csv" AS row
MATCH (order:Order {orderID: row.OrderID})
MATCH (employee:Employee {employeeID: row.EmployeeID})
MERGE (employee)-[:SOLD]->(order);

// Create relationships between products and suppliers
LOAD CSV WITH HEADERS FROM "file:///products.csv" AS row
MATCH (product:Product {productID: row.ProductID})
MATCH (supplier:Supplier {supplierID: row.SupplierID})
MERGE (supplier)-[:SUPPLIES]->(product);

// Create relationships between products and categories
LOAD CSV WITH HEADERS FROM "file:///products.csv" AS row
MATCH (product:Product {productID: row.ProductID})
MATCH (category:Category {categoryID: row.CategoryID})
MERGE (product)-[:PART_OF]->(category);

// Create relationships between employees (reporting hierarchy)
LOAD CSV WITH HEADERS FROM "file:///employees.csv" AS row
MATCH (employee:Employee {employeeID: row.EmployeeID})
MATCH (manager:Employee {employeeID: row.ReportsTo})
MERGE (employee)-[:REPORTS_TO]->(manager);
```

Original Source (with 2.1 neo4j version): https://neo4j.com/developer/guide-importing-data-and-etl/#_how_does_the_graph_model_differ_from_the_relational_model