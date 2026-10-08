# US Candy Distributor – Shipping & Product Analysis

## Introduction

This project analyzes a candy distributor's sales, products, factories, and shipping data to understand shipping efficiency and product profitability.

The analysis focuses on factory-to-customer shipping routes, product margins, sales volume, total gross profit, and whether relocating a product line to another factory could improve its shipping geography.

---

## Background

This dataset was provided by **Maven Analytics** and contains sales transactions along with product, factory, target, and U.S. ZIP-code geographic data.

Before starting the analysis, I cleaned the data and loaded the tables into BigQuery.

During data validation, I noticed a systematic issue with the shipping dates. The raw difference between `ship_date` and `order_date` ranged from **2,000 to 2,011 days** across the dataset. I used three separate checks to confirm that this was a consistent pattern rather than an isolated anomaly.

Subtracting **2,000 days** produced a more plausible shipping-duration range of **0–11 days**, which also aligned logically with the different shipping modes. I therefore used this adjusted value as the shipping-time measure in this analysis.

---

## Tools I Used

* **Google Sheets** – Data cleaning
* **Google BigQuery** – SQL analysis
* **GitHub** – Project documentation and version control

SQL techniques used included CTEs, joins, aggregations, `CASE` logic, window functions, and BigQuery geospatial functions such as `ST_GEOGPOINT()` and `ST_DISTANCE()`.

---

## The Analysis

### 1. Most Efficient Factory-to-Customer Shipping Routes

To measure route efficiency, I calculated the average adjusted shipping time for each **factory-to-customer-state route**.

Routes with fewer than 10 orders were excluded so that very small order volumes would not dominate the results.

```sql
WITH sales_table AS (
  SELECT DISTINCT
    sales.order_id AS order_id,
    sales.order_date AS order_date,
    sales.ship_date AS ship_date,
    sales.ship_mode AS ship_mode,
    sales.state_province AS state_province,
    products.factory AS factory
  FROM us_candy_distributor.candy_sales AS sales
  INNER JOIN us_candy_distributor.candy_products AS products
    ON sales.product_id = products.product_id
)

SELECT
  sales_table.factory AS factory,
  sales_table.state_province AS state_province,
  COUNT(DISTINCT sales_table.order_id) AS orders,
  ROUND(AVG(DATE_DIFF(DATE(sales_table.ship_date), DATE(sales_table.order_date), DAY) - 2000), 2) AS avg_shipping_days
FROM sales_table
GROUP BY
  sales_table.factory,
  sales_table.state_province
HAVING
 orders > 10
ORDER BY
  avg_shipping_days,
  orders DESC
```

The fastest route identified was:

| Factory         | Customer State | Orders | Avg. Shipping Days |
| --------------- | -------------- | -----: | -----------------: |
| Wicked Choccy's | Louisiana      |     14 |           **2.71** |
| Secret Factory  | Illinois       |     13 |               3.00 |
| Lot's O' Nuts   | Nebraska       |     20 |               3.05 |
| Lot's O' Nuts   | Ontario        |     24 |               3.08 |
| Lot's O' Nuts   | Louisiana      |     21 |               3.14 |

**Finding:** Wicked Choccy's to Louisiana had the shortest average adjusted shipping time at **2.71 days** among routes with more than 10 orders.

---

### 2. Least Efficient Factory-to-Customer Shipping Routes

The same route analysis was used to identify the longest average shipping times.

```sql
WITH sales_table AS (
  SELECT DISTINCT
    sales.order_id AS order_id,
    sales.order_date AS order_date,
    sales.ship_date AS ship_date,
    sales.ship_mode AS ship_mode,
    sales.state_province AS state_province,
    products.factory AS factory
  FROM us_candy_distributor.candy_sales AS sales
  INNER JOIN us_candy_distributor.candy_products AS products
    ON sales.product_id = products.product_id
)

SELECT
  sales_table.factory AS factory,
  sales_table.state_province AS state_province,
  COUNT(DISTINCT sales_table.order_id) AS orders,
  ROUND(AVG(DATE_DIFF(DATE(sales_table.ship_date), DATE(sales_table.order_date), DAY) - 2000), 2) AS avg_shipping_days
FROM sales_table
GROUP BY
  sales_table.factory,
  sales_table.state_province
HAVING
 orders > 10
ORDER BY
  avg_shipping_days DESC,
  orders DESC
```

The least efficient route among routes with more than 10 orders was:

| Factory         | Customer State | Orders | Avg. Shipping Days |
| --------------- | -------------- | -----: | -----------------: |
| Lot's O' Nuts   | New Mexico     |     18 |           **4.83** |
| Lot's O' Nuts   | Alberta        |     12 |               4.67 |
| Lot's O' Nuts   | Minnesota      |     38 |               4.61 |
| Lot's O' Nuts   | Iowa           |     14 |               4.57 |
| Wicked Choccy's | Quebec         |     16 |               4.56 |

**Finding:** Lot's O' Nuts to New Mexico had the longest average adjusted shipping time at **4.83 days**.

---

### 3. Product Lines With the Best Product Margin

Product margin was calculated as:

**Gross Profit ÷ Sales × 100**

```sql
ELECT
  sale.product_name,
  ROUND(SAFE_DIVIDE(SUM(sale.gross_profit), SUM(sale.sales)) * 100, 2) AS product_margin
FROM us_candy_distributor.candy_sales AS sale
GROUP BY
  sale.product_name
ORDER BY
  product_margin DESC;
```

The products with the highest margins were:

| Product                           | Product Margin |
| --------------------------------- | -------------: |
| Everlasting Gobstopper            |     **80.00%** |
| Hair Toffee                       |         77.78% |
| Wonka Bar - Nutty Crunch Surprise |         71.35% |
| Wonka Bar -Scrumdiddlyumptious    |         69.44% |
| Wonka Bar - Fudge Mallows         |         66.67% |

**Finding:** Everlasting Gobstopper had the highest product margin at **80.00%**.

However, a high margin does not necessarily mean a product generates the most profit. For example, Everlasting Gobstopper had a very high margin but only **13 units sold**.

---

### 4. Sales Volume and Gross Profit

I also examined sales volume and total gross profit to understand whether high-margin products were also commercially important.

The highest sales volumes were:

```sql
SELECT
  product_name,
  SUM(units) AS sales_volume
FROM us_candy_distributor.candy_sales
GROUP BY
  product_name
ORDER BY
  sales_volume DESC;
```

| Product                           | Units Sold |
| --------------------------------- | ---------: |
| Wonka Bar - Milk Chocolate        |  **8,267** |
| Wonka Bar -Scrumdiddlyumptious    |      7,743 |
| Wonka Bar - Triple Dazzle Caramel |      7,596 |
| Wonka Bar - Fudge Mallows         |      6,914 |
| Wonka Bar - Nutty Crunch Surprise |      6,755 |

The products generating the most gross profit were:

```sql
SELECT
  product_name,
  ROUND(SUM(gross_profit), 2) AS profit
FROM us_candy_distributor.candy_sales
GROUP BY
  product_name
ORDER BY
  profit DESC;
```

| Product                           |   Gross Profit |
| --------------------------------- | -------------: |
| Wonka Bar -Scrumdiddlyumptious    | **$19,357.50** |
| Wonka Bar - Triple Dazzle Caramel |     $18,610.20 |
| Wonka Bar - Milk Chocolate        |     $17,443.37 |
| Wonka Bar - Nutty Crunch Surprise |     $16,819.95 |
| Wonka Bar - Fudge Mallows         |     $16,593.60 |

**Finding:** Milk Chocolate had the highest sales volume, while Scrumdiddlyumptious generated the highest total gross profit.

This shows why margin, sales volume, and total profit should not be viewed as the same metric. A product can have a high margin but contribute very little total profit if its sales volume is low.

---

### 5. Which Product Line Should Be Moved to Another Factory?

For this analysis, I used a **weighted geographic distance** approach.

For each product, I considered the locations where its customers purchased the product and weighted each location by the number of units sold there. I then compared the current factory with every alternative factory.

```sql
WITH product_customers AS (
  SELECT
    sale.product_name AS product_name,
    products.factory AS current_factory,
    sale.postal_code AS postal_code,
    SUM(sale.units) AS units
  FROM us_candy_distributor.candy_sales AS sale
  INNER JOIN us_candy_distributor.candy_products AS products
    ON sale.product_id = products.product_id
  GROUP BY
    sale.product_name,
    products.factory,
    sale.postal_code
),
customers_location AS (
  SELECT
    pc.product_name AS product_name,
    pc.current_factory AS current_factory,
    pc.postal_code AS postal_code,
    pc.units AS units,
    uszip.lat AS customer_lat,
    uszip.lng AS customer_lng
  FROM product_customers AS pc
  INNER JOIN us_candy_distributor.uszips AS uszip
    ON CAST(pc.postal_code AS STRING) = CAST(uszip.zip AS STRING)
),
current_factory_distance AS (
  SELECT
    customers.product_name AS product_name,
    customers.current_factory AS current_factory,
    SUM(customers.units * ST_DISTANCE(ST_GEOGPOINT(factories.longitude, factories.latitude), ST_GEOGPOINT(customers.customer_lng, customers.customer_lat))) / SUM(customers.units) AS
    current_avg_distance
  FROM customers_location AS customers
  INNER JOIN us_candy_distributor.candy_factories AS factories
    ON customers.current_factory = factories.factory
  GROUP BY
    customers.product_name,
    customers.current_factory
),
alternative_factory_distance AS (
  SELECT
  customers.product_name AS product_name,
  customers.current_factory AS current_factory,
  factories.factory AS alternative_factory,
  SUM(customers.units * ST_DISTANCE(ST_GEOGPOINT(factories.longitude, factories.latitude), ST_GEOGPOINT(customers.customer_lng, customers.customer_lat))) / SUM(customers.units) AS
  alternative_avg_distance
  FROM customers_location AS customers
  CROSS JOIN us_candy_distributor.candy_factories AS factories
  WHERE
    customers.current_factory <> factories.factory
  GROUP BY
    customers.product_name,
    customers.current_factory,
    alternative_factory
),
ranked_alternative AS (
  SELECT
    af.product_name AS product_name,
    af.current_factory AS current_factory,
    af.alternative_factory AS alternative_factory,
    af.alternative_avg_distance AS alternative_avg_distance,
    ROW_NUMBER () OVER(
      PARTITION BY product_name, current_factory
      ORDER BY alternative_avg_distance
    ) AS factory_rank
  FROM alternative_factory_distance AS af
)

SELECT
  cf.product_name,
  cf.current_factory,
  ra.alternative_factory,
  ROUND(cf.current_avg_distance / 1609.34, 2) AS current_avg_distance_miles,
  ROUND(ra.alternative_avg_distance / 1609.34, 2) AS alternative_avg_distance_miles,
  ROUND((cf.current_avg_distance - ra.alternative_avg_distance) / 1609.34, 2) AS saved_distance_miles
FROM current_factory_distance AS cf
INNER JOIN ranked_alternative AS ra
  ON cf.product_name = ra.product_name
  AND cf.current_factory = ra.current_factory
WHERE
  ra.factory_rank = 1 AND
  ra.alternative_avg_distance < cf.current_avg_distance
ORDER BY
  saved_distance_miles DESC
```

The results show how much the average geographic distance could potentially be reduced by moving a product to another factory.

| Product                           | Current Factory | Alternative Factory | Current Avg. Distance | Alternative Avg. Distance | Distance Saved |
| --------------------------------- | --------------- | ------------------- | --------------------: | ------------------------: | -------------: |
| Laffy Taffy                       | Sugar Shack     | Wicked Choccy's     |           1,273.06 mi |                 590.51 mi |  **682.55 mi** |
| Nerds                             | Sugar Shack     | Wicked Choccy's     |           1,199.16 mi |                 710.67 mi |      488.50 mi |
| Fun Dip                           | Sugar Shack     | Secret Factory      |             958.48 mi |                 553.87 mi |      404.61 mi |
| Wonka Bar - Nutty Crunch Surprise | Lot's O' Nuts   | Secret Factory      |           1,317.98 mi |                 926.66 mi |      391.32 mi |
| Wonka Bar -Scrumdiddlyumptious    | Lot's O' Nuts   | Secret Factory      |           1,320.50 mi |                 944.25 mi |      376.25 mi |
| Wonka Bar - Fudge Mallows         | Lot's O' Nuts   | Secret Factory      |           1,322.12 mi |                 947.48 mi |      374.64 mi |

**Finding:** Laffy Taffy has the largest potential geographic-distance reduction, with a possible reduction of **682.55 miles** by moving production from Sugar Shack to Wicked Choccy's.

However, Laffy Taffy had only **27 units sold**, so its potential business impact may be limited. Higher-volume products such as Scrumdiddlyumptious and Nutty Crunch Surprise may deserve more attention because they combine substantial potential distance savings with much greater sales volume.

The alternative factory analysis is based on **geographic distance**, so it should be treated as a routing recommendation rather than proof that actual transportation cost or delivery time would decrease by the same amount.

---

## Conclusion

This analysis revealed several differences between product profitability and shipping performance.

Wicked Choccy's to Louisiana was the fastest factory-to-customer-state route, averaging **2.71 adjusted shipping days**, while Lot's O' Nuts to New Mexico was the slowest at **4.83 days** among routes with more than 10 orders.

Everlasting Gobstopper had the highest product margin at **80.00%**, but its low sales volume meant that it generated only **$104.00** in gross profit. In contrast, Wonka Bar -Scrumdiddlyumptious generated the highest total gross profit at **$19,357.50**.

The geographic optimization analysis identified **Laffy Taffy** as having the largest potential reduction in average customer distance if production were moved from Sugar Shack to Wicked Choccy's. However, its low sales volume suggests that higher-volume products may provide a greater operational impact.

Overall, the analysis showed the importance of looking beyond a single metric. **Margin, sales volume, total profit, shipping time, and geographic distance each tell a different part of the business story.**
