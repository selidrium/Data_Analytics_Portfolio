# Scenario: New Pizzeria opening, take-out and delivery

## Stakeholder needs:

Design and build a tailormade bespoke relational database to capture and store all the data the business generates to use for dashboards.

### Main Focuses:
- Customer Orders
- Stock Control Requirements
- Staff

### Orders Data Requested:
- Item name
- Item price
- Quantity
- Customer Name
- Delivery Address

### Actual Data Needed:
- Row ID
- Order ID
- Item name
- Item category
- Item size
- Item price
- Quantity
- Customer first name
- Customer last name
- Delivery address 1
- Delivery address 2
- Delivery city
- Delivery zip code

## Stock Control Requirements:
- **Objective**: Stakeholder wants to know when to order new stock
- **Execution**: We will need more information about:
    - Ingredients
    - Quantity based on sizes
    - Existing stock level

## Staff Data Requirements:
- **Objective**: track employees' schedule and determine pizza cost (ingredients + chefs + delivery)

## Query to create tables:

```sql
CREATE TABLE `orders` (
    `row_id` int  NOT NULL ,
    `order_id` varchar(10)  NOT NULL ,
    `created_at` datetime  NOT NULL ,
    `item_id` varchar(10)  NOT NULL ,
    `quantity` int  NOT NULL ,
    `cust_id` int  NOT NULL ,
    `delivery` boolean  NOT NULL ,
    `add_id` int  NOT NULL ,
    PRIMARY KEY (
        `row_id`
    )
);

CREATE TABLE `customers` (
    `cust_id` int  NOT NULL ,
    `cust_firstname` varchar(50)  NOT NULL ,
    `cust_lastname` varchar(50)  NOT NULL ,
    PRIMARY KEY (
        `cust_id`
    )
);

CREATE TABLE `address` (
    `add_id` int  NOT NULL ,
    `delivery_address1` varchar(200)  NOT NULL ,
    `delivery_address2` varchar(200)  NULL ,
    `delivery_city` varchar(50)  NOT NULL ,
    `delivery_zipcode` varchar(20)  NOT NULL ,
    PRIMARY KEY (
        `add_id`
    )
);

CREATE TABLE `item` (
    `item_id` varchar(10)  NOT NULL ,
    `sku` varchar(20)  NOT NULL ,
    `item_name` varchar(50)  NOT NULL ,
    `item_cat` varchar(50)  NOT NULL ,
    `item_size` varchar(20)  NOT NULL ,
    `item_price` decimal(5,2)  NOT NULL ,
    PRIMARY KEY (
        `item_id`
    )
);

CREATE TABLE `ingredient` (
    `ing_id` varchar(10)  NOT NULL ,
    `ing_name` varchar(200)  NOT NULL ,
    `ing_weight` int  NOT NULL ,
    `ing_meas` varchar(20)  NOT NULL ,
    `ing_price` decimal(5,2)  NOT NULL ,
    PRIMARY KEY (
        `ing_id`
    )
);

CREATE TABLE `recipe` (
    `row_id` int  NOT NULL ,
    `recipe_id` varchar(20)  NOT NULL ,
    `ing_id` varchar(10)  NOT NULL ,
    `quantity` int  NOT NULL ,
    PRIMARY KEY (
        `row_id`
    )
);

CREATE TABLE `inventory` (
    `inv_id` int  NOT NULL ,
    `item_id` varchar(10)  NOT NULL ,
    `quantity` int  NOT NULL ,
    PRIMARY KEY (
        `inv_id`
    )
);

CREATE TABLE `rota` (
    `row_id` int  NOT NULL ,
    `rota_id` varchar(20)  NOT NULL ,
    `date` datetime  NOT NULL ,
    `shift_id` varchar(20)  NOT NULL ,
    `staff_id` varchar(20)  NOT NULL ,
    PRIMARY KEY (
        `row_id`
    )
);

CREATE TABLE `staff` (
    `staff_id` int  NOT NULL ,
    `first_name` varchar(50)  NOT NULL ,
    `last_name` varchar(50)  NOT NULL ,
    `position` varchar(100)  NOT NULL ,
    `hourly_rate` decimal(5,2)  NOT NULL ,
    PRIMARY KEY (
        `staff_id`
    )
);

CREATE TABLE `shift` (
    `shift_id` int  NOT NULL ,
    `day_of_week` varchar(10)  NOT NULL ,
    `start_time` time  NOT NULL ,
    `end_time` time  NOT NULL ,
    PRIMARY KEY (
        `shift_id`
    )
);

ALTER TABLE `customers` ADD CONSTRAINT `fk_customers_cust_id` FOREIGN KEY(`cust_id`)
REFERENCES `orders` (`cust_id`);

ALTER TABLE `address` ADD CONSTRAINT `fk_address_add_id` FOREIGN KEY(`add_id`)
REFERENCES `orders` (`add_id`);

ALTER TABLE `item` ADD CONSTRAINT `fk_item_item_id` FOREIGN KEY(`item_id`)
REFERENCES `orders` (`item_id`);

ALTER TABLE `ingredient` ADD CONSTRAINT `fk_ingredient_ing_id` FOREIGN KEY(`ing_id`)
REFERENCES `recipe` (`ing_id`);

ALTER TABLE `recipe` ADD CONSTRAINT `fk_recipe_recipe_id` FOREIGN KEY(`recipe_id`)
REFERENCES `item` (`sku`);

ALTER TABLE `inventory` ADD CONSTRAINT `fk_inventory_item_id` FOREIGN KEY(`item_id`)
REFERENCES `recipe` (`ing_id`);

ALTER TABLE `rota` ADD CONSTRAINT `fk_rota_date` FOREIGN KEY(`date`)
REFERENCES `orders` (`created_at`);

ALTER TABLE `rota` ADD CONSTRAINT `fk_rota_staff_id` FOREIGN KEY(`staff_id`)
REFERENCES `staff` (`staff_id`);

ALTER TABLE `shift` ADD CONSTRAINT `fk_shift_shift_id` FOREIGN KEY(`shift_id`)
REFERENCES `rota` (`shift_id`);
```

### Dashboard 1: Order Activity
1. Total orders
2. Total sales
3. Total items
4. Average order value
5. Sales by category
6. Top selling items
7. Orders by hour
8. Sales by hour
9. Orders by address
10. Orders by delivery/pickup

**SQL Queries for Dashboard 1:**
```sql
SELECT 
	o.order_id,
	i.item_price,
	o.quantity,
	i.item_cat,
	i.item_name,
	o.created_at,
	a.delivery_address1,
	a.delivery_address2,
	a.delivery_city,
	a.delivery_zipcode,
	o.delivery
	FROM 
		orders o
		LEFT JOIN item i on o.item_id = i.item_id
		LEFT JOIN address a on o.add_id = a.add_id
```

### Dashboard 2: Inventory Management

1. Total quantity by ingredient
2. Total cost of ingredients
3. Calculated cost of pizza
4. Percentage stock remaining by ingredient

**SQL Queries for Dashboard 2:**
-1-3: 

```sql
select 
s1.item_name,
s1.ing_id,
s1.ing_name,
s1.ing_weight,
s1.ing_price,
s1.order_quantity,
s1.recipe_quantity,
s1.order_quantity*s1.recipe_quantity as ordered_weight,
s1.ing_price/s1.ing_weight as unit_cost,
(s1.order_quantity*s1.recipe_quantity)*(s1.ing_price/s1.ing_weight) as ingredient_cost
from (SELECT
	o.item_id,
    i.sku,
	i.item_name,
    r.ing_id,
    ing.ing_name,
    r.quantity AS recipe_quantity,
    sum( o.quantity ) AS order_quantity,
    ing.ing_weight,
    ing.ing_price
FROM
	orders o
    LEFT JOIN item i ON o.item_id = i.item_id
    LEFT JOIN recipe r ON i.sku = r.recipe_id
    LEFT JOIN ingredient ing ON ing.ing_id = r.ing_id
GROUP BY
	o.item_id,
    i.sku,
    i.item_name,
    r.ing_id,
    r.quantity)s1
```
- 4:

```

SELECT 
    s2.ing_name,
    s2.ordered_weight,
    ing.ing_weight*inv.quantity AS total_inv_weight,
    (ing.ing_weight*inv.quantity)-s2.ordered_weight AS remaining_weight


FROM (SELECT
	ing_id,
	ing_name,
    sum( ordered_weight ) as ordered_weight
FROM
	stock1
    group by ing_name, ing_id) s2
    
LEFT JOIN inventory inv ON inv.item_id = s2.ing_id
LEFT JOIN ingredient ing ON ing.ing_id = s2.ing_id

```
### Dashboard 3: Staff Management
1. Cost per employee
2. Schedule availability
3. Shift openings
   
**SQL Queries for Dashboard 3:**
```sql
SELECT
	r.date,
	s.first_name,
	s.last_name,
	s.hourly_rate,
    sh.start_time,
    sh.end_time,
	TIMEDIFF(sh.end_time, sh.start_time) AS hours_worked,
    (TIMESTAMPDIFF(SECOND, sh.start_time, sh.end_time) / 3600) * s.hourly_rate AS staff_cost
FROM rota r	
LEFT JOIN staff s on r.staff_id = s.staff_id
LEFT JOIN shift sh on r.shift_id = sh.shift_id
```
