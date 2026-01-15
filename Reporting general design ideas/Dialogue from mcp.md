 1. Instance Limits/Pagination:
    - For daily reports, showing all instances could be overwhelming //yes
    - Should you limit to "last 30 days" or "last 10 instances"? // last 10 instances
    - Add filtering (date range, status)? //Add filtering (date range, status)
  2. Actions on Definitions vs Instances:
    - On Definition: Edit metadata, manage templates, archive/delete //not yet, will be hardcoded on backend at the momemt
    - On Instance: Create new, view, edit, submit, approve, export //yes
    - Where does "Create New Instance" button live? //in the best place
  3. Template Version Selection:
    - Show version dropdown in definition row? //no
    - Or only when creating a new instance? //yes
    - How to indicate which version each instance used? //indicate at the bottom of instance
  4. Expected vs Actual Instances:
    - Based on cadence, should you show "missing" reports? //yes
    - E.g., Weekly report, but no submission for Week 2 → show placeholder row? //yes
  5. Current Data Model Mismatch:
    - Backend currently returns flat report objects //I know
    - Need to restructure to: Definitions + Instances relationship //That's the tricky part!
    - Or fetch instances separately when expanding a definition? //Think of something good, but explain well after you're done



-------------------------------------------------------------------------
● Perfect! Clear requirements. Let me design the data architecture and explain the approach:

  Proposed Architecture

  Data Flow Strategy: Lazy Load on Expand

  Why this approach:
  - ✅ Fast initial page load (only definitions)
  - ✅ Filtering works naturally (filter when fetching instances)
  - ✅ Scalable (don't load thousands of instances upfront)
  - ✅ Can calculate "missing" instances on-demand

  Frontend Data Structure

  (defonce app-state
    (r/atom {
      ;; List of report definitions (loaded on page load)
      :report-definitions []

      ;; Map of definition-code → instances (lazy loaded)
      ;; Only populated when user expands a definition
      :report-instances {}

      ;; Currently expanded definition code
      :expanded-definition nil

      ;; Filter state per definition
      :instance-filters {}

      ;; Loading states
      :definitions-loading false
      :instances-loading {}  ; {"R001" true, "R002" false}
    }))

  Example data:
  ;; Definitions (fetched once)
  :report-definitions
  [{:code "R001"
    :name "Weekly Sales Report"
    :cadence :weekly
    :owner "Sales Dept"
    :template-current "v2.1"
    :template-versions ["v2.1" "v2.0" "v1.0"]
    :description "Weekly sales by region"}

   {:code "R002"
    :name "Monthly Financial Summary"
    :cadence :monthly
    :owner "Finance"
    :template-current "v1.0"
    :template-versions ["v1.0"]}]

  ;; Instances (fetched on expand)
  :report-instances
  {"R001"
    [{:id "I001"
      :definition-code "R001"
      :period "2025-W52"  ; ISO week or custom period format
      :period-start "2025-12-22"
      :period-end "2025-12-28"
      :status :approved
      :submitted-by "Jane Smith"
      :submitted-at "2025-12-29T10:00:00Z"
      :template-version "v2.1"
      :data {...}}  ; Actual report data

     ;; Placeholder for missing report
     {:id nil
      :definition-code "R001"
      :period "2025-W51"
      :period-start "2025-12-15"
      :period-end "2025-12-21"
      :status :missing  ; ← Generated on frontend/backend
      :submitted-by nil
      :template-version nil}]}

  Backend API Design

  Since definitions are hardcoded for now, we need minimal backend changes:


  ;; NEW endpoints needed:

  ;; 1. Get all report definitions (hardcoded on backend)
  GET /api/report-definitions
  → Returns [{:code "R001" :name "..." :cadence :weekly ...}]

  ;; 2. Get instances for a definition (with filters)
  GET /api/report-instances?definition=R001&limit=10&from-date=2025-01-01&status=approved
  → Returns instances + calculated missing placeholders

  ;; 3. Create new instance
  POST /api/report-instances
  Body: {:definition-code "R001" :period "2025-W52" :template-version "v2.1" ...}

  ;; 4. Update instance
  PUT /api/report-instances/:id
  Body: {:status :submitted :data {...}}

  Frontend Flow

  On page load:
  1. Fetch /api/report-definitions → populate :report-definitions
  2. Render collapsed list of definitions

  On expand definition:
  1. Set :expanded-definition "R001"
  2. Set :instances-loading {"R001" true}
  3. Fetch /api/report-instances?definition=R001&limit=10
  4. Calculate missing instances (if not done by backend)
  5. Populate :report-instances {"R001" [...]}
  6. Render instances subtable

  On filter change:
  1. Update :instance-filters {"R001" {:status :draft}}
  2. Re-fetch instances with filter params
  3. Update :report-instances

  On "Create New Instance":
  4. Show modal/form with:
    - Period selector (based on cadence)
    - Template version dropdown (default to current)
    - Data entry fields
  5. POST to /api/report-instances
  6. Refresh instances list

  Missing Instances Calculation

  Two options:

  Option A: Backend calculates (recommended)
  ;; Backend logic:
  (defn get-instances-with-missing [definition-code limit filters]
    (let [definition (get-definition definition-code)
          actual-instances (fetch-instances definition-code filters)
          expected-periods (calculate-expected-periods (:cadence definition) limit)
          actual-periods (set (map :period actual-instances))
          missing-periods (remove actual-periods expected-periods)
          missing-instances (map make-placeholder-instance missing-periods)]
      (take limit (sort-by :period > (concat actual-instances missing-instances)))))

  Option B: Frontend calculates
  ;; Frontend logic:
  (defn add-missing-instances [definition instances limit]
    (let [cadence (:cadence definition)
          expected-periods (calculate-expected-periods cadence limit)
          actual-periods (set (map :period instances))
          missing (for [period expected-periods
                        :when (not (actual-periods period))]
                    {:id nil :period period :status :missing})]
      (take limit (sort-by :period > (concat instances missing)))))

  I recommend Option A (backend) because:
  - Business logic belongs on backend
  - Easier to test
  - Consistent period calculation
  - Less frontend complexity

  Migration Path

  Phase 1: Hardcoded Backend (now)
  ;; src/clj/reporting_planner/definitions.clj
  (def report-definitions
    [{:code "R001" :name "Weekly Sales" :cadence :weekly ...}
     {:code "R002" :name "Monthly Finance" :cadence :monthly ...}])

  Phase 2: Database-driven (later)
  - Move definitions to report-definitions.edn
  - Add admin UI for editing
  - Keep same API contract

  ---
  Summary

  Frontend changes needed:
  - Restructure app-state (definitions vs instances)
  - Change report-list-view to master-detail
  - Add expand/collapse logic
  - Add filters UI
  - Handle missing instance placeholders

  Backend changes needed:
  - Add /api/report-definitions endpoint (hardcoded data)
  - Modify /api/report-instances to accept filters
  - Add logic to calculate missing instances
  - Keep existing instance CRUD endpoints

  Best place for "Create New Instance" button:
  - In the expanded definition section, above the instances table
  - Shows: "➕ Create [Period] Report" (e.g., "Create Week 1 2026 Report")

  Does this architecture make sense? Should I proceed with implementing this?







 Here is Claude's plan:
╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌ Implementation Plan: Hierarchical Master-Detail Reporting System

 Overview

 Restructure the reporting system from a flat list of reports to a two-tier hierarchical architecture:
 - Master Level: Report definitions (form types like Форма 1, Форма 2) with metadata
 - Detail Level: Report instances (submitted reports for specific periods)

 Key Features:
 - Lazy-load instances when expanding a definition
 - Show last 10 instances per definition
 - Filter instances by date range and status
 - Display missing instances as grayed-out rows
 - Template version selection when creating instances
 - Start with fresh instances.edn (leave reports.edn untouched)

 ---
 Critical Files

 Backend (Clojure)

 1. src/clj/reporting_planner/definitions.clj (NEW) - Hardcoded report definitions
 2. src/clj/reporting_planner/instances.clj (NEW) - Instance management & missing calculation
 3. src/clj/reporting_planner/config.clj (MODIFY) - Add instances.edn path
 4. src/clj/reporting_planner/core.clj (MODIFY) - Add 3 new API endpoints

 Frontend (ClojureScript)

 5. src/cljs/reporting_planner/core.cljs (MODIFY) - Restructure app-state, add API calls, rebuild UI
 6. src/cljs/reporting_planner/i18n.cljs (MODIFY) - Add ~20 new translation keys (EN/RU)

 Data Files

 7. data/instances.edn (NEW) - Will be created automatically on startup

 ---
 Data Structures

 Definition (hardcoded in definitions.clj)

 {:code "DEF-FORMA-1"
  :name "Объем инвестиций"
  :form-type "Форма 1"
  :cadence :monthly  ; :daily, :weekly, :monthly, :quarterly, :yearly
  :owner "Отдел финансирования"
  :template-current "v2"
  :template-versions [{:version "v2" :file "template_forma_1_v2.xlsx" :active-from "2025-01-01"}
                      {:version "v1" :file "template_forma_1_v1.xlsx" :active-from "2024-01-01"}]
  :description "Ежемесячный отчет об объеме инвестиций"
  :fields ["domestic-investment" "foreign-investment" "production-volume"]}

 Instance (stored in instances.edn)

 {:code "INS-001"
  :definition-code "DEF-FORMA-1"
  :period-type "month"
  :period-value "2025-01"
  :status :draft  ; :draft, :submitted, :approved
  :submitted-by "ivanov"
  :submission-date "2025-01-15"
  :template-version "v2"
  :data {"domestic-investment" "1500000" ...}
  :created-at 1736956800000
  :updated-at 1736956800000}

 Missing Instance Placeholder

 {:period-value "2024-12"
  :status :missing
  :definition-code "DEF-FORMA-1"}

 ---
 API Endpoints

 GET /api/report-definitions

 Purpose: Fetch all report definitions (hardcoded)
 Auth: Required (Bearer token)
 Response:
 {
   "success": true,
   "definitions": [
     {"code": "DEF-FORMA-1", "name": "Объем инвестиций", "cadence": "monthly", ...},
     {"code": "DEF-FORMA-2", "name": "Налоговые вычеты", "cadence": "quarterly", ...}
   ]
 }

 GET /api/report-instances

 Purpose: Fetch instances for a definition with filters
 Auth: Required
 Query Params:
 - definition-code (required): e.g., "DEF-FORMA-1"
 - limit (optional): default 10
 - status (optional): "draft", "submitted", "approved"
 - date-from (optional): "2024-01" (month format)
 - date-to (optional): "2025-12"

 Response:
 {
   "success": true,
   "instances": [
     {"code": "INS-001", "period-value": "2025-01", "status": "submitted", ...}
   ],
   "missing": [
     {"period-value": "2024-12", "status": "missing", "definition-code": "DEF-FORMA-1"}
   ]
 }

 POST /api/report-instances

 Purpose: Create new instance
 Auth: Required
 Request Body:
 {
   "definition-code": "DEF-FORMA-1",
   "period-type": "month",
   "period-value": "2025-02",
   "template-version": "v2",
   "data": {
     "domestic-investment": "1600000",
     "foreign-investment": "400000"
   }
 }

 Response:
 {
   "success": true,
   "instance": {"code": "INS-007", ...},
   "message": "Instance created successfully"
 }

 ---
 Frontend App State Changes

 (defonce app-state
   (r/atom {
     ;; NEW: Definitions state
     :definitions []
     :definitions-loading false
     :definitions-failed false

     ;; NEW: Instances state (per-definition)
     :instances {}  ; Map: {definition-code {:instances [] :missing [] :loading false :failed false}}
     :instance-filters {}  ; Map: {definition-code {:status nil :date-from nil :date-to nil}}

     ;; NEW: Expansion tracking
     :expanded-definition nil  ; Currently expanded definition code

     ;; NEW: Create instance form
     :create-instance-form {:definition-code nil
                            :period-type nil
                            :period-value nil
                            :template-version nil
                            :data {}}
     :show-create-instance-modal false

     ;; Existing state remains unchanged
     :logged-in false
     :username nil
     :token nil
     :loading false
     :error nil
     :current-page :report-directory
     ...}))

 ---
 UI Component Hierarchy

 report-list-view (master table showing definitions)
   └─ definition-row (each definition, clickable)
        ├─ definition-summary (collapsed: cert icon, name, owner, cadence, status)
        └─ definition-detail-panel (expanded)
             ├─ definition-metadata (cadence, owner, current template)
             ├─ instance-filters (status dropdown, date-from, date-to, apply button)
             ├─ create-instance-button
             └─ instance-table
                  ├─ instance-row (actual submitted instances)
                  └─ missing-instance-row (grayed out, status "Missing")

 ---
 Implementation Steps

 Phase 1: Backend (2 days)

 1. config.clj: Add instances-file path
 2. definitions.clj: Create with hardcoded definitions for Форма 1, 2, 5
 3. instances.clj: Implement:
   - Storage functions (read/write instances.edn)
   - get-instances-by-definition with filter support
   - calculate-missing-instances based on cadence
   - create-instance! with auto-generated codes (INS-001, INS-002...)
   - initialize-instances-file!
 4. core.clj: Add 3 handlers and update routes
 5. Test: Use curl to verify endpoints work

 Phase 2: Frontend (2 days)

 6. i18n.cljs: Add translations for:
   - Status badges: :status-draft, :status-submitted, :status-approved, :status-missing
   - Filters: :filter-status, :filter-date-from, :filter-date-to, :apply-filters
   - Actions: :create-new-instance, :view-button
   - Labels: :column-period, :column-template-version, :definition-details
   - Messages: :loading, :failed-to-load-definitions, :no-instances
 7. core.cljs: Update app-state structure
 8. core.cljs: Add API functions:
   - fetch-definitions!
   - fetch-instances! (with filter params)
   - create-instance!
 9. core.cljs: Build components:
   - instance-status-badge (colored badges for draft/submitted/approved/missing)
   - missing-instance-row (grayed out with opacity 0.5)
   - instance-row (period, template version, status, submitted-by, view button)
   - instance-filters (3 dropdowns + apply button)
   - instance-table (thead + tbody with conditional rendering)
   - definition-detail-panel (metadata + filters + create button + table)
   - definition-row (clickable row that expands/collapses)
   - Update report-list-view to use new structure
 10. Test: Manual testing in browser

 Phase 3: Integration (1 day)

 11. End-to-end testing:
   - Login → View definitions → Expand → View instances + missing
   - Apply filters → Verify filtered results
   - Create new instance → Verify it appears in table
 12. Verify backward compatibility (old /api/reports still works)

 ---
 Key Implementation Notes

 Backend Patterns

 - Follow existing auth pattern: extract Bearer token → session/session-valid? → session/update-session-access!
 - Use storage/read-edn-file and storage/write-edn-file! for persistence
 - Return {:success true ...} or {:error "..."} with appropriate HTTP status codes
 - All new routes go in api-routes with Transit format

 Frontend Patterns

 - Dereference @i18n/current-locale at component start for reactivity
 - Use cond for loading → error → empty → data states
 - Use for with ^{:key field} metadata for collections
 - Lazy load: fetch instances only when definition is expanded
 - Cache instances in app-state (don't refetch on re-expand unless filters change)

 Missing Instance Logic

 - Backend calculates missing instances based on cadence
 - For monthly: generate last 12 months of "YYYY-MM" strings
 - For quarterly: generate last 4 quarters of "YYYY-Q1" strings
 - Compare with actual instances, return placeholders for gaps
 - Frontend displays missing rows with gray background and "Missing" status

 Styling

 - Use existing win98-button and win98-input style functions
 - Missing rows: {:opacity "0.5" :background-color "#f5f5f5"}
 - Status badges: colored spans with classes (add CSS later if needed)
 - Detail panel: blue border {:border "2px solid #0066cc"} on white background

 ---
 Testing Strategy

 Backend Testing (curl)

 # Get definitions
 curl -H "Authorization: Bearer <token>" http://localhost:3000/api/report-definitions

 # Get instances
 curl -H "Authorization: Bearer <token>" \
   "http://localhost:3000/api/report-instances?definition-code=DEF-FORMA-1&limit=10"

 # Create instance
 curl -X POST -H "Authorization: Bearer <token>" \
   -H "Content-Type: application/transit+json" \
   -d '{"definition-code":"DEF-FORMA-1","period-value":"2025-03",...}' \
   http://localhost:3000/api/report-instances

 Frontend Testing (manual)

 1. Login with valid credentials
 2. Verify definitions load and display in table
 3. Click definition row → Should expand with loading indicator
 4. Verify instances + missing rows appear
 5. Change filters → Click Apply → Verify filtered results
 6. Click "Create New Instance" → Fill form → Submit → Verify new row appears

 ---
 Potential Challenges

 1. Date Arithmetic: Missing instance calculation needs careful date handling
   - Solution: Use simple string generation for periods, no complex date math
 2. Filter State Complexity: Each definition has independent filters
   - Solution: Use nested map structure {:instance-filters {def-code {:status ...}}}
 3. Lazy Loading Performance: Multiple expansions = multiple API calls
   - Solution: Cache in app-state, only fetch once per definition
 4. Backward Compatibility: Old code expects flat reports
   - Solution: Keep /api/reports unchanged, new code uses separate endpoints

 ---
 Success Criteria

 - Backend: 3 new endpoints working with proper auth
 - Backend: Missing instance calculation working for monthly/quarterly cadence
 - Frontend: Master-detail UI with expand/collapse
 - Frontend: Lazy loading of instances on expand
 - Frontend: Filters working (status, date range)
 - Frontend: Missing instances shown as grayed rows
 - Frontend: Create instance button + workflow
 - Both: Template version selection and display
 - Backward compatibility: Old /api/reports still functional

 ---
 Notes

 - Existing reports.edn remains untouched (backward compatibility)
 - New system uses separate instances.edn file
 - Migration helper included in instances.clj for testing (optional use)
 - All new UI text is bilingual (EN/RU) via i18n
 - Follows existing Win98-style design patterns
╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌
 Would you like to proceed?

 ❯ 1. Yes, and auto-accept edits
   2. Yes, and manually approve edits
   3. Type here to tell Claude what to change

 ctrl-g to edit in Notepad