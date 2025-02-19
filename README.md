# MySQL Stored Procedures with Django Integration

## Overview
This document provides a complete guide on how to create, optimize, and use stored procedures in MySQL. Additionally, it covers best practices for indexing and integrating stored procedures with Django.

---

## 📌 What is a Stored Procedure?
A stored procedure is a set of SQL statements stored in the database that can be executed with a single call. Stored procedures improve performance, security, and maintainability.

### ✅ Benefits of Stored Procedures:
- **Better Performance**: Reduces network latency by executing SQL logic on the database server.
- **Code Reusability**: Avoids repetitive queries in the application.
- **Security**: Limits access by granting execution permissions.
- **Encapsulation**: Centralizes database logic.

---

## 📌 Creating a Stored Procedure

### 🔹 **Basic Syntax**
```sql
DELIMITER $$

CREATE PROCEDURE procedure_name(IN param1 DataType, OUT param2 DataType)
BEGIN
    -- SQL Statements
    SELECT COUNT(*) INTO param2 FROM table_name WHERE column_name = param1;
END $$

DELIMITER ;
```

### 🔹 **Example: Get All Users**
```sql
DELIMITER $$

CREATE PROCEDURE get_all_users()
BEGIN
    SELECT * FROM users;
END $$

DELIMITER ;
```
📌 **Execute the Procedure:**
```sql
CALL get_all_users();
```

### 🔹 **Example: Get User by ID**
```sql
DELIMITER $$

CREATE PROCEDURE get_user_by_id(IN user_id INT)
BEGIN
    SELECT * FROM users WHERE id = user_id;
END $$

DELIMITER ;
```
📌 **Execute:**
```sql
CALL get_user_by_id(5);
```

### 🔹 **Example: Get User Count (Using OUT Parameter)**
```sql
DELIMITER $$

CREATE PROCEDURE get_user_count(OUT user_count INT)
BEGIN
    SELECT COUNT(*) INTO user_count FROM users;
END $$

DELIMITER ;
```
📌 **Execute & Retrieve Output:**
```sql
CALL get_user_count(@count);
SELECT @count;
```

---

## 📌 Pagination with Search (Advanced Stored Procedure)
```sql
DELIMITER $$

CREATE PROCEDURE get_users_paginated(
    IN search_text VARCHAR(100),
    IN page_limit INT,
    IN page_offset INT
)
BEGIN
    DECLARE adjusted_limit INT;
    SET adjusted_limit = page_limit + 1;

    SELECT id, username, email,
        IF(row_number > page_limit, TRUE, FALSE) AS has_next
    FROM (
        SELECT id, username, email,
               ROW_NUMBER() OVER (ORDER BY username ASC) AS row_number
        FROM users
        WHERE username LIKE CONCAT(search_text, '%')
        LIMIT adjusted_limit OFFSET page_offset
    ) AS subquery;
END $$

DELIMITER ;
```
📌 **Execute:**
```sql
CALL get_users_paginated('john', 10, 0);
```

---

## 📌 Indexing for Performance Optimization
Indexes significantly improve query performance by reducing the amount of scanned data.

### 🔹 **Create Indexes on Frequently Queried Columns**
```sql
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
```

### 🔹 **Primary Key Index (Auto-Created)**
```sql
ALTER TABLE users ADD PRIMARY KEY (id);
```

### 🔹 **Unique Index (Prevent Duplicates)**
```sql
CREATE UNIQUE INDEX idx_unique_email ON users(email);
```

### 🔹 **Composite Index (Multi-Column Index)**
```sql
CREATE INDEX idx_users_name_email ON users(username, email);
```

---

## 📌 Using Stored Procedures in Django
Django provides `connection.cursor()` to execute raw SQL queries, including stored procedures.

### 🔹 **Calling a Stored Procedure in Django**
```python
from django.db import connection

def get_users(search_text, page_limit, page_offset):
    with connection.cursor() as cursor:
        cursor.callproc('get_users_paginated', [search_text, page_limit, page_offset])
        results = cursor.fetchall()
    
    return [{'id': row[0], 'username': row[1], 'email': row[2], 'has_next': bool(row[3])} for row in results]
```

📌 **Usage in Django View**
```python
data = get_users('john', 10, 0)
print(data)
```

---

## 📌 Modifying or Deleting Stored Procedures

### 🔹 **Modify a Procedure**
```sql
DROP PROCEDURE IF EXISTS get_all_users;

DELIMITER $$

CREATE PROCEDURE get_all_users()
BEGIN
    SELECT id, username, email FROM users;
END $$

DELIMITER ;
```

### 🔹 **Delete a Procedure**
```sql
DROP PROCEDURE IF EXISTS get_all_users;
```

---

## 📌 Error Handling in Stored Procedures
```sql
DELIMITER $$

CREATE PROCEDURE insert_user(IN user_name VARCHAR(100), IN user_email VARCHAR(100))
BEGIN
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        -- Handle error
        ROLLBACK;
    END;

    START TRANSACTION;
    
    INSERT INTO users (username, email) VALUES (user_name, user_email);
    
    COMMIT;
END $$

DELIMITER ;
```
📌 **Call the Procedure:**
```sql
CALL insert_user('JohnDoe', 'johndoe@example.com');
```

---

## 📌 Summary
| Feature | Example |
|---------|---------|
| **Basic Procedure** | `CREATE PROCEDURE get_all_users()` |
| **Input Parameter** | `IN user_id INT` |
| **Output Parameter** | `OUT user_count INT` |
| **Pagination & Search** | `CALL get_users_paginated('john', 10, 0);` |
| **Error Handling** | `DECLARE CONTINUE HANDLER FOR SQLEXCEPTION` |
| **Indexing** | `CREATE INDEX idx_users_email ON users(email);` |
| **Django Integration** | `cursor.callproc('procedure_name', [params])` |

---

## 📌 Final Thoughts
✅ Stored procedures **enhance performance**, **simplify application logic**, and **reduce database load**.
✅ Use **indexes** to speed up queries, especially in large datasets.
✅ Always **test stored procedures** before deploying to production.

🚀 **Happy Coding!** 🚀


```sql
ALTER TABLE pes_oempartnumber ADD FULLTEXT INDEX ft_partnumber (PartNumber);
CREATE INDEX idx_created_at ON pes_oempartnumber(created_at);
```
```sql
USE devpatriosoft_db;

DELIMITER $$

CREATE PROCEDURE `get_oem_part_numbers_paginator`(
    IN search_text VARCHAR(256),
    IN page_limit INT,
    IN last_partnumber VARCHAR(256)  -- Used for keyset pagination
)
BEGIN
    DECLARE adjusted_limit INT;
    SET adjusted_limit = page_limit + 1;

    -- Direct query handling both exact and partial matches
    IF search_text IS NULL OR search_text = '' THEN
        -- No search, fetch all records using keyset pagination
        SELECT 
            id,
            partnumber,
            IF(row_count > page_limit, TRUE, FALSE) AS has_next
        FROM (
            SELECT 
                id,
                partnumber,
                ROW_NUMBER() OVER () as row_count
            FROM devpatriosoft_db.PES_OEMPartNumber
            WHERE (last_partnumber IS NULL OR partnumber > last_partnumber)
            ORDER BY partnumber ASC
            LIMIT adjusted_limit
        ) t;
    ELSE
        -- Search with both exact and partial matches using keyset pagination
        SELECT 
            id,
            partnumber,
            IF(row_count > page_limit, TRUE, FALSE) AS has_next
        FROM (
            SELECT 
                id,
                partnumber,
                ROW_NUMBER() OVER () as row_count
            FROM (
                -- First get exact matches
                SELECT id, partnumber, created_at, 1 AS match_type
                FROM devpatriosoft_db.PES_OEMPartNumber
                WHERE partnumber = search_text

                UNION ALL

                -- Then get partial matches (StartsWith)
                SELECT id, partnumber, created_at, 2 AS match_type
                FROM devpatriosoft_db.PES_OEMPartNumber
                WHERE partnumber LIKE CONCAT(search_text, '%')
                AND partnumber != search_text
            ) combined
            WHERE (last_partnumber IS NULL OR partnumber > last_partnumber)
            ORDER BY match_type, partnumber ASC
            LIMIT adjusted_limit
        ) t;
    END IF;
END
$$ DELIMITER ;

```

```python

def stored_procedures_dropdown(stored_procedures_fun: str,
                                search_text : str, 
                                page_limit : int, 
                                last_text : int)->Tuple[List[Dict],bool,str]:
    with connection.cursor() as cursor:
        cursor.callproc(stored_procedures_fun, [search_text, page_limit, last_text])
        results = cursor.fetchall()  # Fetch all results

    if not results:
        return [], False, last_text  # Handle empty results case

    queryset = [{'id': row[0], 'name': row[1]} for row in results]
    has_next = bool(results[-1][2])  # Extract has_next from the last row
    last_text = results[-1][1]  # Extract last_text from the last row

    return queryset, has_next, last_text

ef get_items_interchanges(request):
    dropdown = request.GET.get('dropdown')
    search = request.GET.get('search')
    if dropdown:
        queryset = []
        next_page_url = None
        last_text = request.GET.get('last_text',None)
        queryset , has_next, last_partnumber = stored_procedures_dropdown('get_oem_part_numbers_paginator',
            search, 10, last_text)
        if has_next:
            next_page_url = reverse('app:name') + f"?dropdown=true&last_text={last_text}"
        return JsonResponse({"data": queryset, "next_page_url": next_page_url}, status=200)
```

