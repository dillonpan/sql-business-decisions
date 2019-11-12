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
GROUP BY 1
ORDER BY 2 DESC
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

It is very noticeable that the most popular genre in the USA is easily Rock as it represents more than half of the tracks sold. Thus we may want to focus on connecting with record labels who focus on Rock artists.

