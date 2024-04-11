# Primary Key: 
A primary key is a unique identifier for each record in a table. It uniquely identifies each row in the table and must contain unique values. Also, a primary key column cannot have NULL values. Each table in a database should have a primary key.

In a table called `Students`, you might have a primary key named `student_id`, which uniquely identifies each student. For example:
```
student_id | student_name |  age   |   city
---------------------------------------------
   1       |   John Doe   |   20   |  New York
   2       |   Jane Smith |   22   |  Boston
   3       |   Alice Lee  |   21   |  Chicago
```
# Foreign Key:
A foreign key is a column or a set of columns in one table that refers to the primary key in another table. It establishes a relationship between two tables, enforcing referential integrity. The foreign key ensures that the values in the referencing column(s) exist in the referenced table's primary key column(s).

Let's say you have another table called `Grades`, and you want to reference the `Students` table. You might have a foreign key `student_id` in the `Grades` table, referring to the primary key `student_id` in the `Students` table:
```
grade_id  | student_id |  course  |  grade
------------------------------------------
   1      |     1      |  Math    |   A
   2      |     2      |  Science |   B
   3      |     3      |  English |   A-
```
In this example, `student_id` in the `Grades` table references the primary key `student_id` in the `Students` table.
# Unique Key: 
A unique key is similar to a primary key in that it ensures that all values in a column or a group of columns are unique. However, unlike a primary key, a table can have multiple unique keys. Unique keys can contain NULL values, but if a column is defined as a unique key, only one NULL value is allowed.

Suppose you have a table called `Employees`, and you want to ensure that each employee has a unique employee number (`emp_id`), but you allow `NULL` values for the `department` column. You could use a unique key on the `emp_id` column:
```
emp_id | employee_name |  department
-------------------------------------
 1001  |   John Smith  |   Sales
 1002  |   Jane Doe    |   NULL
 1003  |   Alice Lee   |   Marketing
```
 In this example, `emp_id` is a unique key, ensuring that each employee has a unique identifier.
# Composite Key:
A composite key is a key that consists of two or more columns to uniquely identify a record in a table. Together, these columns form a unique combination. Unlike a primary key, none of the columns in a composite key individually need to be unique, but the combination of values across the columns must be unique.
  
Imagine you have a table called `Orders`, and you want to uniquely identify each order by a combination of `order_id` and `customer_id`. You could create a composite key using both columns:
```
order_id | customer_id |   order_date   |  total_amount
-------------------------------------------------------
  101    |     1001    |  2024-04-01    |    $50.00
  102    |     1002    |  2024-04-02    |    $75.00
  103    |     1001    |  2024-04-02    |    $30.00
```
In this example, the combination of `order_id` and `customer_id` forms a composite key, ensuring that each order is uniquely identified.