PART A: HOTEL MANAGEMENT SQL
1) Get user_id and last booked room

SELECT b.user_id, b.room_no
FROM bookings b
JOIN (
    SELECT user_id, MAX(booking_date) AS last_booking
    FROM bookings
    GROUP BY user_id
) lb
ON b.user_id = lb.user_id 
AND b.booking_date = lb.last_booking;

2)Get booking_id and total billing amount for Nov 2021

SELECT bc.booking_id,
       SUM(bc.item_quantity * i.item_rate) AS total_amount
FROM booking_commercials bc
JOIN items i ON bc.item_id = i.item_id
WHERE MONTH(bc.bill_date) = 11 AND YEAR(bc.bill_date) = 2021
GROUP BY bc.booking_id;

3)Get bill_id and bill amount > 1000 in Oct 2021

SELECT bc.bill_id,
       SUM(bc.item_quantity * i.item_rate) AS bill_amount
FROM booking_commercials bc
JOIN items i ON bc.item_id = i.item_id
WHERE MONTH(bc.bill_date) = 10 AND YEAR(bc.bill_date) = 2021
GROUP BY bc.bill_id
HAVING SUM(bc.item_quantity * i.item_rate) > 1000;

4)Get most and least ordered item per month

WITH item_orders AS (
    SELECT 
        MONTH(bc.bill_date) AS month,
        bc.item_id,
        SUM(bc.item_quantity) AS total_qty,
        RANK() OVER (PARTITION BY MONTH(bc.bill_date) ORDER BY SUM(bc.item_quantity) DESC) AS rnk_desc,
        RANK() OVER (PARTITION BY MONTH(bc.bill_date) ORDER BY SUM(bc.item_quantity) ASC) AS rnk_asc
    FROM booking_commercials bc
    GROUP BY MONTH(bc.bill_date), bc.item_id
)
SELECT *
FROM item_orders
WHERE rnk_desc = 1 OR rnk_asc = 1;

5). Get customers with 2nd highest bill each month

WITH bill_amounts AS (
    SELECT 
        b.user_id,
        MONTH(bc.bill_date) AS month,
        bc.bill_id,
        SUM(bc.item_quantity * i.item_rate) AS total_bill,
        RANK() OVER (PARTITION BY MONTH(bc.bill_date) ORDER BY SUM(bc.item_quantity * i.item_rate) DESC) AS rnk
    FROM booking_commercials bc
    JOIN items i ON bc.item_id = i.item_id
    JOIN bookings b ON bc.booking_id = b.booking_id
    GROUP BY b.user_id, MONTH(bc.bill_date), bc.bill_id
)
SELECT *
FROM bill_amounts
WHERE rnk = 2;

PART B: CLINIC MANAGEMENT SQL

1) Get revenue from each sales channel
SELECT sales_channel,
       SUM(amount) AS total_revenue
FROM clinic_sales
WHERE YEAR(datetime) = 2021
GROUP BY sales_channel;


2)Get top 10 most valuable customers

SELECT uid,
       SUM(amount) AS total_spent
FROM clinic_sales
WHERE YEAR(datetime) = 2021
GROUP BY uid
ORDER BY total_spent DESC
LIMIT 10;

3)Get month-wise revenue, expense, profit and status
WITH revenue AS (
    SELECT MONTH(datetime) AS month,
           SUM(amount) AS revenue
    FROM clinic_sales
    WHERE YEAR(datetime) = 2021
    GROUP BY MONTH(datetime)
),
expenses_cte AS (
    SELECT MONTH(datetime) AS month,
           SUM(amount) AS expenses
    FROM expenses
    WHERE YEAR(datetime) = 2021
    GROUP BY MONTH(datetime)
)
SELECT r.month,
       r.revenue,
       e.expenses,
       (r.revenue - e.expenses) AS profit,
       CASE 
           WHEN (r.revenue - e.expenses) > 0 THEN 'Profitable'
           ELSE 'Not Profitable'
       END AS status
FROM revenue r
JOIN expenses_cte e ON r.month = e.month;

4)Get most profitable clinic per city
WITH profit_calc AS (
    SELECT c.city, c.cid,
           SUM(cs.amount) - COALESCE(SUM(e.amount),0) AS profit,
           RANK() OVER (PARTITION BY c.city ORDER BY SUM(cs.amount) - COALESCE(SUM(e.amount),0) DESC) AS rnk
    FROM clinics c
    LEFT JOIN clinic_sales cs ON c.cid = cs.cid
    LEFT JOIN expenses e ON c.cid = e.cid
    GROUP BY c.city, c.cid
)
SELECT *
FROM profit_calc
WHERE rnk = 1;

5)Get second least profitable clinic per state

WITH profit_calc AS (
    SELECT c.state, c.cid,
           SUM(cs.amount) - COALESCE(SUM(e.amount),0) AS profit,
           RANK() OVER (PARTITION BY c.state ORDER BY SUM(cs.amount) - COALESCE(SUM(e.amount),0)) AS rnk
    FROM clinics c
    LEFT JOIN clinic_sales cs ON c.cid = cs.cid
    LEFT JOIN expenses e ON c.cid = e.cid
    GROUP BY c.state, c.cid
)
SELECT *
FROM profit_calc
WHERE rnk = 2;



























