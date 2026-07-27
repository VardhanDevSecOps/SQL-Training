
# Patent Citation Hierarchy Analysis

### Objective:

This project focuses on implementing and optimizing patent citation hierarchy analysis in PostgreSQL. It leverages recursive queries, user-defined functions, views, and materialized views to process and evaluate an existing dataset of 1 million patent records.

### Task 1: Create Patent Citation Table

If you want it phrased like a GitHub issue/task in bullet points:

* Create a new table `patent_citations`.
* Add the following columns:

  * `citing_publication_number`
  * `cited_publication_number`
* Define an appropriate **primary key**.
* Add **foreign key constraints** where applicable.
* Create appropriate **indexes** to improve query performance.

```
CREATE SCHEMA IF NOT EXISTS citation;

CREATE TABLE citation.patent_training
(LIKE patents.patents_training INCLUDING ALL);

INSERT INTO citation.patent_training
SELECT *
FROM patents.patents_training;
 
```
```
CREATE TABLE citation.patent_citations (
  citing_publication_number TEXT NOT NULL,
  cited_publication_number TEXT NOT NULL,
  PRIMARY KEY (
    citing_publication_number, cited_publication_number
  ),
  FOREIGN KEY (citing_publication_number) REFERENCES patents.patents_training(publication_number),
  FOREIGN KEY (cited_publication_number) REFERENCES patents.patents_training(publication_number)
);
```

<img width="1175" height="225" alt="image" src="https://github.com/user-attachments/assets/2f9f7b43-bbb0-41ba-b7ad-b2f490f6e167" />

------------------------------------------------------------------------------------------------------------------------------------------
### Insert the data

```
WITH patent_sequence AS
(
    SELECT
        publication_number,
        publication_date,
        ROW_NUMBER() OVER (
            ORDER BY publication_date, publication_number
        ) AS seq_no
    FROM citation.patent_training
)
INSERT INTO citation.patent_citations
(
    citing_publication_number,
    cited_publication_number
)
SELECT
    p.publication_number,
    c.publication_number
FROM patent_sequence p
JOIN patent_sequence c
    ON c.seq_no < p.seq_no
   AND c.seq_no >= GREATEST(1, p.seq_no - 3)
WHERE p.seq_no <= 10000;

```

<img width="728" height="538" alt="Screenshot 2026-07-27 at 3 48 22 PM" src="https://github.com/user-attachments/assets/60bc01fd-9fd0-4269-af86-eacfab402c83" />

------------------------------------------------------------------------------------------------------------------------------------------
### Create Indexs

• Recursive queries repeatedly search both columns, while indexes significantly improve lookup performance.

```
CREATE INDEX idx_citing
ON citation.patent_citations(citing_publication_number);

CREATE INDEX idx_cited
ON citation.patent_citations(cited_publication_number);
```
#### with-out Index

<img width="1089" height="88" alt="image" src="https://github.com/user-attachments/assets/bd76c787-da77-440a-8864-1132cc925523" />

#### with Index

<img width="1045" height="168" alt="image" src="https://github.com/user-attachments/assets/cbfc197a-ebf5-49cd-8740-ed4d9fe05743" />

<img width="1128" height="228" alt="image" src="https://github.com/user-attachments/assets/119f0e4f-7ec2-4eb6-ac99-fc1d6d3cab35" />

------------------------------------------------------------------------------------------------------------------------------------------
### Step 2 - Generate Citation Data

To prevent invalid future citations, sort patents chronologically by publication date, assign each a row number, and permit citations only to patents with a smaller row number, ensuring every citation refers to an older patent.

### Create Ordered Patent List
```
CREATE TEMP TABLE ordered_patents AS
SELECT
    publication_number,
    publication_date,
    ROW_NUMBER() OVER (
        ORDER BY publication_date,
                 publication_number
    ) AS rn
FROM patents.patents_training;
```
<img width="719" height="226" alt="image" src="https://github.com/user-attachments/assets/b001bb23-485f-474d-bcca-77aa72711daf" />

#### This query creates a temporary, ordered copy of your patent data and assigns each patent a sequential rn number, which you can use to easily create citation relationships between newer patents and older patents.

------------------------------------------------------------------------------------------------------------------------------------------
#### Create Index
```
CREATE INDEX idx_ordered_patents_rn
ON ordered_patents(rn);
```
<img width="1060" height="199" alt="image" src="https://github.com/user-attachments/assets/e434b7aa-c514-467f-99fc-50c1f76d9d15" />

------------------------------------------------------------------------------------------------------------------------------------------
### Generate Citation Relationships

For every patent that has at least one older patent, randomly choose 1–3 patents from approximately the previous 1,000 patents, and create citation relationships between the current patent and those older patents.

### Why CROSS JOIN LATERAL?

LATERAL lets each row of `p` independently select 1–5 random older patents and insert their citations, rather than reusing the same patents for every row.
```
INSERT INTO patent_citations
(
    citing_publication_number,
    cited_publication_number
)
SELECT
    p.publication_number,
    c.publication_number
FROM ordered_patents p
CROSS JOIN LATERAL
(
    SELECT DISTINCT
        (
            GREATEST(1, p.rn - 1000) +
            floor(random() * LEAST(1000, p.rn - 1))
        )::bigint AS random_rn
    FROM generate_series(1,10)
    WHERE p.rn > 1
    LIMIT (floor(random() * 3) + 1)::int
) r
JOIN ordered_patents c
  ON c.rn = r.random_rn
WHERE p.rn > 1;

```
------------------------------------------------------------------------------------------------------------------------------------------
### Step 3 - Verify Citation Data

<img width="635" height="823" alt="image" src="https://github.com/user-attachments/assets/8634f9cc-3886-4154-9c54-81f0a593e504" />

