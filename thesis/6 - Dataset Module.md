# Dataset Module

## Plan

- Dataset model
  - Required file structure
  - Output data structures
- Used datasets
  - NIST's C Test Suite
  - Juliet
  - Toy
- Technologies
  - Docker
  - gcc
- CLI

## Existent Content

\item \textbf{Dataset Module} \cite{dataset_repo}: We created hundreds of vulnerable executables from two open source test suites, namely NIST's C Test Suite \cite{nist_c_test_suite} and Juliet \cite{juliet_testcase}. These we're wrapped into a standardized folder structured and managed by a Python 3 script that deals with the compilation and their filtering based on CWEs.

> Dataset model
> From first report
