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
