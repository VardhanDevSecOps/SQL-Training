<img width="617" height="190" alt="Insert Unique Inventors" src="https://github.com/user-attachments/assets/201d2a6c-d51f-4641-851a-9e739a5d9258" /><img width="617" height="190" alt="Insert Unique Inventors" src="https://github.com/user-attachments/assets/f6809d28-a1fc-4178-b089-16f0d82de886" />
**Check your table structure**

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

**Task 1 – Create Inventor Master** 

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

**Insert Unique Inventors**

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
<img width="493" height="122" alt="Index Created" src="https://github.com/user-attachments/assets/944b5149-3356-4064-b8a0-ab5fbe9bcda1" />
------------------------------------------------------------------------------------------------------------------------------------------------------





















