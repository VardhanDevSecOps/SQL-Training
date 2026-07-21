# Day 3 – Complex Data Representation & Analysis Objective

### The objective of this exercise is to explore PostgreSQL's advanced data representation capabilities by storing and querying complex relationships using ARRAY and JSONB data types.

The implementation focuses on:

Aggregating inventor information
Searching within arrays
Comparing patents based on shared inventors
Converting arrays back into normalized rows
Working with structured JSON metadata
Updating JSON fields
Producing API-ready JSON documents
Building hierarchical JSON reports
Optimizing performance using GIN indexes
Evaluating PostgreSQL's support for complex document-style storage

### Database Used

Schema
```
patents
```

### Main Tables
```
patents_training

patent_inventors
```
### Task 1 – Store All Inventors in One Field

#### Purpose

Instead of storing one inventor per row, all inventors belonging to a patent are stored together inside an ARRAY.

This creates a denormalized representation suitable for reporting and read-heavy workloads.

### Dataset Preparation

#### 1 - Create Patent Inventor Table
```sql
CREATE TABLE patents.patent_inventors
(
    publication_number TEXT,
    inventor_name      TEXT,
    inventor_order     SMALLINT,
    PRIMARY KEY (publication_number, inventor_order)
);
```
<img width="775" height="244" alt="1-Patent Inventor Table" src="https://github.com/user-attachments/assets/a09ba1ef-1396-4b69-b1ab-4b6ba3c0f176" />

------------------------------------------------------------------------------------------------------------------------------------------
#### 2 - Populate Patent Inventors

Multiple inventors are randomly selected from the inventor master table and assigned to each patent.
```sql
INSERT INTO patents.patent_inventors
(
    publication_number,
    inventor_name,
    inventor_order
)
SELECT
    p.publication_number,
    x.inventor_name,
    x.rn
FROM patents.patents_training p
CROSS JOIN LATERAL
(
    SELECT
        inventor_name,
        ROW_NUMBER() OVER () AS rn
    FROM
    (
        SELECT inventor_name
        FROM patents.inventor_master
        WHERE inventor_name <> p.inventor_name
        ORDER BY random()
        LIMIT (floor(random()*4)+1)::int
    ) t
) x;
```

<img width="1295" height="654" alt="2-Populate Patent Inventors" src="https://github.com/user-attachments/assets/13d5b7bf-eb30-4c7d-9b4f-f36d1707378f" />

------------------------------------------------------------------------------------------------------------------------------------------
#### Task 1 - Create ARRAY Table

Merge inventor rows into a single array.

```sql
CREATE TABLE patents.patent_inventor_array AS

SELECT
    publication_number,
    ARRAY_AGG(inventor_name ORDER BY inventor_name) AS inventors
FROM patents.patent_inventors
GROUP BY publication_number;
```

<img width="757" height="208" alt="3 - Create ARRAY Table" src="https://github.com/user-attachments/assets/b29d3026-2379-4a80-8913-162e3532b648" />

------------------------------------------------------------------------------------------------------------------------------------------
#### Performance Test Before Index
```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_00467'];
```

<img width="1532" height="425" alt="4-Performance Test Before Index" src="https://github.com/user-attachments/assets/6dc460ed-a582-4ac8-9aa2-0d428cdf6dea" />

------------------------------------------------------------------------------------------------------------------------------------------
### Create GIN Index
```
CREATE INDEX idx_inventors
ON patents.patent_inventor_array
USING GIN(inventors);
```

<img width="532" height="107" alt="5-Create GIN Index" src="https://github.com/user-attachments/assets/3140bccd-29c5-4532-adc3-bd53fd8ba1e2" />

------------------------------------------------------------------------------------------------------------------------------------------
### Performance Test After Index
```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_00467'];
```
<img width="1441" height="393" alt="6-Performance Test After Index" src="https://github.com/user-attachments/assets/48ee8137-4369-4ca9-9149-802b3a34d8fa" />

------------------------------------------------------------------------------------------------------------------------------------------
### Task 2 - Find Patents by One Inventor
```sql
SELECT *
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_00685'];
```
Determines whether the specified inventor exists in the array.

<img width="1065" height="1040" alt="7-Find Patents by One Inventor" src="https://github.com/user-attachments/assets/616c6bce-fb4c-458a-89f6-8e9525c42d9b" />

------------------------------------------------------------------------------------------------------------------------------------------
### Task 3 - Find Patents Matching Any Inventor
```sql
SELECT *
FROM patents.patent_inventor_array
WHERE inventors && ARRAY
[
    'Inventor_06130',
    'Inventor_05418',
    'Inventor_08207'
];
```
The && operator returns rows when both arrays contain one or more common elements.

<img width="1030" height="1040" alt="8-Find Patents Matching Any Inventor" src="https://github.com/user-attachments/assets/8e65c42a-e7fe-4212-a3f0-46f01224ca6e" />

------------------------------------------------------------------------------------------------------------------------------------------
### Task 4 - Find Patents Sharing Inventors

```sql
SELECT
    p1.publication_number,
    p2.publication_number
FROM patents.patent_inventor_array p1
JOIN patents.patent_inventor_array p2
ON p1.inventors && p2.inventors
AND p1.publication_number < p2.publication_number
LIMIT 20;
WITH sample AS
(
SELECT *
FROM patents.patent_inventor_array
LIMIT 1000
)

SELECT *
FROM sample p1
JOIN sample p2
ON p1.inventors && p2.inventors
AND p1.publication_number < p2.publication_number;
```
<img width="630" height="1026" alt="9-Find Patents Sharing Inventors 1" src="https://github.com/user-attachments/assets/b36c8667-8463-440b-853e-6737e5c560aa" />

<img width="1676" height="1033" alt="9-Find Patents Sharing Inventors 2" src="https://github.com/user-attachments/assets/55e01926-cd40-4591-9fe3-cd2596604b93" />

------------------------------------------------------------------------------------------------------------------------------------------
### Task 5 - Convert ARRAY Back to Rows
```sql
SELECT
    publication_number,
    UNNEST(inventors) AS inventor_name
FROM patents.patent_inventor_array

EXCEPT

SELECT
    publication_number,
    inventor_name
FROM patents.patent_inventors;
```
UNNEST transforms each element of an ARRAY into a separate row. A result with no rows indicates that both datasets are identical.

<img width="756" height="371" alt="10-Convert ARRAY Back to Rows" src="https://github.com/user-attachments/assets/9c0cab02-9b97-40cb-be4d-409db29fa57c" />

------------------------------------------------------------------------------------------------------------------------------------------
### Task 6 - Create JSONB Metadata
```sql
CREATE TABLE patents.patent_metadata AS

SELECT
    publication_number,
    jsonb_build_object
    (
        'country','US',
        'category','Mechanical',
        'status','Granted',
        'technology','AI',
        'filing_date','2023-01-01'
    ) AS metadata

FROM patents_training;
```
<img width="795" height="384" alt="11-Create JSONB Metadata" src="https://github.com/user-attachments/assets/52c62ae1-c1de-4016-8a44-e3cd5c712daa" />

------------------------------------------------------------------------------------------------------------------------------------------
### Performance Test Before JSONB Index
```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_metadata
WHERE metadata->>'technology'='AI';
```
<img width="1603" height="400" alt="12-Performance Test Before JSONB Index" src="https://github.com/user-attachments/assets/4f41f10c-5abc-4db5-819c-4ee01e9efb01" />

------------------------------------------------------------------------------------------------------------------------------------------
### Create JSONB GIN Index
```sql
CREATE INDEX idx_metadata
ON patents.patent_metadata
USING GIN(metadata);
```
<img width="742" height="104" alt="13-Create JSONB GIN Index" src="https://github.com/user-attachments/assets/9a8757d3-37e7-4e6a-86bb-9c701c2d9c4c" />

------------------------------------------------------------------------------------------------------------------------------------------
### Performance Test After JSONB Index
```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_metadata
WHERE metadata->>'technology'='AI';
```
<img width="1531" height="391" alt="14-Performance Test After JSONB Index" src="https://github.com/user-attachments/assets/3fa74c6b-6a59-4f7c-9995-dfb3f5d29313" />

------------------------------------------------------------------------------------------------------------------------------------------
### Task 7 - Query JSONB

### Country
```sql
SELECT COUNT(*)
FROM patents.patent_metadata
WHERE metadata->>'country' = 'US';
```
```sql
SELECT *
FROM patents.patent_metadata
WHERE metadata->>'country'='US'
LIMIT 20;
```
<img width="1579" height="890" alt="15-Query JSONB" src="https://github.com/user-attachments/assets/25aefb1b-454e-458d-aeb3-c5b53ce98fff" />

------------------------------------------------------------------------------------------------------------------------------------------
### Technology

```sql
SELECT *
FROM patents.patent_metadata
WHERE metadata->>'technology'='AI';
```
<img width="1587" height="1037" alt="15-Query JSONB (Technology)" src="https://github.com/user-attachments/assets/d7343b6e-e5ce-4c1f-989b-7d809a7292a2" />

------------------------------------------------------------------------------------------------------------------------------------------
### Status
```sql
SELECT *
FROM patents.patent_metadata
WHERE metadata->>'status'='Granted';
```
<img width="1639" height="1034" alt="15-Query JSONB (Status)" src="https://github.com/user-attachments/assets/4af47bf0-9871-488a-b963-a1870b1566e8" />

------------------------------------------------------------------------------------------------------------------------------------------
### Task 8 - Update JSONB
```sql
SELECT publication_number
FROM patents.patents_training
LIMIT 10;
UPDATE patents.patent_metadata
SET metadata =
jsonb_set
(
    metadata,
    '{status}',
    '"Expired"'
)
WHERE publication_number='US0000000001';
```
<img width="500" height="681" alt="16-Update JSONB" src="https://github.com/user-attachments/assets/8e7ec50e-55c0-49a9-8782-573565aa9c1d" />

------------------------------------------------------------------------------------------------------------------------------------------
### Task 9 - Aggregate Inventors as JSON
```sql
SELECT
    p.publication_number,
    p.title,
    jsonb_agg(pi.inventor_name)
FROM patents_training p
JOIN patents.patent_inventors pi
USING (publication_number)
GROUP BY
    p.publication_number,
    p.title
LIMIT 20;
```
<img width="1680" height="1007" alt="17-Aggregate Inventors as JSON" src="https://github.com/user-attachments/assets/0a51fca9-c885-49c1-8f90-29e9eef6f107" />

------------------------------------------------------------------------------------------------------------------------------------------
### Task 10 - Build Nested JSON
```
\x
```

```sql
WITH sample AS
(
    SELECT *
    FROM patents_training
    WHERE EXTRACT(YEAR FROM publication_date)=2023
    LIMIT 100
)

SELECT jsonb_build_object
(
    'year',
    EXTRACT(YEAR FROM publication_date),

    'patents',

    jsonb_agg
    (
        jsonb_build_object
        (
            'publication_number',
            publication_number,

            'title',
            title,

            'inventors',

            (
                SELECT jsonb_agg(inventor_name)
                FROM patents.patent_inventors pi
                WHERE pi.publication_number=s.publication_number
            )
        )
    )
)

AS patent_hierarchy

FROM sample s

GROUP BY
EXTRACT(YEAR FROM publication_date);
```
<img width="1680" height="1001" alt="18-Build Nested JSON" src="https://github.com/user-attachments/assets/1ededc11-323f-4c13-8868-c28c78a13723" />

------------------------------------------------------------------------------------------------------------------------------------------
### Task 11 - Performance Comparison

Execute EXPLAIN ANALYZE both before and after creating the indexes.

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_08709'];
```
<img width="1480" height="432" alt="19-Performance Comparison before" src="https://github.com/user-attachments/assets/065a2094-f412-4dfe-894e-fa981cdcf762" />

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_metadata
WHERE metadata->>'technology'='AI';
```
<img width="1556" height="438" alt="19-Performance Comparison after" src="https://github.com/user-attachments/assets/b5a3f0cd-cd5b-489f-806c-dd1c976d185d" />

------------------------------------------------------------------------------------------------------------------------------------------
### Task 12 - ARRAY Functions

**1. Find Inventor Position**
```sql
SELECT
    publication_number,
    inventors,
    array_position(inventors,'Inventor_08011') AS inventor_position
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_08011'];
```
<img width="1338" height="1040" alt="20 - ARRAY Functions_Inventor Position" src="https://github.com/user-attachments/assets/966a7e8f-ae9e-42b5-9022-9007d2230f46" />

------------------------------------------------------------------------------------------------------------------------------------------
**2. Count Inventors**
```sql
SELECT
    publication_number,
    inventors,
    cardinality(inventors) AS inventor_count
FROM patents.patent_inventor_array
ORDER BY inventor_count DESC
LIMIT 10;

```

<img width="1268" height="999" alt="20 - ARRAY Functions_Count Inventors" src="https://github.com/user-attachments/assets/154639d2-a3ac-4e95-bc28-ef7a8ae607ac" />

------------------------------------------------------------------------------------------------------------------------------------------
### 3. Append Inventor
```sql
SELECT
    publication_number,
    inventors,
    array_append(inventors,'Inventor_99999') AS updated_inventors
FROM patents.patent_inventor_array
WHERE publication_number='US0000001133';
```
<img width="1013" height="244" alt="20 - ARRAY Functions_Append Inventor" src="https://github.com/user-attachments/assets/808eee5d-cd8b-40fa-8c3a-06784fefc153" />

------------------------------------------------------------------------------------------------------------------------------------------
### 4. Remove Inventor
```sql
SELECT
    publication_number,
    inventors,
    array_remove(inventors,'Inventor_06431') AS updated_inventors
FROM patents.patent_inventor_array
WHERE publication_number='US0000001136';
```
<img width="742" height="242" alt="20 - ARRAY Functions_Remove Inventor" src="https://github.com/user-attachments/assets/46043e5a-558e-42cf-8426-75c1abc53a16" />

------------------------------------------------------------------------------------------------------------------------------------------
### 5. Slice Array
```sql
SELECT
    publication_number,
    inventors[1:2] AS first_two_inventors
FROM patents.patent_inventor_array
WHERE cardinality(inventors)>=2
LIMIT 10;

```
<img width="732" height="478" alt="20 - ARRAY Functions_Slice Array" src="https://github.com/user-attachments/assets/18d6262c-b442-4858-b4a3-18a09386438a" />

------------------------------------------------------------------------------------------------------------------------------------------
### 6. Check Whether an Inventor Exists
```sql
SELECT
    publication_number,
    inventors
FROM patents.patent_inventor_array
WHERE 'Inventor_09915'=ANY(inventors);
```
<img width="1127" height="1042" alt="20 - ARRAY Functions_Check Whether an Inventor Exists" src="https://github.com/user-attachments/assets/ca175c3c-f7b6-4f61-bf62-45c1c6f5d1dc" />

------------------------------------------------------------------------------------------------------------------------------------------
### 7. Check whether a key exists
 
SELECT *
FROM patent_metadata
WHERE metadata ? 'technology';

<img width="1611" height="1034" alt="Screenshot 2026-07-21 at 9 40 30 PM" src="https://github.com/user-attachments/assets/5d6ed319-9473-412e-a9e6-ff9da3f820ef" />

------------------------------------------------------------------------------------------------------------------------------------------
### 8. Check multiple keys
 
SELECT *
FROM patent_metadata
WHERE metadata ?& ARRAY['country','technology'];

<img width="1358" height="948" alt="Screenshot 2026-07-21 at 9 41 18 PM" src="https://github.com/user-attachments/assets/9636df48-8673-4582-b192-303526908139" />

------------------------------------------------------------------------------------------------------------------------------------------ 
### 9. Pretty print JSON
 
SELECT jsonb_pretty(metadata)
FROM patent_metadata
LIMIT 5;

<img width="664" height="924" alt="Screenshot 2026-07-21 at 9 42 14 PM" src="https://github.com/user-attachments/assets/1d79c2ab-3d9a-4d43-9ab6-ad8ceb65debe" />




