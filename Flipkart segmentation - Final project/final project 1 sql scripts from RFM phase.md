\----**RFM** 



CREATE TABLE rfm\_final\_segments AS

WITH customer\_metrics AS (

&#x20;   SELECT

&#x20;       c.customer\_id,

&#x20;       -- Reference the latest date in the data for accurate recency

&#x20;       (SELECT MAX(order\_date) FROM orders) - MAX(o.order\_date) AS recency\_val,

&#x20;       COUNT(DISTINCT o.order\_id) AS frequency\_val,

&#x20;       SUM(oi.final\_price \* oi.quantity) AS monetary\_val

&#x20;   FROM customers c

&#x20;   JOIN orders o ON c.customer\_id = o.customer\_id

&#x20;   JOIN order\_items oi ON o.order\_id = oi.order\_id

&#x20;   GROUP BY c.customer\_id

),



rfm\_scores AS (

&#x20;   SELECT

&#x20;       customer\_id,

&#x20;       NTILE(5) OVER (ORDER BY recency\_val DESC) AS r,

&#x20;       NTILE(5) OVER (ORDER BY frequency\_val ASC) AS f,

&#x20;       NTILE(5) OVER (ORDER BY monetary\_val ASC) AS m

&#x20;   FROM customer\_metrics

)



SELECT

&#x20;   customer\_id,

&#x20;   r, f, m,

&#x20;   CASE

&#x20;       -- 1. Champions

&#x20;       WHEN r >= 4 AND f >= 4 AND m >= 4 THEN 'Champions'

&#x20;

&#x20;       -- 2. Cannot Lose Them

&#x20;       WHEN r = 1 AND f >= 4 AND m >= 4 THEN 'Cannot Lose Them'

&#x20;

&#x20;       -- 3. At Risk

&#x20;       WHEN r <= 2 AND f BETWEEN 2 AND 5 AND m BETWEEN 2 AND 5 THEN 'At Risk'

&#x20;

&#x20;       -- 4. New Customers

&#x20;       WHEN r >= 4 AND f = 1 THEN 'New Customers'

&#x20;

&#x20;       -- 5. Potential Loyalists

&#x20;       WHEN r BETWEEN 3 AND 5 AND f BETWEEN 1 AND 3 AND m BETWEEN 1 AND 3 THEN 'Potential Loyalists'

&#x20;

&#x20;       -- 6. Loyal Customers

&#x20;       WHEN r BETWEEN 3 AND 5 AND f BETWEEN 3 AND 5 AND m BETWEEN 3 AND 5 THEN 'Loyal Customers'

&#x20;

&#x20;       -- 7. Promising

&#x20;       WHEN r BETWEEN 3 AND 4 AND f BETWEEN 1 AND 2 AND m BETWEEN 1 AND 2 THEN 'Promising'

&#x20;

&#x20;       -- 8. About to Sleep

&#x20;       WHEN r BETWEEN 2 AND 3 AND f BETWEEN 1 AND 2 AND m BETWEEN 1 AND 2 THEN 'About to Sleep'

&#x20;

&#x20;       -- 9. Need Attention

&#x20;       WHEN r BETWEEN 2 AND 3 AND f BETWEEN 2 AND 3 AND m BETWEEN 2 AND 3 THEN 'Need Attention'

&#x20;

&#x20;       -- 10. Lost Customers

&#x20;       WHEN r = 1 AND f BETWEEN 1 AND 2 AND m BETWEEN 1 AND 2 THEN 'Lost Customers'

&#x20;

&#x20;       ELSE 'Unclassified'

&#x20;   END AS rfm\_segment

FROM rfm\_scores

ORDER BY r DESC, f DESC;



\--- **Entity relationship 1**

\-- 1. Order Items to Orders (The one that caused the error)

ALTER TABLE order\_items DROP CONSTRAINT IF EXISTS fk\_items\_order;

ALTER TABLE order\_items

ADD CONSTRAINT fk\_items\_order

FOREIGN KEY (order\_id) REFERENCES orders (order\_id);



\-- 2. Orders to Customers

ALTER TABLE orders DROP CONSTRAINT IF EXISTS fk\_orders\_customer;

ALTER TABLE orders

ADD CONSTRAINT fk\_orders\_customer

FOREIGN KEY (customer\_id) REFERENCES customers (customer\_id);



\-- 3. Order Items to Products

ALTER TABLE order\_items DROP CONSTRAINT IF EXISTS fk\_items\_product;

ALTER TABLE order\_items

ADD CONSTRAINT fk\_items\_product

FOREIGN KEY (product\_id) REFERENCES products (product\_id);



\-- 4. RFM Segments to Customers

ALTER TABLE industry\_level\_segmentation DROP CONSTRAINT IF EXISTS fk\_segmentation\_customer;

ALTER TABLE industry\_level\_segmentation

ADD CONSTRAINT fk\_segmentation\_customer

FOREIGN KEY (customer\_id) REFERENCES customers (customer\_id);



\---**entity relationship 2**

\-- Link Browsing History to Customers

ALTER TABLE browsing\_events DROP CONSTRAINT IF EXISTS fk\_browsing\_customer;

ALTER TABLE browsing\_events

ADD CONSTRAINT fk\_browsing\_customer

FOREIGN KEY (customer\_id) REFERENCES customers (customer\_id);



\-- Link Campaign Performance to Customers

ALTER TABLE campaign\_responses DROP CONSTRAINT IF EXISTS fk\_campaign\_customer;

ALTER TABLE campaign\_responses

ADD CONSTRAINT fk\_campaign\_customer

FOREIGN KEY (customer\_id) REFERENCES customers (customer\_id);



\----**entity relationship 3**

\-- Link Customers to City Tiers

ALTER TABLE customers DROP CONSTRAINT IF EXISTS fk\_customer\_city;

ALTER TABLE customers

ADD CONSTRAINT fk\_customer\_city

FOREIGN KEY (city) REFERENCES city\_tier\_mapping (city);











\---**orphans correction**

INSERT INTO customers (customer\_id, full\_name)

VALUES ('UNKNOWN\_CUST', 'Orphaned\_Data');



INSERT INTO orders (order\_id, customer\_id, order\_status)

SELECT DISTINCT order\_id, 'UNKNOWN\_CUST', 'System\_Generated'

FROM order\_items

WHERE order\_id NOT IN (SELECT order\_id FROM orders);



ALTER TABLE order\_items

ADD CONSTRAINT fk\_items\_order

FOREIGN KEY (order\_id) REFERENCES orders (order\_id);



\---**check for orphans**

SELECT COUNT(\*) AS remaining\_orphans

FROM order\_items

WHERE order\_id NOT IN (SELECT order\_id FROM orders);

















\-- **phase 3 (1)**

CREATE TABLE rfm\_final\_segments AS

\-- \[Your entire WITH blocks and SELECT statement here]



CREATE TABLE customer\_browsing\_summary AS

SELECT

&#x20;   customer\_id,

&#x20;   -- 1. Identify their primary interest

&#x20;   MODE() WITHIN GROUP (ORDER BY category\_viewed) AS top\_category,

&#x20;

&#x20;   -- 2. Engagement Depth

&#x20;   COUNT(sessions\_id) AS total\_sessions,

&#x20;   SUM(time\_spent\_sec) / 60 AS total\_minutes\_spent,

&#x20;

&#x20;   -- 3. Intent Signals (Binary or Counts)

&#x20;   SUM(CASE WHEN added\_to\_cart = 'Yes' OR added\_to\_cart = '1' THEN 1 ELSE 0 END) AS cart\_additions,

&#x20;   SUM(CASE WHEN added\_to\_wishlist = 'Yes' OR added\_to\_wishlist = '1' THEN 1 ELSE 0 END) AS wishlist\_additions,

&#x20;

&#x20;   -- 4. Behavioral Persona (Research vs. Quick Buy)

&#x20;   CASE

&#x20;       WHEN AVG(time\_spent\_sec) > 300 THEN 'Deep Researcher'

&#x20;       WHEN AVG(time\_spent\_sec) < 60 THEN 'Quick Browser'

&#x20;       ELSE 'Steady User'

&#x20;   END AS browsing\_depth\_type,

&#x20;

&#x20;   -- 5. Technical Context

&#x20;   MODE() WITHIN GROUP (ORDER BY device) AS preferred\_device

FROM browsing\_events

GROUP BY customer\_id;





CREATE TABLE industry\_level\_segmentation AS

SELECT

&#x20;   r.customer\_id,

&#x20;   r.rfm\_segment,

&#x20;   r.r, r.f, r.m,

&#x20;   b.top\_category,

&#x20;   b.browsing\_depth\_type,

&#x20;   b.wishlist\_additions,

&#x20;   b.cart\_additions,

&#x20;   -- Creating a 'Targeted Action' column based on the overlap

&#x20;   CASE

&#x20;       WHEN r.rfm\_segment = 'At Risk' AND b.cart\_additions > 0

&#x20;            THEN 'High Priority: Abandoned Cart Recovery'

&#x20;       WHEN r.rfm\_segment = 'Champions' AND b.wishlist\_additions > 3

&#x20;            THEN 'VIP: Exclusive Early Access'

&#x20;       WHEN r.rfm\_segment = 'About to Sleep' AND b.total\_sessions > 5

&#x20;            THEN 'Re-engagement: Content Personalization'

&#x20;       ELSE 'Standard Marketing'

&#x20;   END AS marketing\_strategy

FROM rfm\_final\_segments r

LEFT JOIN customer\_browsing\_summary b ON r.customer\_id = b.customer\_id;





SELECT

&#x20;   rfm\_segment,

&#x20;   top\_category,

&#x20;   browsing\_depth\_type,

&#x20;   COUNT(customer\_id) AS total\_customers,

&#x20;   ROUND(AVG(cart\_additions), 2) AS avg\_cart\_adds,

&#x20;   ROUND(AVG(wishlist\_additions), 2) AS avg\_wishlist\_adds

FROM industry\_level\_segmentation

GROUP BY rfm\_segment, top\_category, browsing\_depth\_type

ORDER BY rfm\_segment, total\_customers DESC;





SELECT \* FROM industry\_level\_segmentation

LIMIT 10;



\----

WITH customer\_behavior AS (

&#x20;   SELECT

&#x20;       customer\_id,

&#x20;       SUM(time\_spent\_sec) AS total\_time,

&#x20;       COUNT(DISTINCT sessions\_id) AS total\_sessions,

&#x20;       SUM(CASE WHEN added\_to\_cart = 'Yes' OR added\_to\_cart = '1' THEN 1 ELSE 0 END) AS total\_carts,

&#x20;       SUM(CASE WHEN page\_type = 'product\_page' THEN 1 ELSE 0 END) AS total\_product\_views,

&#x20;       SUM(CASE WHEN added\_to\_wishlist = 'Yes' OR added\_to\_wishlist = '1' THEN 1 ELSE 0 END) AS total\_wishlist,

&#x20;       MODE() WITHIN GROUP (ORDER BY device) AS primary\_device

&#x20;   FROM browsing\_events

&#x20;   GROUP BY customer\_id

) -- No semicolon here!

SELECT \* FROM customer\_behavior; -- Semicolon goes at the very end of the final action



\-----

\-- 1. First, get the raw counts per segment and category

, category\_counts AS (

&#x20;   SELECT

&#x20;       s.rfm\_segment,

&#x20;       b.category\_viewed,

&#x20;       COUNT(\*) as view\_count

&#x20;   FROM industry\_level\_segmentation s

&#x20;   JOIN browsing\_events b ON s.customer\_id = b.customer\_id

&#x20;   GROUP BY 1, 2

),

\---

**-- phase 3 (2)**

WITH customer\_behavior AS (

&#x20;   SELECT

&#x20;       customer\_id,

&#x20;       SUM(time\_spent\_sec) AS total\_time,

&#x20;       COUNT(DISTINCT sessions\_id) AS total\_sessions,

&#x20;       -- Count unique sessions where an add-to-cart happened

&#x20;       COUNT(DISTINCT CASE

&#x20;           WHEN LOWER(TRIM(added\_to\_cart::text)) IN ('yes', '1', 'true', 't')

&#x20;           THEN sessions\_id

&#x20;       END) AS sessions\_with\_cart,

&#x20;       SUM(CASE

&#x20;           WHEN LOWER(TRIM(added\_to\_wishlist::text)) IN ('yes', '1', 'true', 't') THEN 1

&#x20;           ELSE 0

&#x20;       END) AS total\_wishlist,

&#x20;       MODE() WITHIN GROUP (ORDER BY device) AS primary\_device

&#x20;   FROM browsing\_events

&#x20;   GROUP BY customer\_id

),



category\_counts AS (

&#x20;   SELECT

&#x20;       s.rfm\_segment,

&#x20;       b.category\_viewed,

&#x20;       COUNT(\*) as view\_count

&#x20;   FROM industry\_level\_segmentation s

&#x20;   JOIN browsing\_events b ON s.customer\_id = b.customer\_id

&#x20;   WHERE b.category\_viewed IS NOT NULL

&#x20;   GROUP BY 1, 2

),



category\_rankings AS (

&#x20;   SELECT

&#x20;       rfm\_segment,

&#x20;       category\_viewed,

&#x20;       ROW\_NUMBER() OVER(PARTITION BY rfm\_segment ORDER BY view\_count DESC) as rank

&#x20;   FROM category\_counts

),



top\_categories AS (

&#x20;   SELECT

&#x20;       rfm\_segment,

&#x20;       STRING\_AGG(category\_viewed, ', ') AS top\_3\_categories

&#x20;   FROM category\_rankings

&#x20;   WHERE rank <= 3

&#x20;   GROUP BY 1

)



SELECT

&#x20;   s.rfm\_segment,

&#x20;   ROUND(AVG(cb.total\_time / NULLIF(cb.total\_sessions, 0)), 2) AS avg\_sec\_per\_session,

&#x20;   COALESCE(tc.top\_3\_categories, 'No Data') AS top\_3\_categories,

&#x20;   -- NEW DENOMINATOR: Total Sessions (Guarantees <= 100%)

&#x20;   COALESCE(

&#x20;       ROUND((SUM(cb.sessions\_with\_cart) \* 100.0 / NULLIF(SUM(cb.total\_sessions), 0)), 2)::TEXT || '%',

&#x20;       '0.00%'

&#x20;   ) AS session\_to\_cart\_rate,

&#x20;   MODE() WITHIN GROUP (ORDER BY cb.primary\_device) AS segment\_device,

&#x20;   ROUND(AVG(cb.total\_wishlist), 2) AS avg\_wishlist\_size

FROM industry\_level\_segmentation s

LEFT JOIN customer\_behavior cb ON s.customer\_id = cb.customer\_id

LEFT JOIN top\_categories tc ON s.rfm\_segment = tc.rfm\_segment

GROUP BY s.rfm\_segment, tc.top\_3\_categories

ORDER BY session\_to\_cart\_rate DESC;



\--**phase 3 (3)**



WITH customer\_browsing\_intent AS (

&#x20;   -- Step 1: Find the top category each customer browses

&#x20;   SELECT

&#x20;       customer\_id,

&#x20;       MODE() WITHIN GROUP (ORDER BY category\_viewed) AS top\_browsed\_category

&#x20;   FROM browsing\_events

&#x20;   GROUP BY 1

),



customer\_purchase\_history AS (

&#x20;   -- Step 2: Find the top category each customer actually buys

&#x20;   -- Joining order\_items with products to get category names

&#x20;   SELECT

&#x20;       o.customer\_id,

&#x20;       MODE() WITHIN GROUP (ORDER BY p.category) AS top\_purchased\_category

&#x20;   FROM order\_items oi

&#x20;   JOIN orders o ON oi.order\_id = o.order\_id

&#x20;   JOIN products p ON oi.product\_id = p.product\_id

&#x20;   GROUP BY 1

)



\-- Step 3: Compare Intent vs. Action by Segment

SELECT

&#x20;   s.rfm\_segment,

&#x20;   bi.top\_browsed\_category AS intent\_category,

&#x20;   ph.top\_purchased\_category AS actual\_purchase\_category,

&#x20;   COUNT(s.customer\_id) AS customer\_count,

&#x20;   -- Identify the Gap

&#x20;   CASE

&#x20;       WHEN bi.top\_browsed\_category = ph.top\_purchased\_category THEN 'Aligned'

&#x20;       ELSE 'Purchase Gap'

&#x20;   END AS alignment\_status

FROM rfm\_final\_segments s

JOIN customer\_browsing\_intent bi ON s.customer\_id = bi.customer\_id

LEFT JOIN customer\_purchase\_history ph ON s.customer\_id = ph.customer\_id

GROUP BY 1, 2, 3

ORDER BY customer\_count DESC;



\---**phase 3 (4)**

WITH campaign\_data\_cleaned AS (

&#x20;   -- Step 1: Fix the casting logic

&#x20;   -- String "TRUE" -> BOOLEAN -> INTEGER (1)

&#x20;   SELECT

&#x20;       r.rfm\_segment,

&#x20;       c.campaign\_type,

&#x20;       COALESCE(c.opened::BOOLEAN::INT, 0) AS is\_open,

&#x20;       COALESCE(c.clicked::BOOLEAN::INT, 0) AS is\_click,

&#x20;       COALESCE(c.converted::BOOLEAN::INT, 0) AS is\_conv,

&#x20;       COALESCE(c.revenue\_generated, 0) AS revenue

&#x20;   FROM rfm\_final\_segments r

&#x20;   JOIN campaign\_responses c ON r.customer\_id = c.customer\_id

),



campaign\_aggregates AS (

&#x20;   -- Step 2: Sum the standardized values

&#x20;   SELECT

&#x20;       rfm\_segment,

&#x20;       campaign\_type,

&#x20;       COUNT(\*) AS total\_delivered,

&#x20;       SUM(is\_open) AS total\_opens,

&#x20;       SUM(is\_click) AS total\_clicks,

&#x20;       SUM(is\_conv) AS total\_conversions,

&#x20;       SUM(revenue) AS revenue\_contribution

&#x20;   FROM campaign\_data\_cleaned

&#x20;   GROUP BY 1, 2

)



\-- Step 3: Calculate rates with 100.0 for float precision

SELECT

&#x20;   rfm\_segment,

&#x20;   campaign\_type,

&#x20;   total\_delivered,

&#x20;

&#x20;   ROUND((total\_opens \* 100.0) / NULLIF(total\_delivered, 0), 2) || '%' AS open\_rate,

&#x20;

&#x20;   ROUND((total\_clicks \* 100.0) / NULLIF(total\_opens, 0), 2) || '%' AS click\_rate,

&#x20;

&#x20;   ROUND((total\_conversions \* 100.0) / NULLIF(total\_clicks, 0), 2) || '%' AS conversion\_rate,

&#x20;

&#x20;   revenue\_contribution

FROM campaign\_aggregates

ORDER BY rfm\_segment, revenue\_contribution DESC;



\--- **phase 4 (1)**

WITH customer\_lifespan\_data AS (

&#x20;   SELECT

&#x20;       r.customer\_id,

&#x20;       r.rfm\_segment,

&#x20;       COUNT(DISTINCT o.order\_id) as total\_orders,

&#x20;       SUM(oi.final\_price) as total\_customer\_revenue,

&#x20;       MIN(o.order\_date) as first\_purchase,

&#x20;       MAX(o.order\_date) as last\_purchase,

&#x20;       -- Subtracting dates gives an integer (days). Divide by 365.0 for years.

&#x20;       -- We use GREATEST to ensure we don't divide by zero for one-day customers.

&#x20;       GREATEST(

&#x20;           (MAX(o.order\_date) - MIN(o.order\_date)) / 365.0,

&#x20;           0.1

&#x20;       ) as tenure\_years

&#x20;   FROM rfm\_final\_segments r

&#x20;   JOIN orders o ON r.customer\_id = o.customer\_id

&#x20;   JOIN order\_items oi ON o.order\_id = oi.order\_id

&#x20;   GROUP BY r.customer\_id, r.rfm\_segment

),

segment\_stats AS (

&#x20;   SELECT

&#x20;       rfm\_segment,

&#x20;       COUNT(customer\_id) as segment\_size,

&#x20;       AVG(total\_customer\_revenue / NULLIF(total\_orders, 0)) as avg\_order\_value,

&#x20;       AVG(total\_orders / tenure\_years) as yearly\_frequency,

&#x20;       -- Churn Proxy: % of customers who haven't ordered in the last 6 months

&#x20;       COUNT(CASE WHEN last\_purchase < (SELECT MAX(order\_date) FROM orders) - INTERVAL '6 months' THEN 1 END)::float

&#x20;           / COUNT(customer\_id) as churn\_rate

&#x20;   FROM customer\_lifespan\_data

&#x20;   GROUP BY rfm\_segment

)

SELECT

&#x20;   rfm\_segment,

&#x20;   segment\_size,

&#x20;   ROUND(avg\_order\_value::numeric, 2) as aov,

&#x20;   ROUND(yearly\_frequency::numeric, 2) as purchase\_freq\_per\_year,

&#x20;   ROUND(LEAST(1 / NULLIF(churn\_rate, 0), 7)::numeric, 1) as estimated\_lifespan\_yrs,

&#x20;   ROUND(

&#x20;       (avg\_order\_value \* yearly\_frequency \* LEAST(1 / NULLIF(churn\_rate, 0), 7))::numeric,

&#x20;       2

&#x20;   ) as customer\_lifetime\_value

FROM segment\_stats

ORDER BY customer\_lifetime\_value DESC;



\---**phase 4 (2)**



SELECT

&#x20;   s.rfm\_segment,

&#x20;   ROUND(AVG(c.revenue\_generated)::numeric, 2) AS avg\_rev\_per\_customer,

&#x20;   SUM(c.revenue\_generated) AS total\_segment\_revenue,

&#x20;   ROUND(

&#x20;       (SUM(c.revenue\_generated) \* 100.0 / SUM(SUM(c.revenue\_generated)) OVER())::numeric,

&#x20;       2

&#x20;   ) AS revenue\_contribution\_pct

FROM rfm\_final\_segments s

JOIN campaign\_responses c ON s.customer\_id = c.customer\_id

GROUP BY s.rfm\_segment

ORDER BY total\_segment\_revenue DESC;

&#x20;

&#x20;

&#x20;---**phase 4 (3)**

&#x20;WITH order\_item\_summary AS (

&#x20;   SELECT

&#x20;       o.customer\_id,

&#x20;       o.order\_id,

&#x20;       o.is\_festive\_order,

&#x20;       o.discount\_code,

&#x20;       oi.final\_price,

&#x20;       p.category

&#x20;   FROM orders o

&#x20;   JOIN order\_items oi ON o.order\_id = oi.order\_id

&#x20;   JOIN products p ON oi.product\_id = p.product\_id

),

order\_level\_aggregation AS (

&#x20;   SELECT

&#x20;       customer\_id,

&#x20;       order\_id,

&#x20;       is\_festive\_order,

&#x20;       discount\_code,

&#x20;       SUM(final\_price) AS total\_order\_value,

&#x20;       -- We take the first category found for the order mix analysis

&#x20;       MODE() WITHIN GROUP (ORDER BY category) AS primary\_category

&#x20;   FROM order\_item\_summary

&#x20;   GROUP BY customer\_id, order\_id, is\_festive\_order, discount\_code

)

SELECT

&#x20;   s.rfm\_segment,

&#x20;   ola.is\_festive\_order,

&#x20;   ROUND(AVG(ola.total\_order\_value)::numeric, 2) AS avg\_order\_value,

&#x20;   ROUND(

&#x20;       COUNT(CASE WHEN ola.discount\_code IS NOT NULL AND ola.discount\_code <> '' THEN 1 END) \* 100.0 / COUNT(\*),

&#x20;       2

&#x20;   ) AS discount\_usage\_rate\_pct,

&#x20;   MODE() WITHIN GROUP (ORDER BY ola.primary\_category) AS dominant\_category\_mix

FROM rfm\_final\_segments s

JOIN order\_level\_aggregation ola ON s.customer\_id = ola.customer\_id

GROUP BY s.rfm\_segment, ola.is\_festive\_order

ORDER BY s.rfm\_segment, ola.is\_festive\_order DESC;



\----**phase 4 (4)**

SELECT

&#x20;   rfm\_segment,

&#x20;   COUNT(DISTINCT customer\_id) AS total\_customers,

&#x20;   ROUND(SUM(final\_price) / COUNT(DISTINCT customer\_id), 2) AS avg\_revenue\_per\_customer,

&#x20;   SUM(final\_price) AS total\_segment\_revenue,

&#x20;   -- Ranking to quickly see the top of each category

&#x20;   RANK() OVER (ORDER BY COUNT(DISTINCT customer\_id) DESC) AS volume\_rank,

&#x20;   RANK() OVER (ORDER BY (SUM(final\_price) / COUNT(DISTINCT customer\_id)) DESC) AS revenue\_rank

FROM (

&#x20;   SELECT

&#x20;       s.rfm\_segment,

&#x20;       s.customer\_id,

&#x20;       oi.final\_price

&#x20;   FROM rfm\_final\_segments s

&#x20;   JOIN order\_items oi ON s.customer\_id = (SELECT customer\_id FROM orders WHERE order\_id = oi.order\_id)

) sub

GROUP BY rfm\_segment

ORDER BY avg\_revenue\_per\_customer DESC;



**phase 5 (1)**



WITH segment\_category\_counts AS (

&#x20;   -- Step 1: Count views per category for each segment

&#x20;   SELECT

&#x20;       s.rfm\_segment,

&#x20;       be.category\_viewed,

&#x20;       COUNT(\*) as view\_count,

&#x20;       -- Step 2: Rank categories within each segment by popularity

&#x20;       RANK() OVER (PARTITION BY s.rfm\_segment ORDER BY COUNT(\*) DESC) as category\_rank

&#x20;   FROM rfm\_final\_segments s

&#x20;   JOIN browsing\_events be ON s.customer\_id = be.customer\_id

&#x20;   WHERE be.category\_viewed IS NOT NULL -- Filters out those \[null] categories

&#x20;   GROUP BY s.rfm\_segment, be.category\_viewed

),

top\_segment\_categories AS (

&#x20;   -- Step 3: Keep only the #1 category for each segment

&#x20;   SELECT rfm\_segment, category\_viewed

&#x20;   FROM segment\_category\_counts

&#x20;   WHERE category\_rank = 1

)

SELECT

&#x20;   s.rfm\_segment,

&#x20;   -- Channel Assignment

&#x20;   CASE

&#x20;       WHEN s.rfm\_segment = 'Champions' THEN 'In-App'

&#x20;       WHEN s.rfm\_segment IN ('At Risk', 'Can''t Lose Them', 'Hibernating') THEN 'SMS'

&#x20;       WHEN s.rfm\_segment IN ('About to Sleep', 'Loyal Customers') THEN 'Email'

&#x20;       ELSE 'Push Notification'

&#x20;   END AS primary\_channel,

&#x20;   -- Offer Assignment

&#x20;   CASE

&#x20;       WHEN s.rfm\_segment = 'Champions' THEN 'Early Access'

&#x20;       WHEN s.rfm\_segment = 'Loyal Customers' THEN 'Loyalty Points'

&#x20;       WHEN s.rfm\_segment IN ('At Risk', 'Hibernating', 'Can''t Lose Them') THEN 'Win-back Coupon'

&#x20;       ELSE 'Percentage Discount'

&#x20;   END AS offer\_type,

&#x20;   -- The "Fix": Grabbing the top category only

&#x20;   COALESCE(tsc.category\_viewed, 'General Trending') AS featured\_category,

&#x20;   -- Discount Depth (Matching your data findings)

&#x20;   CASE

&#x20;       WHEN s.rfm\_segment IN ('At Risk', 'Hibernating') THEN '50%'

&#x20;       WHEN s.rfm\_segment = 'Can''t Lose Them' THEN '40%'

&#x20;       WHEN s.rfm\_segment = 'About to Sleep' THEN '25%'

&#x20;       WHEN s.rfm\_segment IN ('Champions', 'Loyal Customers') THEN '0-10%'

&#x20;       ELSE '15%'

&#x20;   END AS discount\_depth,

&#x20;   -- Justification

&#x20;   CASE

&#x20;       WHEN s.rfm\_segment = 'At Risk' THEN 'High festive AOV (\~36k) + 100% discount reliance; high-impact SMS required.'

&#x20;       WHEN s.rfm\_segment = 'About to Sleep' THEN 'Pivot to Essentials noted in festive analysis; Email for utility.'

&#x20;       WHEN s.rfm\_segment = 'Champions' THEN 'Premium segment; prioritize exclusivity over margin loss.'

&#x20;       ELSE 'Optimized based on segment-wide browsing frequency.'

&#x20;   END AS data\_justification

FROM (SELECT DISTINCT rfm\_segment FROM rfm\_final\_segments) s

LEFT JOIN top\_segment\_categories tsc ON s.rfm\_segment = tsc.rfm\_segment

ORDER BY rfm\_segment;



**---phase 5 (2)**

SELECT

&#x20;   campaign\_type,

&#x20;   COUNT(customer\_id) AS total\_sent,

&#x20;   -- Check if the string matches 'TRUE' (case-insensitive)

&#x20;   ROUND(

&#x20;       (SUM(CASE WHEN UPPER(opened) = 'TRUE' THEN 1 ELSE 0 END) \* 100.0) / NULLIF(COUNT(customer\_id), 0), 2

&#x20;   ) AS open\_rate\_pct,

&#x20;   -- Same logic for converted

&#x20;   ROUND(

&#x20;       (SUM(CASE WHEN UPPER(converted) = 'TRUE' THEN 1 ELSE 0 END) \* 100.0) / NULLIF(COUNT(customer\_id), 0), 2

&#x20;   ) AS conversion\_rate\_pct,

&#x20;   SUM(revenue\_generated) AS total\_revenue

FROM campaign\_responses

GROUP BY campaign\_type

ORDER BY conversion\_rate\_pct DESC;



\------------------**check**

SELECT

&#x20;   tc.table\_name AS child\_table,

&#x20;   kcu.column\_name AS child\_column,

&#x20;   ccu.table\_name AS parent\_table,

&#x20;   ccu.column\_name AS parent\_column

FROM

&#x20;   information\_schema.table\_constraints AS tc

&#x20;   JOIN information\_schema.key\_column\_usage AS kcu

&#x20;     ON tc.constraint\_name = kcu.constraint\_name

&#x20;   JOIN information\_schema.constraint\_column\_usage AS ccu

&#x20;     ON ccu.constraint\_name = tc.constraint\_name

WHERE tc.constraint\_type = 'FOREIGN KEY'

AND tc.table\_name IN ('orders', 'order\_items', 'customers', 'industry\_level\_segmentation', 'browsing\_events', 'campaign\_responses');



\------------**check 2 (for relationships)**



SELECT

&#x20;   tc.table\_name AS child\_table,

&#x20;   kcu.column\_name AS child\_column,

&#x20;   ccu.table\_name AS parent\_table,

&#x20;   ccu.column\_name AS parent\_column

FROM

&#x20;   information\_schema.table\_constraints AS tc

&#x20;   JOIN information\_schema.key\_column\_usage AS kcu

&#x20;     ON tc.constraint\_name = kcu.constraint\_name

&#x20;   JOIN information\_schema.constraint\_column\_usage AS ccu

&#x20;     ON ccu.constraint\_name = tc.constraint\_name

WHERE tc.constraint\_type = 'FOREIGN KEY'

ORDER BY child\_table;



\-----**Master check ER**

CREATE OR REPLACE VIEW v\_master\_customer\_intelligence AS

WITH behavior\_metrics AS (

&#x20;   SELECT

&#x20;       customer\_id,

&#x20;       COUNT(DISTINCT sessions\_id) AS total\_sessions,

&#x20;       SUM(time\_spent\_sec) AS total\_time\_spent,

&#x20;       -- Corrected Session-to-Cart logic

&#x20;       COUNT(DISTINCT CASE WHEN LOWER(TRIM(added\_to\_cart::text)) IN ('yes', '1', 'true', 't') THEN sessions\_id END) AS cart\_sessions,

&#x20;       SUM(CASE WHEN LOWER(TRIM(added\_to\_wishlist::text)) IN ('yes', '1', 'true', 't') THEN 1 ELSE 0 END) AS wishlist\_count

&#x20;   FROM browsing\_events

&#x20;   GROUP BY customer\_id

),

category\_ranking AS (

&#x20;   SELECT

&#x20;       customer\_id,

&#x20;       category\_viewed,

&#x20;       ROW\_NUMBER() OVER(PARTITION BY customer\_id ORDER BY COUNT(\*) DESC) as rank

&#x20;   FROM browsing\_events

&#x20;   WHERE category\_viewed IS NOT NULL

&#x20;   GROUP BY customer\_id, category\_viewed

),

top\_cats AS (

&#x20;   SELECT

&#x20;       customer\_id,

&#x20;       STRING\_AGG(category\_viewed, ', ') AS preferred\_categories

&#x20;   FROM category\_ranking

&#x20;   WHERE rank <= 3

&#x20;   GROUP BY customer\_id

)

SELECT

&#x20;   -- 1. Identity \& Demographics (Updated to your exact column names)

&#x20;   c.customer\_id,

&#x20;   c.full\_name,

&#x20;   c.age,

&#x20;   c.gender,

&#x20;   c.city,

&#x20;   c.state,

&#x20;   ct.tier AS city\_tier, -- Mapping 'tier' to 'city\_tier' for the view

&#x20;   c.signup\_date,

&#x20;

&#x20;   -- 2. RFM Segmentation \& Interests

&#x20;   rfm.rfm\_segment,

&#x20;   tc.preferred\_categories,

&#x20;

&#x20;   -- 3. Sales Performance (Facts)

&#x20;   COUNT(DISTINCT o.order\_id) AS total\_orders,

&#x20;   ROUND(SUM(oi.final\_price)::NUMERIC, 2) AS total\_revenue,

&#x20;

&#x20;   -- 4. Behavioral Metrics

&#x20;   COALESCE(bm.total\_sessions, 0) AS total\_sessions,

&#x20;   COALESCE(ROUND((bm.cart\_sessions \* 100.0 / NULLIF(bm.total\_sessions, 0)), 2), 0) AS session\_to\_cart\_rate,

&#x20;   COALESCE(bm.wishlist\_count, 0) AS total\_wishlist\_items



FROM customers c

LEFT JOIN city\_tier\_mapping ct ON c.city = ct.city

LEFT JOIN industry\_level\_segmentation rfm ON c.customer\_id = rfm.customer\_id

LEFT JOIN orders o ON c.customer\_id = o.customer\_id

LEFT JOIN order\_items oi ON o.order\_id = oi.order\_id

LEFT JOIN behavior\_metrics bm ON c.customer\_id = bm.customer\_id

LEFT JOIN top\_cats tc ON c.customer\_id = tc.customer\_id



GROUP BY

&#x20;   c.customer\_id, c.full\_name, c.age, c.gender, c.city, c.state,

&#x20;   ct.tier, c.signup\_date, rfm.rfm\_segment, tc.preferred\_categories,

&#x20;   bm.total\_sessions, bm.cart\_sessions, bm.wishlist\_count;





\----------- **final master**

SELECT \* FROM v\_master\_customer\_intelligence WHERE total\_revenue > 0 LIMIT 10;





truncate table customer\_preferences;



select  r from rfm\_final\_segments

