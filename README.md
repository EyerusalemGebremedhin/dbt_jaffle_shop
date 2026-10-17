# dbt Jaffle Shop

A sample dbt project from dbt Labs. It loads sample data, transforms it with SQL models, and runs tests to check the data is correct. Uses DuckDB (a simple database that runs in a single file).

## Project structure

- `dbt_project.yml` - dbt configuration
- `models/staging/` - clean raw data (rename columns, fix types)
- `models/orders.sql` - orders with payment split by method
- `models/customers.sql` - customers with order history and lifetime value
- `models/schema.yml` - documentation and tests
- `seeds/` - raw CSV data files
- `profiles.yml` - DuckDB database location

## Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install dbt-core dbt-duckdb
```

## Run

```bash
dbt build
```

This loads the seeds, builds all models, and runs all tests. You should see 29 steps pass: 3 seeds, 3 views, 2 tables, and 21 tests.

## Check the data

```bash
python3 -c "
import duckdb
con = duckdb.connect('jaffle_shop.duckdb', read_only=True)
print(con.execute('SELECT * FROM customers LIMIT 5').fetchall())
"
```

## Models explained

### Staging models (models/staging/)
These take raw CSV data and make it clean:
- `stg_customers.sql` - renames `id` to `customer_id`, reads from `raw_customers`
- `stg_orders.sql` - renames `id` to `order_id`, `user_id` to `customer_id`, reads from `raw_orders`
- `stg_payments.sql` - converts `amount` from cents to dollars, reads from `raw_payments`

Each uses `{{ ref('raw_...') }}` to get its own seed data.

### orders.sql
Joins orders with payments. Splits the total payment into four columns: `credit_card_amount`, `coupon_amount`, `bank_transfer_amount`, `gift_card_amount`. Uses a Jinja loop over the list of payment methods.

### customers.sql
Joins customers with their orders and payments. Calculates `first_order`, `most_recent_order`, `number_of_orders`, and `customer_lifetime_value`. Also calculates `average_order_value` (see below).

## What ref() does

`ref()` means "use the output of another model". It creates a dependency. For example, `orders.sql` says `select * from {{ ref('stg_orders') }}`. This means: "first build stg_orders, then use it in orders". dbt uses this to figure out the correct build order.

## Tests

Tests are defined in `schema.yml`. They check:
- **unique** - no duplicate values (e.g. every customer_id appears once)
- **not_null** - no empty values (e.g. order_id must exist)
- **accepted_values** - only allowed values (e.g. status must be placed/shipped/completed/return_pending/returned)
- **relationships** - foreign key exists in another table (e.g. orders.customer_id exists in customers.customer_id)

## My change: average_order_value

I added a new column to `customers.sql`:

```sql
case
    when customer_orders.number_of_orders > 0
    then customer_payments.total_amount / customer_orders.number_of_orders
    else 0
end as average_order_value
```

This calculates the average amount spent per order. The `CASE WHEN` prevents division by zero (if a customer has no orders, it returns 0). I also documented the column in `schema.yml` and added a `not_null` test.

## Break/fix test exercise

To prove the tests work, I:
1. Added a duplicate customer_id to `seeds/raw_customers.csv`
2. Ran `dbt build` - the `unique` test on customer_id FAILED with: "Got 1 result, configured to fail if != 0"
3. Removed the duplicate line
4. Ran `dbt build` again - all 29 tests PASSED

## Why we test data

Tests catch problems before they reach dashboards or AI tools. If a customer_id is duplicated, the lifetime value is wrong. If an order has no customer_id, we cannot attribute revenue correctly. Tests give us confidence that the data is trustworthy.

## Loading vs transforming

**Loading** (my fx-pipeline): Gets data from an API and puts it into a database. Fast and simple. Example: GET exchange rates from an API, save to SQLite.

**Transforming** (dbt): Cleans, joins, and organizes data that is already in the database. More complex, makes data useful. Example: join orders with payments, calculate lifetime value per customer.