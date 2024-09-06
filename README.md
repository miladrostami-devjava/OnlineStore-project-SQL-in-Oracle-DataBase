# Scenario: online store order management system
As a PL/SQL programmer for a large online store, 
you are responsible for managing the database of customer orders.
The store contains various sections such as customers, products,
orders, and invoices. Your goal is to optimize and implement PL/SQL 
queries and procedures to manage this data.



create user online_store_milad identified by Milad13711992;
grant connect ,resource to online_store_milad;
alter session set current_schema = ONLINE_STORE_MILAD;

-- create table space
create tablespace online_store_tablespace
DATAFILE 'C:\Users\Parse\Desktop\MiladTask\JavaCoreTask\OnlineStore-project-SQL-in-Oracle\src\main\resources\online_store_tablespace_file.dbf' size 100m;
ALTER USER ONLINE_STORE_MILAD DEFAULT TABLESPACE online_store_tablespace;


GRANT UNLIMITED TABLESPACE TO ONLINE_STORE_MILAD;


CREATE TABLE CUSTOMERS (
CUSTOMER_ID NUMBER PRIMARY KEY NOT NULL ,
FIRST_NAME VARCHAR2(30) NOT NULL ,
LAST_NAME VARCHAR2(30) NOT NULL ,
EMAIL VARCHAR2(50) NOT NULL UNIQUE ,
REGISTRATION_DATE DATE NOT NULL CHECK ( REGISTRATION_DATE > TO_DATE('01-01-2024' ,'DD-MM-YYYY')  )
);

CREATE TABLE PRODUCTS (
PRODUCT_ID NUMBER PRIMARY KEY NOT NULL ,
PRODUCT_NAME VARCHAR2(100) NOT NULL UNIQUE ,
PRICE NUMBER(10, 2) NOT NULL CHECK ( PRICE > 5 ),
STOCK_QUANTITY NUMBER NOT NULL CHECK ( STOCK_QUANTITY > 0 )
);

CREATE TABLE ORDERS (
ORDER_ID NUMBER PRIMARY KEY NOT NULL ,
CUSTOMER_ID NUMBER NOT NULL ,
ORDER_DATE DATE NOT NULL ,
TOTAL_AMOUNT NUMBER(10, 2),
FOREIGN KEY (CUSTOMER_ID) REFERENCES CUSTOMERS(CUSTOMER_ID)
);

CREATE TABLE ORDER_ITEMS (
ORDER_ITEM_ID NUMBER PRIMARY KEY,
ORDER_ID NUMBER,
PRODUCT_ID NUMBER,
QUANTITY NUMBER,
UNIT_PRICE NUMBER(10, 2),
FOREIGN KEY (ORDER_ID) REFERENCES ORDERS(ORDER_ID),
FOREIGN KEY (PRODUCT_ID) REFERENCES PRODUCTS(PRODUCT_ID)
);
CREATE TABLE INVOICES (
INVOICE_ID NUMBER PRIMARY KEY,
ORDER_ID NUMBER NOT NULL,
INVOICE_DATE DATE NOT NULL,
TOTAL_AMOUNT NUMBER(10, 2) NOT NULL,
STATUS VARCHAR2(20) DEFAULT 'Issued',
CONSTRAINT fk_order FOREIGN KEY (ORDER_ID) REFERENCES ORDERS(ORDER_ID)
);


SELECT * FROM CUSTOMERS,PRODUCTS,ORDERS,ORDER_ITEMS,INVOICES;


/*First-question:  Retrieving the information of customers
who have ordered more than 500 dollars*/

select c.first_name,c.last_name ,sum(o.total_amount) as total_amount
from CUSTOMERS c
join ORDERS o on c.CUSTOMER_ID = o.ORDER_ID
where o.ORDER_DATE > to_date('1-JAN-2024' , 'DD-MM-YYYY')
GROUP BY c.first_name, c.last_name
HAVING SUM(o.TOTAL_AMOUNT) >500;

/*Second-question: You must write a PL/SQL procedure called
CREATE_ORDER that creates a new order for a specific customer
and inserts the order items into the ORDER_ITEMS table.
This procedure should receive parameters such as the customer ID,
the list of products and their number, and if successful,
create the order and its items.*/


-- Create sequences for ORDER_ITEMS , ORDERS Tables
/*For the ORDERS and ORDER_ITEMS tables,
you need sequences that automatically generate
new unique values for the primary keys.*/
CREATE SEQUENCE ORDERS_SEQ
START WITH 1
INCREMENT BY 1
NOCACHE
NOCYCLE;
CREATE SEQUENCE ORDER_ITEMS_SEQ
START WITH 1
INCREMENT BY 1
NOCACHE
NOCYCLE;



CREATE OR REPLACE PROCEDURE CREATE_ORDER(
P_CUSTOMER_ID IN NUMBER,
P_PRODUCT_ID IN NUMBER,
P_QUANTITY IN NUMBER
) IS
V_UNIT_PRICE NUMBER(10, 2);
V_TOTAL_AMOUNT NUMBER(10, 2);
BEGIN
-- Fetch the product price
SELECT PRICE INTO V_UNIT_PRICE FROM PRODUCTS WHERE PRODUCT_ID = P_PRODUCT_ID;

    -- Calculate the total amount
    V_TOTAL_AMOUNT := V_UNIT_PRICE * P_QUANTITY;

    -- Insert a new order into the ORDERS table
    INSERT INTO ORDERS (ORDER_ID, CUSTOMER_ID, ORDER_DATE, TOTAL_AMOUNT)
    VALUES (ORDERS_SEQ.NEXTVAL, P_CUSTOMER_ID, SYSDATE, V_TOTAL_AMOUNT);

    -- Insert order details into the ORDER_ITEMS table
    INSERT INTO ORDER_ITEMS (ORDER_ITEM_ID, ORDER_ID, PRODUCT_ID, QUANTITY, UNIT_PRICE)
    VALUES (ORDER_ITEMS_SEQ.NEXTVAL, ORDERS_SEQ.CURRVAL, P_PRODUCT_ID, P_QUANTITY, V_UNIT_PRICE);

    -- Update the product stock quantity
    UPDATE PRODUCTS SET STOCK_QUANTITY = STOCK_QUANTITY - P_QUANTITY WHERE PRODUCT_ID = P_PRODUCT_ID;

    COMMIT;
EXCEPTION
WHEN OTHERS THEN
ROLLBACK;
DBMS_OUTPUT.PUT_LINE('Error creating order: ' || SQLERRM);
END;

/*Fourth-question: Write a PL/SQL function to calculate the discount
Write a function that receives the order amount and applies a
discount if certain conditions are met.

Analysis:
The function should accept an input amount and calculate the discount based on it.
IF and RETURN structures are used to apply discount conditions.*/

CREATE OR REPLACE FUNCTION CALCULATE_DISCOUNT (
p_total_amount IN NUMBER
) RETURN NUMBER IS
v_discount NUMBER := 0;
BEGIN
IF p_total_amount >= 1000 THEN
v_discount := p_total_amount * 0.1; -- 10% discount
ELSIF p_total_amount >= 500 THEN
v_discount := p_total_amount * 0.05; -- 5% discount
END IF;

    RETURN v_discount;
END;


    /*the fifth-question : Error handling in PL/SQL
Problem: Write an error handling procedure that, in case of errors,
records the appropriate message to the log and rolls back the transaction.*/
--Create the ERROR_LOG Table
CREATE TABLE ERROR_LOG (
LOG_ID NUMBER PRIMARY KEY, -- Unique identifier for each error
ERROR_MESSAGE VARCHAR2(4000), -- Error message
ERROR_DATE DATE -- Date the error occurred
);

-- Create the ERROR_LOG_SEQ Sequence
CREATE SEQUENCE ERROR_LOG_SEQ
START WITH 1
INCREMENT BY 1
NOCACHE
NOCYCLE;
--Write the LOG_ERROR Procedure
CREATE OR REPLACE PROCEDURE LOG_ERROR(p_error_message IN VARCHAR2) IS
BEGIN
INSERT INTO ERROR_LOG (LOG_ID, ERROR_MESSAGE, ERROR_DATE)
VALUES (ERROR_LOG_SEQ.NEXTVAL, p_error_message, SYSDATE);

    COMMIT;
EXCEPTION
WHEN OTHERS THEN
-- Do nothing if an error occurs while logging the error
NULL;
END;


CREATE OR REPLACE PROCEDURE LOG_ERROR(p_error_message IN VARCHAR2) IS
BEGIN
INSERT INTO ERROR_LOG (LOG_ID, ERROR_MESSAGE, ERROR_DATE)
VALUES (ERROR_LOG_SEQ.NEXTVAL, p_error_message, SYSDATE);

    COMMIT;
END;



--Error handling in the main procedure:

-- Error Handling in the Main Procedure
BEGIN
NULL ;

    -- Perform operations
    -- (Replace this section with your actual operations)
    EXCEPTION
    WHEN OTHERS THEN
        LOG_ERROR(SQLERRM); -- Log the error in the table
ROLLBACK; -- Rollback the transaction
RAISE; -- Re-raise the error
END;


/*the sixth-question :   Optimizing queries
To optimize queries:

Using indexes: Create indexes for frequently
used fields such as CUSTOMER_ID, ORDER_ID.*/
CREATE INDEX idx_orders_customer_id ON ORDERS (CUSTOMER_ID);
CREATE INDEX idx_order_items_order_id ON ORDER_ITEMS (ORDER_ID);

/*Using the EXPLAIN PLAN tool: Use EXPLAIN PLAN to review
how queries are executed and identify the most optimal paths.*/

EXPLAIN PLAN FOR
SELECT C.CUSTOMER_ID, C.FIRST_NAME, C.LAST_NAME, SUM(OI.QUANTITY * OI.UNIT_PRICE) AS TOTAL_AMOUNT
FROM CUSTOMERS C
JOIN ORDERS O ON C.CUSTOMER_ID = O.CUSTOMER_ID
JOIN ORDER_ITEMS OI ON O.ORDER_ID = OI.ORDER_ID
GROUP BY C.CUSTOMER_ID, C.FIRST_NAME, C.LAST_NAME;



