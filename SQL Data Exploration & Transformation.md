# Day 2 – SQL Data Exploration & Transformation Challenge 

**Check your table structure**

1. First see the available columns.
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
**Screenshot**

<img width="657" height="272" alt="Table creation" src="https://github.com/user-attachments/assets/2ec1c5b3-571b-4663-9cfe-adc3c000822f" />

------------------------------------------------------------------------------------------------------------------------------------------------------------

This SQL script generates and inserts **1,000,000 synthetic patent records** into the `patents.patents_training` table.

**What it does:**

* Inserts data into these columns:

  * `publication_number`
  * `inventor_name`
  * `publication_date`
  * `title`
  * `abstract`
* Uses `generate_series(1,1000000)` to create **1 million rows**.
* Generates random values for:

  * **Publication number:** `US0000000001` to `US0001000000`
  * **Inventor name:** Random names like `Inventor_01234`
  * **Publication date:** Random date between **2010-01-01** and roughly **2025-12-31**
  * **Title:** Randomly combines words from predefined arrays (technology, action, domain, architecture, etc.) to create patent-like titles.
  * **Abstract:** Creates a short patent-style description using the same random word arrays.

**Key techniques used:**

* `generate_series()` → Generates 1 million rows.
* `random()` → Selects random words and dates.
* `ARRAY[]` → Stores word lists for random selection.
* `concat_ws()` → Concatenates words into readable titles and abstracts.
* `CROSS JOIN` → Makes the arrays available to every generated row.

**Purpose:**

* Populate a database with **large-scale dummy patent data** for testing, benchmarking, search indexing, machine learning, or performance testing.

**Script**

```
INSERT INTO patents.patents_training
(
    publication_number,
    inventor_name,
    publication_date,
    title,
    abstract
)
SELECT
    'US' || LPAD(gs::text, 10, '0'),

    'Inventor_' || LPAD((1 + floor(random() * 10000))::int::text, 5, '0'),

    DATE '2010-01-01' + (random() * 5843)::int,

    concat_ws(
        ' ',
        adjectives[(random()*19+1)::int],
        technologies[(random()*19+1)::int],
        actions[(random()*19+1)::int],
        'for',
        domains[(random()*19+1)::int],
        'using',
        technologies[(random()*19+1)::int],
        connectors[(random()*19+1)::int],
        features[(random()*19+1)::int],
        architectures[(random()*19+1)::int],
        'with',
        benefits[(random()*19+1)::int],
        qualities[(random()*19+1)::int],
        'performance',
        'and',
        security[(random()*19+1)::int],
        'based',
        'system',
        'architecture',
        versions[(random()*19+1)::int]
    ),

    concat_ws(
        ' ',
        'This invention provides',
        adjectives[(random()*19+1)::int],
        technologies[(random()*19+1)::int],
        'for',
        domains[(random()*19+1)::int],
        'using',
        architectures[(random()*19+1)::int],
        'to improve',
        benefits[(random()*19+1)::int],
        'performance, scalability, availability and security.'
    )

FROM generate_series(1,1000000) gs

CROSS JOIN
(
SELECT

ARRAY[
'Machine','Cloud','Distributed','Neural','Artificial',
'Quantum','Blockchain','Battery','Electric','Medical',
'Image','Wireless','Sensor','Edge','Cyber',
'Speech','Network','Autonomous','Virtual','Digital'
] technologies,

ARRAY[
'System','Method','Framework','Architecture','Platform',
'Engine','Application','Solution','Algorithm','Model',
'Protocol','Mechanism','Workflow','Service','Module',
'Controller','Interface','Process','Device','Analytics'
] actions,

ARRAY[
'Healthcare','Manufacturing','Finance','Retail',
'Agriculture','Education','Automotive','IoT',
'Cloud','Robotics','Satellite','Security',
'Telecommunication','Energy','Payments',
'Logistics','SupplyChain','Medical','SmartCity','Defence'
] domains,

ARRAY[
'advanced','intelligent','scalable','distributed',
'secure','adaptive','dynamic','predictive',
'robust','optimized','highspeed','cloudnative',
'autonomous','wireless','efficient','reliable',
'virtualized','parallel','automated','flexible'
] adjectives,

ARRAY[
'through','via','leveraging','utilizing',
'combining','integrating','supporting','enabling',
'optimizing','processing','executing','monitoring',
'controlling','managing','analyzing','predicting',
'detecting','improving','accelerating','transforming'
] connectors,

ARRAY[
'realtime','streaming','analytics','monitoring',
'optimization','automation','processing','storage',
'routing','prediction','classification','compression',
'authentication','authorization','encryption','synchronization',
'visualization','coordination','diagnostics','recovery'
] features,

ARRAY[
'framework','architecture','pipeline','database',
'cluster','engine','platform','gateway',
'controller','scheduler','processor','interface',
'network','application','repository','cache',
'queue','service','module','workflow'
] architectures,

ARRAY[
'high','better','enhanced','maximum',
'greater','improved','efficient','fast',
'optimized','reliable','stable','consistent',
'predictable','secure','scalable','flexible',
'faulttolerant','continuous','intelligent','dynamic'
] benefits,

ARRAY[
'overall','operational','computational','system',
'business','enterprise','network','application',
'resource','processing','storage','service',
'cloud','database','transaction','runtime',
'deployment','production','execution','workflow'
] qualities,

ARRAY[
'protection','encryption','authentication','validation',
'authorization','monitoring','verification','integrity',
'privacy','compliance','governance','availability',
'confidentiality','resilience','isolation','backup',
'auditing','detection','prevention','recovery'
] security,

ARRAY[
'v1','v2','v3','generation',
'nextgeneration','release','edition','model',
'series','prototype','variant','revision',
'phase1','phase2','phase3','alpha',
'beta','gamma','enterprise','premium'
] versions

) x;
```
**Screenshot**

<img width="1443" height="907" alt="Screenshot 2026-07-17 at 4 34 26 PM" src="https://github.com/user-attachments/assets/91f0348c-261a-47ac-b2cc-a9e464bca60f" />

------------------------------------------------------------------------------------------------------------------------------------------------------------

**2 – Create Inventor Master** 

Purpose

Instead of storing duplicate inventor names millions of times, create a master table containing only unique inventors.

**Create Table**

```
CREATE TABLE patents.inventor_master
(
    inventor_id BIGSERIAL PRIMARY KEY,
    inventor_name TEXT UNIQUE
);
```

**Screenshot**

<img width="586" height="245" alt="Inventor Master" src="https://github.com/user-attachments/assets/0a1e93d0-c3c1-4b0c-8698-453741e7c33b" />
------------------------------------------------------------------------------------------------------------------------------------------------------

**3. Insert Unique Inventors**

```
INSERT INTO patents.inventor_master(inventor_name)
SELECT DISTINCT TRIM(inventor_name)
FROM patents.patents_training
WHERE inventor_name IS NOT NULL
AND TRIM(inventor_name) <> '';
```
**Screenshot**

<img width="617" height="190" alt="Insert Unique Inventors" src="https://github.com/user-attachments/assets/52b5d806-865d-4bd3-a12a-20acc1b39e03" />

**Verify**

<img width="635" height="205" alt="Verify" src="https://github.com/user-attachments/assets/837df9ce-07b4-4f10-aca2-f7e033911ca3" />

------------------------------------------------------------------------------------------------------------------------------------------------------

**Create Index**

Although UNIQUE already creates an index, you can create another only if searching differently.

```
CREATE INDEX idx_inventor_name
ON patents.inventor_master(inventor_name);
```
Index created

<img width="493" height="122" alt="Index Created" src="https://github.com/user-attachments/assets/944b5149-3356-4064-b8a0-ab5fbe9bcda1" />
------------------------------------------------------------------------------------------------------------------------------------------------------

**4 – Patent Title Analysis**
```
Create table
CREATE TABLE patents.title_word_frequency
(
    word TEXT PRIMARY KEY,
    frequency BIGINT,
    rank INT
);
```
**Table created**

<img width="653" height="221" alt="Table created" src="https://github.com/user-attachments/assets/8f97b423-021e-4096-ba14-2111c66d315d" />

**Extract words**

**PostgreSQL regular expression:**
```
regexp_split_to_table(lower(title), '[^a-z0-9]+')
```
splits using

• spaces
• commas
• hyphens
• slashes
• brackets
• punctuation
• Insert Top 100 words

**Insert Top 100 words**
```
INSERT INTO patents.title_word_frequency(word, frequency, rank)

WITH words AS
(
    SELECT
        regexp_split_to_table(lower(title), '[^a-z0-9]+') AS word
    FROM patents.patents_training
),

filtered AS
(
    SELECT word
    FROM words
    WHERE length(word) > 3
),

freq AS
(
    SELECT
        word,
        COUNT(*) AS frequency
    FROM filtered
    GROUP BY word
)

SELECT
    word,
    frequency,
    DENSE_RANK() OVER
    (
        ORDER BY frequency DESC
    ) AS rank
FROM freq
ORDER BY frequency DESC
LIMIT 100;
```

View Top Words
```
SELECT *
FROM patents.title_word_frequency
ORDER BY rank
LIMIT 50;
```

**Concepts Covered**

regexp_split_to_table()
Regular Expressions
Common Table Expressions (CTEs)
Aggregation
Window Functions
DENSE_RANK()

<img width="675" height="1039" alt="View Top words" src="https://github.com/user-attachments/assets/5cfdedbc-86e7-462a-9534-86be5192ea5d" />
------------------------------------------------------------------------------------------------------------------------------------------------------

**5 - Patent Coverage Analysis**

Find patents whose titles do not contain any of the Top 100 frequent words.
```
SELECT *

FROM patents.patents_training p

WHERE NOT EXISTS
(
    SELECT 1

    FROM patents.title_word_frequency t

    WHERE lower(p.title) ~

    ('(^|[^a-z0-9])' ||

     t.word ||

     '([^a-z0-9]|$)')
);
```

```
Explanation

For every patent

↓

Check Top100 words

↓

If one matches

↓

Reject

↓

Otherwise

↓

Return patent
```

Alternative (more accurate)

Split title into words and compare word-by-word instead of LIKE.

**Screenshot**

<img width="870" height="517" alt="Patent Coverage Analysis" src="https://github.com/user-attachments/assets/e6f3def9-b50a-4ddb-9b77-00dd0286737a" />

------------------------------------------------------------------------------------------------------------------------------------------------------

**6 - Patent Trend Analysis**

Analyze publication trends over the years.

Annual Patent Count
```
SELECT

    EXTRACT(YEAR FROM publication_date)::INT AS year,

    COUNT(*) AS patent_count

FROM patents.patents_training

GROUP BY year

ORDER BY patent_count DESC

LIMIT 10;
```
<img width="535" height="653" alt=" Patent Trend Analysi" src="https://github.com/user-attachments/assets/57ca3ca6-8ce2-4a86-9db5-c3e3b1901e3d" />

------------------------------------------------------------------------------------------------------------------------------------------------------
**Most Recent Patent Publication Year**
```
SELECT

    EXTRACT(YEAR FROM publication_date)::INT AS year,

    COUNT(*) AS patent_count

FROM patents.patents_training

GROUP BY year

ORDER BY patent_count DESC

LIMIT 1;
```
<img width="562" height="462" alt="Most Recent Patent" src="https://github.com/user-attachments/assets/d765bb8c-5c50-464e-933f-79cf25cc8323" />

------------------------------------------------------------------------------------------------------------------------------------------------------
**Year-over-Year Performance Analysis Using SQL**
```
WITH yearly
AS
(
    SELECT

        EXTRACT(YEAR FROM publication_date)::INT AS year,

        COUNT(*) AS patent_count

    FROM patents.patents_training

    GROUP BY year
)

SELECT

    year,

    patent_count,

    LAG(patent_count)

    OVER (ORDER BY year)

    AS previous_year,

    ROUND
    (

        (

            patent_count -

            LAG(patent_count)

            OVER (ORDER BY year)

        )

        *100.0

        /

        NULLIF

        (

            LAG(patent_count)

            OVER (ORDER BY year),

            0

        ),

        2

    ) AS growth_percentage

FROM yearly

ORDER BY year;
```
**Core Concepts:**
LAG(), Window Functions, Growth Percentage Calculation, NULLIF, Common Table Expressions (CTEs)

<img width="933" height="1042" alt="Year-over-Year" src="https://github.com/user-attachments/assets/bdb87245-2138-40ff-8c32-1151d6399533" />

------------------------------------------------------------------------------------------------------------------------------------------------------
**7 - Performance Optimization**

Index on Publication Date
```
CREATE INDEX idx_patent_date
ON patents.patents_training(publication_date);
```
Purpose
Makes filtering by publication date more effective.

**Screenshot**

<img width="675" height="143" alt="Index Publication date" src="https://github.com/user-attachments/assets/51546f3b-9943-44de-80d0-8b2439a5b93e" />

------------------------------------------------------------------------------------------------------------------------------------------------------
**Functional GIN Index on Title**

```
CREATE INDEX idx_lower_title

ON patents.patents_training

USING gin

(
    to_tsvector('simple', lower(title))
);
```

Purpose
Enhances full-text search performance for patent titles.

**Screenshot**

<img width="745" height="296" alt="Functional GIN Index on Title" src="https://github.com/user-attachments/assets/7f46a4e6-274e-43d2-b052-9c314b7187a4" />

------------------------------------------------------------------------------------------------------------------------------------------------------
**Update Statistics**

ANALYZE patents.patents_training;

Purpose
Update planner statistics for improved execution plans.

<img width="568" height="113" alt="Analyze Update statistics" src="https://github.com/user-attachments/assets/8647165e-6ebc-46ab-abae-3d029a5073ab" />

------------------------------------------------------------------------------------------------------------------------------------------------------
 **8- Query Performance Benchmarking**

Use EXPLAIN ANALYZE to compare execution plans.

**Example 1**
```
EXPLAIN ANALYZE

SELECT

    EXTRACT(YEAR FROM publication_date),

    COUNT(*)

FROM patents.patents_training

GROUP BY 1;
```
<img width="1652" height="685" alt="Example 1" src="https://github.com/user-attachments/assets/acebf317-c317-4325-bf57-93d1cf1b48e2" />

------------------------------------------------------------------------------------------------------------------------------------------------------
**Example 2**
```
EXPLAIN ANALYZE

SELECT *
FROM patents.patents_training
WHERE publication_date >= DATE '2020-01-01';
```
<img width="1341" height="429" alt="Example 2" src="https://github.com/user-attachments/assets/1cc383cb-ec9d-45f1-8921-2e25178ae2a4" />

------------------------------------------------------------------------------------------------------------------------------------------------------
**Example 3**
```
EXPLAIN ANALYZE

SELECT *

FROM patents.patents_training

WHERE publication_date
BETWEEN '2020-01-01'
AND '2020-12-31';
```
<img width="1366" height="536" alt="Example 3" src="https://github.com/user-attachments/assets/e9ad95dc-df43-4428-818b-58208a284b86" />

------------------------------------------------------------------------------------------------------------------------------------------------------
Example 4
```
EXPLAIN ANALYZE

SELECT *
FROM patents.patents_training
WHERE title LIKE '%battery%';
```
<img width="1317" height="500" alt="Example 4" src="https://github.com/user-attachments/assets/088fdf7a-5966-4330-9e47-eb6e18bd107f" />

------------------------------------------------------------------------------------------------------------------------------------------------------

















