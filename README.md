# 🧠 Oracle PL/SQL Concepts

This document contains important PL/SQL concepts, interview notes, and practical examples including BULK COLLECT, FORALL, TRIGGERS, and EXECUTE IMMEDIATE.

---

# 🔐 Database Access (SQL*Plus / PL/SQL Developer)

Login steps:

Open PL/SQL Developer  
→ SYS user  
→ Password  
→ Connect as SYSDBA  

```sql
SELECT * FROM dba_users;

Task- implement mcp server and project , check with Rahul


Variables:
%type - single cpolumn type
%rowtype -> entire row type

--select * from HR.employees;
DECLARE
 v_ename varchar(50);
 v_sal HR.employees.salary%type;
BEGIN
 select first_name || ' '||last_name, SALARY
 INTO v_ename, v_sal
 from HR.EMPLOYEES
 where EMPLOYEE_ID=101;
 dbms_output.put_line(v_ename||'  '||v_sal);
 EXCEPTION
   when no_data_found then
     dbms_output.PUT_LINE('Employee not present');    
END;

--Using %rowtype variable -
DECLARE
 v_emp HR.EMPLOYEES%rowtype;
BEGIN
 select * INTO v_emp 
 from HR.EMPLOYEES
 where EMPLOYEE_ID=101;

 dbms_output.put_line(v_emp.first_name||' '||v_emp.salary||' '||v_emp.department_Id);   
END;


--USE of IF -> ELSE  -> END IF
DECLARE
 v_emp HR.EMPLOYEES%rowtype;
BEGIN
 select * INTO v_emp 
 from HR.EMPLOYEES
 where EMPLOYEE_ID=101;
 dbms_output.put_line(v_emp.first_name||' '||v_emp.salary||' '||v_emp.department_Id);

 IF v_emp.salary >= 15000 then
    dbms_output.put_line('salary '||' '||v_emp.salary|| 'is higher.');
 ELSE
    dbms_output.put_line('salary '||' '||v_emp.salary|| 'is lower.');
 END IF;
 
END;


DECLARE
 v_num number(5):= 0;
BEGIN
    
    for v_num in 1 .. 10 LOOP
        DBMS_OUTPUT.PUT_LINE('Numbers: '|| v_num);
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('*******************');
    v_num := 100000;
    WHILE v_num <= 50 LOOP
        DBMS_OUTPUT.PUT_LINE('Nums: '||v_num);
        v_num := v_num+1;
    END LOOP;
    EXCEPTION
        WHEN OTHERS THEN
            DBMS_OUTPUT.PUT_LINE('Error: '|| SQLERRM);
END;

NOTE:  EXCEPTION block supports WHEN -> THEN clause, we can't use IF instead of WHEN, but can use it in nested clasue like below.

EXCEPTION
    WHEN OTHERS THEN
        IF SQLCODE = -1403 THEN
            DBMS_OUTPUT.PUT_LINE('Employee not found.');
        ELSE
            DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
        END IF;


CURSOR:

A cursor is a pointer to a private SQL area in Oracle that stores information about the execution of a SQL statement. For a SELECT statement, the cursor provides access to the result set (rows returned by the query), allowing PL/SQL to process those rows one at a time (using FETCH).

Cursor c_cur_demo
 |
 v
+----------------------+
| Private SQL Area |
|----------------------|
| SQL Text |
| Execution State |
| Result Set |
| Current Row Pointer |
+----------------------+

NOTE:- For explicit cursors, Oracle doesn't necessarily copy all rows into memory at once. Internally, Oracle may fetch rows in batches as needed. Conceptually, however, you can think of the cursor as giving you access to the query result set and maintaining the current position within it.
So:
	• Private SQL area = query + execution state + fetch information + access to result rows. 
	• Cursor = handle/pointer to that private SQL area. 
	• Result set = all rows returned by the query (Steven, Neena, Lex, etc.). 
	• FETCH = retrieves the next row from that result set and advances the cursor position. 


DECLARE
    v_FIRST_NAME HR.employees.FIRST_NAME%type;
    v_salary HR.employees.salary%type;
    v_department_id HR.employees.department_id%type;
    cursor c_cur_demo IS
    select FIRST_NAME, salary, department_id
    from HR.EMPLOYEES
    where DEPARTMENT_ID=90;
BEGIN
    open c_cur_demo;
    LOOP
        fetch c_cur_demo INTO v_FIRST_NAME, v_salary, v_department_id;
        EXIT when c_cur_demo%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE(v_FIRST_NAME||'  '||v_salary||'  '||v_department_id);
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('Total rowcounts that are fetched by cursor are: '|| c_cur_demo%rowcount );
    close c_cur_demo;
    EXCEPTION
        when others THEN
            if c_cur_demo%isopen THEN
                close c_cur_demo;
            end if;
        dbms_output.put_line('Error:  '||SQLERRM);
END;


REF cursor ( Dynamic Cursor):-
Ref cursor is a cursor where the query is not fixed but can be passed dynamically during compile time.

--select * from HR.employees;
DECLARE
    c_ref_cur SYS_REFCURSOR;

    v_department_id HR.EMPLOYEES.department_id%type := 60;
    v_emp HR.EMPLOYEES%rowtype;

BEGIN
    
    if v_department_id is not null THEN
        open c_ref_cur FOR
        select * from HR.employees
        where department_id = v_department_id;
    ELSE
        open c_ref_cur FOR
        SELECT * FROM
        HR.EMPLOYEES;
    END IF;

    LOOP
        FETCH c_ref_cur INTO v_emp;
        EXIT when c_ref_cur%notfound;
        DBMS_OUTPUT.PUT_LINE(v_emp.first_name ||' '||v_emp.salary);
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('cursor row count: '|| c_ref_cur%rowcount);

    close c_ref_cur;

    EXCEPTION
        when others THEN
            if c_ref_cur%isopen THEN
                close c_ref_cur;
            end if;
        dbms_output.put_line('Error:  '||SQLERRM);
END;

HR.employees%ROWTYPE  --> This represents one row of the EMPLOYEES table.

TYPE v_table_emp IS TABLE OF HR.employees%ROWTYPE  --> This defines a collection type (nested table) whose elements are employees rows.

Cursor with BULK COLLECT (Performance Boost ⚡)
Instead of fetching one row at a time, fetch all rows at once into a collection. 
Dramatically reduces context switches between SQL and PL/SQL engines.

DECLARE
    --define collection type
    TYPE v_table_emp IS TABLE OF HR.employees%rowtype;
    v_emp v_table_emp;
    cursor c_cur IS
    select * from HR.employees
    where salary > 10000;
 --   c_num number(2):= 1;
BEGIN
    open c_cur;
    FETCH c_cur BULK COLLECT INTO v_emp;
    DBMS_OUTPUT.PUT_LINE('Fetched rows count: '|| v_emp.count );
    FOR i in 1 .. v_emp.count LOOP
        DBMS_OUTPUT.PUT_LINE('Row '||i||': ---> '||v_emp(i).first_name||' '||v_emp(i).department_id||' '||v_emp(i).salary);
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('Cursor row counts: '||c_cur%rowcount );
    close c_cur;
END;



BULK COLLECT with LIMIT (Memory-Safe): Used for batch processing

DECLARE
    TYPE t_emp_table IS TABLE OF employees%ROWTYPE;
    v_employees  t_emp_table;

    CURSOR c_emp IS SELECT * FROM employees;

    c_batch_size CONSTANT NUMBER := 100;  -- Process 100 rows at a time

BEGIN
    OPEN c_emp;
    LOOP
        -- Fetch in batches to avoid memory issues
        FETCH c_emp BULK COLLECT INTO v_employees LIMIT c_batch_size;

        EXIT WHEN v_employees.COUNT = 0;

        -- Process each batch
        FOR i IN 1 .. v_employees.COUNT LOOP
            -- Your processing logic here
            NULL;
        END LOOP;

        DBMS_OUTPUT.PUT_LINE('Processed batch of: ' || v_employees.COUNT);

        EXIT WHEN c_emp%NOTFOUND;
    END LOOP;
    CLOSE c_emp;
END;
/

--select * from hr.employees;

create or replace procedure proc-demo(
    v_sal number(10),
    v_empno number(5)
)
is
begin
    update hr.employees set salary=v_sal
    where employee_id=v_empno;
end;

exec proc-demo(10000,101);


Prepare these thoroughly:
	1. Difference between Procedure and Function 
	2. Cursor vs Bulk Collect 
	3. Bulk Collect vs FORALL 
	4. Package Advantages 
	5. Trigger Types 
	6. Exception Handling 
	7. Dynamic SQL 
	8. Mutating Table Error 
	9. Autonomous Transaction 
	10. Explain Plan 
	11. Index Types 
	12. Context Switching 
	13. Collections 
	14. SQL Tuning 
	15. Real-time production issue you resolved

Open plsql developer -> sys -> password - system -> XE -> sys_as_sysdba

Select * from dba_users;

Diffeent excceptions:

NO_DATA_FOUND
TOO_MANY_ROWS
ZERO_DIVIDE

• BULK COLLECT retrieves all employee IDs in one operation. 
• FORALL sends all update statements to the SQL engine in a single batch. 
• Context switching is dramatically reduced.

	Q. WAQ to update the salary in bulk in emp2 table…each dept should get increment with specific amount that should be passed by parameter.

	Create below plsql block with table type-
	
	CREATE OR REPLACE TYPE empno_list IS TABLE OF number;
	
	
	--create this function to get the list of employee_id for a department_Id -> use cursor with bulk collect to minimize context switching, for faster execution-
	
	CREATE OR REPLACE FUNCTION get_emp ( v_deptno NUMBER)
	RETURN empno_list
	IS
	--    TYPE empno_list IS TABLE OF HR.emp2.employee_id%TYPE;
	    v_empno empno_list;
	    
	    cursor c_cur is
	    select employee_id
	    FROM hr.emp2
	    WHERE department_id = v_deptno;
	    
	BEGIN
	    open c_cur;
	    fetch c_cur bulk collect into v_empno;
	
	    RETURN v_empno;
	
	    exception
	       when others then
	         dbms_output.put_line('Errors :'|| sqlerrm ||sqlcode);
	END;
	

-- create procedure to update salary for these list of employees, by calling the function.
create or replace procedure update_sal (
  percentage number, v_deptno NUMBER
)
IS

v_empid empno_list;

BEGIN
  
  v_empid := get_emp(v_deptno);
 
  Forall i in 1 .. v_empid.count
    update HR.emp2
    SET salary=salary*(1 + percentage/100)
    where employee_id=v_empid(i);

    dbms_output.put_line('row affeted :'||sql%rowcount);
end;


Call the above procedure from a pl/sql block-
begin
  update_sal(10,90);
end;
/

Triggers - A trigger in PL/SQL is a stored program that automatically executes when a specific event occurs on a table, view, schema, or database. Events are insert, update, delete.

Row-level trigger - trigger once per 
Statement level trigger - trigger once for each events.

DML trigger - it triggers after/before of insert, update, delete.

After trigger/before trigger -

create or replace trigger bef_trg
before update of salary
on HR.emp2
for each row
begin
  if :OLD.salary <= :NEW.salary then
     dbms_output.put_line('salary update from old salary :'||:old.salary||' to new salry '||:NEW.salary);
  else
      RAISE_APPLICATION_ERROR(-2001,'salary cant be lesser.');
  end if;
end;

Key point
An AFTER ROW trigger can still prevent the update by raising an exception. The word "AFTER" means "after Oracle has processed the row internally," not "after the transaction is permanently committed."

create or replace trigger aft_trg2
  after insert or update or delete
  on emp2 
  for each row
declare
  -- local variables here
begin
  dbms_output.put_line('old salary :'||:OLD.salary);
  dbms_output.put_line('New salary :'||:NEW.salary);
  
  if :OLD.salary <= :NEW.salary then
     dbms_output.put_line('salary update from old salary :'||:old.salary||'to new salry '||:NEW.salary);
  else
      RAISE_APPLICATION_ERROR(-20001,'salary cant be lesser.');
  end if;
end aft_trg2;



An INSTEAD OF trigger is used on a view, usually when the view is not directly updatable.
Oracle cannot automatically determine how to perform the INSERT, UPDATE, or DELETE on the underlying tables for view, so the trigger tells Oracle what to do instead.

create or replace triggere insteadOf_trg_emp2_view
instead of insert
on emp2_view
for each row
begin
  insert into emp2( first_name, job)
  values(:NEW.first_name, :NEW.job);
end;

Let say we don't have a trigger on a view, if we want to do below insert it can't be done.
After creating above view we can insert above column values into view -

Insert into emp2_view(first_name, job) VALUES ('Dipu', 50000)  -> internally executes trigger emp2_view and value gets inserted into base table.

Common Interview Example
Prevent employees from being inserted on weekends:

create or replace trigger trg_weekend_denied
before insert
on emp2
for each row
begin
  if to_char(susdate,'DY') in ( 'SAT','SUN') then
    RAISE_APPLICATION_ERROR(-20005,'Employee entry restricted in weekends');
  end IF;
end;

RAISE_APPLICATION_ERROR error codes ranges between -20000 to -20999.

EXECUTE IMMEDIATE - It is used to execute the dynamic sql in runtime. We used to store the sql statement in a string at runtime and used to execute through it.
EXECUTE IMMEDIATE in PL/SQL is one of the most important features for dynamic SQL execution.
Think of it as:
	"Build an SQL statement as a string at runtime and execute it."
This is useful when the SQL statement is not known when the PL/SQL code is compiled.

CREATE OR REPLACE PROCEDURE drop_table_dyn (
    p_table_name IN VARCHAR2
)
AS
    v_sql VARCHAR2(500);
BEGIN
    v_sql := 'DROP TABLE ' || p_table_name;

    EXECUTE IMMEDIATE v_sql;

    DBMS_OUTPUT.PUT_LINE('Table dropped: ' || p_table_name);
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/

Execute the proc from pl/ql block
BEGIN
    drop_table_dyn('EMP_TEST');
END;
/





<img width="1153" height="4277" alt="image" src="https://github.com/user-attachments/assets/90b924f6-ce61-48bd-b429-b8d907dd1b8a" />
