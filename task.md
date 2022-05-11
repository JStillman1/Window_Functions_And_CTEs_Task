# Task - Employee Records

The major corporation BusinessCorp&#8482; wants to do some analysis of varioius metrics around its employees and have contracted the job out to you. They have provided you with two SQL files, you will need to write the queries which will extract the data they are looking for.

## Setup

- Create a database to run the files against
- Use the `psql -d database -f file` command to run the SQL files
- Before starting to write the queries take some time to look at the data and figure out the relationships between the tables - maybe even draw an ERD to help.

## Tasks

### CTEs

1) Calculate the average salary of all employees

```sql
SELECT AVG(salary) FROM employees
```

2) Calculate the average salary of the employees in each team (hint: you'll need to `JOIN` and `GROUP BY` here)

```sql
WITH average_salaries (id, average) AS (
SELECT departments.id, ROUND(AVG(employees.salary), 2)  FROM departments
INNER JOIN employees
ON departments.id = employees.department_id
GROUP BY departments.id)

SELECT departments.id, average, name FROM average_salaries JOIN departments ON departments.id = average_salaries.id

```

3) Using a CTE find the ratio of each employees salary to their team average, eg. an employee earning £55000 in a team where the average is £50000 has a ratio of 1.1

```sql
WITH average_salaries (id, team_average) AS (
SELECT departments.id, ROUND(AVG(employees.salary), 2)  FROM departments
INNER JOIN employees
ON departments.id = employees.department_id
GROUP BY departments.id)

SELECT employees.id, first_name, last_name, salary, team_average, ROUND(salary/team_average, 2) AS ratio FROM average_salaries JOIN employees ON employees.department_id = average_salaries.id
```

4) Find the employee with the highest ratio in Argentina

```sql
WITH average_salaries (id, team_average) AS (
SELECT departments.id, ROUND(AVG(employees.salary), 2)  FROM departments
INNER JOIN employees
ON departments.id = employees.department_id
GROUP BY departments.id)

SELECT MAX(ROUND(salary/team_average, 2)) AS ratio FROM average_salaries JOIN employees ON employees.department_id = average_salaries.id WHERE country = 'Argentina'
```

5) **Extension:** Add a second CTE calculating the average salary for each country and add a column showing the difference between each employee's salary and their country average

```sql
WITH country_average (country, country_average_salary) AS (
SELECT country, ROUND(AVG(salary),2) FROM employees 
JOIN departments ON employees.department_id = departments.id
GROUP BY country)

SELECT id, first_name, last_name, salary, employees.country, country_average_salary, salary - country_average_salary AS difference FROM country_average JOIN employees 
ON employees.country = country_average.country
```

---

### Window Functions

1) Find the running total of salary costs as the business has grown and hired more people

```sql
SELECT start_date, salary, SUM(salary)OVER (ORDER BY start_date) AS running_salary_total FROM employees 
```

2) Determine if any employees started on the same day (hint: some sort of ranking may be useful here)

```sql
SELECT COUNT(DISTINCT(start_date)) - COUNT(*) FROM employees
```

3) Find how many employees there are from each country

```sql
SELECT country, COUNT(*) FROM employees GROUP BY country

<!-- Or, using Window functions-->
SELECT DISTINCT(country), COUNT(*) OVER(PARTITION BY country) FROM employees 
```

4) Show how the average salary cost for each department has changed as the number of employees has increased

```sql
SELECT departments.id, departments.name, SUM(salary) OVER (PARTITION BY departments.id ORDER BY  start_date) FROM employees 
JOIN departments
ON departments.id = employees.department_id

```

5) **Extension:** Research the `EXTRACT` function and use it in conjunction with `PARTITION` and `COUNT` to show how many employees started working for BusinessCorp&#8482; each year. If you're feeling adventurous you could further partition by month...

```sql
<!-- By year -->
SELECT DISTINCT(EXTRACT(year FROM start_date)), COUNT(*) OVER(PARTITION BY EXTRACT(year FROM start_date)) FROM  employees ORDER BY EXTRACT (year FROM start_date)

<!-- By month as well -->
SELECT DISTINCT(DATE_TRUNC('month', start_date)) AS month, COUNT(*) OVER(PARTITION BY DATE_TRUNC('month', start_date)) FROM  employees ORDER BY DATE_TRUNC('month', start_date)

```

---

### Combining the two

1) Find the maximum and minimum salaries

```sql
SELECT MAX(salary), MIN(salary) FROM employees
```

2) Find the difference between the maximum and minimum salaries and each employee's own salary

```sql
WITH min_max (id, minimum_salary, maximum_salary) AS (
SELECT employees.id, MIN(salary) OVER (), MAX(salary) OVER () FROM employees)

SELECT employees.id, salary, salary - minimum_salary AS more_than_min, maximum_salary - salary AS less_than_max FROM employees JOIN min_max ON employees.id = min_max.id
```

3) Order the employees by start date. Research how to calculate the **median** salary value and the **standard deviation** in salary values and show how these change as more employees join the company

```sql
CREATE OR REPLACE FUNCTION _final_median(numeric[])
   RETURNS numeric AS
$$
   SELECT AVG(val)
   FROM (
     SELECT val
     FROM unnest($1) val
     ORDER BY 1
     LIMIT  2 - MOD(array_upper($1, 1), 2)
     OFFSET CEIL(array_upper($1, 1) / 2.0) - 1
   ) sub;
$$
LANGUAGE 'sql' IMMUTABLE;

CREATE AGGREGATE median(numeric) (
  SFUNC=array_append,
  STYPE=numeric[],
  FINALFUNC=_final_median,
  INITCOND='{}'
);

WITH medians (id, median) AS (SELECT employees.id, median(salary) OVER (ORDER BY start_date) FROM employees),
stddevs (id, standard_deviations) AS (SELECT employees.id, ROUND(STDDEV(salary) OVER (ORDER BY start_date), 2) FROM employees)

SELECT * FROM employees 
JOIN medians ON medians.id = employees.id 
JOIN stddevs ON employees.id = stddevs.id 


```

4) Limit this query to only Research & Development team members and show a rolling value for only the 5 most recent employees.

```sql
<!-- With the function as above already created -->

WITH medians (id, median) AS (SELECT employees.id, median(salary) OVER (ORDER BY start_date) FROM employees WHERE department_id = 3),
stddevs (id, standard_deviations) AS (SELECT employees.id, ROUND(STDDEV(salary) OVER (ORDER BY start_date), 2) FROM employees WHERE department_id = 3)

SELECT employees.*, departments.name AS department_name, medians.median, stddevs.standard_deviations FROM departments
JOIN employees ON departments.id = employees.department_id
JOIN medians ON medians.id = employees.id 
JOIN stddevs ON employees.id = stddevs.id 
ORDER BY start_date DESC LIMIT 5

```

