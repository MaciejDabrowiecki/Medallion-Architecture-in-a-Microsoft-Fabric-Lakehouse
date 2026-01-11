# Medallion Architecture in Microsoft Fabric (Lakehouse)
(architecture image will be inserted here)
## Executive Summary

After identifying a need for modern, scalable data storage and analytics capabilities, and completing a thorough Proof of Concept, the company’s IT Steering Committee approved the implementation of a centralized analytics platform based on Microsoft Fabric.

The goal of this initiative was to design and validate an end-to-end Lakehouse solution that supports:

* Structured and governed data ingestion

* Scalable transformation pipelines

* Auditability and replay-safe processing

* Business-ready semantic modeling

* Executive-level reporting in Power BI

This repository documents the **architecture, design decisions, and implementation artifacts** of that solution, built using a **Medallion Architecture (Bronze → Silver → Gold)** pattern and aligned with modern data engineering best practices.

## Business Objectives

The solution was designed to address the following business needs:

* Centralize analytical data in a governed Lakehouse

* Enable reliable and repeatable ingestion pipelines

* Support historical data retention and reprocessing

* Provide a clean semantic model for self-service analytics

* Deliver executive-level reporting with consistent KPIs

* Prepare the foundation for Dev/Test/Prod deployment and CI/CD

## High-Level Architecture
**Core platform:** Microsoft Fabric

**Storage:** Lakehouse

**Modeling approach:** Star schema (Gold layer)

**Reporting:** Power BI (Fabric semantic model)

The architecture consists of:

* External data sources (CSV files simulating operational systems)

* Fabric Lakehouse for storage and processing (File directories - Bronze, Delta Tables - Silver & Gold)

* Fabric Data Pipelines for orchestration

* Notebooks (Spark) for transformation logic

* Audit and monitoring Delta Tables

* Fabric Semantic Model

* Power BI report for consumption

**Below are the Fabric items located in dedicated workspace ws_sales_analytics_dev:**

<img width="711" height="875" alt="obraz" src="https://github.com/user-attachments/assets/65619919-6810-4adc-8880-fb7dbddb4651" />


## Data Sources

The dataset is based on **AdventureWorks-style business data**, including:

* Sales (multiple years)

* Product master data (products, categories, subcategories)

* Customers

* Territories

* Calendar

* Returns

[Data stored in project GitHub repository](https://github.com/MaciejDabrowiecki/Medallion-Architecture-in-a-Microsoft-Fabric-Lakehouse/tree/main/Data)

These CSV files simulate structured extracts from upstream operational systems.

## Medallion Architecture

To create whole Lakehouse **ls_sales** architecture notebook **_nb_00_setup_lakehouse.ipynb_** was used. Result as below:

<img width="324" height="538" alt="obraz" src="https://github.com/user-attachments/assets/4e48f084-2b7a-450f-a042-1a3e3c0db2da" />
<img width="319" height="607" alt="obraz" src="https://github.com/user-attachments/assets/18448dcf-0f2d-42d4-98fb-ada4f0677692" />


### Bronze Layer – Raw Ingestion

**Purpose:**

* Capture raw data exactly as received

* Preserve history

* Enable replay and reprocessing

**Key characteristics:**

* File-based storage in Lakehouse (Files/Bronze)

* One folder per entity

* Subfolders partitioned by ingestion date and batch ID

* No transformations applied

**Ingestion design highlights:**

* Config-driven ingestion using JSON: **_bronze_ingestion_config.json_**

* Parameterized pipelines

* Dynamic target paths

* Idempotent batch processing

Bronze file directory subfolders dynamically created for each csv file, run date and specific run ID as below:

<img width="406" height="589" alt="obraz" src="https://github.com/user-attachments/assets/a94417f0-c653-4881-8feb-10adba3ffa55" />

## Audit & Monitoring Framework

To ensure enterprise-grade reliability, the solution implements **explicit audit tables:**

* audit.ingestion_batches

* audit.ingestion_files

These tables track:

* Batch start and end times

* Pipeline name and run ID

* Execution status (SUCCESS / FAILED)

* File-level ingestion status

This enables:

* Operational monitoring

* Failure analysis

* Replay-safe processing

* Historical traceability

Table ingestion_batches structure as below:

<img width="1306" height="217" alt="obraz" src="https://github.com/user-attachments/assets/5f360c9f-2618-4021-a62a-f36c3d599b9b" />

Note: For above runs pipelines were triggered manually, but in final solution the runs are scheduled and triggered automatically.

## Silver Layer – Cleansed & Conformed Data

**Purpose:**

* Clean and standardize data

* Apply business rules

* Enforce schemas

* Prepare data for analytics

**Key transformations include:**

* Data type casting

* Date normalization

* Null handling

* Column renaming and reordering

* Deduplication

* Conforming dimensions

**Loading strategy:**

* Delta tables

* Upsert (MERGE) logic

* Natural business keys

* SCD Type 1 behavior where appropriate

Silver Delta Table example(products - merged from 3 csv's) below:

<img width="1778" height="732" alt="obraz" src="https://github.com/user-attachments/assets/883d2204-0f15-4021-a2b0-abb5b92fd6f8" />

## Gold Layer – Business Model

**Purpose:**

* Provide analytics-ready data

* Implement a star schema

* Optimize for BI performance

**Model design:**

+ 2 fact tables:

  * fact_sales

  * fact_returns

+ 4 dimensions:

  * dim_date

  * dim_product

  * dim_customer

  * dim_territory

The Gold layer contains **no raw attributes**, only business-ready fields and keys required for analytics.

Below is the visual of the Star Schema - as a data model from Fabric semantic model **_sales_analytics_model_**:

<img width="987" height="602" alt="obraz" src="https://github.com/user-attachments/assets/7f8a2f8c-08fd-4e3e-bcd4-517f7fbb199e" />

## Orchestration & Pipelines

The solution uses **Microsoft Fabric Data Pipelines** for orchestration.

### Pipelines Implemented

**1. To Bronze(pl_to_bronze.sales):**

<img width="1537" height="287" alt="obraz" src="https://github.com/user-attachments/assets/40010f64-edaa-4f34-b8ff-cd9b046c2488" />


Notebook **_nb_01_register_batch.ipynb_** was used to register batch information in **audit.ingestion_batches** table. Batch ID was then passed to the pipeline **variable**. Next **Lookup** activity uses **bronze_ingestion_config.json** to find csv files and set up parameters.
Those are passed to **For Each** activity which uses them in **Copy Data** activities creating dedicated subfolders for each new file with every run. After that each new file is registered in **audit.ingestion_files** thanks to **_nb_02_log_ingestion.ipynb_** notebook.
At last, at Completion a notebook **_nb_03_finalize_batch.ipynb_** updates information about current batch - when it was completed, either succesfully or not. On the failure an email notification is send.

**2. Bronze to Silver(pl_to_silver.sales):**

<img width="1421" height="138" alt="obraz" src="https://github.com/user-attachments/assets/338b89d7-16e1-446c-b85e-7c196fec94ba" />

In this pipeline batch is handled identically. All the transformations are performed with **_nb_04_bronze_to_silver_sales.ipynb_** notebook.

**3. Silver to Gold(pl_to_gold.sales)**

<img width="1414" height="138" alt="obraz" src="https://github.com/user-attachments/assets/0260b182-282d-4135-8af4-4f57d704d1a7" />

In this pipeline batch is handled identically. All the transformations are performed with **_nb_05_silver_to_gold_sales.ipynb_** notebook.

**4. Parent Orchestration Pipeline(pl_parent_sales)**

<img width="837" height="126" alt="obraz" src="https://github.com/user-attachments/assets/a45a36e1-a853-4ad4-94c6-f2145a7fd417" />

This parent pipeline orchestrates other pipeline by invoking them.

## Parameterization & Batch Handling

A key design focus was **parameter-driven orchestration**:

+ Pipelines parameters and varaibles:

  * pipeline_name

  * pipeline_run_id

  * batch_id

+ Notebooks use Fabric parameter cells

+ Batch ID is generated once and reused across activities

This ensures:

* Consistent batch tracking

* Safe re-execution

* Clear lineage across layers

<img width="710" height="328" alt="obraz" src="https://github.com/user-attachments/assets/0e184738-1df3-4d3c-b410-48c8a7438b2e" />

## Semantic Model

A Fabric Semantic Model was created on top of the Gold layer.

**Design principles:**

* One semantic model per business domain

* Measures defined centrally

* Relationships managed in the model (not reports)

(Data model shown earlier under Architecture section)

## Power BI Report (Sales Performance Overview)

A minimalistic executive-level Power BI report was built on top of the semantic model.

**Report characteristics:**

* Single executive overview page

* KPI cards (Sales, Returns, Volumes)

* Time-series trend analysis

* Geographic analysis (map)

* Product performance matrix

* Toggleable filter pane for slicers

Below a snapshot from executive report:

<img width="1432" height="803" alt="obraz" src="https://github.com/user-attachments/assets/00290110-c811-406b-bba5-bcced3d3cf5a" />

## Deployment Strategy

A **Fabric Deployment Pipeline** is used to move artifacts across environments - each being a dedicated workspace - all under one Domain:

* Development

* Test

* Production

Artifacts include:

* Lakehouse

* Notebooks

* Pipelines

* Semantic model

* Reports

<img width="1580" height="857" alt="obraz" src="https://github.com/user-attachments/assets/447590cf-f827-4869-984d-81576b683a0a" />


## Security & Governance

While not fully implemented in this PoC, the architecture accounts for:

* Azure Key Vault for secrets management

* Workspace-level access control and Role-based access

* Separation of environments

* Audit logging

## Limitations & Fabric Constraints

Due to current Microsoft Fabric limitations:

* Pipelines cannot be exported as code

* Semantic models cannot be versioned

* Reports created in Fabric cannot be downloaded as PBIX

These are addressed through:

* Extensive documentation

* Architecture diagrams

* Notebook-first development

## Future Enhancements & Enterprise-Scale Extensions

While this project delivers a robust and production-ready Proof of Concept in Microsoft Fabric, the architecture intentionally allows for further enterprise-scale enhancements beyond the current scope.

One key future improvement would be the introduction of **Infrastructure as Code (IaC)** using tools such as **Terraform** or **Bicep**. IaC would enable automated, repeatable provisioning of Fabric workspaces, Lakehouses, pipelines, and supporting Azure resources, improving environment consistency across Development, Test, and Production while reducing manual configuration risks.

From a governance perspective, integration with **Microsoft Purview** would enhance metadata management, lineage visibility, and data discovery across Lakehouse layers. Combined with row-level security and sensitivity labeling, this would further strengthen compliance and enterprise trust.

Performance and cost optimization could be expanded through **incremental data processing**, **partitioning strategies**, and **semantic model optimizations**, ensuring scalable performance as data volumes grow.

Finally, the platform could be extended to support **advanced analytics and machine learning workloads**, leveraging Fabric notebooks for feature engineering and model training, with results published back into the Gold layer for business consumption in Power BI.

## Conclusion

This project demonstrates an **enterprise-grade data platform design** using Microsoft Fabric, covering ingestion, transformation, modeling, orchestration, governance, and analytics.

It reflects real-world data engineering practices rather than simplified tutorials and is intended to showcase architectural thinking, not just tooling familiarity.

Note: LLM was used in couple of README sections for rephrasing purposes.

## Author

Maciej Dąbrowiecki

Data Engineering Enthusiast/ Analytics Engineer and PMO
