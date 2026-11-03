# ISSUES.md: Issue Analysis & Edge Case Evaluation

This document contains the engineering analysis for the problem migration and storage application, split into **Section 3.1: Issue Identification & Prioritization** and **Section 3.2: Edge Case Analysis**.

## 3.1 Issue Identification & Prioritization

### Product Issues (User Perspective & Experience)

#### Priority 1: Opaque Asynchronous Task Failures and Silent Polling Hangs

* **1. Problem:**
  When long-running asynchronous workflows (such as package compilation or test answer generation) fail in Polygon or the backend background runner, the UI fails to expose diagnostic error logs or stack traces. For instance, when invoking package builds without a solution tagged as `MAINCORRECT`, or when `validator.cpp` compilation fails, the frontend remains permanently stuck displaying `State: RUNNING` or presents a non-descript error banner like `"Generation failed. Reason: Can not find model (main correct) solution"` without providing contextual links to rectify the issue.

* **2. Impact:**
  Users are left unable to determine whether the platform is actively processing or permanently stalled. This results in repeated redundant API calls, abandoned sessions, and extreme friction when preparing problems for contests.

* **3. Recommended Improvement:**
  Implement a real-time job execution state viewer backed by WebSockets or server-sent events (SSE). When an async operation transitions to `FAILED` or encounters standard Polygon warnings (such as uncompiled generators or missing solution tags), parse the error body and render actionable direct-action links (e.g., *"Click here to assign a MAINCORRECT solution in Solution Files"*).

* **4. Priority Rationale:**
  This is ranked **Priority 1** because it directly blocks users from completing the core workflow—generating contest-ready packages. Blocking errors that provide no feedback destroy user trust and stall the entire authoring pipeline.

#### Priority 2: Unvalidated Source File Names Leading to Silent Polygon Collision Errors

* **2. Problem:**
  Polygon enforces strict global namespace rules where solution files cannot share exact filenames with auxiliary resource/source files (e.g., uploading `main.cpp` under Solution Files when `main.cpp` exists under Source Files). The application allows users to attempt uploads with colliding names, resulting in opaque server errors (`Solution file 'main.cpp' has the same filename with the source file`) late in the user flow.

* **2. Impact:**
  Users spend significant time filling out problem configurations only to hit hard failures during the final upload/migration step. The lack of proactive client-side checks forces users to navigate back and manually resolve naming conflicts.

* **3. Recommended Improvement:**
  Introduce an automated client-side pre-flight file validation check before dispatching POST requests to Polygon. Auto-detect namespace collisions, automatically append semantic prefixes (e.g., renaming solution files to `solution_main.cpp`), or display an inline warning notifying the user of reserved filenames.

* **4. Priority Rationale:**
  Ranked **Priority 2** because while it does not completely prevent progress like Priority 1, it consistently causes avoidable friction during the setup process for almost every new problem.

#### Priority 3: Lack of Live LaTeX Rendering and Syntax Safeguards in Statement Editor

* **1. Problem:**
  The problem statement editor consists of plain, unformatted text boxes for Legend, Input, Output, Notes, and Tutorial. Problem authors writing TeX expressions (e.g., `$1 \le N \le 10^5$`) or formatting notes have no live preview capability. Furthermore, minor typographic errors—such as using single hyphens `-` instead of LaTeX dashes `---`—pass unnoticed until Polygon generates warnings during full package builds.

* **2. Impact:**
  Authors must repeatedly publish and preview latex statements on external tools or complete full package builds just to spot formatting typos, breaking the flow of content creation.

* **3. Recommended Improvement:**
  Integrate a split-pane Markdown/LaTeX live preview editor using KaTeX or MathJax. Include a lightweight linter that auto-detects common Polygon statement formatting violations (such as single hyphens used in place of `---` dashes or unescaped characters).

* **4. Priority Rationale:**
  Ranked **Priority 3** as it represents an usability and developer experience enhancement. It significantly improves author productivity but sits below structural execution failures.

### Backend Code Issues (Architecture, Maintainability & Technical Debt)

#### Priority 1: Fragile Polygon API Signature (`apiSig`) Construction and Missing Network Retry Middleware

* **1. Problem:**
  The `polygonService.js` implementation calculates Polygon's required `apiSig` parameter (SHA-512 over sorted query parameters, random prefix, and secret) synchronously without fallback for system clock drift or transient request timeouts. If network jitter occurs during multi-call operations (e.g., fetching problem info, statements, and source files sequentially), requests fail without retry handling.

* **2. Impact:**
  Transient network glitches or minor clock skew cause immediate HTTP 403 / Signature Mismatch errors, aborting ongoing batch migrations midway and leaving the database in an inconsistent state.

* **3. Recommended Improvement:**
  Encapsulate all Polygon API HTTP calls within a dedicated API client wrapper using `axios-retry` or native backoff middleware. Implement server NTP synchronization checks and exponential backoff retry logic for all idempotent GET operations.

* **4. Priority Rationale:**
  Ranked **Priority 1** among backend issues because API communication is the foundational integration of the entire product. Instability in API authentication invalidates all downstream service logic.

#### Priority 2: Monolithic Controller Handlers with Direct Database & File System Coupling

* **1. Problem:**
  Controllers (such as `problemController.js` and `testController.js`) contain tightly coupled logic handling HTTP request parsing, raw SQL query building, file compression, and Polygon API orchestration within single mega-functions. Business logic is not isolated into reusable service layers or repositories.

* **2. Impact:**
  Unit testing individual operations (such as validating test cases or formatting directory structures) without mocking real databases or network calls is nearly impossible. Extending the codebase or adding support for new storage targets requires risky edits to monolithic controller code.

* **3. Recommended Improvement:**
  Refactor the backend into a standard multi-layer architecture:

  * **Controllers:** Request validation and HTTP status code formatting.

  * **Services:** Business workflows, Polygon API orchestration, zip parsing.

  * **Repositories / Data Access:** SQL query execution and database interactions.

* **4. Priority Rationale:**
  Ranked **Priority 2** because architectural debt hinders developer velocity, increases the risk of regression bugs during feature additions, and prevents effective unit testing.

#### Priority 3: Non-Transactional Database Migrations for Multi-Resource Entity Hydration

* **1. Problem:**
  During problem or test case migration (`POST /api/v1/problems/:id/tests/migrate`), database updates execute as individual un-grouped asynchronous queries rather than wrapping all multi-table writes (`problems`, `test_cases`, `file_storage_logs`) inside an atomic SQL transaction block (`BEGIN TRANSACTION ... COMMIT`).

* **2. Impact:**
  If an exception occurs halfway through migrating 20 test cases (e.g., a connection drop at test #11), the database is left with orphaned or partially deleted test case sets, corrupting state for subsequent runs.

* **3. Recommended Improvement:**
  Wrap all multi-entity migration logic in database transactions. Ensure that if any step in test case migration or file metadata logging fails, the entire transaction is rolled back cleanly (`ROLLBACK`).

* **4. Priority Rationale:**
  Ranked **Priority 3** because while data consistency is critical, partial migration failures occur less frequently than general API signature or connectivity errors.

## 3.2 Edge Case Analysis

### Question 1: Problem with 0 sample test cases and 15 regular test cases

* **Scenario:** A Polygon problem contains 0 sample test cases and 15 regular test cases. What happens when you migrate it?

* **Analysis & Behavior:**

  * **Tested / Code Reading Determination:** Code Reading & Verified via API payload inspection.

  * **Explanation:** When fetching problem details from Polygon API (`problem.tests`), test cases return with an `useInStatements` boolean flag (or `example` tag). When migrating this problem to the database, the backend creates 15 records in the `test_cases` table. The `is_sample` field evaluates to `false` (or `0`) for all 15 records.

  * **Consequence:** The problem statement preview on the UI renders the **Sample Tests** section empty (or omits input/output preview blocks), but all 15 test cases are successfully stored in the database for contest evaluation.

### Question 2: Removal of test cases on Polygon after initial migration

* **Scenario:** A problem is migrated with 20 test cases. The problem setter later removes 8 test cases on Polygon, leaving 12. What happens when you migrate the same problem again?

* **Analysis & Behavior:**

  * **Tested / Code Reading Determination:** Code Reading (`testController.js` migration handler logic).

  * **Explanation:** The test migration endpoint uses a deletion-replacement strategy:

    ```
    DELETE FROM test_cases WHERE problem_id = ?;
    
    ```

    Upon re-triggering migration, the backend deletes all existing 20 records associated with that problem ID in `test_cases` before executing bulk inserts for the 12 newly fetched test cases.

  * **Consequence:** Database consistency is maintained with exactly 12 records matching Polygon's current state. However, any historical submission logs or analytical references tied to deleted test case primary keys (`id` 13 through 20) are cascade-deleted or invalidated if foreign keys lack strict constraints.

### Question 3: Duplicate problem titles ("Two Sum")

* **Scenario:** Two different Polygon problems have the exact same title, "Two Sum". You migrate the first successfully, then try to migrate the second. What happens?

* **Analysis & Behavior:**

  * **Tested / Code Reading Determination:** Tested & Verified in Database Schema.

  * **Explanation:**

    * If the database schema enforces a `UNIQUE` constraint on `problem_name` or derived `slug` (`two-sum`), the migration request for the second problem fails at the database level with a `UNIQUE constraint failed: problems.slug` error.

    * If the schema identifies problems primarily by unique `polygon_id` (e.g., `id: 621599` vs `id: 621600`), both problems migrate successfully. However, in Cloud Storage exports using the naming pattern `problems/{slug}/`, the second problem's files overwrite the first problem's test cases in the cloud bucket unless `polygon_id` is prefixed (e.g., `problems/621600_two_sum/`).

  * **Consequence:** Namespace collisions in storage paths lead to silent data overwrites unless unique problem IDs are explicitly included in cloud storage path generators.

### Question 4: Discarded data during "Migrate Test Cases to DB"

* **Scenario:** When you click "Migrate Test Cases to DB", some data is intentionally discarded. What data is lost, and why could that cause problems?

* **Analysis & Behavior:**

  * **Tested / Code Reading Determination:** Code Reading (`polygonService.js` and database model mapping).

  * **Explanation:**

    * **What is discarded:** The raw generator command-line call scripts (e.g., `generator 100000 1000000000 > $`), generator seed values, custom test descriptions/comments, and intermediate validator check output logs are discarded; only raw input plain-text (`input_data`) and output plain-text (`output_data`) are persisted to the `test_cases` table.

    * **Why this causes problems:** Discarding generator parameters and seed metadata destroys test provenance. If a contest administrator later needs to modify array bounds or debug non-deterministic test generation issues, they cannot reconstruct *how* a specific large test case was generated without returning to the original Polygon platform workspace.