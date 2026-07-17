# Contribution [#1]: [[Bug]: Adding system versioning to a table makes foreign keys unusable
 #20095]

**Contribution Number:** 1

**Student:** Mariamawit Berta

**Issue:** [GitHub Issue #20095](https://github.com/phpmyadmin/phpmyadmin/issues/20095)

**Status:** Phase IV Complete — PR Updated Per Maintainer Feedback

---

## Why I Chose This Issue

I chose this issue because it combines two areas I'm passionate about: SQL/database work and debugging. As a computer science student and data enthusiast, I've worked extensively with data — including processing 2.5B+ records for a USPS performance analysis project — so understanding how foreign keys and table structures work is familiar territory.

I also wanted an issue that touches backend/database logic rather than just frontend UI. While I've built AI applications with Next.js and OpenAI, I'm eager to learn more about how database tools like phpMyAdmin work under the hood. This issue feels like a perfect learning opportunity for a beginer to open source like myself as it's well-scoped and has a clear reproduction path.

Finally, I chose this issue because a maintainer already gave a helpful clue about PERIOD FOR SYSTEM_TIME being added before constraints. That tells me the community is supportive and the fix is likely achievable for a first-time contributor.

---

## Understanding the Issue

### Problem Description

When a MariaDB table has system versioning enabled (using ADD SYSTEM VERSIONING), phpMyAdmin stops recognizing and displaying foreign keys on that table. The foreign keys still exist in the database and appear in SHOW CREATE TABLE, but phpMyAdmin's UI doesn't show them in "Relation View" or create clickable links in the "Browse" tab.

### Expected Behavior

Foreign keys should appear in the "Relation View" section and in the "Browse" tab, foreign key values should be clickable links that navigate to the referenced table — exactly as they do for non-versioned tables.

### Current Behavior

phpMyAdmin acts as if there are no foreign keys on the table, even though they still exist in the database and are visible in SHOW CREATE TABLE.

### Affected Components

The issue likely affects the foreign key parser and the table structure display logic. Based on the maintainer's clue, I suspect the problem is in how `SHOW CREATE TABLE` output is parsed — specifically when a `PERIOD FOR SYSTEM_TIME` line appears before foreign key constraints.

I'll identify the exact files after setting up the development environment locally.

Update:

The issue affects src/ConfigStorage/Relation.php in phpMyAdmin, specifically the getForeignKeysData() method at line 429. This method calls SHOW CREATE TABLE and passes the result to the phpmyadmin/sql-parser library. When MariaDB system versioning is enabled, the SHOW CREATE TABLE output includes two new constructs that the SQL parser library has never been taught to handle — GENERATED ALWAYS AS ROW START/END columns and a PERIOD FOR SYSTEM_TIME clause — causing the parser to fail silently and return no foreign keys.

---

## Reproduction Process

### Environment Setup

The local environment requires four tools working together: Docker Desktop to run MariaDB in an isolated container without installing it directly on the machine, PHP to serve phpMyAdmin's codebase, Composer to install phpMyAdmin's PHP dependencies, and MariaDB 10.11 (running inside Docker) as the actual database engine to reproduce the bug against.


Docker Desktop — runs MariaDB as a container so there is no need to install a database server directly on the machine
PHP 8.2+ — required to run phpMyAdmin locally via its built-in development server
Composer 2.x — phpMyAdmin's dependency manager; installs all required PHP libraries from composer.json
MariaDB 10.11 (via Docker) — the specific database version needed to test system versioning behaviour
Node.js / Yarn — required to build phpMyAdmin's frontend assets

### Setup Challenges

I ran into OS compatibility issues that blocked both Docker and PHP from installing. My machine was on macOS 13 (Ventura), which the latest Docker Desktop no longer supports, and Homebrew's PHP builds were also failing mid-compilation for the same reason. I tried installing an older Docker version as a workaround but that failed too. The fix was straightforward once I identified the root cause — I upgraded to **macOS Tahoe (macOS 15)** and both tools installed without issues. It was a good reminder that environment setup in open source rarely goes smoothly on the first try.


### Steps to Reproduce

### Step 1 — Fork and Clone phpMyAdmin

1. Go to [github.com/phpmyadmin/phpmyadmin](https://github.com/phpmyadmin/phpmyadmin)
2. Click **Fork** (top right) to create your own copy
3. Clone your fork locally:

```bash
git clone https://github.com/YOUR_USERNAME/phpmyadmin.git
cd phpmyadmin
```

4. Create a branch for your fix:

```bash
git checkout -b fix/system-versioning-foreign-keys
```

### Step 2 — Install PHP Dependencies

```bash
composer install
```

This reads `composer.json` and downloads all required libraries into a `vendor/` folder.

### Step 3 — Start MariaDB with Docker

Create a file called `docker-compose.yml` in the root of the project:

```yaml
services:
  mariadb:
    image: mariadb:10.11
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: testdb
    ports:
      - "3306:3306"
```

Start it:

```bash
docker compose up -d
```

### Step 4 — Configure phpMyAdmin to Connect

Copy the sample config:

```bash
cp config.sample.inc.php config.inc.php
```

Edit `config.inc.php` and set the host to `127.0.0.1`:

```php
$cfg['Servers'][$i]['host'] = '127.0.0.1';
$cfg['Servers'][$i]['port'] = '3306';
```

### Step 5 — Start phpMyAdmin's Built-in Server

```bash
php -S localhost:8080
```

Open your browser to `http://localhost:8080` and log in with:
- **Username:** root
- **Password:** root

### Step 6 — Create the Test Tables

In phpMyAdmin's SQL tab, run:

```sql
-- Referenced table
CREATE TABLE customers (
  id INT PRIMARY KEY,
  name VARCHAR(100)
) ENGINE=InnoDB;

-- Table with a foreign key
CREATE TABLE orders (
  id INT PRIMARY KEY,
  customer_id INT,
  ts TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  te TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME(ts, te),
  CONSTRAINT fk_customer FOREIGN KEY (customer_id)
    REFERENCES customers(id)
) ENGINE=InnoDB WITH SYSTEM VERSIONING;
```

### Step 7 — Observe the Bug

1. In phpMyAdmin, navigate to the `orders` table
2. Click the **Structure** tab
3. Click **Relation view** at the bottom
4. **Expected:** The `fk_customer` foreign key constraint should be listed
5. **Actual:** The relation view is empty — no foreign keys shown

You can also confirm what MariaDB returns by running:

```sql
SHOW CREATE TABLE orders;
```

Notice the output includes `PERIOD FOR SYSTEM_TIME (`ts`, `te`),` appearing before the `CONSTRAINT` line — this is what breaks phpMyAdmin's parser.


### Reproduction Evidence 

- Bug: Relation View shows empty Foreign key constraints section despite the foreign key existing in the database
  <img width="1440" height="900" alt="Screenshot 2026-06-26 at 2 31 03 PM" src="https://github.com/user-attachments/assets/60a704a4-5d65-4a7b-a5fd-b61a78b30662" />

- Root cause confirmed: SHOW CREATE TABLE terminal output shows PERIOD FOR SYSTEM_TIME and GENERATED ALWAYS AS ROW START/END present before the CONSTRAINT line
 
- Fix verified: After applying the fix, Relation View correctly shows fk_customer with customer_id → testdb.customers.id (screenshot)

<img width="1435" height="777" alt="Screenshot 2026-06-28 at 2 49 03 PM" src="https://github.com/user-attachments/assets/b7d7ef94-1f31-4ead-80ac-92479d4eea3c" />
<img width="1440" height="900" alt="Screenshot 2026-06-28 at 3 18 30 PM" src="https://github.com/user-attachments/assets/de2eb724-39f1-462f-94f7-215112965254" />
<img width="1440" height="900" alt="Screenshot 2026-06-28 at 3 28 24 PM" src="https://github.com/user-attachments/assets/7f7a2d11-1a69-484d-92de-2bb760350e00" />




---

## Solution Approach

### Analysis

phpMyAdmin reads the output of SHOW CREATE TABLE and passes it to the phpmyadmin/sql-parser library to extract foreign key information. When MariaDB system versioning is enabled, two new constructs appear that the parser library was never taught to handle:


GENERATED ALWAYS AS ROW START/END — MariaDB adds these to the timestamp columns used for versioning. The parser knows GENERATED ALWAYS but not the AS ROW START/END variant. When it encounters ROW, it treats it as part of a column definition, enters the wrong state, and drops all fields that follow — including the CONSTRAINT line.
PERIOD FOR SYSTEM_TIME (ts, te) — The parser has no knowledge of this clause at all. It treats PERIOD as an unknown column name, enters the wrong parsing state, and fails before reaching the foreign key constraint.


Both failures cause getForeignKeysData() to return an empty array, making phpMyAdmin act as if no foreign keys exist.

I confirmed this by grepping the entire phpmyadmin/sql-parser source for any mention of PERIOD FOR SYSTEM_TIME, ROW START, or ROW END — zero results. The library was simply never updated for MariaDB system versioning, which was added in MariaDB 10.3.4 in 2018.

### Proposed Solution

Pre-process the SHOW CREATE TABLE string in getForeignKeysData() to strip the two MariaDB system versioning constructs before passing it to the parser. This only affects the string used for foreign key detection and does not change any database data or any other phpMyAdmin functionality.

php// Strip GENERATED ALWAYS AS ROW START/END columns (MariaDB system versioning)
$showCreateTable = preg_replace(
    '/^\s*`?\w+`?\s+\w+(?:\(\d+\))?\s+GENERATED ALWAYS AS ROW (?:START|END)[^\n]*,?\n?/im',
    '',
    $showCreateTable
);

// Strip PERIOD FOR SYSTEM_TIME clause (MariaDB system versioning)
$showCreateTable = preg_replace(
    '/^\s*PERIOD\s+FOR\s+SYSTEM_TIME\s*\([^)]*\),?\n?/im',
    '',
    $showCreateTable
);

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** When MariaDB system versioning is enabled, SHOW CREATE TABLE includes constructs the SQL parser library does not know about, causing it to silently fail and return no foreign keys

**Match:** Pre-processing the input string before parsing is the safest approach — it keeps the change contained to a single method in phpMyAdmin and avoids modifying the parser library (which would require a separate PR to a different repository).

**Plan:** [Step-by-step implementation plan]
1. Locate getForeignKeysData() in src/ConfigStorage/Relation.php (line 429)
2. Add two preg_replace calls to strip the versioning constructs before parsing
3. Add unit tests covering the fix
4. Run the full test suite (composer test) to confirm no regressions
   
**Implement:** Fix applied to src/ConfigStorage/Relation.php (branch link to be added)

**Review:** Change is minimal, focused, and does not modify any parser library code. Only affects the string pre-processing step in one method.

**Evaluate:** 
- [ ] Foreign keys shown correctly in Relation View for system-versioned tables ✅ (verified visually)
- [ ] Foreign keys still work correctly for non-versioned tables (no regression)
- [ ] New test case passes
- [ ] All existing tests still pass
  
---

## Testing Strategy

### Unit Tests

- [ ] getForeignKeysData() correctly returns foreign keys when SHOW CREATE TABLE includes PERIOD FOR SYSTEM_TIME
- [ ] getForeignKeysData() correctly returns foreign keys when SHOW CREATE TABLE includes GENERATED ALWAYS AS ROW START/END columns
- [ ] getForeignKeysData() still works correctly for non-versioned tables (regression check)
- [ ] Regex handles both backtick-quoted and unquoted column names

### Integration Tests

- [ ]  Relation View on a system-versioned table shows the foreign key correctly
- [ ]  Browse tab shows clickable FK links for system-versioned tables
- [ ]  Non-versioned tables are unaffected


### Manual Testing

Manually reproduced and verified the fix in phpMyAdmin running locally against MariaDB 10.11 via Docker. Before the fix: the Relation View for the orders table showed an empty Foreign key constraints section. After applying the fix to src/ConfigStorage/Relation.php: the Relation View correctly displays the fk_customer constraint with customer_id → testdb.customers.id.
---

## Implementation Notes

### Week 3 Progress

[This week I focused on deeply understanding the issue, setting up the local development environment, and planning the fix. I read through the GitHub issue thread, understood the role of SHOW CREATE TABLE and how phpMyAdmin parses it, and identified the root cause as the PERIOD FOR SYSTEM_TIME clause breaking the foreign key regex.

I ran into significant environment setup challenges — Docker and PHP both failed to install due to macOS 13 (Ventura) compatibility issues. After multiple troubleshooting attempts including trying older Docker versions and debugging Homebrew compilation failures, I identified the OS as the root cause and upgraded to macOS Tahoe (macOS 15). Environment setup will be completed and the fix implemented once the OS upgrade finishes.



### Week 4 Progress

Completed full reproduction, root cause analysis, and fix implementation.

Reproduction: Set up phpMyAdmin locally with MariaDB 10.11 via Docker. Created a system-versioned table with a foreign key and confirmed the bug visually in the Relation View. Used SHOW CREATE TABLE via terminal to identify the exact SQL constructs causing the failure.

Root cause discovery: Traced the code path from phpMyAdmin's Relation View → getForeignKeysData() in src/ConfigStorage/Relation.php → phpmyadmin/sql-parser library. Discovered that the parser library has zero knowledge of PERIOD FOR SYSTEM_TIME or GENERATED ALWAYS AS ROW START/END. Grepped the entire library source — no results for either construct. The library was never updated for MariaDB system versioning (added in 2018).

Fix attempts: Initially tried fixing the parser library directly in vendor/phpmyadmin/sql-parser/src/Parsers/CreateDefinitions.php. Successfully handled PERIOD FOR SYSTEM_TIME with a skip block in the state machine. However, GENERATED ALWAYS AS ROW START/END failed at a deeper level — the AS option parser expected parentheses-delimited expressions and had no way to handle the ROW START/END tokens that follow without parentheses. Multiple attempts to patch the state machine all hit new failure points.

Final fix: Switched to pre-processing the SHOW CREATE TABLE string in Relation.php before passing to the parser — two preg_replace calls strip both problematic constructs. Verified the fix end-to-end in the phpMyAdmin UI — the Relation View now correctly shows the foreign key for system-versioned tables.

### Code Changes

- **File modified:** `src/ConfigStorage/Relation.php`
- **Branch:** [fix/system-versioning-foreign-keys](https://github.com/Maria-Berta/phpmyadmin/tree/fix/system-versioning-foreign-keys)
- **Key commit:** [Fix foreign key parsing broken by MariaDB system versioning](https://github.com/Maria-Berta/phpmyadmin/commit/2aac8520cc)
- **Change: Two preg_replace calls strip GENERATED ALWAYS AS ROW START/END columns and PERIOD FOR SYSTEM_TIME clause from the SHOW CREATE TABLE string before it is passed to the SQL parser
- **Approach decision:** Pre-processing in `Relation.php` is cleaner than patching the parser library — it is more readable, easier to review, and keeps the change contained to the one method where the data is consumed.

---

## Pull Request

**PR Link:** [Fix foreign key parsing broken by MariaDB system versioning](https://github.com/phpmyadmin/phpmyadmin/pull/20346)

**PR Description:** 
Fix foreign key parsing broken by MariaDB system versioning

When a MariaDB table has system versioning enabled, SHOW CREATE TABLE
includes constructs that the sql-parser library does not handle:

1. GENERATED ALWAYS AS ROW START/END columns
2. PERIOD FOR SYSTEM_TIME (...) clause

These cause the parser to return no fields, making getForeignKeysData()
return an empty array. The Relation View then appears empty for any
system-versioned table even though the foreign keys exist.

Fix: Pre-process the SHOW CREATE TABLE string in getForeignKeysData()
to strip these MariaDB-specific constructs before passing to the parser.
The foreign keys themselves are unaffected.

Fixes: #20095

Signed-off-by: Mariamawit Berta <mariamawit21geremew@gmail.com>

**Summary of Contribution:**
Fixed a bug where phpMyAdmin's Relation View showed no foreign keys for MariaDB tables with system versioning enabled. The root cause was that `SHOW CREATE TABLE` output for system-versioned tables includes two constructs — `GENERATED ALWAYS AS ROW START/END` columns and a `PERIOD FOR SYSTEM_TIME` clause — that the `phpmyadmin/sql-parser` library does not handle. The fix pre-processes the SQL string in `getForeignKeysData()` to strip these constructs before parsing, allowing foreign keys to be correctly detected and displayed.

**Maintainer Feedback:**
- **July 7, 2026:** Maintainer requested hard reset to `QA_5_2` branch, cherry-pick of fix, and addition of PHPUnit tests
- **July 7, 2026:** Completed all requested changes — rebased to `QA_5_2`, added 4 PHPUnit tests, all CI checks passing, no merge conflicts

**Code Changes (updated):**
- **File modified:** `libraries/classes/ConfigStorage/Relation.php` (QA_5_2 path)
- **Test file updated:** `test/classes/ConfigStorage/RelationTest.php` — 4 new tests added
- **Branch:** [fix/system-versioning-foreign-keys](https://github.com/Maria-Berta/phpmyadmin/tree/fix/system-versioning-foreign-keys)
- **Key commit:** [Fix foreign key parsing broken by MariaDB system versioning](https://github.com/Maria-Berta/phpmyadmin/commit/86bd09b9e1)

**Status:** Awaiting review after maintainer-requested changes
---

## Learnings & Reflections

### Technical Skills Gained

- Learned how phpMyAdmin reads and parses SHOW CREATE TABLE output using the phpmyadmin/sql-parser library
- Deepened understanding of MariaDB system versioning — what it does, when it was introduced (MariaDB 10.3.4, 2018), and how it changes table DDL output
- Learned to trace a bug through multiple layers: UI → controller → service method → third-party library
- Learned how to read and understand a state machine parser in PHP
- Practiced writing and debugging PHP regex patterns for SQL string pre-processing
- Learned how open source PHP projects use Composer, Yarn, and webpack together
- Learned phpMyAdmin's contribution requirements: Signed-off-by tags, test requirements, and branch targeting
- Learned how to respond to maintainer feedback — rebasing onto a stable branch (`QA_5_2`), writing PHPUnit tests, and iterating on a PR until CI passes and conflicts are resolved
  
### Challenges Overcome

The biggest technical challenge was discovering that the bug was deeper than expected. I initially assumed a simple regex fix in one file, but tracing the code revealed the issue was in a third-party parser library that had no knowledge of MariaDB system versioning. I attempted to fix the parser's state machine directly and hit multiple failure points — each fix exposed a new problem in a deeper layer of the parser. The pre-processing approach in Relation.php was ultimately the cleaner and more maintainable solution.

The environment setup was also unexpectedly difficult — what should have been a 30-minute setup took several sessions due to macOS version incompatibilities and a full disk.

### What I'd Do Differently Next Time

I would grep the dependency library for the relevant keywords before writing any code — knowing that phpmyadmin/sql-parser had zero knowledge of PERIOD FOR SYSTEM_TIME upfront would have pointed me to the pre-processing approach immediately. I also learned that checking system requirements for all tools before starting setup would save significant time.

---

## Resources Used

- [phpMyAdmin GitHub Issue #20095](https://github.com/phpmyadmin/phpmyadmin/issues/20095)
- [MariaDB System Versioning documentation](https://mariadb.com/kb/en/system-versioned-tables/)
- [phpMyAdmin CONTRIBUTING.md](https://github.com/phpmyadmin/phpmyadmin/blob/master/CONTRIBUTING.md)
- [phpmyadmin/sql-parser repository](https://github.com/phpmyadmin/sql-parser)
- [Homebrew Support Tiers](https://docs.brew.sh/Support-Tiers)

  

# Contribution [#2]: [ComplexCopier missing support for FixedSizeBinary columns — #559]

**Contribution Number:** 2

**Student:** Mariamawit Berta

**Issue:** [GitHub Issue #559](https://github.com/apache/arrow-java/issues/559)

**Status:** Phase II Complete.

---

## Why I Chose This Issue

I chose this issue because it sits at the intersection of Java — my strongest language — and Apache Arrow, one of the most impactful data engineering projects in the open source world. Apache Arrow is used widely in data pipelines and analytics tools, which connects directly to my background in data analysis (USPS 2.5B+ record pipeline project) and my AI Fellow experience building predictive models.

The bug is a clear, well-scoped Java issue — a missing type case causing an UnsupportedOperationException — which is exactly the kind of contained, reproducible problem I've learned to look for after my phpMyAdmin contribution experience. I also specifically wanted a project with active, responsive maintainers after experiencing slow response times on my first contribution. Apache Arrow has a large, active contributor community with regular releases and a welcoming contributor guide.

Additionally, having an Apache Arrow contribution on my resume is a strong signal — Arrow is used at companies like Databricks, Pandas, and Snowflake, all of which align with the data engineering career path I'm building.

---


## Understanding the Issue

### Problem Description

ComplexCopier in Apache Arrow Java does not support copying FixedSizeBinary columns. When a user tries to copy a ListVector that contains a FixedSizeBinary field, it throws:

java.lang.UnsupportedOperationException: FIXEDSIZEBINARY
    at org.apache.arrow.vector.complex.impl.ComplexCopier.getListWriterForReader(ComplexCopier.java:824)
    at org.apache.arrow.vector.complex.impl.ComplexCopier.writeValue(ComplexCopier.java:93)
    at org.apache.arrow.vector.complex.impl.ComplexCopier.copy(ComplexCopier.java:76)
    at org.apache.arrow.vector.complex.ListVector.copyFrom(ListVector.java:409)

### Expected Behavior

ComplexCopier should support all Arrow types including FixedSizeBinary, just like it supports other binary and variable-width types. Copying a ListVector containing FixedSizeBinary fields should work without throwing an exception.

### Current Behavior

Any attempt to copy a vector containing FixedSizeBinary fields using ComplexCopier throws UnsupportedOperationException: FIXEDSIZEBINARY.

### Affected Components

The issue is in ComplexCopier.java in the getListWriterForReader() method, which uses a switch/case over Arrow types to return the appropriate writer. FIXEDSIZEBINARY is simply missing from the switch — the default case throws the UnsupportedOperationException.

---

## Reproduction Process

### Steps to reproduce:

1. Clone `apache/arrow-java` and build the `vector` module:
```bash
   git clone https://github.com/apache/arrow-java.git
   cd arrow-java
   mvn -pl vector -am install -DskipTests
```

2. Open `vector/src/test/java/org/apache/arrow/vector/complex/impl/TestComplexCopier.java`
   and add the following test method inside the `TestComplexCopier` class:

```java
   @Test
   public void testCopyListOfFixedSizeBinary() {
     final int byteWidth = 4;
     try (ListVector from = ListVector.empty("v", allocator);
         ListVector to = ListVector.empty("v", allocator);
         ArrowBuf buf = allocator.buffer(byteWidth)) {

       from.addOrGetVector(FieldType.nullable(new ArrowType.FixedSizeBinary(byteWidth)));

       UnionListWriter listWriter = from.getWriter();
       listWriter.allocate();

       FixedSizeBinaryHolder holder = new FixedSizeBinaryHolder();
       holder.byteWidth = byteWidth;
       holder.buffer = buf;

       for (int i = 0; i < COUNT; i++) {
         listWriter.setPosition(i);
         listWriter.startList();

         buf.setBytes(0, new byte[] {1, 2, 3, 4});
         listWriter.fixedSizeBinary().write(holder);

         buf.setBytes(0, new byte[] {5, 6, 7, 8});
         listWriter.fixedSizeBinary().write(holder);

         listWriter.endList();
       }
       from.setValueCount(COUNT);

       FieldReader in = from.getReader();
       FieldWriter out = to.getWriter();
       UnsupportedOperationException e =
           assertThrows(
               UnsupportedOperationException.class,
               () -> {
                 for (int i = 0; i < COUNT; i++) {
                   in.setPosition(i);
                   out.setPosition(i);
                   ComplexCopier.copy(in, out);
                 }
               });
       assertTrue(e.getMessage().contains("FIXEDSIZEBINARY"));
     }
   }
```

3. Run the test:
```bash
   mvn -pl vector test -Dtest=TestComplexCopier#testCopyListOfFixedSizeBinary
```

4. Confirm the bug: the test passes, because it explicitly asserts that
   `ComplexCopier.copy()` throws `UnsupportedOperationException` containing
   `"FIXEDSIZEBINARY"` — a green result here confirms the bug is present, not
   that it's fixed.Expected output:
[INFO] Running org.apache.arrow.vector.complex.impl.TestComplexCopier
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.385 s -- in org.apache.arrow.vector.complex.impl.TestComplexCopier
[INFO] Results:
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS

The root cause is in `getListWriterForReader()` in
   `vector/src/main/codegen/templates/ComplexCopier.java` — its `switch`
   statement over `MinorType` values doesn't include a case for
   `FIXEDSIZEBINARY`, so it falls through to the `default` branch and throws.
   This matches [issue #559](https://github.com/apache/arrow-java/issues/559).

   
### Reproduction Evidence 
[GH-559-complex-copier-fixed-size-binary](https://github.com/Maria-Berta/arrow-java/tree/GH-559-complex-copier-fixed-size-binary)


## Solution Approach (Initial Analysis)

### Root Cause Hypothesis

getListWriterForReader() in ComplexCopier.java has a switch statement that maps Arrow MinorType values to their corresponding writer. FIXEDSIZEBINARY is not listed as a case, so it falls through to the default case which throws UnsupportedOperationException.

The fix should involve:


Adding a FIXEDSIZEBINARY case to getListWriterForReader() that returns the appropriate FixedSizeBinaryWriter
Similarly checking writeValue() (line 93) and any other methods in ComplexCopier.java that have type switches missing FIXEDSIZEBINARY
Adding a unit test that copies a ListVector containing FixedSizeBinary fields and asserts it completes without error


### Implementation Plan

Using the UMPIRE framework:

**Understand:** ComplexCopier doesn't have a case for FIXEDSIZEBINARY in its type switch, causing all copy operations on vectors containing that type to fail with an exception.

**Match:** The fix follows the same pattern as all other type cases already in the switch — find how VARBINARY or FIXEDWIDTHTYPE cases are implemented and replicate that pattern for FIXEDSIZEBINARY.

**Plan:**


1. Clone apache/arrow-java and build locally with Maven
2. Read ComplexCopier.java and locate all switch statements involving MinorType
3. Find existing FixedSizeBinaryWriter usage elsewhere in the codebase for reference
4. Add FIXEDSIZEBINARY case to getListWriterForReader() and any other missing locations
5. Find the existing ComplexCopierTest.java and model a new test on it
6. Run mvn test to confirm fix and no regressions


**Implement:** (branch link to be added)

**Review:** Follow Apache Arrow Java contribution guidelines — code style, license headers, test coverage

**Evaluate:** New test passes, no existing tests broken, UnsupportedOperationException no longer thrown for FixedSizeBinary vectors


### Week 5 Progress

This week I focused on issue selection — evaluating multiple open source projects and issues against criteria including maintainer responsiveness, skill match, scope, and resume value. I identified Apache Arrow Java #559 as the best fit: pure Java, Apache brand, active contributor community, open since January 2025 with no one currently working on it, and directly connected to my data engineering background.

I commented on the issue to express interest and ask for guidance on the correct approach and relevant test files. Awaiting maintainer confirmation before beginning implementation.

Key decisions made:


1. Chose Apache Arrow Java over phpMyAdmin (second issue) due to maintainer responsiveness concerns — phpMyAdmin PR #20346 has received no review despite being submitted and followed up on
2. Chose Apache Arrow over other candidates (Apache Burr, Angular, k-NN) because Java is my strongest language and Arrow's data engineering focus aligns most closely with my experience



## Resources Used


- [Apache Arrow Java GitHub Issue #559](https://github.com/apache/arrow-java/issues/559)
- [Apache Arrow Contributor Guide](https://arrow.apache.org/docs/developers/guide/index.html)
- [Apache Arrow Java Repository](https://github.com/apache/arrow-java)
