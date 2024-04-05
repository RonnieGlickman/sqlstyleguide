## Objective

This style guide is intended to standardize how an analytics team writes production code. Ad-hoc queries can be written in whatever format pleases the developer.  Once the code needs to go in production, the style guide should be applied.

In order to make applying these styling principles easier for developers, there is a SQL linter called [SQL Fluff](https://sqlfluff.com/) that will easily apply these changes to whatever file a dev is changing. Find more information about integrating SQLFluff into your Github or Gitlab development process [here](https://github.com/sqlfluff/sqlfluff-github-actions) 
  
## General Principles for Writing SQL

- Optimize for readability and maintainability rather than fewer lines of code
- Write code with the WET (write everything twice) programming principle in mind
	- It's okay to write the same logic in two different places.  Once we've written it a third time that logic should be standardized in a single upstream table
- Be explicit rather than implicit
	- Instead of short form alias, column names, and table names - focus on enhancing your code by being explicit with the naming and intention of code blocks

## Query Structure and Syntax Styling

- Lowercase everything
	- This removes the need for devs to hit shift or caps lock to swap back and forth between cases
    - Common IDEs highlight keywords and functions to make it easier for readers to delineate between keywords and non-keyword syntax
- Trailing commas only
	- Improves readability of the code as it's the natural way lists are written
    - Snowflake supports trailing commas, so the ability to comment out the last line of syntax is now supported making using trailing commas easier
- Put each column on its own line and each SQL keyword on it's own line
- If there is a distinct  qualifier, put it on the same line as `select`
- Each new column line should be indented with either 1 indent or 4 spaces
- Left align select  statements if they're not in a CTE

```
with cte_name as (
    select 
        column_name
    from 
        table_name
)
select
    column_name
from
    cte_name
```

- Avoid using `select *`  outside of import CTEs
	- `select *` isn't explicit as to what columns are being used and some tables could have hundreds of columns which makes debugging and understanding the intent of the code difficult

### CTEs

- Utilize CTEs instead of subqueries
	- CTEs help to break your code into more readable blocks
	- Each CTE should ideally perform one logical body of work
- CTEs should be placed at the beginning of the query
- Use import CTEs to filter datasets at the beginning of your query
	- Import CTEs are useful when you need to filter a dataset in order to utilize it. 
	- Filtering can help with overall ETL performance, especially when joining tables later in the script

```
with import_table_name as (
    select 
        * 
    from 
        database.schema.table_name 
    where 
        created_date >= '2024-01-01'
),
final as (
    select
        column_one,
        column_two
    from
        import_table_name
)
select * from final
```

- Be explicit with CTE names. The CTE name should indicate what that code block is trying to accomplish
- Indent the select statement within your CTE 1 tab, or 4 spaces

```
-- Good
with active_customer_accounts as (
    select
        customer_id,
        account_id,
        account_status
    from
        customers
    where
        account_status = 'active'
)

-- Bad
with customers as (
    select
        customer_id,
        account_id,
        account_status
    from
        customers
    where
        account_status = 'active'
)

-- Really Bad
with cte_1 as (
    select
        customer_id,
        account_id,
        account_status
    from
        customers
    where
        account_status = 'active'
)
```


- Use a final CTE before your final select statement.
	- This comes in handy when debugging a script where you have multiple CTEs since you can easily isolate the CTE that has a bug by replacing the `select * from final` with `select * from cte_name`
```
Using a final CTE 

with active_customer_accounts as (
    select
        customer_id,
        account_id,
        account_status
    from
        customers
    where
        account_status = 'active'
),
active_customer_accounts_pending_orders as (
    select
        active_customers.customer_id,
        active_customers.account_id,
        active_customers.account_status,
        orders.id as order_id,
        orders.create_date as order_create_date,
        orders.order_status
    from
        active_customer_accounts as active_customers
    inner join
        orders on active_customers.customer_id = orders.customer_id
    where
        orders.order_status = 'pending'
),
final as (
    select
        customer_id,
        account_id,
        account_status,
        order_id,
        order_create_date,
        order_status
    from
        active_customer_accounts_pending_orders
)
select * from final
;
```
### CASE Statements

- Case statements with one when  clause can be written on one line
- Case statements with multiple when  clauses should be written with each when  clause on it's own line
- If a when  clause has multiple conditions the first condition should be written on the same line as the when  clause, each subsequent condition should go on a new line and indented from the first line
```
-- Good

select 
    case when status_code = 1 then 'Active' else 'Inactive' end as customer_status,

-- Good

select
    case
        when status_code = 1 and deleted_at is null then 'Active'
        else 'Inactive'
    end as customer_status,

/* Good */

select
    case
        when status_code = 1 then 'Active'
        when status_code = 0 then 'Inactive'
        when status_code = 0 and cancelled_at is not null then 'Cancelled'
        else 'No Status'
    end as customer_status,

/* Bad */

select
    case when status_code = 1 and deleted_at is null then 'Active' when status_code = 0 then 'Inactive' 
    when status_code = 0 and cancelled_at is not null then 'Cancelled' else 'No Status' 
    end as customer_status
```
 
### Join Conditions

- Be explicit in JOIN conditions - use `inner join`  instead of `join`
- If using an `inner join`, apply your filtering logic in the `where` clause of the `select` statement
- Avoid using `right join`  - instead, restructure your query to use `left join`
- The left table in a `left join` condition should be referenced first 
```
select
    table_a.id
from
    table_a
left join
    table_b on table_a.id = table_b.id
```
- Joins with single conditions should be written on one line
- Joins with multiple conditions should have each condition written on it's own line
```
-- Good

from 
    customers
left join 
    orders on customers.id = orders.customer_id

-- Good

from
    customers
left join
    orders on customers.id = orders.customer_id
        and orders.order_status = 'pending'

```

### Aggregation, Filtering, and Ordering

- Use column names for `group by` and `order by` statements
- `group by all`  is your best friend if you need to group by a lot of columns
	- Make sure to put your aggregates at the end of your column list so it's understood what columns are being aggregated  
- Each column in the `group by` , `order by` , and `where`  clause should have it's own line unless there is only one column being utilized
```
-- Good

where alert_type = 'new'

-- Good

where
    alert_type = 'new'
    and alert_created_date = '2023-01-01'

-- Good

group by alert_type

-- Good

group by
    alert_type,
    alert_created_date

-- Good

order by alert_type

-- Good

order by
    alert_type,
    alert_created_date
```
- Use lateral column aliasing in your `group by`
```
-- Good

select
    date_trunc('month', alert_created_at) as alert_created_month,
    count(*) as total_alerts
from 
    companies
group by
    alert_created_month

-- Bad

select
    date_trunc('month', alert_created_at) as alert_created_month,
    count(*) as total_alerts
from 
    companies
group by
    date_trunc('month', alert_created_at)
```

- Avoid using `order by` to reduce compute costs - leave that to the BI tools or Excel/GSheet user to do that
- Break long lists of `in` values into multiple lines with one value per line
```
-- Good

where email_address in (
    'user_1@sample.com',
    'user_2@sample.com',
    'user_3@sample.com'
)

-- Bad

where email_address in ('user_1@sample.com','user_2@sample.com','user_3_@sample.com')
```


### Table and Column Naming Conventions

- Be **explicit** with column, table, and alias names
- It's okay if these names become obnoxiously long - embrace being explicit
- When aliasing columns use `as` to define the alias name
```
-- Good

select
    id as customer_id
from
    customers

-- Bad

select
    a.column_name
from
    table_name a
```
- Avoid using non-descriptive naming conventions such as `id` , `name` , and `type`  - prefix these columns with what is being identified or named
- Booleans should start with `has_`  or `is_`
- Timestamps should end with `_at`
- Dates should end with `_date`
- Avoid reserved keywords like `date`  or `month`  as column names
- Always rename aggregates
```
-- Good

count(transaction_id) as transactions_count

-- Bad

count(transaction_id)
```


## Sample SQL

```
with sample_cte as (
    select
        column_name,
        is_boolean
    from
        table_name
),

final as (
    select
        table_alias.column_name,
		cte_alias.is_boolean,
		table_alias.created_at,
		table_alias.created_date,
		case
            when table_alias.column_name = 'hello' then 'world'
			when cte_alias.column_name = 'world' then 'hello'
			else null
		end as column_name,
		count(table_alias.column_name) as total_column_name
	from
        example_table as table_alias
	inner join
		sample_cte as cte_alias on table_alias.column_name = sample_cte.column_name
	left join
		left_join_table_name as table_name on table_alias.column_name = table_name.column_name
	group by
        table_alias.column_name,
		cte_alias.column_name,
		column_name,
		table_alias.created_at,
		table_alias.created_date
)
select * from final
;

```
