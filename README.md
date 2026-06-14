# Contribution [#]: [[Bug]: Adding system versioning to a table makes foreign keys unusable
 #20095]

**Contribution Number:** [1]  
**Student:** [Mariamawit Berta]  
**Issue:** [[GitHub issue link](https://github.com/phpmyadmin/phpmyadmin/issues/20095)]  
**Status:** [Phase I Complete]

---

## Why I Chose This Issue

[I chose this issue because it combines two areas I'm passionate about: SQL/database work and debugging. As a computer science student and data enthusiast, I've worked extensively with data — including processing 2.5B+ records for a USPS performance analysis project — so understanding how foreign keys and table structures work is familiar territory.

I also wanted an issue that touches backend/database logic rather than just frontend UI. While I've built AI applications with Next.js and OpenAI, I'm eager to learn more about how database tools like phpMyAdmin work under the hood. This issue feels like a perfect learning opportunity for a beginer to open source like myself as it's well-scoped and has a clear reproduction path.

Finally, I chose this issue because a maintainer already gave a helpful clue about PERIOD FOR SYSTEM_TIME being added before constraints. That tells me the community is supportive and the fix is likely achievable for a first-time contributor.]

---

## Understanding the Issue

### Problem Description

[When a MariaDB table has system versioning enabled (using ADD SYSTEM VERSIONING), phpMyAdmin stops recognizing and displaying foreign keys on that table. The foreign keys still exist in the database and appear in SHOW CREATE TABLE, but phpMyAdmin's UI doesn't show them in "Relation View" or create clickable links in the "Browse" tab.]

### Expected Behavior

[Foreign keys should appear in the "Relation View" section and in the "Browse" tab, foreign key values should be clickable links that navigate to the referenced table]

### Current Behavior

[phpMyAdmin acts as if there are no more foreign keys on this table, but they are still there]

### Affected Components

[The issue likely affects the foreign key parser and the table structure display logic. Based on the maintainer's clue, I suspect the problem is in how `SHOW CREATE TABLE` output is parsed — specifically when a `PERIOD FOR SYSTEM_TIME` line appears before foreign key constraints.

I'll identify the exact files after setting up the development environment locally.]

---

## Reproduction Process

### Environment Setup

[The local environment requires four tools working together: Docker Desktop to run MariaDB in an isolated container without installing it directly on the machine, PHP to serve phpMyAdmin's codebase, Composer to install phpMyAdmin's PHP dependencies, and MariaDB 10.11 (running inside Docker) as the actual database engine to reproduce the bug against.


Docker Desktop — runs MariaDB as a container so there is no need to install a database server directly on the machine
PHP 8.2+ — required to run phpMyAdmin locally via its built-in development server
Composer 2.x — phpMyAdmin's dependency manager; installs all required PHP libraries from composer.json
MariaDB 10.11 (via Docker) — the specific database version needed to test system versioning behaviour]

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


### Reproduction Evidence (Will be adding on this)

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [Observation during reproduction]

---

## Solution Approach

### Analysis

[phpMyAdmin reads the output of `SHOW CREATE TABLE` as a text string and uses a **regular expression (regex)** to find foreign key constraints. The regex looks for patterns like:

```
CONSTRAINT `name` FOREIGN KEY (`col`) REFERENCES `table` (`col`)
```

When MariaDB system versioning is enabled on a table, the `SHOW CREATE TABLE` output gains an additional line:

```sql
PERIOD FOR SYSTEM_TIME (`ts`, `te`),
```

This line appears **immediately before** the `CONSTRAINT` lines. The existing regex pattern does not account for this clause, causing it to fail to match the foreign key constraints entirely.]

### Proposed Solution

[**Option A — Pre-process the string (simplest)**

Strip the `PERIOD FOR SYSTEM_TIME` line before running the regex:

```php
// Remove PERIOD FOR SYSTEM_TIME clause before parsing foreign keys
$createQuery = preg_replace(
    '/\bPERIOD\s+FOR\s+SYSTEM_TIME\s*\([^)]*\),?\s*/i',
    '',
    $createQuery
);
```

**Option B — Update the regex to tolerate the clause**

Modify the existing regex so it doesn't break when `PERIOD FOR SYSTEM_TIME` appears before a `CONSTRAINT` line. This is more surgical but requires understanding the full regex pattern first.

**Recommended approach:** Option A — it is simpler, easier to review, and less likely to introduce unintended side effects.]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [When a MariaDB table has system versioning enabled (using ADD SYSTEM VERSIONING), phpMyAdmin stops recognizing and displaying foreign keys on that table. The foreign keys still exist in the database and appear in SHOW CREATE TABLE, but phpMyAdmin's UI doesn't show them in "Relation View" or create clickable links in the "Browse" tab.]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]
- [ ] Foreign keys are correctly shown in phpMyAdmin's Relation view for system-versioned tables
- [ ] Foreign keys still work correctly for non-versioned tables (no regression)
- [ ] New test case passes
- [ ] All existing tests still pass
- [ ] Code change is minimal and focused — no unrelated formatting or whitespace changes

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
