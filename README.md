# sql-test-
Q1. List top 5 customers by total order amount.
Retrieve the top 5 customers who have spent the most across all sales orders. Show CustomerID, CustomerName, and TotalSpent.

----ANSWER----

SELECT TOP 5 
    c.CustomerID, 
    c.Name AS CustomerName, 
    SUM(so.TotalAmount) AS TotalSpent
FROM dbo.Customer c
INNER JOIN dbo.SalesOrder so ON c.CustomerID = so.CustomerID
GROUP BY c.CustomerID, c.Name
ORDER BY TotalSpent DESC;



Q2. Find the number of products supplied by each supplier.
Display SupplierID, SupplierName, and ProductCount. Only include suppliers that have more than 10 products

-----ANSWER-----


SELECT 
    s.SupplierID, 
    s.Name AS SupplierName, 
    COUNT(DISTINCT pod.ProductID) AS ProductCount
FROM dbo.Supplier s
INNER JOIN dbo.PurchaseOrder po ON s.SupplierID = po.SupplierID
INNER JOIN dbo.PurchaseOrderDetail pod ON po.OrderID = pod.OrderID
GROUP BY s.SupplierID, s.Name
HAVING COUNT(DISTINCT pod.ProductID) > 10;


Q3. Identify products that have been ordered but never returned.
Show ProductID, ProductName, and total order quantity.



----ANSWER----


SELECT 
    p.ProductID, 
    p.Name AS ProductName, 
    SUM(sod.Quantity) AS TotalOrderQuantity
FROM dbo.Product p
INNER JOIN dbo.SalesOrderDetail sod ON p.ProductID = sod.ProductID
WHERE p.ProductID NOT IN (
    SELECT DISTINCT ProductID 
    FROM dbo.ReturnDetail
)
GROUP BY p.ProductID, p.Name;


Q4. For each category, find the most expensive product.
Display CategoryID, CategoryName, ProductName, and Price. Use a subquery to get the max price per category

---ANSWER---


SELECT 
    c.CategoryID, 
    c.Name AS CategoryName, 
    p.Name AS ProductName, 
    p.Price
FROM dbo.Product p
INNER JOIN dbo.Category c ON p.CategoryID = c.CategoryID
WHERE p.Price = (
    SELECT MAX(subP.Price) 
    FROM dbo.Product subP 
    WHERE subP.CategoryID = p.CategoryID
);

Q5. List all sales orders with customer name, product name, category, and supplier.

-----ANSWER----

SELECT 
    so.OrderID, 
    c.Name AS CustomerName, 
    p.Name AS ProductName, 
    cat.Name AS CategoryName, 
    s.Name AS SupplierName, 
    sod.Quantity
FROM dbo.SalesOrder so
  INNER JOIN dbo.Customer c ON so.CustomerID = c.CustomerID
INNER JOIN dbo.SalesOrderDetail sod ON so.OrderID = sod.OrderID
    INNER JOIN dbo.Product p ON sod.ProductID = p.ProductID
 LEFT JOIN dbo.Category cat ON p.CategoryID = cat.CategoryID
  LEFT JOIN dbo.PurchaseOrderDetail pod ON p.ProductID = pod.ProductID
 LEFT JOIN dbo.PurchaseOrder po ON pod.OrderID = po.OrderID
      LEFT JOIN dbo.Supplier s ON po.SupplierID = s.SupplierID;





      Q6. Find all shipments with details of warehouse, manager, and products shipped.
Display:
ShipmentID, WarehouseName, ManagerName, ProductName, QuantityShipped, and TrackingNumber.

-----ANSWER----

SELECT 
    s.ShipmentID, 
    l.Name AS WarehouseLocationName, 
    e.Name AS ManagerName,          
    p.Name AS ProductName,     
    sd.Quantity AS QuantityShipped,
    s.TrackingNumber
FROM dbo.Shipment s
INNER JOIN dbo.Warehouse w ON s.WarehouseID = w.WarehouseID
LEFT JOIN dbo.Location l ON w.LocationID = l.LocationID
LEFT JOIN dbo.Employee e ON w.ManagerID = e.EmployeeID
INNER JOIN dbo.ShipmentDetail sd ON s.ShipmentID = sd.ShipmentID
INNER JOIN dbo.Product p ON sd.ProductID = p.ProductID;



Q7. Find the top 3 highest-value orders per customer using RANK(). Display CustomerID, CustomerName, OrderID, and TotalAmount.


-----ANSWER----

WITH RankedOrders AS (
    SELECT 
        c.CustomerID, 
        c.Name AS CustomerName, 
        so.OrderID, 
        so.TotalAmount,
        RANK() OVER (PARTITION BY so.CustomerID ORDER BY so.TotalAmount DESC) AS OrderRank
    FROM dbo.Customer c
    INNER JOIN dbo.SalesOrder so ON c.CustomerID = so.CustomerID
)
SELECT CustomerID, CustomerName, OrderID, TotalAmount
FROM RankedOrders
WHERE OrderRank <= 3


Q8. For each product, show its sales history with the previous and next sales quantities (based on order date). Display ProductID, ProductName, OrderID, OrderDate, Quantity, PrevQuantity, and NextQuantity.


-----ANSWER----

SELECT 
    p.ProductID, 
    p.Name AS ProductName, 
    so.OrderID, 
    so.OrderDate, 
    sod.Quantity,
    LAG(sod.Quantity) OVER (PARTITION BY p.ProductID ORDER BY so.OrderDate) AS PrevQuantity,
    LEAD(sod.Quantity) OVER (PARTITION BY p.ProductID ORDER BY so.OrderDate) AS NextQuantity
FROM dbo.Product p
INNER JOIN dbo.SalesOrderDetail sod ON p.ProductID = sod.ProductID
INNER JOIN dbo.SalesOrder so ON sod.OrderID = so.OrderID;

Q9. Create a view named vw_CustomerOrderSummary that shows for each customer:
CustomerID, CustomerName, TotalOrders, TotalAmountSpent, and LastOrderDate.


------ANSWER------

GO
CREATE OR ALTER VIEW dbo.vw_CustomerOrderSummary AS
SELECT 
    c.CustomerID, 
    c.Name AS CustomerName, 
    COUNT(so.OrderID) AS TotalOrders, 
    SUM(so.TotalAmount) AS TotalAmountSpent, 
    MAX(so.OrderDate) AS LastOrderDate
FROM dbo.Customer c
LEFT JOIN dbo.SalesOrder so ON c.CustomerID = so.CustomerID
GROUP BY c.CustomerID, c.Name;
GO

SELECT * FROM dbo.vw_CustomerOrderSummary;


Q10. Write a stored procedure sp_GetSupplierSales that takes a takes a SupplierID as input and returns the total sales amount for all products supplied by that supplier.

GO
CREATE OR ALTER PROCEDURE dbo.sp_GetSupplierSales
    @SupplierID INT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        @SupplierID AS SupplierID,
        ISNULL(SUM(sod.TotalAmount), 0) AS TotalSalesAmount
    FROM dbo.Product p
    INNER JOIN dbo.PurchaseOrderDetail pod ON p.ProductID = pod.ProductID
    INNER JOIN dbo.PurchaseOrder po ON pod.OrderID = po.OrderID
    INNER JOIN dbo.SalesOrderDetail sod ON p.ProductID = sod.ProductID
    WHERE po.SupplierID = @SupplierID;
END;
GO

EXEC dbo.sp_GetSupplierSales @SupplierID = 5;
