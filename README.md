# sql-business-decisions
SQL project using the Python package "pandas" and "sqlite3" to read and print queries from a .db file

# Project Details:
For this project, imagine that we work for the Chinook Record Store and our data is stored within the "chinook.db" (The Chinook database is actually a popular practice database for SQL practice, available on Github). We're going to use the database to answer a few business questions. The database schema is below with the highlighted being primary keys and the connections being foreign keys:

![image](https://user-images.githubusercontent.com/57373723/68708541-cb035980-0548-11ea-8594-cd28770acb46.png)


# Creating helper functions

We're going to create two helper functions to better seperate the different queries we're going to run. The run_query() function will be used for queries in which we just want to retrieve information. The run_command() function will be used specifically when we want to make changes to the database:
```python
import sqlite3
import pandas

def run_query(q):
    # using a with statement will provide better exception handling, it also automatically closes the file.
    # this "context manager" ensures we don't accidentally make changes to the database if one of our queries has an error
    with sqlite3.connect('chinook.db') as conn:
        return pandas.read_sql(q, conn)


def run_command(c):
    with sqlite3.connect('chinook.db') as conn:
        # tells SQLite to autocommit any changes
        conn.isolation_level = None
        conn.execute(c)
        
```

#  Most Popular Genre in the USA

Let's pretend we are currently in search of new record labels to work with and distribute for in the USA. Many record labels generally focus on a certain genre and thus we want to see what our most popular genres in the USA are (top 10):

```python

# create a sub-query using a WITH statement to pull only the customers in the USA

popular_usa_genre = '''
WITH usa_tracks_sold AS
   (
    SELECT il.*
    FROM invoice_line il
    INNER JOIN invoice i on il.invoice_id = i.invoice_id
    INNER JOIN customer c on i.customer_id = c.customer_id
    WHERE c.country = "USA"
   )


SELECT g.name genre, COUNT(uts.invoice_line_id) tracks_sold, CAST(COUNT(uts.invoice_line_id) AS FLOAT) / (SELECT COUNT(*)
                                                                                                      FROM usa_tracks_sold
                                                                                                       ) percentage_sold
FROM usa_tracks_sold uts
INNER JOIN track t on t.track_id = uts.track_id
INNER JOIN genre g on g.genre_id = t.genre_id
GROUP BY g.name
ORDER BY tracks_sold DESC
LIMIT 10;
'''

print(run_query(popular_usa_genre))
```

| |               genre|  tracks_sold|  percentage_sold|
|---|------------------|-------------|-----------------|
|0|                Rock|          561|         0.533777|
|1|  Alternative & Punk|          130|         0.123692|
|2|               Metal|          124|         0.117983|
|3|            R&B/Soul|           53|         0.050428|
|4|               Blues|           36|         0.034253|
|5|         Alternative|           35|         0.033302|
|6|               Latin|           22|         0.020932|
|7|                 Pop|           22|         0.020932|
|8|         Hip Hop/Rap|           20|         0.019029|
|9|                Jazz|           14|         0.013321|

Now that we have the info, we can go ahead and visualize it in a graph.

Note: The following was run on Jupyter Notebook, which has a "maigc command" (%matplotlib inline) that will print the graphs in order of creation. If not using Jupyter Notebook, you will use the "matplotlib.pyplot.show()" function under the matplotlib package at the end. This will print out all of the created graphs at once:

```python
%matplotlib inline # delete if not using Jupyter Notebook
import matplotlib.pyplot # delete if using Jupyter Notebook

run_query(popular_usa_genre).plot.bar('genre', 'percentage_sold', rot=25) #rot is just rotating the x-axis titles to a 25 degree angle
```
![image](https://user-images.githubusercontent.com/57373723/68715033-f17bc180-0555-11ea-9fde-1be083182c89.png)

It is very noticeable that the most popular genre in the USA is easily Rock as it represents more than half of the tracks sold (53.37%). Thus we may want to focus on connecting with record labels who focus on Rock artists.

# Analyzing Employee Sales Performance
Each customer for the Chinook store gets assigned to a sales support agent within the company when they first make a purchase. We've been tasked by upper management to check the purchases connected to each service agent to see if anyone is underperfoming/overperforming.
```python

# Create a subquery to total the value of purchases per customer as they keep the same sales rep for each purchase

employee_sales_performance = '''
WITH
    total_per_cust AS (SELECT customer.customer_id, customer.support_rep_id, SUM(invoice.total) AS 'customer_total'
                        FROM invoice
                        LEFT JOIN customer ON customer.customer_id = invoice.customer_id
                        GROUP BY customer.customer_id)

SELECT employee.first_name || ' ' || employee.last_name AS 'employee_name',
    SUM(total_per_cust.customer_total) AS 'agent_total'
FROM total_per_cust
LEFT JOIN employee ON employee.employee_id = total_per_cust.support_rep_id
GROUP BY total_per_cust.support_rep_id
ORDER BY agent_total DESC;
'''

print(run_query(employee_sales_performance))
run_query(employee_sales_performance).plot.barh('employee_name', 'agent_total')
```
||employee_name|agent_total|
|---|-----|-----------|
|0|Jane Peacock|1731.51|
|1|Margaret Park|1584.00|
|2|Steve Johnson|1393.92|

![image](https://user-images.githubusercontent.com/57373723/68901737-587eaf00-06eb-11ea-8d3b-24fab5a9e1e6.png)

From the looks of it, it does seem like Jane is outperforming Steve by almost 25%. However, we overlooked one important detail, when were they hired. This can easily be added by revising the main query as hire_date is under the employee table. Thus we can revise the following:
```python
SELECT employee.first_name || ' ' || employee.last_name AS 'employee_name', SUM(total_per_cust.customer_total) AS 'agent_total'
```
to 
```python
# added the hire date column at the end
SELECT employee.first_name || ' ' || employee.last_name AS 'employee_name', SUM(total_per_cust.customer_total) AS 'agent_total', employee.hire_date
```

||employee_name|agent_total|hire_date|
|---|-----|-----------|---------|
|0|Jane Peacock|1731.51|2017-04-01 00:00:00|
|1|Margaret Park|1584.00|2017-05-03 00:00:00|
|2|Steve Johnson|1393.92|2017-10-17 00:00:00|

Now we can see that Jane was hired the earliest of the three and thus had a headstart in accumulating sales. Based on the current date/month, we could create some form of an average to see who is truly achieving the most/least.

# Analyzing Sales by Country
Next, let's take a look and analyze sales on a country by country level. First, let's define country by the customer themselves and not by which country is listed on the invoices. For any country with just a 1 single customer, let's put them together in a "other" category. Management has determined we need to find the following to analyze sales in each country:

- total number of customers
- total value of sales
- average value of sales per customer
- average order value

Let's order the table by the amount of customers but place the "Other" section at the very end:

```python

# Create a subquery categorizing whether or not the country should be under "Other"

sales_per_country = '''
WITH 
    country_or_other AS (SELECT CASE WHEN (
                 SELECT count(*)
                 FROM customer
                 where country = c.country
                ) = 1 THEN "Other"
           ELSE c.country
       END AS country,
       c.customer_id,
       il.*
     FROM invoice_line il
     INNER JOIN invoice i ON i.invoice_id = il.invoice_id
     INNER JOIN customer c ON c.customer_id = i.customer_id
    )
    
SELECT country, count(distinct customer_id) customers, SUM(unit_price) total_sales,
SUM(unit_price) / count(distinct customer_id) customer_lifetime_value,
SUM(unit_price) / count(distinct invoice_id) average_order, CASE WHEN country = "Other"
                                                                THEN 1
                                                                ELSE 0
                                                            END AS sort
FROM country_or_other
GROUP BY country
ORDER BY sort ASC, total_sales DESC
'''

print(run_query(sales_per_country))
run_query(sales_per_country).plot.bar('country', 'average_order', rot=0)

```
| |country|customers|total_sales|customer_lifetime_value|average_order|sort|
|---|---|---|---|---|---|---|
|0|USA|13|1040.49|80.037692|7.942672|0|
|1|Canada|8|535.59|66.948750|7.047237|0|
|2|Brazil|5|427.68|85.536000|7.011148|0|
|3|France|5|389.07|77.814000|7.781400|0|
|4|Germany|4|334.62|83.655000|8.161463|0|
|5|Czech Republic|2|273.24|136.620000|9.108000|0|
|6|United Kingdom|3|245.52|81.840000|8.768571|0|
|7|Portugal|2|185.13|92.565000|6.383793|0|
|8|India|2|183.15|91.575000|8.721429|0|
|9|Other|15|1094.94|72.996000|7.448571|1|

![image](https://user-images.githubusercontent.com/57373723/68904796-a26b9300-06f3-11ea-8436-a6e43b65972d.png)

USA has the most customers but the average customer/order is generally middle of the pack. What really stands out is that we have 15 different countries where we had just a single customer. Unfortunately, there isn't a high enough count of customers in each country to determine much.

