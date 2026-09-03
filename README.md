# AML Transaction Analysis

Data profiling, integrity validation, wrangling and analysis on a financial transactions dataset, aimed at anti-money laundering (AML) analysis.

This is an ongoing project. The goal is to explore a large financial-transactions dataset and, ultimately, to detect potentially suspicious (money-laundering) transactions — building each phase on a rigorously validated foundation.

## Project phases

The project is organized in phases. Each phase has its own write-up (the reasoning and findings) and shares the accompanying notebook (the code and outputs).

[Part I — Profiling and Integrity Validation](01-profiling-and-integrity-validation.md) | Structural exploration of both tables: cardinality profiling, key identification, functional dependencies, and cross-table referential integrity. Validates the data is structurally sound before modeling. |
[Part II — Cleaning](02-cleaning.md) | Structural arrangements, data-type conversion, filtering (scope), string cleaning, and null/duplicate handling. Prepares the data for the next phase. |
[Part III — Typology Analysis: Fan-In](03-typology-analysis-fan-in.md) | Detection of the fan-in typology using a temporal definition: deriving a time threshold from the data, clustering incoming transactions with DBSCAN to isolate bursts, and comparing the results against the simulator's own structural labels. |
Part IV — Typology Analysis: Fan-Out | *(upcoming)* |

The full write-up for Part III, including all charts and output screenshots, is also available as **III Typology Analysis.pdf** in this repository.

## Dataset

The dataset used is the **IBM Transactions for Anti-Money Laundering (AML)** synthetic dataset by Erik Altman, available on Kaggle:

https://www.kaggle.com/datasets/ealtman2019/ibm-transactions-for-anti-money-laundering-aml

This project uses the **LI-Small** variant — specifically the `LI-Small_accounts.csv` (accounts table) and `LI-Small_Trans.csv` (transactions table) files.

Due to its size (millions of rows), the dataset is not included in this repository. To reproduce the analysis, download the files from the link above and place them alongside the notebook (or update the file paths in the notebook accordingly).

## Tools

Python · Pandas · scikit-learn (DBSCAN) · Power BI

