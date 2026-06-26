## 🛒 E-Commerce Business Intelligence Analytics (BigQuery)

This repository module documents advanced relational database views compiled in **Google BigQuery** to extract high-level 
strategic insights on customer lifecycle value, operational performance, and transactional behavior.

## 📈 1. Month-over-Month (MoM) Revenue Growth
**Business Goal:** Identify seasonal sales trends and track the platform's compounding growth rate.
```sql
WITH monthly_revenue AS (
    SELECT 
        DATE_TRUNC(DATE(order_purchase_timestamp), MONTH) AS revenue_month,
        ROUND(SUM(price), 2) AS total_sales
    FROM `dsml-scaler-ml.olist_e_commerce.orders` o
    JOIN `dsml-scaler-ml.olist_e_commerce.order_items` i ON o.order_id = i.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY 1
)
SELECT 
    revenue_month,
    total_sales,
    LAG(total_sales) OVER (ORDER BY revenue_month) AS previous_month_sales,
    ROUND(((total_sales - LAG(total_sales) OVER (ORDER BY revenue_month)) / LAG(total_sales) OVER (ORDER BY revenue_month)) * 100, 2) AS mom_growth_pct
FROM monthly_revenue
ORDER BY revenue_month;
```
## 🚚 2. Shipping Performance & Delivery Lag by State
**Business Goal:** Spot geographic logistics bottlenecks by calculating the variance between estimated and actual delivery dates.
```sql
CREATE OR REPLACE VIEW `dsml-scaler-ml.olist_e_commerce.view_shipping_logistics_performance_by_state` AS
SELECT 
  c.customer_state,
  COUNT(o.order_id) AS total_orders,
  ROUND(AVG(DATE_DIFF(DATE(o.order_delivered_customer_date), DATE(o.order_purchase_timestamp), DAY)), 1) AS avg_actual_delivery_days,
  ROUND(AVG(DATE_DIFF(o.order_estimated_delivery_date, DATE(o.order_purchase_timestamp), DAY)), 1) AS avg_estimated_delivery_days,
  ROUND((SUM(CASE WHEN DATE(o.order_delivered_customer_date) > o.order_estimated_delivery_date THEN 1 ELSE 0 END) / COUNT(o.order_id)) * 100, 2) AS late_delivery_percentage
FROM `dsml-scaler-ml.olist_e_commerce.orders` o
JOIN `dsml-scaler-ml.olist_e_commerce.customers` c ON o.customer_id = c.customer_id
WHERE o.order_status = 'delivered' 
  AND o.order_delivered_customer_date IS NOT NULL
GROUP BY c.customer_state
ORDER BY late_delivery_percentage DESC;
```

## 3. Customer RFM Segmentation
**Business Goal:** Segment customers into behavioral cohorts (Recency, Frequency, Monetary) 
using window functions (NTILE) to identify high-value loyalists and churn risks.
```sql
CREATE OR REPLACE VIEW `dsml-scaler-ml.olist_e_commerce.view_customer_rfm_segmentation` AS
WITH customer_raw_metrics AS (
  SELECT
    c.customer_unique_id,
    DATE_DIFF(DATE('2018-09-03'), MAX(DATE(o.order_purchase_timestamp)), DAY) AS recency_days,
    COUNT(DISTINCT o.order_id) AS frequency,
    ROUND(SUM(i.price), 2) AS monetary_value
  FROM `dsml-scaler-ml.olist_e_commerce.orders` o
  JOIN `dsml-scaler-ml.olist_e_commerce.customers` c ON o.customer_id = c.customer_id
  JOIN `dsml-scaler-ml.olist_e_commerce.order_item` i ON o.order_id = i.order_id
  WHERE o.order_status = 'delivered'
  GROUP BY c.customer_unique_id
),
rfm_scores AS (
  SELECT
    customer_unique_id,
    recency_days,
    frequency,
    monetary_value,
    -- Recency: Lower days since last purchase is better -> Score 5
    NTILE(5) OVER (ORDER BY recency_days DESC) AS r_score,
    -- Frequency: Higher number of orders is better -> Score 5
    NTILE(5) OVER (ORDER BY frequency ASC) AS f_score,
    -- Monetary: Higher total spend is better -> Score 5
    NTILE(5) OVER (ORDER BY monetary_value ASC) AS m_score
  FROM customer_raw_metrics
)
SELECT
  customer_unique_id,
  recency_days,
  frequency,
  monetary_value,
  r_score,
  f_score,
  m_score,
  (r_score + f_score + m_score) AS total_rfm_score,
  CASE
    WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
    WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN 'Loyal Customers'
    WHEN r_score >= 4 AND f_score <= 2 THEN 'Recent New Customers'
    WHEN r_score <= 2 AND m_score >= 4 THEN 'At Risk / Big Spenders'
    WHEN r_score <= 1 THEN 'Lost / Churned'
    ELSE 'Regular / Hibernating'
  END AS customer_segment
FROM rfm_scores;
```
## 📦 4. Product Category Revenue & Order Volume Analysis
**Business Goal:** Determine the top-performing product categories by total sales volume and revenue generation
to optimize inventory procurement.
```sql
CREATE OR REPLACE VIEW `dsml-scaler-ml.olist_e_commerce.view_product_category_performance` AS
SELECT 
  COALESCE(p.product_category_name, 'unknown/unlabeled') AS product_category,
  COUNT(i.item_id) AS total_items_sold,
  COUNT(DISTINCT o.order_id) AS total_unique_orders,
  ROUND(SUM(i.price), 2) AS total_revenue,
  ROUND(AVG(i.price), 2) AS avg_item_price
FROM `dsml-scaler-ml.olist_e_commerce.orders` o
JOIN `dsml-scaler-ml.olist_e_commerce.order_item` i ON o.order_id = i.order_id
LEFT JOIN `dsml-scaler-ml.olist_e_commerce.products` p ON i.product_id = p.product_id
WHERE o.order_status = 'delivered'
GROUP BY 1
ORDER BY total_revenue DESC;
```
## 💳 5. Payment Method Preferences & Installment Trends
**Business Goal:** Analyze the distribution of preferred payment types (credit card, boleto, voucher) and check if higher order values correlate
                  with a greater number of credit card installments.
```sql
CREATE OR REPLACE VIEW `dsml-scaler-ml.olist_e_commerce.view_payment_method_installments` AS
SELECT 
  p.payment_type,
  COUNT(o.order_id) AS total_orders,
  ROUND(SUM(p.payment_value), 2) AS total_payment_amount,
  ROUND(AVG(p.payment_value), 2) AS avg_order_payment_value,
  -- Calculate average installments specifically for credit card transactions
  ROUND(AVG(CASE WHEN p.payment_type = 'credit_card' THEN p.payment_installments ELSE NULL END), 1) AS avg_credit_card_installments,
  -- Track the maximum installments utilized
  MAX(p.payment_installments) AS max_installments_used
FROM `dsml-scaler-ml.olist_e_commerce.orders` o
JOIN `dsml-scaler-ml.olist_e_commerce.order_payments` p ON o.order_id = p.order_id
WHERE o.order_status = 'delivered'
GROUP BY 1
ORDER BY total_payment_amount DESC;
```

## ⭐️ 6. Review Score Correlations with Delivery Delay
**Business Goal:** Evaluate how customer satisfaction (review scores) drops when actual delivery dates exceed estimated 
                  delivery timelines,establishing data-driven logistics SLAs (service Level Agreements ) targets.
```sql
CREATE OR REPLACE VIEW `dsml-scaler-ml.olist_e_commerce.view_review_score_delivery_correlation` AS
SELECT 
  r.review_score,
  COUNT(o.order_id) AS total_orders,
  -- Calculate the average discrepancy between actual and estimated delivery (positive = late, negative = early)
  ROUND(AVG(DATE_DIFF(DATE(o.order_delivered_customer_date), date(o.order_estimated_delivery_date), DAY)), 1) AS avg_days_vs_estimate,
  -- Calculate the percentage of orders in this score bucket that arrived late
  ROUND((SUM(CASE WHEN DATE(o.order_delivered_customer_date) > date(o.order_estimated_delivery_date) THEN 1 ELSE 0 END) / COUNT(o.order_id)) * 100, 2) AS late_delivery_percentage
FROM `dsml-scaler-ml.olist_e_commerce.orders` o
JOIN `dsml-scaler-ml.olist_e_commerce.order_reviews` r ON o.order_id = r.order_id
WHERE o.order_status = 'delivered' 
  AND o.order_delivered_customer_date IS NOT NULL
GROUP BY 1
ORDER BY r.review_score DESC;
```
