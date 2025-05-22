Project: SQL Queries – Chinook Database
📘 About the Database
The Chinook database simulates a digital music store. It contains tables for artists, albums, tracks, customers, invoices, and more. It's often used for learning SQL due to its realistic yet manageable complexity.

### 1. Top Revenue-Generating Tracks

**Business Case:**  
The business wants to highlight tracks generating the most revenue to prioritize them in top recommendations.

**Objective:**  
Display the top 10 tracks with the highest total revenue, including their name, artist, genre, and revenue amount.

**SQL Query:**
```sql
SELECT i.TrackId as TrackId
	, t.Name as TrackName
	, art.Name as ArtistName
	, g.Name as GenreName
	, SUM(i.Quantity * i.UnitPrice) as Revenue
FROM InvoiceLine as i
	LEFT JOIN Track as t ON i.TrackId = t.TrackId
	LEFT JOIN Album as a ON t.AlbumId = a.AlbumId
	LEFT JOIN Artist as art ON a.ArtistId = art.ArtistId
	LEFT JOIN Genre as g ON g.GenreId = t.GenreId
GROUP BY i.TrackId, t.Name, art.Name, g.Name
ORDER BY Revenue DESC
LIMIT 10
```

### 2. Strategic Markets

**Business Case:**  
The business wants to identify the most profitable countries to prioritize strategic investments.

**Objective:**  
Display the top 5 countries with the highest total revenue, average invoice amount, and number of unique customers.

**SQL Query:**
```sql
WITH group_invoice as(
	SELECT InvoiceId
		, SUM(Quantity * UnitPrice) as invoice_total
	FROM InvoiceLine 
	GROUP BY InvoiceId)

SELECT i.BillingCountry as Country
	, SUM(gi.invoice_total) as TotalRevenue
	, ROUND(AVG(gi.invoice_total),2) as AvgInvoice
	, COUNT(DISTINCT c.CustomerId) as ClientsCount
FROM group_invoice as gi
	JOIN Invoice as i USING(InvoiceId)
	JOIN Customer as c USING(CustomerId)
GROUP BY i.BillingCountry
ORDER BY TotalRevenue DESC, AvgInvoice DESC
LIMIT 5
```

### 3. Top Customers by Lifetime Value

**Business Case:**  
The retention team wants to identify the most valuable customers to invite into a VIP loyalty program.

**Objective:**  
List the top customers ranked by total lifetime value (total purchases).

**SQL Query:**
```sql
SELECT c.FirstName||' '||c.LastName as FullName
	, c.Email as Email
	, c.Country as Country
	, COALESCE(SUM(il.Quantity*il.UnitPrice),0) as TotalSpent
	, DENSE_RANK() OVER(ORDER BY COALESCE(SUM(il.Quantity*il.UnitPrice),0) DESC) as Rank
FROM Customer as c
	LEFT JOIN Invoice as i ON c.CustomerId = i.CustomerId
	LEFT JOIN InvoiceLine as il ON i.InvoiceId = il.InvoiceId
GROUP BY 1, 2, 3
```

### 4. Top Genre per Customer

**Business Case:**  
The business is preparing personalized email campaigns based on each customer's music preferences.

**Objective:**  
Identify the genre with the highest total spending for each customer. Show one top genre per customer.

**SQL Query:**
```sql
WITH cte AS (
    SELECT i.CustomerId
         , g.Name as GenreName 
         , SUM(il.Quantity * il.UnitPrice) as Genre_total
         , RANK() OVER(PARTITION BY i.CustomerId ORDER BY SUM(il.Quantity * il.UnitPrice) DESC) as Rank
    FROM InvoiceLine as il
        JOIN Track as t ON il.TrackId = t.TrackId
        JOIN Genre as g ON g.GenreId = t.GenreId
        JOIN Invoice as i ON il.InvoiceId = i.InvoiceId
    GROUP BY 1, 2)
    
SELECT c.FirstName || ' ' || c.LastName as FullName
     , c.Email
     , cte.GenreName as TopGenre
     , cte.Genre_total as GenreSpend
FROM Customer as c
JOIN cte USING(CustomerId)
WHERE cte.Rank = 1
ORDER BY 4 DESC
```

### 5. Seasonality of Purchases

**Business Case:**  
The business wants to understand seasonal revenue dynamics for planning discounts and promotions.

**Objective:**  
Show total revenue and number of orders by year and month.

**SQL Query:**
```sql
SELECT strftime('%Y', i.InvoiceDate) AS Year
	, strftime('%m', i.InvoiceDate) AS Month
	, SUM(il.Quantity * il.UnitPrice) AS TotalRevenue
	, COUNT(DISTINCT i.InvoiceId) AS OrderCount
	, CASE WHEN SUM(il.Quantity * il.UnitPrice) > (SELECT AVG(MonthlyRevenue)
            FROM (SELECT SUM(il2.Quantity * il2.UnitPrice) AS MonthlyRevenue
                 FROM Invoice i2
                 JOIN InvoiceLine il2 USING(InvoiceId)
                 GROUP BY strftime('%Y-%m', i2.InvoiceDate)
            )) THEN 'YES'
				ELSE 'NO' END as AboveAverage
FROM Invoice AS i
JOIN InvoiceLine AS il USING(InvoiceId)
GROUP BY Year, Month
ORDER BY Year, Month
```

### 6. Inefficient Catalog

**Business Case:**  
The business needs to clean the track database from unpopular tracks that never sold.

**Objective:**  
Find tracks with zero sales.

**SQL Query:**
```sql
SELECT t.Name as TrackName
	, a.Title as AlbumTitle
	, g.Name as GenreName
	, ROUND(t.Milliseconds/60000,2) as Duration_min
FROM Track as t 
	JOIN Album as a ON t.AlbumId = a.AlbumId
	JOIN Genre as g ON t.GenreId = g.GenreId
	LEFT JOIN InvoiceLine as il ON t.TrackId = il.TrackId
WHERE il.TrackId IS NULL
```

### 7. Repeat Purchases

**Business Case:**  
The business wants to measure customer return rates.

**Objective:**  
Find customers with more than one order. Show full name, email, orders count, first and last purchase dates.

**SQL Query:**
```sql
SELECT c.FirstName||' '||c.LastName as FullName
	, c.Email as Email
	, COUNT(DISTINCT i.InvoiceID) as OrdersCount
	, DATE(MIN(i.InvoiceDate)) as FirstOrderDate
	, DATE(MAX(i.InvoiceDate)) as LastOrderDate
FROM Customer as c
	LEFT JOIN Invoice as i USING(CustomerID)
GROUP BY c.CustomerId
HAVING COUNT(DISTINCT i.InvoiceID)>1
```

### 8. Geography of Jazz Fans

**Business Case:**  
The business plans a new series of Jazz compilations targeting regions with the greatest interest.

**Objective:**  
Calculate revenue, client count, and track count by country for Jazz purchases.

**SQL Query:**
```sql
SELECT i.BillingCountry as Country
	, SUM(il.Quantity*il.UnitPrice) as Revenue
	, COUNT(DISTINCT i.CustomerId) as Clients
	, COUNT(DISTINCT il.TrackId) as Tracks
FROM InvoiceLine as il
	JOIN Track as t ON il.TrackId = t.TrackId
	JOIN Genre as g ON t.GenreId = g.GenreId
	JOIN Invoice as i ON i.InvoiceId = il.InvoiceId
WHERE g.Name = 'Jazz'
GROUP BY 1 
ORDER BY Revenue DESC
```

### 9. Listening Duration by Genre

**Business Case:**  
Analysts want to evaluate which genre takes the most listening time among users.

**Objective:**  
Calculate total duration (in minutes) of tracks purchased by users, grouped by genre.

**SQL Query:**
```sql
SELECT g.Name as Genre
	, ROUND(SUM(t.Milliseconds)/60000, 2) as TotalMinutes
FROM InvoiceLine as il
	LEFT JOIN Track as t ON il.TrackId = t.TrackId
	JOIN Genre as g ON t.GenreId = g.GenreId
GROUP BY 1
ORDER BY 2 DESC
```

### 10. Top Tracks with Repeat Purchases

**Business Case:**  
The business is interested in tracks that have multiple purchases and exceed average revenue levels.

**Objective:**  
Show tracks purchased at least twice with revenue higher than the average revenue of all tracks.

**SQL Query:**
```sql
SELECT t.Name as TrackName
	, COUNT(*) as Purchases
	, SUM(il.Quantity * il.UnitPrice) as Revenue
FROM InvoiceLine as il
	JOIN Track as t USING(TrackId)
GROUP BY il.TrackId
HAVING COUNT(*) > 1
	AND SUM(il.Quantity * il.UnitPrice) > (
		SELECT AVG(TrackRevenue)
		FROM (
			SELECT SUM(Quantity * UnitPrice) as TrackRevenue
			FROM InvoiceLine
			GROUP BY TrackId
		) as avg_revenue)
```

### 11. Sales by Managers

**Business Case:**  
Management wants to evaluate the effectiveness of each account manager.

**Objective:**  
List manager name, number of clients they manage, total sales amount, and average invoice value.

**SQL Query:**
```sql
SELECT e.FirstName||' '||e.LastName as ManagerName
	, COUNT(DISTINCT c.CustomerId) as ClientsCount
	, COALESCE(ROUND(SUM(i.Total), 2),0) as TotalSales
	, COALESCE(ROUND(AVG(i.Total), 2),0) as AvgInvoice
FROM Employee as e
	LEFT JOIN Customer as c ON e.EmployeeId = c.SupportRepId
	LEFT JOIN Invoice as i ON c.CustomerId = i.CustomerId
GROUP BY e.EmployeeId 
ORDER BY 2 DESC
```

### 12. Customers Without Orders

**Business Case:**  
The CRM team wants to identify inactive contacts for follow-up or removal.

**Objective:**  
Find customers who registered but made no orders.

**SQL Query:**
```sql
SELECT c.FirstName||' '||c.LastName as FullName
	, c.Email as Email
	, c.Country as Country
FROM Customer as c
	LEFT JOIN Invoice as i ON c.CustomerId = i.CustomerId
WHERE i.InvoiceId IS NULL 
```

### 13. Suspicious Spending Behavior

**Business Case:**  
The finance team wants to identify customers who made unusually large purchases.

**Objective:**  
Find customers who have at least one invoice that exceeds their own average order value by 50%.

**SQL Query:**
```sql
SELECT c.FirstName||' '||c.LastName as FullName
	, c.Email as Email
	, i.InvoiceId as InvoiceID
	, i.Total as Total
FROM Invoice as i
	JOIN Customer as c ON i.CustomerId = c.CustomerId
GROUP BY c.CustomerId, i.InvoiceId
HAVING i.Total > (SELECT AVG(Total)*1.5
				 FROM Invoice i2
				 WHERE c.CustomerId = i2.CustomerId )
ORDER BY 1
```

### 14. Similar Music Taste Clusters

**Business Case:**  
The business wants to identify groups of customers with similar music preferences.

**Objective:**  
Find pairs of customers who share at least 6 common purchased genres.

**SQL Query:**
```sql
WITH CustomerGenre as (SELECT c.FirstName||' '||c.LastName as FullName
		, c.CustomerId 
		, t.GenreId
	FROM Customer as c
		JOIN Invoice as i ON i.CustomerId = c.CustomerId
		JOIN InvoiceLine as il ON i.InvoiceId = il.InvoiceId
		JOIN Track as t ON il.TrackId = t.TrackId
	GROUP BY c.CustomerId, t.GenreId)
	
SELECT cg.FullName as ClientA
	, cg2.FullName as ClientB
	, COUNT(*) as CommonGenres
FROM CustomerGenre cg,
	CustomerGenre cg2
WHERE cg.GenreId = cg2.GenreId 
	AND cg.CustomerId < cg2.CustomerId
GROUP BY cg.CustomerId, cg2.CustomerId 
HAVING COUNT(*) >5
ORDER BY 3 DESC
```

### 15. Playlist for New Customers

**Business Case:**  
The business is preparing a playlist based on purchases from recently registered customers.

**Objective:**  
Find unique tracks purchased by customers who registered within the last 42 months of the latest invoice date.

**SQL Query:**
```sql
WITH CustomerReg AS(SELECT CustomerId 
						, MIN(InvoiceDate) as FirstOrder
					FROM Invoice
					GROUP BY 1)
					
SELECT DISTINCT
	  t.Name as TrackName 
	, a.Title as AlbumTitle
	, g.Name as GenreName
FROM  Invoice as i 
	  JOIN InvoiceLine as il ON i.InvoiceId = il.InvoiceId
	  JOIN Track as t ON il.TrackId = t.TrackId
	  JOIN Album as a ON a.AlbumId = t.AlbumId
	  JOIN Genre as g ON t.GenreId = g.GenreId
WHERE i.CustomerId IN (SELECT CustomerId
	  FROM CustomerReg
	  WHERE FirstOrder >= DATE((SELECT MAX(InvoiceDate) as LastOrder
								FROM Invoice), '-42 months'))
```