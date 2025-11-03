# <p align="center" style="margin-top: 0px;">🍜 Case Study #2: Pizza Runner 
## <p align="center"><img src="https://github.com/RonaldF-Z/SQL-Case_Studies/blob/main/8WeeksSQLChallenge/2%20-%20Pizza%20Runner/resources/c2.png?raw=true" alt="Image" width="400" height="420">

## <p align="center">Notes
- ##### Case study's data and questions were extracted from this link: [here](https://8weeksqlchallenge.com/case-study-2/). 
- ##### All questions were solved using MSSQL Server and PostgreSQL

## <p align="center">Introduction
Did you know that over 115 million kilograms of pizza is consumed daily worldwide??? (Well according to Wikipedia anyway…)

Danny was scrolling through his Instagram feed when something really caught his eye - “80s Retro Styling and Pizza Is The Future!”

Danny was sold on the idea, but he knew that pizza alone was not going to help him get seed funding to expand his new Pizza Empire - so he had one more genius idea to combine with it - he was going to Uberize it - and so Pizza Runner was launched!

Danny started by recruiting “runners” to deliver fresh pizza from Pizza Runner Headquarters (otherwise known as Danny’s house) and also maxed out his credit card to pay freelance developers to build a mobile app to accept orders from customers.

## <p align="center">Questions
This case study has LOTS of questions - they are broken up by area of focus including:

- Pizza Metrics
- Runner and Customer Experience
- Ingredient Optimisation
- Pricing and Ratings
- Bonus DML Challenges (DML = Data Manipulation Language)

Each of the following case study questions can be answered using a single SQL statement.
Again, there are many questions in this case study - please feel free to pick and choose which ones you’d like to try!
Before you start writing your SQL queries however - you might want to investigate the data, you may want to do something with some of those null values and data types in the customer_orders and runner_orders tables!

## MSSQL

### <p align="center">A. Pizza Metrics
#### 1.How many pizzas were ordered?
```sql
select count(pizza_id) total_pizzas_ordered from customer_orders;
```

#### 2.How many unique customer orders were made?
```sql
select count(distinct customer_id) nro_customers from customer_orders;
```

#### 3.How many successful orders were delivered by each runner?
```sql
update runner_orders set cancellation = case when cancellation='' then null
                                              when UPPER(cancellation)='NULL' then null
                                              else cancellation end;
select runner_id,count(distinct order_id) orders_delivered from runner_orders
where cancellation is null
group by runner_id
```
#### 4.How many of each type of pizza was delivered?
```sql
alter table customer_orders alter column order_id integer;
alter table customer_orders alter column pizza_id integer;
alter table runner_orders alter column order_id integer;
alter table runner_orders alter column runner_id integer;
alter table pizza_names alter column pizza_name varchar(15);

select p.pizza_name,count(co.pizza_id) nro_pizza_delivered from customer_orders co
inner join pizza_names p on co.pizza_id=p.pizza_id
inner join runner_orders ro on co.order_id=ro.order_id
where ro.cancellation is null
group by p.pizza_name
```
#### 5.How many Vegetarian and Meatlovers were ordered by each customer?
```sql
select c.customer_id,
  sum(case when p.pizza_name="Meatlovers" then 1 else 0 end) Meatlovers,
  sum(case when p.pizza_name="Vegetarian" then 1 else 0 end) Vegetarian
from customer_orders c
inner join pizza_names p on c.pizza_id=p.pizza_id
group by c.customer_id
order by c.customer_id
```
#### 6.What was the maximum number of pizzas delivered in a single order?
```sql
select c.order_id,count(c.pizza_id) nro_pizza_delivered from customer_orders c
inner join runner_orders r on c.order_id=r.order_id
where r.cancellation is null
group by c.order_id
order by 2 desc
```
#### 7.For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
```sql
update customer_orders set exclusions = case when exclusions = '' then null
                                              when UPPER(exclusions)='NULL' then null
                                              else exclusions end;
update customer_orders set extras = case when extras = '' then null
                                              when UPPER(extras)='NULL' then null
                                              else extras end;                                              
select customer_id,
  sum(case when exclusions is not null or extras is not null then 1 else 0 end) with_changes,
    sum(case when exclusions is null and extras is null then 1 else 0 end) no_changes
from customer_orders c
inner join runner_orders r on c.order_id = r.order_id
where r.cancellation is null
group by customer_id
```
#### 8.How many pizzas were delivered that had both exclusions and extras?
```sql
select customer_id,
  sum(case when exclusions is not null and extras is not null then 1 else 0 end) both_changes
from customer_orders c
inner join runner_orders r on c.order_id = r.order_id
where r.cancellation is null
group by customer_id
```
#### 9.What was the total volume of pizzas ordered for each hour of the day?
```sql
with hour_cte as (
  select 0 as hour_order_time
  union all
  select hour_order_time+1 from hour_cte
  where hour_order_time<24
)
select hcte.hour_order_time,count(c.pizza_id) nro_pizzas from customer_orders c
right join hour_cte hcte on datepart(hour,c.order_time)=hcte.hour_order_time
group by hcte.hour_order_time
```
#### 10.What was the volume of orders for each day of the week?
```sql
select datepart(weekday,order_time) date_num,datename(weekday,order_time) date_name,count(distinct order_id) nro_orders from customer_orders
group by datename(weekday,order_time),datepart(weekday,order_time)
order by datepart(weekday,order_time)
```
### <p align="center">B. Runner and Customer Experience
#### 1.How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
```sql
select datepart(week,registration_date) week,count(runner_id) nro_runners from runners
group by datepart(week,registration_date)
order by 1
```
#### 2.What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
```sqlwith cte as (
  select datediff(minute,c.order_time,r.pickup_time) time_preparation_order,r.runner_id from customer_orders c
  inner join runner_orders r on c.order_id = r.order_id
  where r.cancellation is null
)
select runner_id,round(avg(time_preparation_order*1.0),2) avg_arrive from cte
group by runner_id
order by 1
```
#### 3.Is there any relationship between the number of pizzas and how long the order takes to prepare?
```sql
update runner_orders set pickup_time = case when UPPER(pickup_time)='NULL' then null else pickup_time end;
alter table runner_orders alter column pickup_time datetime;

with cte as (
  select datediff(minute,c.order_time,r.pickup_time) time_preparation_order,r.order_id,c.pizza_id from customer_orders c
  inner join runner_orders r on c.order_id = r.order_id
  where r.cancellation is null
)
select order_id,time_preparation_order,count(pizza_id) nro_pizzas from cte
group by order_id,time_preparation_order
order by 1
```
 . R: El tiempo de preparación de una pizza suele ser 10 minutos y al ordenar más cantidades el tiempo suele subir en una proporción aproximadamente equivalente, excepto en la orden 8
#### 4.What was the average distance travelled for each customer?
```sql
update runner_orders set distance=case when UPPER(distance)='NULL' then null else distance end;

with cte as (
  select c.customer_id,c.order_id,
    distance,case when charindex('k',distance)!=0 then charindex('k',distance) else len(distance)+1 end flag_index
  from customer_orders c
  inner join runner_orders r on c.order_id = r.order_id
  where cancellation is null
)
select customer_id,avg(cast(left(distance,flag_index-1) as decimal(3,1))) as distance_km from cte
group by customer_id
order by 1
```
#### 5.What was the difference between the longest and shortest delivery times for all orders?
```sql
update runner_orders set duration=case when UPPER(duration)='NULL' then null else duration end;

select max(cast(left(duration,2) as int))-min(cast(left(duration,2) as int)) diff_max_min_duration from runner_orders
where cancellation is null
```
#### 6.What was the average speed for each runner for each delivery and do you notice any trend for these values?
```sql
with cte as (
  select runner_id,pickup_time,
    case when charindex('k',distance)!=0 then charindex('k',distance) else len(distance)+1 end flag_index,
    distance,duration,order_id
  from runner_orders
  where cancellation is null
),speed_cte as (
  select runner_id,order_id,pickup_time,
  cast(left(distance,flag_index-1) as decimal(3,1)) distance,cast(left(duration,2) as int)*60 duration_secs from cte
)
select runner_id,order_id,pickup_time,distance*1000/duration_secs speed_ms from speed_cte
order by 1,3
```
 . R: Se puede notar que todos los runners empiezan con una velocidad lenta al repartir los primeros dias y mientras trancurre los dias su velocidad va aumentando
#### 7.What is the successful delivery percentage for each runner?
```sql
with cte as (
  select runner_id,
    sum(case when cancellation is null then 1 else 0 end) as nro_successful_delivery,
    count(runner_id) total_delivery from runner_orders
  group by runner_id
)
select runner_id,nro_successful_delivery*100.0/total_delivery successful_delivery_percentage from cte
```

### <p align="center"> C. Ingredient Optimisation
#### 1.What are the standard ingredients for each pizza?
```sql
with pizza_recipes_cte as (
  select pizza_id,value as topping from pizza_recipes
  cross apply string_split(replace(toppings,' ',''),',') r
)
select pcte.pizza_id,string_agg(pt.topping_name,', ') toppings from pizza_recipes_cte pcte
inner join pizza_toppings pt on pcte.topping=pt.topping_id
group by pcte.pizza_id
```
#### 2.What was the most commonly added extra?
```sql
with cte as (
    select value as extra_topping from customer_orders 
    cross apply string_split(replace(extras,' ',''),',') as s 
)
select extra_topping as extras,count(extra_topping) times_extras from cte
group by extra_topping
```
#### 3.What was the most common exclusion?
```sql
with cte as (
    select value as exclude_topping from customer_orders 
    cross apply string_split(replace(exclusions,' ',''),',') as s 
)
select exclude_topping as exclusions,count(exclude_topping) times_exclusions from cte
group by exclude_topping
```
#### 4.Generate an order item for each record in the customers_orders table in the format of one of the following:
  - Meat Lovers
  - Meat Lovers - Exclude Beef
  - Meat Lovers - Extra Bacon
  - Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers*/
```sql
alter table pizza_toppings alter column topping_name varchar(15);

with exclusion_extra_toppings as (
  select order_id,customer_id,pizza_id,exclusions,extras,
    cast(coalesce(exc.value,exclusions) as int) exclude_topping,
    cast(coalesce(ext.value,extras) as int) extra_topping from customer_orders 
  cross apply string_split(replace(exclusions,' ',''),',') as exc
  cross apply string_split(replace(extras,' ',''),',') as ext
  union all
  select order_id,customer_id,pizza_id,exclusions,extras,
    cast(coalesce(exclusions,null) as int) exclude_topping,
    cast(coalesce(extras,null) as int) extra_topping from customer_orders
  where extras is null or exclusions is null
),exclude_toppings_cte as (
  select distinct cte.order_id,cte.customer_id,cte.pizza_id,cte.exclude_topping,p1.topping_name from exclusion_extra_toppings cte
  left join pizza_toppings p1 on cte.exclude_topping=p1.topping_id
  inner join runner_orders r on cte.order_id=r.order_id
  where cte.exclude_topping is not null
),extra_toppings_cte as (
  select distinct cte.order_id,cte.customer_id,cte.pizza_id,cte.extra_topping,p1.topping_name from exclusion_extra_toppings cte
  left join pizza_toppings p1 on cte.extra_topping=p1.topping_id
  inner join runner_orders r on cte.order_id=r.order_id
  where cte.extra_topping is not null
),exclude_topping_names as (
  select order_id,customer_id,pizza_id,
    string_agg(exclude_topping,', ') within group (order by exclude_topping) exclude_toppings,
    string_agg(topping_name,', ') within group (order by exclude_topping) exclude_topping_names from exclude_toppings_cte
  group by order_id,customer_id,pizza_id
),extra_topping_names as (
  select order_id,customer_id,pizza_id,
    string_agg(extra_topping,', ') within group (order by extra_topping) extra_toppings,
    string_agg(topping_name,', ') within group (order by extra_topping) extra_topping_names from extra_toppings_cte
  group by order_id,customer_id,pizza_id
),final_cte as (
  select c.order_id,c.customer_id,c.pizza_id,c.exclusions,c.extras,
    case when c.pizza_id=1 then 'Meat Lovers' else 'Vegetarian' end type_customer,
    "Exclude "+exc.exclude_topping_names exclude_flag,"Extra "+ext.extra_topping_names extra_flag from customer_orders c
  left join exclude_topping_names exc on c.order_id=exc.order_id and c.customer_id=exc.customer_id and c.pizza_id=exc.pizza_id
  left join extra_topping_names ext on c.order_id=ext.order_id and c.customer_id=ext.customer_id and c.pizza_id=ext.pizza_id
)
select order_id,customer_id,pizza_id,exclusions,extras,
  CONCAT_WS(' - ',NULLIF(type_customer,''),NULLIF(exclude_flag,''),NULLIF(extra_flag,'')) format_flag from final_cte
order by order_id
```
#### 5.Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
  For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"*/
```sql
alter table pizza_recipes alter column toppings varchar(25);

with customer_orders_rn as (
  select row_number() over(order by order_id) rn,* from customer_orders
),exclusion_extra_toppings as (
  select rn,order_id,customer_id,pizza_id,exclusions,extras,
    coalesce(exc.value,exclusions,null) exclude_topping,
    coalesce(ext.value,extras,null) extra_topping from customer_orders_rn 
  cross apply string_split(replace(exclusions,' ',''),',') as exc
  cross apply string_split(replace(extras,' ',''),',') as ext
  union all
  select rn,order_id,customer_id,pizza_id,exclusions,extras,
    coalesce(exclusions,null) exclude_topping,
    coalesce(extras,null) extra_topping from customer_orders_rn
  where exclusions is null or extras is null
),toppings_cte as (
  select pizza_id,toppings,cast(value as int) as topping from pizza_recipes 
  cross apply string_split(replace(toppings,' ',''),',') as s 
),all_ingredients_cte as (
  select c.rn,c.order_id,c.customer_id,c.pizza_id,t.topping all_ingredients from customer_orders_rn c
  inner join toppings_cte t on c.pizza_id=t.pizza_id
  where t.topping not in (select distinct c1.exclude_topping from exclusion_extra_toppings c1
                          where c.rn=c1.rn and c.order_id=c1.order_id and c.customer_id=c1.customer_id 
                          and c.pizza_id=c1.pizza_id and exclude_topping is not null)
  union all
  select distinct rn,order_id,customer_id,pizza_id,cast(extra_topping as int) all_ingredients from exclusion_extra_toppings
  where extra_topping is not null
),final_cte as (
  select cte.*,t.topping_name,count(all_ingredients) nro_ingredients from all_ingredients_cte cte
  inner join pizza_toppings t on cte.all_ingredients=t.topping_id
  group by rn,order_id,customer_id,pizza_id,all_ingredients,t.topping_name
)
select order_id,customer_id,pizza_id,
  string_agg(case when nro_ingredients=1 then topping_name else concat(nro_ingredients,'x',topping_name) end,', ') within group (order by topping_name) format_flag 
from final_cte
group by rn,order_id,customer_id,pizza_id
order by 1,2,3,4
```
#### 6.What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?
```sql
with customer_orders_rn as (
  select row_number() over(order by order_id) rn,* from customer_orders
),exclusion_extra_toppings as (
  select rn,order_id,customer_id,pizza_id,exclusions,extras,
    coalesce(exc.value,exclusions,null) exclude_topping,
    coalesce(ext.value,extras,null) extra_topping from customer_orders_rn 
  cross apply string_split(replace(exclusions,' ',''),',') as exc
  cross apply string_split(replace(extras,' ',''),',') as ext
  union all
  select rn,order_id,customer_id,pizza_id,exclusions,extras,
    coalesce(exclusions,null) exclude_topping,
    coalesce(extras,null) extra_topping from customer_orders_rn
  where exclusions is null or extras is null
),toppings_cte as (
  select pizza_id,toppings,cast(value as int) as topping from pizza_recipes 
  cross apply string_split(replace(toppings,' ',''),',') as s 
),all_ingredients_cte as (
  select c.rn,c.order_id,c.customer_id,c.pizza_id,t.topping all_ingredients from customer_orders_rn c
  inner join toppings_cte t on c.pizza_id=t.pizza_id
  where t.topping not in (select distinct c1.exclude_topping from exclusion_extra_toppings c1
                          where c.rn=c1.rn and c.order_id=c1.order_id and c.customer_id=c1.customer_id 
                          and c.pizza_id=c1.pizza_id and exclude_topping is not null)
  union all
  select distinct rn,order_id,customer_id,pizza_id,cast(extra_topping as int) all_ingredients from exclusion_extra_toppings
  where extra_topping is not null
)
select cte.all_ingredients,count(cte.all_ingredients) nro_ingredients from all_ingredients_cte cte
inner join runner_orders r on cte.order_id=r.order_id
where r.cancellation is null
group by cte.all_ingredients
order by 2 desc,1
```

### <p align="center">D. Pricing and Ratings
#### 1.If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
```sql
select sum(case when pizza_id=1 then 12 else 10 end) revenue from customer_orders c 
inner join runner_orders r on c.order_id = r.order_id
where r.cancellation is null
```
#### 2.What if there was an additional $1 charge for any pizza extras?
  - Add cheese is $1 extra*/
```sql
with price_with_extra as (
  select c.order_id,pizza_id,case when pizza_id=1 then 12 else 10 end price,extras,coalesce(len(extras)-len(replace(extras,',',''))+1,0) nro_extras from customer_orders c 
  inner join runner_orders r on c.order_id = r.order_id
  where r.cancellation is null 
)
select sum(price+nro_extras) revenue from price_with_extra
```
#### 3.The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
```sql
drop table if exists runner_rating;
create table runner_ratings (
  order_id integer,
  runner_id integer,
  rating int
)

insert into runner_ratings values
  (1,1,2),
  (2,1,2),
  (3,1,2),
  (4,2,1),
  (5,3,2),
  (6,3,null),
  (7,2,3),
  (8,2,5),
  (9,2,null),
  (10,1,3);

select * from runner_ratings
```
#### 4.Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
  - customer_id
  - order_id
  - runner_id
  - rating
  - order_time
  - pickup_time
  - Time between order and pickup
  - Delivery duration
  - Average speed
  - Total number of pizzas*/
```sql
with cte as (
  select runner_id,case when charindex('k',distance)!=0 then charindex('k',distance) else len(distance)+1 end flag_index,
    distance,duration,order_id
  from runner_orders
  where cancellation is null
),cte2 as (
  select runner_id,order_id,cast(left(distance,flag_index-1) as decimal(3,1)) distance,cast(left(duration,2) as int)*60 duration_secs from cte
),speed_cte as (
  select runner_id,order_id,distance*1000/duration_secs speed_ms from cte2
),request_cte as (
  select c.customer_id,c.order_id,ro.runner_id,rr.rating,c.order_time,ro.pickup_time,
  datediff(minute,c.order_time,ro.pickup_time) time_btw_order_pickup,cast(left(ro.duration,2) as int) duration_mins,pizza_id
  from customer_orders c
  inner join runner_orders ro on c.order_id = ro.order_id
  inner join runner_ratings rr on ro.order_id=rr.order_id and ro.runner_id=rr.runner_id
  where ro.cancellation is null
)
select cte1.customer_id,cte1.order_id,cte1.runner_id,cte1.rating,cte1.order_time,cte1.pickup_time,cte1.duration_mins,
  avg(s.speed_ms) avg_speed,count(cte1.pizza_id) nro_pizzas from request_cte cte1
inner join speed_cte s on cte1.order_id=s.order_id and cte1.runner_id=s.runner_id
group by cte1.customer_id,cte1.order_id,cte1.runner_id,cte1.rating,cte1.order_time,cte1.pickup_time,cte1.duration_mins
order by 2
```
#### 5.If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?
```sql
with cte as (
  select runner_id,order_id,
    distance,case when charindex('k',distance)!=0 then charindex('k',distance) else len(distance)+1 end flag_index
  from runner_orders 
  where cancellation is null
),runner_delivery_cost_cte as (
  select order_id,runner_id,cast(left(distance,flag_index-1) as decimal(3,1))*0.3 as runner_delivery_cost from cte
),request_cte as (
  select c.order_id,sum(case when c.pizza_id=1 then 12 else 10 end) price,rc.runner_delivery_cost from customer_orders c
  inner join runner_orders r on c.order_id = r.order_id
  inner join runner_delivery_cost_cte rc on r.order_id=rc.order_id and r.runner_id=rc.runner_id
  where r.cancellation is null
  group by c.order_id,rc.runner_delivery_cost
)
select sum(price-runner_delivery_cost) revenue from request_cte
```
### <p align="center">E. Bonus Questions
#### 1.If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an INSERT statement to demonstrate what would happen if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?
```sql
alter table pizza_recipes alter column toppings varchar(50);
insert into pizza_recipes(pizza_id,toppings) select 3 pizza_id,string_agg(topping_id,', ') from pizza_toppings
insert into pizza_names values(3,'Supreme Pizza');
```

## POSTGRESQL

### <p align="center">A. Pizza Metrics
#### 1.How many pizzas were ordered?
```sql
select count(pizza_id) total_pizzas from pizza_runner.customer_orders;
```
| total_pizzas |
|---------------|
| 14            |

#### 2.How many unique customer orders were made?
```sql
select count(distinct customer_id) total_customers_order from pizza_runner.customer_orders;
```
| total_customers_order |
|------------------------|
| 5                      |

#### 3.How many successful orders were delivered by each runner?
```sql
select distinct cancellation from pizza_runner.runner_orders;
update pizza_runner.runner_orders set cancellation = case when cancellation = '' then null
															when lower(cancellation) = 'null' then null
															else cancellation end;
select runner_id,count(order_id) total_success_order from pizza_runner.runner_orders
where cancellation is null
group by runner_id;
```
| runner_id | total_success_order |
|------------|---------------------|
| 2          | 3                   |
| 1          | 4                   |
| 3          | 1                   |

#### 4.How many of each type of pizza was delivered?
```sql
select n.pizza_id,n.pizza_name,count(n.pizza_id) total_pizza_delivered from pizza_runner.customer_orders co
inner join pizza_runner.runner_orders ro on co.order_id=ro.order_id
inner join pizza_runner.pizza_names n on co.pizza_id=n.pizza_id
where ro.cancellation is null
group by n.pizza_id,n.pizza_name
```
| pizza_id | pizza_name  | total_pizza_delivered |
|-----------|--------------|----------------------|
| 1         | Meatlovers   | 9                    |
| 2         | Vegetarian   | 3                    |

#### 5.How many Vegetarian and Meatlovers were ordered by each customer?
```sql
select c.customer_id,n.pizza_name,count(c2.order_id) total_pizza_ordered from (
	select distinct customer_id from pizza_runner.customer_orders) c
cross join pizza_runner.pizza_names n
left join pizza_runner.customer_orders c2 on c.customer_id=c2.customer_id and c2.pizza_id=n.pizza_id
group by c.customer_id,n.pizza_name
order by c.customer_id,n.pizza_name;

select co.customer_id,
	sum(case when n.pizza_name='Meatlovers' then 1 else 0 end) Meatlovers,
	sum(case when n.pizza_name='Vegetarian' then 1 else 0 end) Vegetarian
from pizza_runner.customer_orders co
inner join pizza_runner.pizza_names n on co.pizza_id=n.pizza_id
group by co.customer_id
order by 1;
```
| customer_id | pizza_name     | total_pizza_ordered |
|--------------|----------------|---------------------|
| 101          | Meatlovers     | 2                   |
| 101          | Supreme pizza  | 0                   |
| 101          | Vegetarian     | 1                   |
| 102          | Meatlovers     | 2                   |
| 102          | Supreme pizza  | 0                   |
| 102          | Vegetarian     | 1                   |
| 103          | Meatlovers     | 3                   |
| 103          | Supreme pizza  | 0                   |
| 103          | Vegetarian     | 1                   |
| 104          | Meatlovers     | 3                   |
| 104          | Supreme pizza  | 0                   |
| 104          | Vegetarian     | 0                   |
| 105          | Meatlovers     | 0                   |
| 105          | Supreme pizza  | 0                   |
| 105          | Vegetarian     | 1                   |

#### 6.What was the maximum number of pizzas delivered in a single order?
```sql
select c.order_id,count(c.pizza_id) total_pizzas_delivered from pizza_runner.customer_orders c
inner join pizza_runner.runner_orders r on c.order_id=r.order_id
where r.cancellation is null
group by c.order_id
order by 2 desc limit 1;
```
| order_id | total_pizzas_delivered |
|-----------|------------------------|
| 4         | 3                      |

#### 7.For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
```sql
update pizza_runner.customer_orders set exclusions = case when exclusions ~* '^(|null)$' then null else exclusions end;
update pizza_runner.customer_orders set extras = case when extras ~* '^(|null)$' then null else extras end;
with changes_cte as (
	select c.customer_id,c.pizza_id,c.exclusions,extras,
		case when c.exclusions is null and c.extras is null then 'False' else 'True' end changes 
	from pizza_runner.customer_orders c
	inner join pizza_runner.runner_orders r on c.order_id=r.order_id
	where r.cancellation is null
)
select customer_id,changes,count(pizza_id) total_pizzas_changes from changes_cte
group by customer_id,changes
order by customer_id;

select customer_id,
	sum(case when exclusions is not null or extras is not null then 1 else 0 end) with_changes,
    sum(case when exclusions is null and extras is null then 1 else 0 end) no_changes
from pizza_runner.customer_orders c
inner join pizza_runner.runner_orders r on c.order_id = r.order_id
where r.cancellation is null
group by customer_id;
```
| customer_id | changes | total_pizzas_changes |
|--------------|----------|----------------------|
| 101          | False    | 2                    |
| 102          | False    | 3                    |
| 103          | True     | 3                    |
| 104          | False    | 1                    |
| 104          | True     | 2                    |
| 105          | True     | 1                    |

or 

| customer_id | with_changes | no_changes |
|--------------|---------------|-------------|
| 101          | 0             | 2           |
| 102          | 0             | 3           |
| 103          | 3             | 0           |
| 104          | 2             | 1           |
| 105          | 1             | 0           |


#### 8.How many pizzas were delivered that had both exclusions and extras?
```sql
select count(c.pizza_id) from pizza_runner.customer_orders c
inner join pizza_runner.runner_orders r on c.order_id=r.order_id
where r.cancellation is null and (exclusions is not null and extras is not null);
```
| count |
|--------|
| 1      |

#### 9.What was the total volume of pizzas ordered for each hour of the day?
```sql
with recursive all_hours as (
	select 0 as hour_order
	union
	select hour_order+1 as hour_order from all_hours
	where hour_order<24
),pizza_hour_orders as (
	select pizza_id,extract(hour from order_time) hour_order 
	from pizza_runner.customer_orders
)
select h.hour_order,count(p.pizza_id) total_pizzas_ordered from all_hours h
left join pizza_hour_orders p on h.hour_order=p.hour_order
group by h.hour_order
order by 1;
```
| hour_order | total_pizzas_ordered |
|-------------|----------------------|
| 0           | 0                    |
| 1           | 0                    |
| 2           | 0                    |
| 3           | 0                    |
| 4           | 0                    |
| 5           | 0                    |
| 6           | 0                    |
| 7           | 0                    |
| 8           | 0                    |
| 9           | 0                    |
| 10          | 0                    |
| 11          | 1                    |
| 12          | 0                    |
| 13          | 3                    |
| 14          | 0                    |
| 15          | 0                    |
| 16          | 0                    |
| 17          | 0                    |
| 18          | 3                    |
| 19          | 1                    |
| 20          | 0                    |
| 21          | 3                    |
| 22          | 0                    |
| 23          | 3                    |
| 24          | 0                    |

#### 10.What was the volume of orders for each day of the week?
```sql
with recursive day_of_week as (
	select 0 as day_ow
	union
	select day_ow+1 as day_ow from day_of_week
	where day_ow<6
)
select cte.day_ow,count(distinct order_id) total_orders from day_of_week cte
left join pizza_runner.customer_orders c on cte.day_ow=extract(dow from c.order_time)
group by cte.day_ow
order by 1
```
| day_ow | total_orders |
|---------|---------------|
| 0       | 0             |
| 1       | 0             |
| 2       | 0             |
| 3       | 5             |
| 4       | 2             |
| 5       | 1             |
| 6       | 2             |

### <p align="center">B. Runner and Customer Experience

#### 1.How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
```sql
with recursive weeks_cte as (
	select '2021-01-01'::date as start_week
	union
	select start_week+7 as start_week from weeks_cte
	where start_week<'2021-12-31'
),signed_cte as (
	select r.runner_id,r.registration_date,cte.start_week,cte.start_week+6 end_week from weeks_cte cte
	left join pizza_runner.runners r on r.registration_date between cte.start_week and cte.start_week+6
)
select start_week,count(runner_id) total_runner_signedup from signed_cte
group by start_week
order by start_week;
```
| start_week  | total_runner_signedup |
|--------------|------------------------|
| 2021-01-01   | 2                      |
| 2021-01-08   | 1                      |
| 2021-01-15   | 1                      |
| 2021-01-22   | 0                      |
| 2021-01-29   | 0                      |
| 2021-02-05   | 0                      |
| 2021-02-12   | 0                      |
| 2021-02-19   | 0                      |
| 2021-02-26   | 0                      |
| 2021-03-05   | 0                      |
| 2021-03-12   | 0                      |
| 2021-03-19   | 0                      |
| 2021-03-26   | 0                      |
| 2021-04-02   | 0                      |
| 2021-04-09   | 0                      |
| 2021-04-16   | 0                      |
| 2021-04-23   | 0                      |
| 2021-04-30   | 0                      |
| 2021-05-07   | 0                      |
| 2021-05-14   | 0                      |
| 2021-05-21   | 0                      |
| 2021-05-28   | 0                      |
| 2021-06-04   | 0                      |
| 2021-06-11   | 0                      |
| 2021-06-18   | 0                      |
| 2021-06-25   | 0                      |
| 2021-07-02   | 0                      |
| 2021-07-09   | 0                      |
| 2021-07-16   | 0                      |
| 2021-07-23   | 0                      |
| 2021-07-30   | 0                      |
| 2021-08-06   | 0                      |
| 2021-08-13   | 0                      |
| 2021-08-20   | 0                      |
| 2021-08-27   | 0                      |
| 2021-09-03   | 0                      |
| 2021-09-10   | 0                      |
| 2021-09-17   | 0                      |
| 2021-09-24   | 0                      |
| 2021-10-01   | 0                      |
| 2021-10-08   | 0                      |
| 2021-10-15   | 0                      |
| 2021-10-22   | 0                      |
| 2021-10-29   | 0                      |
| 2021-11-05   | 0                      |
| 2021-11-12   | 0                      |
| 2021-11-19   | 0                      |
| 2021-11-26   | 0                      |
| 2021-12-03   | 0                      |
| 2021-12-10   | 0                      |
| 2021-12-17   | 0                      |
| 2021-12-24   | 0                      |
| 2021-12-31   | 0                      |

#### 2.What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
```sql
update pizza_runner.runner_orders set pickup_time = case when pickup_time='null' then null else pickup_time end;
alter table pizza_runner.runner_orders alter column pickup_time type timestamp
using pickup_time::timestamp;

with order_pickup_time as (
	select distinct r.order_id,r.runner_id,c.order_time,r.pickup_time from pizza_runner.runner_orders r
	inner join pizza_runner.customer_orders c on r.order_id=c.order_id
	where r.cancellation is null
)
select runner_id,round(avg(extract(minutes from pickup_time-order_time)),2) avg_pickup from order_pickup_time
group by runner_id
order by 1;
```
| runner_id | avg_pickup |
|------------|------------|
| 1          | 14.00      |
| 2          | 19.67      |
| 3          | 10.00      |

#### 3.Is there any relationship between the number of pizzas and how long the order takes to prepare?
```sql
with order_total_pizzas as (
	select c.order_id,c.pizza_id,c.order_time,r.pickup_time from pizza_runner.customer_orders c
	inner join pizza_runner.runner_orders r on c.order_id=r.order_id
	where r.cancellation is null
)
select order_id,count(pizza_id) total_pizzas_ordered,round(avg(extract(minutes from pickup_time-order_time)),2) avg_prepare_time 
from order_total_pizzas
group by order_id
order by 1
```
| order_id | total_pizzas_ordered | avg_prepare_time |
|-----------|----------------------|------------------|
| 1         | 1                    | 10.00            |
| 2         | 1                    | 10.00            |
| 3         | 2                    | 21.00            |
| 4         | 3                    | 29.00            |
| 5         | 1                    | 10.00            |
| 7         | 1                    | 10.00            |
| 8         | 1                    | 20.00            |
| 10        | 2                    | 15.00            |

##### A: Asumiendo que el tiempo que toma preparar cada pizza es la diferencia entre pickup_time(tiempo que el repartidor toma el pedido) y order_time(tiempo que se ordenó la pizza), hay una relacion directamente proporcional aproximada que se incrementa de acuerdo al numero de pizzas por orden, excepto en las ordenes 8 y 10.
#### 4.What was the average distance travelled for each customer?
```sql
update pizza_runner.runner_orders set distance = case when distance like '%km' then left(distance,length(distance)-2)
												when distance='null' then null else distance end;
alter table pizza_runner.runner_orders alter column distance type numeric using distance::numeric;

with distance_customer as (
	select distinct c.customer_id,c.order_id,c.pizza_id,r.distance from pizza_runner.runner_orders r
	inner join pizza_runner.customer_orders c on r.order_id=c.order_id
	where r.cancellation is null
)
select customer_id,round(avg(distance),2) avg_distance from distance_customer
group by customer_id
order by 1;
```
| customer_id | avg_distance |
|--------------|--------------|
| 101          | 20.00        |
| 102          | 16.73        |
| 103          | 23.40        |
| 104          | 10.00        |
| 105          | 25.00        |

#### 5.What was the difference between the longest and shortest delivery times for all orders?
```sql
update pizza_runner.runner_orders set duration = case when duration ~ '(minutes|minute|mins)$' then left(duration,2)
														when duration = 'null' then null else duration end;
alter table pizza_runner.runner_orders alter column duration type numeric using duration::numeric;

select max(duration)-min(duration) from pizza_runner.runner_orders
where cancellation is null;
```
| diff |
|-------|
| 30    |

#### 6.What was the average speed for each runner for each delivery and do you notice any trend for these values?
```sql
select runner_id,round(avg(distance*1000/(duration*60)),2) speed_ms from pizza_runner.runner_orders
where cancellation is null
group by runner_id
order by runner_id;

select runner_id,order_id,round(distance*1000/(duration*60),2) speed_ms from pizza_runner.runner_orders
where cancellation is null
order by runner_id,pickup_time;
```
| runner_id | order_id | speed_ms |
|------------|-----------|----------|
| 1          | 1         | 10.42    |
| 1          | 2         | 12.35    |
| 1          | 3         | 11.17    |
| 1          | 10        | 16.67    |
| 2          | 4         | 9.75     |
| 2          | 7         | 16.67    |
| 2          | 8         | 26.00    |
| 3          | 5         | 11.11    |

##### A: No se puede analizar una tendencia entre los valores pedidos debido a que el promedio abarca todos los registros de delivery de cada repartidor; sin embargo se puede observar que los repartidores 1 y 2 van aumentando la velocidad cuando más tiempo lleva de repartidor
#### 7.What is the successful delivery percentage for each runner?
```sql
select runner_id,round(count(pickup_time)*100.0/count(*),2) success_prctg from pizza_runner.runner_orders
group by runner_id
order by 2 desc
```
| runner_id | success_prctg |
|------------|----------------|
| 1          | 100.00          |
| 2          | 75.00           |
| 3          | 50.00           |

### <p align="center">C. Ingredient Optimisation

#### 1.What are the standard ingredients for each pizza?
```sql
with pizza_toppings_Cte as (
	select pizza_id,unnest(string_to_Array(toppings,','))::int topping from pizza_runner.pizza_recipes
)
select cte.pizza_id,t.topping_name from pizza_toppings_cte cte
inner join pizza_runner.pizza_toppings t on cte.topping=t.topping_id
order by cte.pizza_id;

with recursive pizza_toppings_cte as (
	select pizza_id,ltrim(substring(toppings,1,position(',' in toppings)-1))::int topping,
		right(toppings,length(toppings)-position(',' in toppings)) miss_toppings,toppings
	from pizza_runner.pizza_recipes
	union
	select pizza_id,ltrim(substring(miss_toppings,1,position(',' in miss_toppings)-1))::int topping,
		right(miss_toppings,length(miss_toppings)-position(',' in miss_toppings)) miss_toppings,toppings
	from pizza_toppings_cte
	where position(',' in miss_toppings)!=0
),all_toppings_pizza as (
	select * from pizza_toppings_cte
	union
	select pizza_id,miss_toppings::int topping,null miss_toppings,toppings from pizza_toppings_cte
	where position(',' in miss_toppings)=0
)
select cte.pizza_id,cte.topping,t.topping_name from all_toppings_pizza cte
inner join pizza_runner.pizza_toppings t on cte.topping=t.topping_id
order by 1;
```
| pizza_id | topping | topping_name  |
|-----------|----------|---------------|
| 1         | 6        | Mushrooms     |
| 1         | 5        | Chicken       |
| 1         | 10       | Salami        |
| 1         | 2        | BBQ Sauce     |
| 1         | 3        | Beef          |
| 1         | 8        | Pepperoni     |
| 1         | 1        | Bacon         |
| 1         | 4        | Cheese        |
| 2         | 12       | Tomato Sauce  |
| 2         | 4        | Cheese        |
| 2         | 6        | Mushrooms     |
| 2         | 7        | Onions        |
| 2         | 9        | Peppers       |
| 2         | 11       | Tomatoes      |
| 3         | 7        | Onions        |
| 3         | 11       | Tomatoes      |
| 3         | 3        | Beef          |
| 3         | 8        | Pepperoni     |
| 3         | 12       | Tomato Sauce  |
| 3         | 9        | Peppers       |
| 3         | 2        | BBQ Sauce     |
| 3         | 10       | Salami        |
| 3         | 5        | Chicken       |
| 3         | 6        | Mushrooms     |
| 3         | 4        | Cheese        |
| 3         | 1        | Bacon         |

#### 2.What was the most commonly added extra?
```sql
with recursive extras_toppings_cte as (
	select order_id,pizza_id,ltrim(substring(extras,1,position(',' in extras)-1))::int extra,
		right(extras,length(extras)-position(',' in extras)) miss_extras,extras
	from pizza_runner.customer_orders
	where position(',' in extras)!=0
	union
	select order_id,pizza_id,ltrim(substring(miss_extras,1,position(',' in miss_extras)-1))::int extra,
		right(miss_extras,length(miss_extras)-position(',' in miss_extras)) miss_extras,extras
	from extras_toppings_cte
	where position(',' in miss_extras)!=0
),all_extras_pizza as (
	select * from extras_toppings_cte
	union
	select order_id,pizza_id,miss_extras::int extra,null miss_extras,extras from extras_toppings_cte
	where position(',' in miss_extras)=0
	union 
	select order_id,pizza_id,extras::int extra, null extras,extras from pizza_runner.customer_orders
	where position(',' in extras)=0
)
select extra,count(extra) cnt_extra from all_extras_pizza
group by extra
order by 2 desc;
```
| extra | cnt_extra |
|--------|------------|
| 1      | 4          |
| 5      | 1          |
| 4      | 1          |

#### 3.What was the most commonly exclusions?
```sql
with recursive exclusions_toppings_cte as (
	select order_id,pizza_id,ltrim(substring(exclusions,1,position(',' in exclusions)-1))::int exclusion,
		right(exclusions,length(exclusions)-position(',' in exclusions)) miss_exclusions,exclusions
	from pizza_runner.customer_orders
	where position(',' in exclusions)!=0
	union
	select order_id,pizza_id,ltrim(substring(miss_exclusions,1,position(',' in miss_exclusions)-1))::int exclusion,
		right(miss_exclusions,length(miss_exclusions)-position(',' in miss_exclusions)) miss_exclusions,exclusions
	from exclusions_toppings_cte
	where position(',' in miss_exclusions)!=0
),all_exclusions_pizza as (
	select * from exclusions_toppings_cte
	union
	select order_id,pizza_id,miss_exclusions::int exclusions,null miss_exclusions,exclusions from exclusions_toppings_cte
	where position(',' in miss_exclusions)=0
	union 
	select order_id,pizza_id,exclusions::int extra, null exclusions,exclusions from pizza_runner.customer_orders
	where position(',' in exclusions)=0
)
select exclusion,count(exclusions) cnt_exclusions from all_exclusions_pizza
group by exclusion
order by 2 desc
```
| exclusion | cnt_exclusions |
|------------|----------------|
| 4          | 3              |
| 6          | 1              |
| 2          | 1              |

#### 4.Generate an order item for each record in the customers_orders table in the format of one of the following:
	Meat Lovers
	Meat Lovers - Exclude Beef
	Meat Lovers - Extra Bacon
	Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers
```sql
with recursive order_pizza_row_number as (
	select row_number() over(order by order_time) rn,* from pizza_runner.customer_orders
),extras_toppings_cte as (
	select rn,order_id,pizza_id,ltrim(substring(extras,1,position(',' in extras)-1))::int extra,
		right(extras,length(extras)-position(',' in extras)) miss_extras,extras
	from order_pizza_row_number
	where position(',' in extras)!=0
	union
	select rn,order_id,pizza_id,ltrim(substring(miss_extras,1,position(',' in miss_extras)-1))::int extra,
		right(miss_extras,length(miss_extras)-position(',' in miss_extras)) miss_extras,extras
	from extras_toppings_cte
	where position(',' in miss_extras)!=0
),all_extras_pizza as (
	select * from extras_toppings_cte
	union
	select rn,order_id,pizza_id,miss_extras::int extra,null miss_extras,extras from extras_toppings_cte
	where position(',' in miss_extras)=0
	union 
	select rn,order_id,pizza_id,extras::int extra, null extras,extras from order_pizza_row_number
	where position(',' in extras)=0
),exclusions_toppings_cte as (
	select rn,order_id,pizza_id,ltrim(substring(exclusions,1,position(',' in exclusions)-1))::int exclusion,
		right(exclusions,length(exclusions)-position(',' in exclusions)) miss_exclusions,exclusions
	from order_pizza_row_number
	where position(',' in exclusions)!=0
	union
	select rn,order_id,pizza_id,ltrim(substring(miss_exclusions,1,position(',' in miss_exclusions)-1))::int exclusion,
		right(miss_exclusions,length(miss_exclusions)-position(',' in miss_exclusions)) miss_exclusions,exclusions
	from exclusions_toppings_cte
	where position(',' in miss_exclusions)!=0
),all_exclusions_pizza as (
	select * from exclusions_toppings_cte
	union
	select rn,order_id,pizza_id,miss_exclusions::int exclusions,null miss_exclusions,exclusions from exclusions_toppings_cte
	where position(',' in miss_exclusions)=0
	union 
	select rn,order_id,pizza_id,exclusions::int extra, null exclusions,exclusions from order_pizza_row_number
	where position(',' in exclusions)=0
),cte as (
select c.*,extras.extra,t1.topping_name extra_name,exclusions.exclusion,t2.topping_name exclusion_name,n.pizza_name 
from order_pizza_row_number c
left join all_extras_pizza extras on c.order_id=extras.order_id and c.pizza_id=extras.pizza_id
left join all_exclusions_pizza exclusions on c.order_id=exclusions.order_id and c.pizza_id=exclusions.pizza_id
left join pizza_runner.pizza_toppings t1 on extras.extra=t1.topping_id
left join pizza_runner.pizza_toppings t2 on exclusions.exclusion=t2.topping_id
left join pizza_runner.pizza_names n on c.pizza_id=n.pizza_id
),generated_cte as (
	select rn,order_id,customer_id,pizza_id,exclusions,extras,order_time,
		case when extras is null and exclusions is null then pizza_name
			when extras is not null and exclusions is null then pizza_name||concat(' - Extra ',string_agg(distinct extra_name,', '))
			when extras is null and exclusions is not null then pizza_name||concat(' - Exclusion ',string_agg(distinct exclusion_name,', '))
			else pizza_name||concat(' - Exclusion ',string_agg(distinct exclusion_name,', '))||concat(' - Extra ',string_agg(distinct extra_name,', ')) end generated
	from cte
	group by rn,order_id,customer_id,pizza_id,extras,exclusions,order_time,pizza_name
)
select order_id,customer_id,pizza_id,exclusions,extras,order_time,generated from generated_cte cte;
```
| order_id | customer_id | pizza_id | exclusions  | extras   | order_time           | generated                                                             |
|-----------|--------------|-----------|-------------|----------|----------------------|------------------------------------------------------------------------|
| 1         | 101          | 1         |             |          | 2020-01-01 18:05:02  | Meatlovers                                                            |
| 2         | 101          | 1         |             |          | 2020-01-01 19:00:52  | Meatlovers                                                            |
| 3         | 102          | 1         |             |          | 2020-01-02 23:51:23  | Meatlovers                                                            |
| 3         | 102          | 2         |             |          | 2020-01-02 23:51:23  | Vegetarian                                                            |
| 4         | 103          | 1         | 4           |          | 2020-01-04 13:23:46  | Meatlovers - Exclusion Cheese                                         |
| 4         | 103          | 1         | 4           |          | 2020-01-04 13:23:46  | Meatlovers - Exclusion Cheese                                         |
| 4         | 103          | 2         | 4           |          | 2020-01-04 13:23:46  | Vegetarian - Exclusion Cheese                                         |
| 5         | 104          | 1         |             | 1        | 2020-01-08 21:00:29  | Meatlovers - Extra Bacon                                              |
| 6         | 101          | 2         |             |          | 2020-01-08 21:03:13  | Vegetarian                                                            |
| 7         | 105          | 2         |             | 1        | 2020-01-08 21:20:29  | Vegetarian - Extra Bacon                                              |
| 8         | 102          | 1         |             |          | 2020-01-09 23:54:33  | Meatlovers                                                            |
| 9         | 103          | 1         | 4           | 1, 5     | 2020-01-10 11:22:59  | Meatlovers - Exclusion Cheese - Extra Bacon, Chicken                  |
| 10        | 104          | 1         |             |          | 2020-01-11 18:34:49  | Meatlovers                                                            |
| 10        | 104          | 1         | 2, 6        | 1, 4     | 2020-01-11 18:34:49  | Meatlovers - Exclusion BBQ Sauce, Mushrooms - Extra Bacon, Cheese     |

#### 5.Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"
```sql
with pizza_toppings_Cte as (
	select pizza_id,unnest(string_to_Array(toppings,','))::int topping from pizza_runner.pizza_recipes
),all_pizza_toppings as (
	select cte.pizza_id,t.topping_id,t.topping_name from pizza_toppings_cte cte
	inner join pizza_runner.pizza_toppings t on cte.topping=t.topping_id
	order by cte.pizza_id
),customer_order_rn as (
	select row_number() over(order by order_time) rn,c.*,n.pizza_name
	from pizza_runner.customer_orders c
	inner join pizza_runner.pizza_names n on c.pizza_id=n.pizza_id
),extras_exclusions_pizzas as (
	select rn,order_id,customer_id,pizza_id,
		unnest(string_to_Array(exclusions,','))::int exclusion,
		unnest(string_to_Array(extras,','))::int extra
	from customer_order_rn
),grouping_all_ingredients as (
	(
	select c.rn,c.order_id,c.customer_id,c.pizza_id,a.topping_id,a.topping_name topping from all_pizza_toppings a
	inner join customer_order_rn c on a.pizza_id=c.pizza_id
	union all
	select rn,order_id,customer_id,pizza_id,extras.extra topping_id,t.topping_name topping from extras_exclusions_pizzas extras
	inner join pizza_runner.pizza_toppings t on extras.extra=t.topping_id
	)
	except all
	select rn,order_id,customer_id,pizza_id,exclusions.exclusion topping_id,t.topping_name topping from extras_exclusions_pizzas exclusions
	inner join pizza_runner.pizza_toppings t on exclusions.exclusion=t.topping_id
),topping_orders as (
	select rn,order_id,customer_id,pizza_id,topping_id,topping,count(topping) total_toppings
	from grouping_all_ingredients
	group by rn,order_id,customer_id,pizza_id,topping_id,topping
),concat_topping_orders as (
	select rn,order_id,customer_id,pizza_id,topping_id,topping,total_toppings,
		case when total_toppings>1 then total_toppings||'x'||topping else topping end topping_times
	from topping_orders
)
select final_cte.order_id,final_cte.customer_id,final_cte.pizza_id,
	n.pizza_name||': '||string_agg(topping_times,',' order by topping) 
from concat_topping_orders final_cte
inner join pizza_runner.pizza_names n on final_cte.pizza_id=n.pizza_id
group by final_cte.rn,final_cte.order_id,final_cte.customer_id,final_cte.pizza_id,n.pizza_name
order by final_cte.rn;
```
| order_id | customer_id | pizza_id | ?column?                                                                                   |
|-----------|--------------|-----------|---------------------------------------------------------------------------------------------|
| 1         | 101          | 1         | Meatlovers: Bacon,BBQ Sauce,Beef,Cheese,Chicken,Mushrooms,Pepperoni,Salami                 |
| 2         | 101          | 1         | Meatlovers: Bacon,BBQ Sauce,Beef,Cheese,Chicken,Mushrooms,Pepperoni,Salami                 |
| 3         | 102          | 1         | Meatlovers: Bacon,BBQ Sauce,Beef,Cheese,Chicken,Mushrooms,Pepperoni,Salami                 |
| 3         | 102          | 2         | Vegetarian: Cheese,Mushrooms,Onions,Peppers,Tomato Sauce,Tomatoes                          |
| 4         | 103          | 1         | Meatlovers: Bacon,BBQ Sauce,Beef,Chicken,Mushrooms,Pepperoni,Salami                        |
| 4         | 103          | 1         | Meatlovers: Bacon,BBQ Sauce,Beef,Chicken,Mushrooms,Pepperoni,Salami                        |
| 4         | 103          | 2         | Vegetarian: Mushrooms,Onions,Peppers,Tomato Sauce,Tomatoes                                 |
| 5         | 104          | 1         | Meatlovers: 2xBacon,BBQ Sauce,Beef,Cheese,Chicken,Mushrooms,Pepperoni,Salami               |
| 6         | 101          | 2         | Vegetarian: Cheese,Mushrooms,Onions,Peppers,Tomato Sauce,Tomatoes                          |
| 7         | 105          | 2         | Vegetarian: Bacon,Cheese,Mushrooms,Onions,Peppers,Tomato Sauce,Tomatoes                    |
| 8         | 102          | 1         | Meatlovers: Bacon,BBQ Sauce,Beef,Cheese,Chicken,Mushrooms,Pepperoni,Salami                 |
| 9         | 103          | 1         | Meatlovers: 2xBacon,BBQ Sauce,Beef,2xChicken,Mushrooms,Pepperoni,Salami                    |
| 10        | 104          | 1         | Meatlovers: Bacon,BBQ Sauce,Beef,Cheese,Chicken,Mushrooms,Pepperoni,Salami                 |
| 10        | 104          | 1         | Meatlovers: 2xBacon,Beef,2xCheese,Chicken,Pepperoni,Salami                                 |

#### 6.What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?
```sql
with pizza_toppings_Cte as (
	select pizza_id,unnest(string_to_Array(toppings,','))::int topping from pizza_runner.pizza_recipes
),all_pizza_toppings as (
	select cte.pizza_id,t.topping_id,t.topping_name from pizza_toppings_cte cte
	inner join pizza_runner.pizza_toppings t on cte.topping=t.topping_id
	order by cte.pizza_id
),customer_order_rn as (
	select row_number() over(order by order_time) rn,c.*
	from pizza_runner.customer_orders c
),extras_exclusions_pizzas as (
	select rn,order_id,customer_id,pizza_id,
		unnest(string_to_Array(exclusions,','))::int exclusion,
		unnest(string_to_Array(extras,','))::int extra
	from customer_order_rn
),grouping_all_ingredients as (
	(
	select c.rn,c.order_id,c.customer_id,c.pizza_id,a.topping_id,a.topping_name topping from all_pizza_toppings a
	inner join customer_order_rn c on a.pizza_id=c.pizza_id
	union all
	select rn,order_id,customer_id,pizza_id,extras.extra topping_id,t.topping_name topping from extras_exclusions_pizzas extras
	inner join pizza_runner.pizza_toppings t on extras.extra=t.topping_id
	)
	except all
	select rn,order_id,customer_id,pizza_id,exclusions.exclusion topping_id,t.topping_name topping from extras_exclusions_pizzas exclusions
	inner join pizza_runner.pizza_toppings t on exclusions.exclusion=t.topping_id
),topping_orders as (
	select rn,order_id,customer_id,pizza_id,topping_id,topping,count(topping) total_toppings
	from grouping_all_ingredients
	group by rn,order_id,customer_id,pizza_id,topping_id,topping
)
select cte.topping_id,t.topping_name,count(cte.topping_id) total_toppings from topping_orders cte
inner join pizza_runner.runner_orders r on cte.order_id=r.order_id
inner join pizza_runner.pizza_toppings t on cte.topping_id=t.topping_id
where r.cancellation is null
group by cte.topping_id,t.topping_name
order by total_toppings desc
```
| topping_id | topping_name | total_toppings |
|-------------|---------------|----------------|
| 6           | Mushrooms     | 11             |
| 1           | Bacon         | 10             |
| 4           | Cheese        | 9              |
| 5           | Chicken       | 9              |
| 8           | Pepperoni     | 9              |
| 10          | Salami        | 9              |
| 3           | Beef          | 9              |
| 2           | BBQ Sauce     | 8              |
| 12          | Tomato Sauce  | 3              |
| 7           | Onions        | 3              |
| 9           | Peppers       | 3              |
| 11          | Tomatoes      | 3              |

### <p align="center">D. Pricing and Ratings
#### 1.If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
```sql
select coalesce(n.pizza_name,'Total revenue'),
	sum(case when pizza_name='Meatlovers' then 12 else 10 end) revenue from pizza_runner.customer_orders c
inner join pizza_runner.pizza_names n on c.pizza_id=n.pizza_id
group by rollup (n.pizza_name)
order by case when n.pizza_name in (select pizza_name from pizza_runner.pizza_names) then n.pizza_name else 'z' end;
```
| coalesce       | revenue |
|----------------|----------|
| Meatlovers     | 120      |
| Vegetarian     | 40       |
| Total revenue  | 160      |

#### 2.What if there was an additional $1 charge for any pizza extras?
```sql
with extra_count as (
	select *,length(extras)-length(replace(extras,',',''))+1 total_extras from pizza_runner.customer_orders
)
select coalesce(n.pizza_name,'Total revenue'),
	sum(case when pizza_name='Meatlovers' then 12+coalesce(total_extras,0) else 10+coalesce(total_extras,0) end) revenue from extra_count c
inner join pizza_runner.pizza_names n on c.pizza_id=n.pizza_id
group by rollup (n.pizza_name)
order by case when n.pizza_name in (select pizza_name from pizza_runner.pizza_names) then n.pizza_name else 'z' end;
```
| coalesce       | revenue |
|----------------|----------|
| Meatlovers     | 125      |
| Vegetarian     | 41       |
| Total revenue  | 166      |

#### 3.The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
```sql
drop table if exists pizza_runner.runner_ratings;
create table pizza_runner.runner_ratings (
	order_id int not null,
	runner_id int not null,
	rating decimal(2,1) not null check (rating between 1 and 5),
	rating_reason varchar(100),
	constraint order_runner_unique unique(order_id,runner_id)
);

insert into pizza_runner.runner_ratings (order_id,runner_id,rating)
with recursive runner_rating_generator as (
	select order_id,runner_id,round((random()*(5-1)+1)::decimal(2,1),1) rating from pizza_runner.runner_orders
	where order_id=1 and cancellation is null
	union
	select r.order_id,r.runner_id,round((random()*(5-1)+1)::decimal(2,1),1) rating from runner_rating_generator cte
	inner join pizza_runner.runner_orders r on cte.order_id+1=r.order_id
	where r.order_id<=10
)
select * from runner_rating_generator;

select * from pizza_runner.runner_ratings;
```
| order_id | runner_id | rating | rating_reason |
|----------|-----------|--------|---------------|
| 1        | 1         | 3.8    |               |
| 2        | 1         | 4.4    |               |
| 3        | 1         | 3.1    |               |
| 4        | 2         | 3.5    |               |
| 5        | 3         | 2.5    |               |
| 6        | 3         | 3.0    |               |
| 7        | 2         | 2.3    |               |
| 8        | 2         | 4.3    |               |
| 9        | 2         | 1.9    |               |
| 10       | 1         | 1.3    |               |

#### 4.Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
	customer_id
	order_id
	runner_id
	rating
	order_time
	pickup_time
	Time between order and pickup
	Delivery duration
	Average speed
	Total number of pizzas
```sql
select c.customer_id,c.order_id,r.runner_id,rtg.rating,c.order_time,r.pickup_time,
	extract(minutes from r.pickup_time-c.order_time) time_between_actions,r.duration,
	round(avg(r.distance*1000.0/(r.duration*60.0)),2) avg_ms_speed,
	count(c.pizza_id) total_pizzas_ordered
from pizza_runner.customer_orders c
inner join pizza_runner.runner_orders r on c.order_id=r.order_id
inner join pizza_runner.runner_ratings rtg on r.order_id=rtg.order_id
where r.cancellation is null
group by c.customer_id,c.order_id,r.runner_id,rtg.rating,c.order_time,r.pickup_time,r.duration
order by 1,2;
```
| customer_id | order_id | runner_id | rating | order_time           | pickup_time          | time_between_actions | duration | avg_ms_speed | total_pizzas_ordered |
|-------------|----------|-----------|--------|---------------------|---------------------|---------------------|----------|--------------|---------------------|
| 101         | 1        | 1         | 3.8    | 2020-01-01 18:05:02 | 2020-01-01 18:15:34 | 10                  | 32       | 10.42        | 1                   |
| 101         | 2        | 1         | 4.4    | 2020-01-01 19:00:52 | 2020-01-01 19:10:54 | 10                  | 27       | 12.35        | 1                   |
| 102         | 3        | 1         | 3.1    | 2020-01-02 23:51:23 | 2020-01-03 00:12:37 | 21                  | 20       | 11.17        | 2                   |
| 102         | 8        | 2         | 4.3    | 2020-01-09 23:54:33 | 2020-01-10 00:15:02 | 20                  | 15       | 26.00        | 1                   |
| 103         | 4        | 2         | 3.5    | 2020-01-04 13:23:46 | 2020-01-04 13:53:03 | 29                  | 40       | 9.75         | 3                   |
| 104         | 5        | 3         | 2.5    | 2020-01-08 21:00:29 | 2020-01-08 21:10:57 | 10                  | 15       | 11.11        | 1                   |
| 104         | 10       | 1         | 1.3    | 2020-01-11 18:34:49 | 2020-01-11 18:50:20 | 15                  | 10       | 16.67        | 2                   |
| 105         | 7        | 2         | 2.3    | 2020-01-08 21:20:29 | 2020-01-08 21:30:45 | 10                  | 25       | 16.67        | 1                   |

#### 5.If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled how much money does Pizza Runner have left over after these deliveries?
```sql
select c.order_id,sum(case when n.pizza_name='Meatlovers' then 12+distance*0.3 else 10+distance*0.3 end) order_amount
from pizza_runner.customer_orders c
inner join pizza_runner.pizza_names n on c.pizza_id=n.pizza_id
inner join pizza_runner.runner_orders r on c.order_id=r.order_id
where r.cancellation is null
group by rollup (c.order_id)
order by 1
```
| order_id | order_amount |
|----------|--------------|
| 1        | 18.0         |
| 2        | 18.0         |
| 3        | 30.04        |
| 4        | 55.06        |
| 5        | 15.0         |
| 7        | 17.5         |
| 8        | 19.02        |
| 10       | 30.0         |
|          | 202.62       |

### <p align="center">E. Bonus Questions

#### 1.If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an INSERT statement to demonstrate what would happen if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?
```sql
insert into pizza_runner.pizza_recipes
values (3,(select string_agg(topping_id::varchar(2),', ') from pizza_runner.pizza_toppings));
insert into pizza_runner.pizza_names values (3,'Supreme pizza');
select * from pizza_runner.pizza_names;
```
| pizza_id | pizza_name      |
|----------|----------------|
| 1        | Meatlovers      |
| 2        | Vegetarian     |
| 3        | Supreme pizza  |
