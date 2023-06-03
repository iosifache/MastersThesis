# Dataset Module

The first module of the cyber reasining system is the dataset one, whom purpose is the create and manage **datasets of vulnerable programs** that can be analyzed by the next modules. A further condition for the resulted binaries is that they should be **tagged** with information related to their vulnerabilities. This will help in computing the accuracy of the entire system, namely by comparing the vulnerabilities that are (correctly or incorrectly) detected by OpenCRS and the intended ones, as labeled in the initial datasets.

> **TODO**: Insert diagram here

## Architecture

### Embedded Test Suites

We started by researching the public domain for **well-known test suites with vulnerable C programs**. As OpenCRS analyzes entire executables with static and dynamic analysis techniques, we filtered out the datasets that contain non-compilable code. For example, some articles in the Academia [dataset.1] are using only the code changes introduced by a commit, property which is not suitable for the **executable-level, black-box analysis** performed by our CRS. On the other hand, we decided to use programs that were tagged with **precise vulnerability tags**, for example by generating syntetic, vulerable-by-design programs. This contradicts the approach of using a static code analyzer (like SonarCloud [dataset.8]) to detect potential vulnerabilities in code and to produce the labels.

Created in a collaboration between the National Institute of Standards and Technology and Alexander Hoole from University of Victoria, Canada, **C Test Suite for Source Code Analyzer v2 - Vulnerable** [dataset.2] is one of the test suite integrated in the dataset modules. It comprises 54 C source codes and a manifest file. The latter respects the XML format, and has a `<testcase>` entry for each program in the test suite, with the following relevant child tags. As each program has a vulnerability, it is mentioned in each tags present on the XPath `/testcase/file[dataset.0]/flaw` and described in the `line` (i.e. the line on which the security issue arrises) and `name` (which is, in fact, the Common Weakness Enumeration entry).

Despite the test suite having attached the build command in `/testcase/@instruction`, we preffered to move the build logic into a separate `Makefile`. This permits the executables creation without initially processing the XML file. Beside the test suite, we published this change to a separate repository [dataset.4] on our GitHub organization. This permitted us to attach it as a submodule to dataset repository.

The second test suite is **Juliet C/C++ 1.3** [dataset.5], already available as a GitHub repository [dataset.6]. It is created by the same American institution, but in collaboration with the National Security Agency's Center for Assured Software. It consists of 64123 vulnerable test cases and an XML manifest file, with small differences than the previously-mentioned one (e.g. not having the build command in `/testcase/@instruction`).

During the testing of other modules, the need of having custom binaries increased. We ended up by creating a separate, **custom test suite** in the dataset module, `toy_test_suite`, with a basic folder structure: a source with vulnerable code and a `cwe.txt` file which has the corresponding CWE ID. We placed there small executables, like stack buffer overflows via `stdin` input and `NULL` pointer dereferencing triggered by program arguments.

### Test Suites Building. Results Retrieval

These three datasets were linked as submodules in the `raw_testsuites` folder of the `dataset` repository [dataset.7]. For each of them, a separate class (named **parser**) is created in the `dataset.parsers` module. As it should inherit the `BaseParser` class, the class should implement the following abstract methods:

- `_get_all_sources`: Parses the folder structure of the dataset in `raw_testsuites/<testsuite_id>`, and returns a list of `Source` objects. The latter contains information such as the full path of the source file, and the embedded CWEs.
- `preprocess`: Having the `Source` objects already created, the method deals with preprocessing the sources with **`gcc`**. The resulted files are placed in the `sources` folder.
- `_generate_gcc_command`: Based on the preprocessed sources in `sources`, uses `gcc` to compile the executables in `executables`. Further details are placed in a CSV file, `vulnerables.csv`, which has columns indicating the executable ID, the test suite it cames from, its CWEs, and a boolean indicating if it is built or not.

All `gcc`-related operations (both preprocessing and building) are executed in a **Docker container** having Ubuntu as operating system. This provides replicability of the building process, and isolation from the host operating system. We prefer to communicate with the container by leveraging volumes (for sharing files) and Docker API calls. This suited the scenario better than having a gRPC service because the module's code is on the same host with the build container and the communciation delays are minimized by avoiding to send large files (e.g. the sources and executables) over (a virtualized) network.

When the build functionality is triggered from command-line interface or by calling the specific method from the Python library, a parsers manager is activated. It selects the required parsers, and calls the preprocessing and building methods. The results of the process can be queried by using other operation, in which `vulnerable.csv` is parsed again.

## Testing

All modules were set into an Ubuntu virtual machine, were we could verify their functionality separately and conjunctively, by aggregating them in OpenCRS.

For the dataset module, we leveraged the CLI to build the test suites and query the dataset:

```bash
➜ poetry run dataset build --testsuite TOY_TEST_SUITE
Successfully built 5 executables.

➜ poetry run dataset get
The available executables are:

┏━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ ID               ┃ CWEs                        ┃ Parent Database ┃ Full Path                        ┃
┡━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ toy_test_suite_0 │ Stack-based Buffer Overflow │ toy_test_suite  │ executables/toy_test_suite_0.elf │
│ toy_test_suite_1 │                             │ toy_test_suite  │ executables/toy_test_suite_1.elf │
│ toy_test_suite_2 │ NULL Pointer Dereference    │ toy_test_suite  │ executables/toy_test_suite_2.elf │
│ toy_test_suite_3 │ NULL Pointer Dereference    │ toy_test_suite  │ executables/toy_test_suite_3.elf │
│ toy_test_suite_4 │                             │ toy_test_suite  │ executables/toy_test_suite_4.elf │
└──────────────────┴─────────────────────────────┴─────────────────┴──────────────────────────────────┘
```
