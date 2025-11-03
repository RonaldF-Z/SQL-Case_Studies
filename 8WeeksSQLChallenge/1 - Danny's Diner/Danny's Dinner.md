# <p align="center" style="margin-top: 0px;">🍜 Case Study #1: Danny's Dinner 
## <p align="center"><img src="https://github.com/RonaldF-Z/SQL-Case_Studies/blob/main/8WeeksSQLChallenge/1%20-%20Danny's%20Diner/resources/c1.png?raw=true" alt="image" width="400" heigth="420">
## <p align="center">Notes
- ##### Case study's data and questions were extracted from this link: [here](https://8weeksqlchallenge.com/case-study-1/). 
- ##### All questions were solved using MSSQL Server
- ##### Answers in PostgreSQL using schema named "danny_dinner" 

## <p align="center">Introduction
Danny seriously loves Japanese food so in the beginning of 2021, he decides to embark upon a risky venture and opens up a cute little restaurant that sells his 3 favourite foods: sushi, curry and ramen.

Danny’s Diner is in need of your assistance to help the restaurant stay afloat - the restaurant has captured some very basic data from their few months of operation but have no idea how to use their data to help them run the business.

## <p align="center">Questions

#### 1. What is the total amount each customer spent at the restaurant?
```sql
select s.customer_id,sum(m.price) total_amount from danny_dinner.sales s
inner join danny_dinner.menu m on s.product_id=m.product_id
group by s.customer_id
```
| customer_id | total_amount |
|--------------|--------------|
| B            | 74           |
| C            | 36           |
| A            | 76           |

#### 2. How many days has each customer visited the restaurant?
```sql
select customer_id,count(distinct order_date) days_visited from sales
group by customer_id
```
| customer_id | nro_days |
|--------------|-----------|
| A            | 4         |
| B            | 6         |
| C            | 2         |

#### 3. What was the first item from the menu purchased by each customer?
##### MSSQL
```sql
with cte as (
  select s.customer_id,s.order_date,m.product_name,row_number() over(partition by s.customer_id order by s.order_date) rn
  from sales s
  inner join menu m on s.product_id = m.product_id
)
select customer_id,product_name first_menu from cte
where rn=1
```
##### POSTGRESQL
```sql
select distinct s1.customer_id,m.product_name from danny_dinner.sales s1
inner join danny_dinner.menu m on s1.product_id=m.product_id
where s1.order_date = (select min(s2.order_date) from danny_dinner.sales s2 
						where s1.customer_id=s2.customer_id
						group by s2.customer_id);
```
| customer_id | product_name |
|--------------|--------------|
| A            | sushi        |
| A            | curry        |
| B            | curry        |
| C            | ramen        |


#### 4.What is the most purchased item on the menu and how many times was it purchased by all customers?
##### MSSQL
```sql
with cte as (
  select s.customer_id,m.product_name,count(m.product_name) times_purchased,
    rank() over(partition by s.customer_id order by count(m.product_name) desc) rnk
  from sales s
  inner join menu m on s.product_id = m.product_id
  group by s.customer_id,m.product_name
)
select customer_id,product_name,times_purchased from cte where rnk=1
```
##### POSTGRESQL
```sql
select s.product_id,m.product_name,count(s.product_id) cnt_product from danny_dinner.sales s
inner join danny_dinner.menu m on s.product_id=m.product_id
group by s.product_id,m.product_name
order by 3 desc limit 1;
```
| product_id | product_name | cnt_product |
|-------------|--------------|--------------|
| 3           | ramen        | 8            |

#### 5.Which item was the most popular for each customer?
```sql
with cte as (
  select s.customer_id,m.product_name,count(m.product_name) times_purchased,
    rank() over(partition by s.customer_id order by count(m.product_name) desc) rnk
  from sales s
  inner join menu m on s.product_id = m.product_id
  group by s.customer_id,m.product_name
)
select customer_id,product_name from cte where rnk=1
```
| customer_id | product_name | cnt_prod_per_cust |
|--------------|--------------|-------------------|
| A            | ramen        | 3                 |
| B            | sushi        | 2                 |
| B            | curry        | 2                 |
| B            | ramen        | 2                 |
| C            | ramen        | 3                 |

#### 6.Which item was purchased first by the customer after they became a member?
```sql
with cte as (
  select s.customer_id,menu.product_name,rank() over(partition by s.customer_id order by s.order_date) rnk from sales s
  inner join members m on s.customer_id=m.customer_id and s.order_date>=m.join_date
  inner join menu on s.product_id=menu.product_id
)
select customer_id,product_name from cte where rnk=1
```
| customer_id | product_name |
|--------------|--------------|
| A            | curry        |
| B            | sushi        |

#### 7.Which item was purchased just before the customer became a member?
##### MSSQL
```sql
with cte as (
  select s.customer_id,max(s.order_date) max_date from sales s
  inner join members m on s.customer_id=m.customer_id and s.order_date<=m.join_date
  group by s.customer_id
)
select s.customer_id,m.product_name item_bef_memb from sales s
inner join cte c on s.customer_id=c.customer_id and s.order_date=c.max_date
inner join menu m on s.product_id=m.product_id
```
##### POSTGRESQL
```sql
with rnk_after_join as (
	select s.customer_id,mn.product_name,s.order_date,mm.join_date,
		rank() over(partition by s.customer_id order by s.order_date desc) rnk_date
	from danny_dinner.sales s
	inner join danny_dinner.members mm on s.customer_id=mm.customer_id and s.order_date<mm.join_date
	inner join danny_dinner.menu mn on s.product_id=mn.product_id
)
select customer_id,product_name from rnk_after_join where rnk_date=1;
```
| customer_id | product_name |
|--------------|--------------|
| A            | sushi        |
| A            | curry        |
| B            | sushi        |

#### 8.What is the total items and amount spent for each member before they became a member?
```sql
select s.customer_id,count(s.product_id) total_items,sum(menu.price) total_spent from sales s
inner join members m on s.customer_id=m.customer_id and s.order_date<=m.join_date
inner join menu on s.product_id=menu.product_id
group by s.customer_id
```
| customer_id | total_items | total_amount |
|--------------|--------------|--------------|
| B            | 3            | 34           |
| A            | 4            | 51           |

#### 9.If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
##### MSSQL
```sql
with cte as (
  select s.customer_id,s.order_date,m.product_name,
    row_number() over(partition by customer_id order by order_date) rn,
    case when m.product_name="sushi" then 0 else m.price*10 end points
  from sales s
  inner join menu m on s.product_id = m.product_id
),cte2 as (
  select rn,customer_id,order_date,product_name,points,points total_points from cte
  where rn=1
  union all
  select cte.rn,cte.customer_id,cte.order_date,cte.product_name,cte.points,
    case when cte.product_name="sushi" then total_points*2 else total_points+cte.points end total_points
  from cte2
  inner join cte on cte2.customer_id=cte.customer_id and cte2.rn+1=cte.rn
),last_points_cte as (
  select customer_id,max(rn) max_rn from cte2
  group by customer_id
)
select cte2.customer_id,cte2.total_points from cte2
inner join last_points_cte lcte on cte2.rn=lcte.max_rn and cte2.customer_id=lcte.customer_id
order by cte2.customer_id
```
##### POSTGRESQL
```sql
with recursive cte_rnk_date as (
	select s.customer_id,s.order_date,s.product_id,m.product_name,m.price,
		row_number() over(partition by s.customer_id order by order_date) rnk_date from danny_dinner.sales s
	inner join danny_dinner.menu m on s.product_id=m.product_id
),cte as (
	select customer_id,order_date,product_id,product_name,price,rnk_date,
	case when product_name='sushi' then 0 else price*10 end points 
	from cte_rnk_date where rnk_date=1
	union
	select c2.customer_id,c2.order_date,c2.product_id,c2.product_name,c2.price,c2.rnk_date,
		case when c2.product_name='sushi' then c1.points*2 else c1.points+c2.price*10 end
	from cte c1
	inner join cte_rnk_date c2 on c1.rnk_date+1=c2.rnk_date and c1.customer_id=c2.customer_id
),rnk_desc as (
	select *,rank() over(partition by customer_id order by rnk_date desc) rnk_desc from cte
)
select customer_id,points from rnk_desc where rnk_desc=1;
```
| customer_id | points |
|--------------|---------|
| A            | 810     |
| B            | 1440    |
| C            | 360     |

#### 10.In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
##### MSSQL
```sql
with cte as (
  select s.customer_id,s.order_date,me.price*10 points,m.join_date,
    row_number() over(partition by s.customer_id order by s.order_date) rn from sales s
  inner join members m on s.customer_id = m.customer_id
  inner join menu me on s.product_id = me.product_id
),cte2 as (
  select rn,customer_id,order_date,join_date,points,datediff(day,join_date,order_date) as diff,points total_points from cte
  where rn=1
  union all
  select cte.rn,cte.customer_id,cte.order_date,cte.join_date,cte.points,datediff(day,cte.join_date,cte.order_date) diff,
    case when datediff(day,cte.join_date,cte.order_date)<=7 and datediff(day,cte.join_date,cte.order_date)>=0 
      then total_points*2 else total_points+cte.points end total_points 
  from cte2
  inner join cte on cte2.customer_id=cte.customer_id and cte2.rn+1=cte.rn
),last_points_cte as (
  select customer_id,max(order_date) last_day_january,max(rn) max_rn from cte2
  where month(order_date)=1
  group by customer_id
)
select cte2.customer_id,cte2.total_points from cte2
inner join last_points_cte lcte on cte2.customer_id=lcte.customer_id 
                                    and cte2.order_date=lcte.last_day_january
                                    and cte2.rn=lcte.max_rn
```
##### POSTGRESQL
```sql
with recursive cte_rnk_date as (
	select s.customer_id,s.order_date,mm.join_date,s.product_id,m.product_name,m.price,
		row_number() over(partition by s.customer_id order by order_date) rnk_date from danny_dinner.sales s
	inner join danny_dinner.menu m on s.product_id=m.product_id
	inner join danny_dinner.members mm on s.customer_id=mm.customer_id
),cte as (
	select customer_id,order_date,join_date,product_id,product_name,price,rnk_date,
		case when order_date-join_date between 0 and 6 then 'True' else 'False' end bonus,
		case when order_date-join_date between 0 and 6 then 0 else price*10 end points
	from cte_rnk_date
	where rnk_date=1
	union
	select c2.customer_id,c2.order_date,c2.join_date,c2.product_id,c2.product_name,c2.price,c2.rnk_date,
		case when c2.order_date-c2.join_date between 0 and 6 then 'True' else 'False' end bonus,
		case when c2.order_date-c2.join_date between 0 and 6 then c1.points*2 else c1.points+c2.price*10 end points
	from cte c1
	inner join cte_rnk_date c2 on c1.customer_id=c2.customer_id and c1.rnk_date+1=c2.rnk_date
),last_january_points as (
	select *,rank() over(partition by customer_id order by rnk_date desc) rnk_points from cte 
	where extract(month from order_date)=1
)
select customer_id,points from last_january_points
where rnk_points=1;
```
| customer_id | points |
|--------------|---------|
| A            | 4000    |
| B            | 920     |

### <p align='center'>Bonus Questions
#### 1. Join All The Things
The following questions are related creating basic data tables that Danny and his team can use to quickly derive insights without needing to join the underlying tables using SQL.

| customer_id | order_date | product_name | price | member |
|-------------|------------|--------------|-------|--------|
| A           | 2021-01-01 | curry        | 15    | N      |
| A           | 2021-01-01 | sushi        | 10    | N      |
| A           | 2021-01-07 | curry        | 15    | Y      |
| A           | 2021-01-10 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| B           | 2021-01-01 | curry        | 15    | N      |
| B           | 2021-01-02 | curry        | 15    | N      |
| B           | 2021-01-04 | sushi        | 10    | N      |
| B           | 2021-01-11 | sushi        | 10    | Y      |
| B           | 2021-01-16 | ramen        | 12    | Y      |
| B           | 2021-02-01 | ramen        | 12    | Y      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-07 | ramen        | 12    | N      |

##### MSSQL
```sql
select s.customer_id,s.order_date,me.product_name,me.price,
  case when s.customer_id in (select customer_id from members) and m.join_date<=s.order_date then 'Y' else 'N' end members from sales s
left join members m on s.customer_id = m.customer_id
inner join menu me on s.product_id = me.product_id
```
##### POSTGRESQL
```sql
select s.customer_id,s.order_date,m.product_name,m.price, 
	case when s.order_date>=mm.join_date then 'Y' else 'N' end member
into danny_dinner.temp_all_things from danny_dinner.sales s
inner join danny_dinner.menu m on s.product_id=m.product_id
left join danny_dinner.members mm on s.customer_id=mm.customer_id
order by customer_id,order_date;
select * from danny_dinner.temp_all_things;
```
| customer_id | order_date  | product_name | price | member |
|--------------|-------------|---------------|--------|---------|
| A            | 2021-01-01  | sushi         | 10     | N       |
| A            | 2021-01-01  | curry         | 15     | N       |
| A            | 2021-01-07  | curry         | 15     | Y       |
| A            | 2021-01-10  | ramen         | 12     | Y       |
| A            | 2021-01-11  | ramen         | 12     | Y       |
| A            | 2021-01-11  | ramen         | 12     | Y       |
| B            | 2021-01-01  | curry         | 15     | N       |
| B            | 2021-01-02  | curry         | 15     | N       |
| B            | 2021-01-04  | sushi         | 10     | N       |
| B            | 2021-01-11  | sushi         | 10     | Y       |
| B            | 2021-01-16  | ramen         | 12     | Y       |
| B            | 2021-02-01  | ramen         | 12     | Y       |
| C            | 2021-01-01  | ramen         | 12     | N       |
| C            | 2021-01-01  | ramen         | 12     | N       |
| C            | 2021-01-07  | ramen         | 12     | N       |

#### 2. Rank All The Things
Danny also requires further information about the ranking of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ranking values for the records when customers are not yet part of the loyalty program.
```sql
with cte as (
  select s.customer_id,s.order_date,me.product_name,me.price,
    case when s.customer_id in (select customer_id from members) and m.join_date<=s.order_date then 'Y' else 'N' end members from sales s
  left join members m on s.customer_id = m.customer_id
  inner join menu me on s.product_id = me.product_id
),ranking_cte as (
  select *,rank() over(partition by customer_id order by order_date) rnk from cte
  where members='Y'
  union all
  select *,null rnk from cte
  where members='N'
)
select * from ranking_cte
order by customer_id,order_date
```
| customer_id | order_date  | product_name | price | member | rnk_flag |
|--------------|-------------|---------------|--------|---------|-----------|
| A            | 2021-01-01  | sushi         | 10     | N       |           |
| A            | 2021-01-01  | curry         | 15     | N       |           |
| A            | 2021-01-07  | curry         | 15     | Y       | 1         |
| A            | 2021-01-10  | ramen         | 12     | Y       | 2         |
| A            | 2021-01-11  | ramen         | 12     | Y       | 3         |
| A            | 2021-01-11  | ramen         | 12     | Y       | 3         |
| B            | 2021-01-01  | curry         | 15     | N       |           |
| B            | 2021-01-02  | curry         | 15     | N       |           |
| B            | 2021-01-04  | sushi         | 10     | N       |           |
| B            | 2021-01-11  | sushi         | 10     | Y       | 1         |
| B            | 2021-01-16  | ramen         | 12     | Y       | 2         |
| B            | 2021-02-01  | ramen         | 12     | Y       | 3         |
| C            | 2021-01-01  | ramen         | 12     | N       |           |
| C            | 2021-01-01  | ramen         | 12     | N       |           |
| C            | 2021-01-07  | ramen         | 12     | N       |           |
