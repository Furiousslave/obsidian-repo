# Inner Join
Returns only the rows that have matching values in both tables.
![[Pasted image 20240402134213.png]]

Example: Consider two tables, "employees" and "departments".
```
employees table:
| emp_id | emp_name | department_id |
|--------|----------|---------------|
| 1      | John     | 101           |
| 2      | Alice    | 102           |
| 3      | Bob      | 101           |
| 4      | Emma     | 103           |

departments table:
| department_id | department_name |
|---------------|-----------------|
| 101           | HR              |
| 102           | IT              |
| 103           | Sales           |
```
Inner Join query:
```
SELECT employees.emp_id, employees.emp_name, departments.department_name
FROM employees
INNER JOIN departments ON employees.department_id = departments.department_id;
```
This will result in:
```
| emp_id | emp_name | department_name |
|--------|----------|-----------------|
| 1      | John     | HR              |
| 2      | Alice    | IT              |
| 3      | Bob      | HR              |
| 4      | Emma     | Sales           |
```
# Left Join
Returns all rows from the left table and matching rows from the right table. If there are no matching rows in the right table, NULL values are returned.
![[Pasted image 20240402134235.png]]

Example: Using the same tables as above.
Left Join query:
```
SELECT employees.emp_id, employees.emp_name, departments.department_name
FROM employees
LEFT JOIN departments ON employees.department_id = departments.department_id;
```
This will result in:
```
| emp_id | emp_name | department_name |
|--------|----------|-----------------|
| 1      | John     | HR              |
| 2      | Alice    | IT              |
| 3      | Bob      | HR              |
| 4      | Emma     | Sales           |
```
# Right Join
Returns all rows from the right table and matching rows from the left table. If there are no matching rows in the left table, NULL values are returned.

Example: Using the same tables as above.
Right Join query:
```
SELECT employees.emp_id, employees.emp_name, departments.department_name
FROM employees
RIGHT JOIN departments ON employees.department_id = departments.department_id;
```
This will result in:
```
| emp_id | emp_name | department_name |
|--------|----------|-----------------|
| 1      | John     | HR              |
| 2      | Alice    | IT              |
| 3      | Bob      | HR              |
| 4      | Emma     | Sales           |
```
# Outer Join
![[Pasted image 20240402134402.png]]