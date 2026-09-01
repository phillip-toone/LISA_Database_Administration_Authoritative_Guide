\# LISA Database Administration Authoritative Guide



A version-controlled database administration, maintenance, troubleshooting, and operational knowledge base for the OSHA Laboratory Information System (LISA/LIMS).



\## Purpose



This repository consolidates the available LISA database administration documentation into a single operational reference for maintaining and troubleshooting the legacy LISA system.



The documentation is intended to be usable by both:



\- a human database administrator responsible for LISA; and

\- an LLM assisting a DBA with investigation, troubleshooting, or maintenance.



The guide emphasizes safe database administration practices, explicit verification of database state, and preservation of the distinction between inherited documentation, live-verified behavior, historical information, and inference.



\## Authoritative Guide



The primary document in this repository is:



\*\*\[LISA Database Administration Authoritative Guide](LISA\_Database\_Administration\_Authoritative\_Guide.md)\*\*



This is the current operational reference. Git history should be used to track revisions rather than creating date- or version-specific copies of the guide.



The guide covers, among other topics:



\- LISA/LIMS architecture and relational data model

\- Oracle transaction and schema-discovery practices

\- sample lookup, assignment, unassignment, and deletion

\- analyte corrections and missing analysis targets

\- sampling-sheet corrections

\- unique-constraint troubleshooting

\- QC administration in `QC\_DEV2`

\- QC unposting and theoretical/ASM value corrections

\- OIS-to-LISA and LISA-to-OIS integration

\- MapForce and batch-process troubleshooting

\- analyst and Oracle-user administration

\- LIMS table, key, constraint, and relationship references

\- known documentation defects and inconsistencies

\- procedures for safely investigating undocumented behavior



\## Repository Structure



```text

.

├── .gitattributes

├── LISA\_Database\_Administration\_Authoritative\_Guide.md

└── docs/

&#x20;   └── source/

&#x20;       ├── DSD12.pdf

&#x20;       ├── ERD5.pdf

&#x20;       ├── LISA-Maintenance.pptx

&#x20;       └── LISA\_Maintenance.pptx

```



\### `docs/source/`



Contains the inherited documentation from which the authoritative guide was initially derived.



These files are retained as source material and historical evidence. They should not automatically be treated as more authoritative than the current guide or the live database. Some of the inherited documentation is old, internally inconsistent, or contains examples that should not be executed as written.



\## Evidence and Provenance



The authoritative guide distinguishes among several types of information:



\- \*\*SOURCE-DOCUMENTED\*\* — stated or shown in the inherited LISA documentation.

\- \*\*LIVE-VERIFIED\*\* — confirmed against the LISA database during actual DBA work.

\- \*\*SOURCE-DOCUMENTED + LIVE-VERIFIED\*\* — supported by both.

\- \*\*HISTORICAL\*\* — useful legacy information that may no longer reflect the current environment.

\- \*\*ENVIRONMENT-DEPENDENT\*\* — infrastructure or configuration that must be verified before use.

\- \*\*INFERRED\*\* — a reasoned conclusion that has not yet been independently established.

\- \*\*SOURCE-DEFECT\*\* — an apparent error or contradiction in the inherited documentation.



When historical documentation conflicts with verified behavior or current database metadata, the discrepancy should be documented rather than silently reconciled.



\## Using This Guide with an LLM



When providing this repository to an LLM for assistance with LISA database administration, the authoritative guide should be supplied as the primary operating context.



The LLM should:



1\. never invent database identifiers, relationships, status meanings, or procedure behavior;

2\. use read-only queries to investigate unknown database state;

3\. verify affected rows before modifying data;

4\. keep changes uncommitted until they have been verified;

5\. inspect stored procedure source before executing unfamiliar procedures;

6\. distinguish documented facts, live observations, and inference; and

7\. preserve newly verified operational knowledge for future inclusion in the guide.



The complete operating rules are maintained in the authoritative guide.



\## Maintaining the Documentation



The guide is intended to evolve as LISA administration work reveals additional behavior.



When a new procedure or behavior is verified, document:



\- the problem or administrative task;

\- the objects and relationships involved;

\- discovery and pre-change queries;

\- required preconditions;

\- modification steps;

\- expected affected-row counts;

\- verification queries;

\- commit and rollback behavior;

\- known side effects;

\- relevant application forms, jobs, or stored procedures; and

\- whether the information is source-documented, live-verified, inferred, or historical.



Changes should be committed to Git with messages that describe the knowledge or procedure being added or corrected.



\## Important Note



LISA is a legacy system and portions of the inherited documentation describe historical infrastructure, configuration, and behavior. Server addresses, filesystem paths, Oracle roles, profiles, schedules, and other environment-dependent information should be verified against the current environment before operational use.



The live database and observed application behavior remain the ultimate sources for determining the current state of the system.

