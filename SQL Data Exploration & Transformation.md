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


------------------------------------------------------------------------------
