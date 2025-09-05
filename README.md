# SQL Работа с тренировочной базой данных по доставке продуктов
Здесь вы можете ознакомиться с демонстрацией моих навыков работы в SQL и решением практических задач. 

Со схемой базы данных можно ознакомиться в репозитории в файле "Доставка продуктов";

##  Задания:

№1
Для каждой даты в таблицах рассчитайте следующие показатели:
  1. Выручку на пользователя (ARPU) за текущий день;
  2. Выручку на платящего пользователя (ARPPU) за текущий день;
  3. Выручку с заказа, или средний чек (AOV) за текущий день;

```sql
WITH prod_price AS
  (SELECT date, order_id, unn.product_id, price
    FROM   (SELECT creation_time::date as date, order_id, UNNEST(product_ids) as product_id
           FROM   orders
            WHERE  order_id not in (SELECT order_id FROM   user_actions WHERE  action = 'cancel_order')) unn
    JOIN products using (product_id)
  ),
  daily_revenue AS
  (SELECT date, SUM(price) as revenue
  FROM   prod_price
  GROUP BY date
  ),
  pay_users (date, paying_users) AS
  (SELECT time::date as date, COUNT(distinct user_id)
  FROM   user_actions
  WHERE  order_id not in (SELECT order_id FROM   user_actions  WHERE  action = 'cancel_order')
  GROUP BY date
  ),
  all_users AS
  (SELECT time::date as date, COUNT(distinct user_id) as all_users
  FROM   user_actions
  GROUP BY date
  ),
  sum_orders AS
  (SELECT creation_time::date as date, COUNT(distinct order_id) as sum_orders
  FROM   orders
  WHERE  order_id not in (SELECT order_id FROM   user_actions WHERE  action = 'cancel_order')
  GROUP BY date)
SELECT DISTINCT date,
                ROUND(revenue::decimal/all_users, 2) as arpu,
                ROUND(revenue::decimal/paying_users, 2) as arppu,
                ROUND(revenue::decimal/sum_orders, 2) as aov
FROM   prod_price 
  JOIN daily_revenue USING (date)
  JOIN pay_users USING (date) 
  JOIN all_users USING (date) 
  JOIN sum_orders USING (date);
```


№2
Для каждого дня недели рассчитайте следующие показатели:
  1. Выручку на пользователя (ARPU);
  2. Выручку на платящего пользователя (ARPPU);
  3. Выручку на заказ (AOV);

```sql
WITH prod_price AS
  (SELECT date, order_id, unn.product_id, price
  FROM   (SELECT creation_time::date as date, order_id, UNNEST(product_ids) as product_id
          FROM   orders
          WHERE  order_id not in (SELECT order_id FROM user_actions WHERE  action = 'cancel_order')) unn
  JOIN products USING (product_id)
  ),
  daily_revenue AS
  (SELECT to_char(date, 'Day') as weekday, EXTRACT('isodow' FROM   date) as weekday_number, SUM(price) as revenue
  FROM   prod_price
  WHERE  date BETWEEN '2022-08-26' AND '2022-09-08'
  GROUP BY weekday, weekday_number
  ),
  pay_users AS
  (SELECT to_char(time, 'Day') as weekday, EXTRACT('isodow' FROM   time) as weekday_number, COUNT(distinct user_id) as paying_users
  FROM   user_actions
  WHERE  order_id not in (SELECT order_id FROM   user_actions WHERE  action = 'cancel_order') AND time::date BETWEEN '2022-08-26' AND '2022-09-08'
  GROUP BY weekday, weekday_number
  ), 
  all_users AS
  (SELECT to_char(time, 'Day') as weekday, EXTRACT('isodow' FROM   time) as weekday_number, COUNT(distinct user_id) as all_users
  FROM   user_actions
  WHERE  time::date BETWEEN '2022-08-26' AND '2022-09-08'
  GROUP BY weekday, weekday_number
  ),
  sum_orders AS
  (SELECT to_char(creation_time, 'Day') as weekday, EXTRACT('isodow' FROM   creation_time) as weekday_number, COUNT(distinct order_id) as sum_orders
  FROM   orders
  WHERE  order_id not in (SELECT order_id FROM   user_actions WHERE  action = 'cancel_order') AND creation_time::date BETWEEN '2022-08-26' AND '2022-09-08'
  GROUP BY weekday, weekday_number
  )
SELECT weekday,
       weekday_number,
       ROUND(revenue::decimal/all_users, 2) as arpu,
       ROUND(revenue::decimal/paying_users, 2) as arppu,
       ROUND(revenue::decimal/sum_orders, 2) as aov
FROM   daily_revenue 
  JOIN pay_users USING (weekday, weekday_number)
  JOIN all_users USING (weekday, weekday_number) 
  JOIN sum_orders USING (weekday, weekday_number)
ORDER BY weekday_number asc;

```

№3
Для каждого товара за весь период времени рассчитайте следующие показатели:
  1. Суммарную выручку, полученную от продажи этого товара за весь период;
  2. Долю выручки от продажи этого товара в общей выручке, полученной за весь период;
Товары, округлённая доля которых в выручке составляет менее 0.5%, объедините в общую группу с названием «ДРУГОЕ», просуммировав округлённые доли этих товаров.

```sql
SELECT product_name,
       SUM(revenue) as revenue,
       SUM(share_in_revenue) as share_in_revenue
FROM   (SELECT CASE
        WHEN ROUND(100 * revenue / SUM(revenue) OVER (), 2) >= 0.5 THEN name
        ELSE 'ДРУГОЕ'
        END as product_name,
        revenue,
        ROUND(100 * revenue / SUM(revenue) OVER (), 2) as share_in_revenue
        FROM   (SELECT name, SUM(price) as revenue
                FROM   (SELECT order_id, UNNEST(product_ids) as product_id
                        FROM   orders
                        WHERE  order_id not in (SELECT order_id FROM   user_actions WHERE  action = 'cancel_order')) q1
          LEFT JOIN products using(product_id)
        GROUP BY name) q2) q3
GROUP BY product_name
ORDER BY revenue desc;

```

№4
Для каждого дня рассчитайте следующие показатели:
  1. Число платящих пользователей;
  2. Число активных курьеров;
  3. Долю платящих пользователей в общем числе пользователей на текущий день;
  4. Долю активных курьеров в общем числе курьеров на текущий день;

```sql
WITH act_co (date, active_couriers) AS
  (SELECT time::date as date, COUNT(distinct courier_id) FILTER (WHERE action = 'accept_order' or action = 'deliver_order')
  FROM   courier_actions
  WHERE  order_id not in (SELECT order_id FROM   user_actions WHERE  action = 'cancel_order')
  GROUP BY date
  ),
  pay_us (date, paying_users) AS
  (SELECT time::date as date,  COUNT(DISTINCT user_id)
  FROM   user_actions
  WHERE  order_id not in (SELECT order_id FROM   user_actions WHERE  action = 'cancel_order')
  GROUP BY date
  ),
  first_user AS
  (SELECT user_id, MIN(time::date) as date
  FROM   user_actions
  GROUP BY user_id
  ),
  first_courier AS
  (SELECT courier_id, MIN(time::date) as date
  FROM   courier_actions
  GROUP BY courier_id
  ),
  first_all AS
  (SELECT COALESCE(fu.date, fc.date) as date, COUNT(DISTINCT fu.user_id) as new_users, COUNT(DISTINCT fc.courier_id) as new_couriers
  FROM   first_user fu full
    OUTER JOIN first_courier fc ON fu.date = fc.date
  GROUP BY COALESCE(fu.date, fc.date)
  ),
  total_all (date, total_users, total_couriers) AS
  (SELECT date, SUM(new_users) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)::integer as total_users,
  SUM(new_couriers) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)::integer as total_couriers
  FROM   first_all
  )
SELECT date,
       paying_users,
       active_couriers,
       ROUND(paying_users*100.00/total_users, 2) as paying_users_share,
       ROUND(active_couriers*100.00/total_couriers, 2) as active_couriers_share
FROM   act_co 
  JOIN pay_us USING (date)
  JOIN total_all USING (date)
ORDER BY date asc;
```

№5
Для каждого дня рассчитайте, за сколько минут в среднем курьеры доставляли свои заказы.

```SQL
SELECT date,
       ROUND(AVG(delivery_time))::integer as minutes_to_deliver
FROM   (SELECT order_id,
               MAX(time::date) as date,
               EXTRACT(epoch FROM MAX(time) - MIN(time))/60 as delivery_time
        FROM   courier_actions
        WHERE  order_id not in (SELECT order_id FROM   user_actions WHERE  action = 'cancel_order')
        GROUP BY order_id) q1
GROUP BY date
ORDER BY date;

```
№6
Для каждого часа в сутках рассчитайте следующие показатели:
  1. Число успешных (доставленных) заказов;
  2. Число отменённых заказов;
  3. Долю отменённых заказов в общем числе заказов (cancel rate);

```sql
WITH success AS 
  (SELECT hour, COUNT(order_id) as successful_orders
  FROM  (SELECT EXTRACT(hour FROM creation_time)::integer as hour, order_id
        FROM   orders
        WHERE  order_id not in (SELECT order_id FROM   user_actions WHERE  action = 'cancel_order')) q1
  GROUP BY hour
  ), 
  cancel AS 
  (SELECT hour,COUNT(order_id) as canceled_orders
  FROM  (SELECT EXTRACT(hour FROM creation_time)::integer as hour, order_id
        FROM   orders
        WHERE  order_id in (SELECT order_id FROM   user_actions WHERE  action = 'cancel_order')) q2
  GROUP BY hour
  )
SELECT hour,
       successful_orders,
       canceled_orders,
       ROUND(canceled_orders::decimal/(successful_orders+canceled_orders), 3) as cancel_rate
FROM   cancel 
  JOIN success USING (hour)
ORDER BY hour asc;
```

## Спасибо, что просмотрели мою работу до конца!



