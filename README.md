# Employee Management System - SQL Functions, Procedures, Triggers, and Views

## Project Description

The **Employee Management System (EMS)** is a collection of SQL functions, procedures, triggers, and views designed to manage and maintain employee data. The system enforces business rules to ensure data integrity and smooth operations. Features include:

- Preventing the dropping of tables.
- Handling employee insertions with validation.
- Deleting linked employee data when an employee is removed.
- Protecting department rows from deletion.
- Archiving deleted employee data.
- Displaying employee hierarchies.
- Ranking employees by salary within departments.

## Features

### 1. Prevent Dropping of Tables
```sql
CREATE OR REPLACE FUNCTION prevent_drop()
RETURNS event_trigger
LANGUAGE plpgsql
AS
$$
	BEGIN
		IF TG_TAG ='DROP TABLE' THEN
			RAISE EXCEPTION 'Cannot drop the table';
		END IF;
	END;
$$

---TRIGGER
CREATE EVENT TRIGGER prevent_Drop
ON ddl_command_start
WHEN TAG IN ('DROP TABLE')
EXECUTE FUNCTION prevent_drop();

---TEST
DROP TABLE employees;
```
### 2.Handling employee insertions with validation.
```sql
CREATE OR REPLACE PROCEDURE Insert_Employee
(
	first_n varchar
	,last_n varchar
	,phone_number varchar(20)
	,job_title varchar
	,dept_id int
	,manager_id INT
	,salary NUMERIC(10,2)

)
LANGUAGE plpgsql
AS
$$
	DECLARE emp_id INT;
	DECLARE counter_dept INT;
	DECLARE email varchar;
	DECLARE checker varchar;
	BEGIN
		--Email for new employee
		email=concat(first_n,'.',last_n,'@company.com');
		--Validate data before Inserting
		SELECT first_name 
		FROM employees O 
		WHERE first_name=first_n AND last_name=last_n 
		INTO checker;
		--
		SELECT MAX(employee_id) 
		FROM employees 
		INTO emp_id;
	
		SELECT MAX(department_id) 
		FROM departments 
		INTO counter_dept;
		
		emp_id = emp_id + 1;
		IF dept_id <= counter_dept  AND manager_id <= emp_id and checker is null THEN
			INSERT INTO employees VALUES(emp_id,first_n,last_n,job_title,dept_id,manager_id);
			INSERT INTO salary VALUES(emp_id,salary);
			INSERT INTO employee_contact VALUES(emp_id,phone_number,email);
		ELSE
			RAISE 'INCORRECT INFORMATION ADDED, PLEASE FIX AND TRY AGAIN';
		END IF;
		
		EXCEPTION WHEN OTHERS THEN
			RAISE;
	END;
$$

---TEST
CALL Insert_Employee('Sanele','Zondo','555-567-8901','Intern',3,2,35000);
```

### 3. Deleting linked employee data when an employee is removed.
```sql
CREATE OR REPLACE FUNCTION f_delete()
RETURNS TRIGGER
LANGUAGE plpgsql
AS
$$
	BEGIN
	
		DELETE FROM employee_contact
		WHERE employee_id=old.employee_id;
		
		DELETE FROM salary
		WHERE employee_id=old.employee_id;
		
		RETURN OLD;
	END;
$$
--TRIGGER
CREATE OR REPLACE TRIGGER tr_delete
BEFORE DELETE
ON employees
FOR EACH ROW 
EXECUTE FUNCTION f_delete();

---TEST
DELETE FROM employees
WHERE employee_id=16;
```

### 4. Protecting department rows from deletion.
```sql
CREATE OR REPLACE FUNCTION prevent_Delete_DEP()
RETURNS TRIGGER
LANGUAGE plpgsql
AS
$$
	BEGIN
		RAISE EXCEPTION 'Cannot delete Department Row';
	END;
$$

---TRIGGER
CREATE TRIGGER prevent_Delete_DEP
BEFORE DELETE
ON departments
FOR EACH ROW 
EXECUTE FUNCTION prevent_Delete_DEP();

---TEST 
DELETE FROM departments
WHERE department_id = 4;
```
### 5. Archiving deleted employee data.
```sql
CREATE OR REPLACE FUNCTION archives()
RETURNS TRIGGER
LANGUAGE plpgsql
AS
$$
	DECLARE _checker int;
	BEGIN
		SELECT 
			CASE 
				WHEN first_name<> old.first_name and last_name<> old.last_name THEN 1 ELSE 0 
			END 
		FROM employees_archives INTO _checker;
		
		IF _checker IN (1) THEN
			INSERT INTO employees_archives VALUES(old.employee_id,old.first_name,old.last_name,old.job_title,old.department_id,old.manager_id);
			RETURN OLD;
		ELSE
			RAISE 'Employee Added Already';
		END IF;
	END;
$$

--TRIGGER
CREATE OR REPLACE TRIGGER archives_employees
AFTER DELETE
ON employees
FOR EACH ROW
EXECUTE FUNCTION archives();

--TEST
DELETE FROM employees
where employee_id=15
```
### 6. A view to display data.
```sql
CREATE OR REPLACE VIEW vw_view_data
AS
(
	SELECT 
		O.employee_id
		,O.first_name
		,O.last_name
		,O.job_title
		,null::varchar(50) AS manager
		,C.department_name 
		,D.salary
	FROM employees O
	INNER JOIN departments C
	ON
		O.department_id=C.department_id
	INNER JOIN salary D
	ON
		O.employee_id =D.employee_id
	WHERE O.manager_id IS NULL
	
	UNION
	
	SELECT 
		O2.employee_id
		,O2.first_name
		,O2.last_name
		,O2.job_title
		,O1.first_name AS manager
		,O3.department_name 
		,O4.salary
	FROM employees O1
	INNER JOIN employees O2
	ON
		O1.employee_id = O2.manager_id 
	INNER JOIN departments O3
	ON
		O2.department_id=O3.department_id
	INNER JOIN salary O4
	ON
		O2.employee_id =O4.employee_id
	ORDER BY 1 ASC
	
)
---TEST
SELECT * FROM vw_view_data
```
### 7. Displaying employee hierarchies.
```sql
CREATE OR REPLACE FUNCTION Employee_hierarchy(employee int)
RETURNS TABLE (Name VARCHAR ,Hieraechy_Level INT)
LANGUAGE plpgsql
AS
$$
	BEGIN
		RETURN QUERY
		WITH RECURSIVE cte_Employee_hierarchy
		AS
		(
			SELECT 
				employee_id
				,first_name
				,last_name
				,job_title
				,department_id
				,manager_id
				,1 as Level
			FROM employees
			WHERE employee_id = employee

			UNION ALL

			SELECT 
				 O.employee_id
				,O.first_name
				,O.last_name
				,O.job_title
				,O.department_id
				,O.manager_id
				,Level+1
			FROM employees O
			INNER JOIN cte_Employee_hierarchy C
			ON
				C.employee_id = O.manager_id
		)
		SELECT
			first_name,level
		FROM cte_Employee_hierarchy;
	END;
$$
---TEST
SELECT * from Employee_hierarchy(1)
```
### 8. Ranking employees by salary within departments.
```sql
SELECT 
    employee_id,
    first_name,
    last_name,
    job_title,
    department_name,
    manager,
    salary,
    -- Calculate the percentage contribution of the employee's salary to the department's total salary
    ROUND(salary / SUM(salary) OVER (PARTITION BY department_name) * 100, 2) AS "%_CONTRIBUTION",
    -- Rank employees based on their salary in descending order within each department
    RANK() OVER (PARTITION BY department_name ORDER BY salary DESC) AS RANK_SALARIES_BY_DEPT
FROM vw_view_data;
```
------------------------------------------------END--------------------------------------------------------
