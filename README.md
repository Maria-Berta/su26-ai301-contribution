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

[phpmyadmin acts as if there was no more foreign keys on this table, but they are still there]

### Affected Components

[The issue likely affects the foreign key parser and the table structure display logic. Based on the maintainer's clue, I suspect the problem is in how `SHOW CREATE TABLE` output is parsed — specifically when a `PERIOD FOR SYSTEM_TIME` line appears before foreign key constraints.

I'll identify the exact files after setting up the development environment locally.]

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

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
