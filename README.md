# ABAP
ABAP — A Comprehensive, Practical, and Detailed Introduction (in English)

ABAP (Advanced Business Application Programming) is the primary programming language used to build applications on the SAP platform. It powers the business logic for SAP ERP, S/4HANA, and countless customer-specific extensions and integrations. This guide gives you a deep, practical tour of ABAP: its history and role, core language concepts, data structures, database access, procedural and object-oriented paradigms, modern ABAP features (CDS, RAP, AMDP), integration and extension points (BAPI, RFC, IDoc, OData), tooling and deployment, performance best practices, testing and debugging, security considerations, and suggested learning path and exercises. Examples and idiomatic patterns are included so you can apply the ideas in real SAP work.

1. What is ABAP and why it matters

ABAP is a high-level, fourth-generation language created by SAP to write business applications that run inside the SAP application server. For decades ABAP has been the dominant way to:

Implement business processes inside SAP ERP (finance, logistics, HR, etc.).

Customize and extend standard SAP behavior using enhancements, exits, and user-exits.

Build reports, batch jobs, forms, and integration interfaces.

Create application servers logic that interacts with the SAP database via Open SQL.

Key reasons ABAP remains central:

Tight integration with SAP data model and metadata (tables, structures, DDIC).

Access to SAP runtime services: transport management, authorization, job scheduling, update tasks, application logs.

Evolution with the platform: ABAP has modernized to support object-orientation, HANA-specific optimizations, CDS, and cloud-ready programming models (RAP).

2. SAP runtime and where ABAP runs

ABAP executes on the SAP NetWeaver / ABAP application server. The typical stack:

Database layer (Oracle, SQL Server, DB2, or SAP HANA): where persistent tables live.

ABAP application server: executes ABAP code, handles presentation via SAP GUI / Web UI, manages sessions, and coordinates transactions.

Presentation layer: SAP GUI, Web Dynpro, Fiori (SAPUI5), or external clients via OData/RFC.

In modern deployments, SAP S/4HANA runs ABAP optimized for the HANA in-memory database, and ABAP developers use tools such as Eclipse with the ABAP Development Tools (ADT) plugin or the Web IDE for SAP Cloud Platform.

3. ABAP language overview — style and basic program structure

ABAP has evolved from procedural roots into a fully object-oriented language while retaining procedural constructs. Programs come in many forms: reports, class pools, function modules, module pools (Dynpro), and include files.

A minimal classical report (procedural) example:

REPORT zhello_world.

WRITE: / 'Hello, ABAP world!'.


A basic class (OO ABAP) example:

CLASS lcl_greeter DEFINITION.
  PUBLIC SECTION.
    METHODS: constructor IMPORTING iv_name TYPE string,
             greet.
  PRIVATE SECTION.
    DATA: mv_name TYPE string.
ENDCLASS.

CLASS lcl_greeter IMPLEMENTATION.
  METHOD constructor.
    mv_name = iv_name.
  ENDMETHOD.
  METHOD greet.
    WRITE: / 'Hello,', mv_name.
  ENDMETHOD.
ENDCLASS.

START-OF-SELECTION.
  DATA(lo) = NEW lcl_greeter( iv_name = 'Alice' ).
  lo->greet( ).


ABAP is case-insensitive for symbols: WRITE, write, and Write are the same. Modern ABAP introduced inline declarations, expressions, and richer constructs to improve readability.

4. Core data types, declarations, and data dictionary
4.1 Basic types

Elementary types: I (integer), F (floating), P (packed decimal), C (character), N (numeric text), D (date), T (time), STRING, XSTRING.

ABAP Dictionary (DDIC): central repository of types, tables, structures, domains. You typically reference DDIC types in code, e.g. TYPE zcustomers-country TYPE kunnr or LIKE and TYPE keywords.

4.2 Variables and declarations

Old style:

DATA lv_counter TYPE i.
DATA ls_item TYPE ztable_line.


Inline declarations (since ABAP 7.40+):

DATA(lv_count) = 0.
DATA(lo) = NEW lcl_object( ).

4.3 Structures and internal tables

Structures: TYPES: BEGIN OF ty_item, id TYPE i, name TYPE string, END OF ty_item.

Internal tables: in-memory tables for processing sets of rows. Key kinds: STANDARD TABLE, SORTED TABLE, HASHED TABLE. Example:

DATA: lt_items TYPE STANDARD TABLE OF ty_item WITH DEFAULT KEY.


READ TABLE, APPEND, INSERT, DELETE, LOOP AT are core operations on internal tables. Modern idioms use FOR expressions and table expressions when available.

4.4 Field-symbols and data references

Field-symbols are like pointers: FIELD-SYMBOLS <fs> TYPE ty. They avoid copying and are essential for performance when manipulating internal table rows.

Data references: REF TO types, CREATE DATA, and dereferencing with ->*.

5. Control flow, expressions, and modern constructs

Classic ABAP control constructs: IF, CASE, WHILE, DO, LOOP AT. Modern ABAP adds richer expressions:

String templates: |Hello { lv_name }|

Inline declarations & table expressions: DATA(value) = lines( lt_items ).

Value constructors: VALUE #( id = 1 name = 'A' )

FOR expressions (functional mapping/filtering):

DATA(lt_names) = VALUE string_table( FOR wa IN lt_items ( wa-name ) ).


Exception handling uses TRY / CATCH / ENDTRY for class-based exceptions. MESSAGE statements produce messages of types I, W, E, S, A, and are frequently used in UI contexts.

6. Internal tables — the workhorse of ABAP data processing

Internal tables are central. Important points:

Table types:

STANDARD TABLE: unsorted, suitable for simple buffer-like use or position-based processing.

SORTED TABLE: keeps entries sorted by key; fast retrieval by key.

HASHED TABLE: optimized for key-based access; O(1) lookups for properly hashed keys.

Keys:

Primary key is declared in the table type. Using proper keys leverages READ TABLE ... WITH KEY and DELETE TABLE operations efficiently.

WITH UNIQUE KEY enforces uniqueness.

Operations:

LOOP AT lt_tab INTO DATA(ls_row). — iterate.

READ TABLE lt_tab WITH KEY id = lv_id INTO DATA(ls_row).

COLLECT, SORT, DELETE ADJACENT DUPLICATES, MODIFY TABLE.

Performance:

Avoid repeated READ TABLE ... inside loops when you can use hashed tables or build an index mapping.

Use field-symbols and ASSIGN to avoid copying rows.

7. Database access — Open SQL, Native SQL, and HANA specifics
7.1 Open SQL (the ABAP staple)

Open SQL is an ABAP-embedded SQL that is database-agnostic and safe; it integrates with ABAP types and the DDIC:

SELECT id name
  FROM zcustomers
  INTO TABLE @DATA(lt_customers)
  WHERE region = @lv_region.


Notes:

Use the @ host-variable prefix (newer syntax) to distinguish ABAP variables inside SQL expressions.

INTO TABLE populates an internal table, INTO a structure or fields.

UP TO n ROWS or ORDER BY supported.

JOIN, GROUP BY, HAVING, FOR ALL ENTRIES exist but must be used carefully for performance.

7.2 Native SQL

If you need vendor-specific SQL or database hints (rare in portable applications), you can use Native SQL — but it breaks portability.

7.3 ABAP on HANA — CDS views, AMDP, and pushdown

When running on HANA, ABAP has optimizations and constructs to exploit in-memory capabilities:

Core Data Services (CDS): a declarative layer to define semantically rich views using a DDL-like syntax. CDS views can expose associations, aggregations, and annotations consumed by UI frameworks and OData. Example (CDS DDL simplified):

@AbapCatalog.sqlViewName: 'ZCDS_CUST'
define view Z_CDS_Customer as select from zcustomers {
  key id,
  name,
  region
}


AMDP (ABAP Managed Database Procedures): allows writing database procedures in SQLScript and invoking them from ABAP to push complex computations to HANA.

Code pushdown: use CDS, SQLScript, and vectorized processing to push work to HANA and reduce network/ABAP server overhead.

8. Object-oriented ABAP (OO ABAP)

OO ABAP has matured into the primary recommended way of structuring application logic.

8.1 Classes and interfaces

Class declaration: CLASS lcl_example DEFINITION PUBLIC CREATE PRIVATE. for controlled instantiation.

Methods: instance (METHODS) and static (CLASS-METHODS) methods, with parameters IMPORTING, EXPORTING, CHANGING, RETURNING.

Visibility: PUBLIC, PROTECTED, PRIVATE.

Constructors: CONSTRUCTOR.

Inheritance: INHERITING FROM (single inheritance), IMPLEMENTING for interfaces.

Interfaces: declare method signatures without implementation.

8.2 Exceptions and events

Exceptions are classes that inherit from CX_ROOT.

Events are declared in classes and raised with RAISE EVENT ....

8.3 Design patterns in ABAP

Common patterns: service classes, factory patterns for instantiation, repositories for data access, and strategy patterns for business rule variability. Modern ABAP emphasizes clean layered architecture: UI -> Application Layer (services) -> Domain -> Persistence (DDIC/CDS).

9. Reports, Dynpro, ALV, and UI integration
9.1 Reports

Classical reports produce lists and support selection screens:

REPORT zreport.

PARAMETERS: p_date TYPE sy-datum.

START-OF-SELECTION.
  " Business logic
  WRITE: / 'Report data:'.

9.2 Module pool (Dynpro) programming

Dynpro is the older screen programming model with screen painter, PBO/PAI modules, and is still used in many SAP applications. It's event-driven and tightly coupled with screen elements.

9.3 ALV (ABAP List Viewer)

ALV provides reusable list/grid display with sorting, filtering, and export features. CL_GUI_ALV_GRID or CL_SALV_TABLE are frequently used classes for table display. ALV drastically improves user productivity over raw WRITE outputs.

9.4 Modern UI — OData and SAP Fiori

Modern ABAP applications expose OData services (via Gateway) or use the ABAP RESTful Programming Model (RAP) and CDS to provide business services consumed by SAP Fiori (SAPUI5) frontends.

10. Integration points — BAPI, RFC, IDoc, and APIs

ABAP is used extensively in integration and interfaces.

BAPI (Business Application Programming Interface): standardized function module interfaces for business objects, used by external systems or other SAP systems.

RFC (Remote Function Call): ABAP can expose RFC-enabled function modules for remote calls (synchronous/asynchronous).

IDoc (Intermediate Document): message format for asynchronous EDI-like integration between SAP systems or external partners.

OData: REST-style services often generated from CDS/Service Builder and consumed by web UIs or external clients.

SOAP / Web services: legacy but supported through proxies.

Working with integration points requires careful transaction and error handling, idempotency awareness, and monitoring of communication channels (SM58, BD87 transactions, etc.).

11. Extensibility — enhancements, user-exits, BAdIs

SAP supports customer-specific extensions without modifying standard code:

User-exits / customer-exits: hooks inserted in standard code to call customer-provided logic.

BAdIs (Business Add-Ins): object-oriented enhancement frameworks allowing multiple implementation instances; GET and METHOD patterns control behavior.

Enhancement framework: supports implicit and explicit enhancements, spots, and sections inserted into standard programs.

Shadow includes and append structures: allow extending standard tables with new fields while preserving upgradeability.

Use these extensibility hooks to safely extend SAP behavior and keep custom code upgrade-friendly.

12. Transactions, locking, and update handling

ABAP transactions must coordinate database state and application consistency.

COMMIT WORK and ROLLBACK WORK control database commits in ABAP programs. Many SAP frameworks manage commit points; manual commits require care.

Enqueue/Dequeue (ENQUEUE/DEQUEUE): explicit locking API to serialize critical sections at application level.

Update tasks: background update function modules ensure DB consistency and asynchronous processing.

Be cautious: misuse of commits can break transactional integrity; use update function modules and officially recommended patterns where possible.

13. Performance, tuning, and HANA best practices

Performance is a major concern in ABAP systems. Key principles:

13.1 Reduce database roundtrips

Fetch necessary columns (avoid SELECT *).

Use INTO TABLE and process data in memory.

Use joins or CDS-based views to let the database do the heavy lifting.

13.2 Use appropriate internal table types

For frequent key lookups, prefer HASHED or SORTED with properly declared keys.

Avoid READ TABLE inside loops—try joins or create maps first.

13.3 HANA pushdown

Use CDS views and AMDPs to push complex calculations into HANA (vectorized processing).

Avoid procedural row-by-row processing in ABAP for large datasets.

13.4 Tracing and analysis tools

ST05 SQL trace to see SQL executed and measure time.

SAT runtime analysis and SE30 older runtime profiler.

ABAP Trace (SE30 / SAT) and Runtime Analysis help find bottlenecks.

13.5 Memory and garbage considerations

Watch internal table sizes, use FREE to free memory, and process large datasets in chunks or via cursor-like patterns.

14. Testing, quality assurance, and CI
14.1 ABAP Unit

ABAP Unit is the unit testing framework (xUnit style). Example:

CLASS ltcl_test DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS test_sum FOR TESTING.
ENDCLASS.

CLASS ltcl_test IMPLEMENTATION.
  METHOD test_sum.
    cl_abap_unit_assert=>assert_equals( act = 3 exp = 1 + 2 ).
  ENDMETHOD.
ENDCLASS.


Unit tests enable regression control and are integrated into development pipelines.

14.2 Code inspector and ATC

Code Inspector and ATC (ABAP Test Cockpit) check code quality, performance, security, and naming conventions.

Use ATC in CI/CD to enforce quality gates before transports are released to higher environments.

14.3 Continuous Integration

Modern SAP landscapes integrate ADT-based builds, Git-based source control (abapGit or the built-in Git integrations), and pipeline automation to run ATC, unit tests, and automated transports.

15. Security and authorization

Security is critical in enterprise systems.

AUTHORITY-CHECK enforces SAP authorization objects; always check authorization before sensitive operations.

Input validation: guard against injection or malformed inputs; although Open SQL is parameterized, dynamic SQL and native SQL require sanitization.

Secure storage: protect credentials and use secure channels (SAProuter, SNC).

Transport security: restrict who can move transports and use change management processes.

ABAP applications should follow SAP Secure Programming Guidelines.

16. Deployment and transports

SAP uses a transport-based change management:

Development objects are bundled in change requests (transport requests).

Transport system moves changes from DEV -> QAS -> PROD.

Package and transport design is important: choose correct package, place objects in request, ensure dependencies are included.

Use CTS (Change and Transport System) and coordinate with Basis/Change Management teams for production deployments.

17. Modern ABAP technologies and the ABAP programming model

Key modern trends:

CDS (Core Data Services): semantic layer and central for SAP's data modeling.

RAP (RESTful ABAP Programming Model): recommended for building transactional and analytical SAP Fiori apps and services; integrates CDS, behavior definitions, and service consumption.

OData services: generated from CDS and consumed by UI5/Fiori.

Cloud and S/4HANA: ABAP adapts to cloud environments (SAP BTP) and S/4-specific patterns; some older extensions are discouraged in cloud scenarios.

Learn RAP and CDS to build modern SAP applications.

18. Practical best practices and coding guidelines

Prefer OO ABAP with clear separation (services, domain, persistence).

Keep database logic in CDS or repository classes; avoid scattered SQL.

Use typed exceptions and avoid generic MESSAGE as the only error handling.

Keep performance in mind: use batch processing, avoid nested loops with DB calls, and leverage HANA pushdown.

Make code upgrade-friendly: use enhancements and BAdIs rather than modifying standard SAP code.

Write unit tests and enforce ATC checks in CI pipelines.

19. Learning path, tools, and resources
Tools

SAP GUI: classic environment for many transactions and debugging.

Eclipse + ABAP Development Tools (ADT): modern IDE for ABAP development; supports class editor, refactoring, debugging, and Git integration.

abapGit: open-source Git integration for ABAP.

SAP Web IDE / Business Application Studio: for cloud development and Fiori.

Suggested learning path

Learn ABAP syntax and procedural programming (reports, selection screens).

Master internal tables and Open SQL.

Learn OO ABAP: classes, interfaces, unit testing.

Explore DDIC and table design.

Learn CDS and RAP for modern service development.

Understand integration: RFC, BAPI, IDoc, and OData.

Study performance analysis tools (ST05, SAT) and HANA-specific optimizations.

Practice transports, change management, and security checks.

Practice exercises

Build a report that reads from a custom table, sorts results, and displays via ALV.

Create a service class and expose a function module via RFC.

Model a business object with CDS and expose it as an OData service consumed by a simple SAPUI5 app.

Convert a procedural legacy report into OO ABAP with unit tests.

Profile and optimize a dataset processing job using ST05 and refactor to use CDS pushdown.

20. Conclusion

ABAP remains the backbone of SAP application development. It is a mature language that has grown from procedural roots to a modern, object-oriented, and HANA-aware programming environment. Whether you maintain legacy SAP systems, build bespoke extensions, or create cloud-ready Fiori services, ABAP knowledge is indispensable in the SAP ecosystem. Key skills include mastering internal tables and Open SQL, practicing OO ABAP and CDS, understanding integration patterns, and applying performance best practices—especially in HANA environments.
