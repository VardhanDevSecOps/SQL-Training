
Step 1: Check your table structure

First see the available columns.
```
\d patents.all_patents
```
If you don't know the inventor column name:
```
SELECT *
FROM patents.all_patents
LIMIT 5;
```

Create a Table name with patents.patents_training
```
CREATE TABLE patents.patents_training
(
    publication_number TEXT PRIMARY KEY,
    inventor_name      TEXT,
    publication_date   DATE,
    title              TEXT,
    abstract           TEXT
);
```
<img width="657" height="272" alt="Table creation" src="https://github.com/user-attachments/assets/2ec1c5b3-571b-4663-9cfe-adc3c000822f" />

------------------------------------------------------------------------------
