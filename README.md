-- *********  B: USER DEFINED FUNCTION ************

CREATE OR REPLACE FUNCTION rent_sort_month(rental_date TIMESTAMP)
RETURNS TEXT
LANGUAGE plpgsql
AS
$$
DECLARE
rental_month TEXT;
BEGIN
rental_month := TO_CHAR(rental_date, 'Month');
RETURN rental_month;
END;
$$;

-- ********* C. Create "detailed" and "summary" tables *******

-- ******** DETAILED TABLE CREATION ***********

CREATE TABLE detailed_table (
rental_id INT,
store_id INT,
payment_date VARCHAR(25),
payment_amount numeric (7,2) );

-- ********* SUMMARY TABLE CREATION *********

CREATE TABLE summary_table (
store_id INT,
payment_month VARCHAR(25),
total_revenue numeric(10,2) );

-- ********* TABLE CREATION TEST **********

SELECT * FROM detailed_table;
SELECT * FROM summary_table;

-- ******** D: DATA INSERTION ********

INSERT INTO detailed_table
SELECT p.rental_id, s.store_id, rent_sort_month(p.payment_date), 
p.amount
FROM payment p
JOIN rental r ON p.rental_id = r.rental_id
JOIN staff s ON r.staff_id = s.staff_id
ORDER BY 2,3,4;

-- ********* E. TRIGGER FUNCTION *********

CREATE OR REPLACE FUNCTION data_trigger()
  RETURNS TRIGGER 
  LANGUAGE plpgsql 
AS
$$
BEGIN

DELETE FROM summary_table;
INSERT INTO summary_table
SELECT store_id, payment_date, SUM(payment_amount)
FROM detailed_table
GROUP BY store_id, payment_date
ORDER BY 1,2;
RETURN NEW;

END;
$$;

CREATE TRIGGER updated_summary
AFTER INSERT
ON detailed_table
FOR EACH STATEMENT
EXECUTE PROCEDURE data_trigger();


-- ******* F: STORED PROCEDURE - REFRESH ********.

CREATE OR REPLACE PROCEDURE data_refresh()
LANGUAGE plpgsql
AS $$
BEGIN

    DELETE FROM detailed_table;
    DELETE FROM summary_table;

    INSERT INTO detailed_table
    SELECT p.rental_id, s.store_id, rent_sort_month(p.payment_date), 
        p.amount
        FROM payment p
        JOIN rental r ON p.rental_id = r.rental_id
        JOIN staff s ON r.staff_id = s.staff_id
        ORDER BY 2,3,4;

RETURN;
END;
$$;


-- ***** RANDOM INSERTION *********
SELECT count(*) FROM detailed_table;
SELECT count(*) FROM summary_table;

INSERT INTO detailed_table 
VALUES ('0', '0', 'January', '11.11'); 

-- ******* TABLE CHECK 1 **********
SELECT * FROM detailed_table;
SELECT count(*) FROM detailed_table;

SELECT * FROM summary_table;
SELECT count(*) FROM summary_table;



-- ****** EXECUTE STORED PROCEDURE - REFRESH ********

CALL data_refresh()

--- ******* TABLE CHECK 2 *******
SELECT * FROM detailed_table;
SELECT count(*) FROM detailed_table;

SELECT * FROM summary_table;
SELECT count(*) FROM summary_table;
