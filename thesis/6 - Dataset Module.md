# Dataset Module

The initial component of the cyber reasoning system is the dataset module. The aim of this component is to create and manage collections of vulnerable software programs utilizing public test suites.

An additional prerequisite for the resulting binary files is that they must be labeled with appropriate information pertaining to their vulnerabilities. This will aid in calculating the overall system's accuracy by comparing the vulnerabilities found (correctly or wrongly) by OpenCRS with the intended ones, as designated in the initial datasets.

## Architecture

### Embedded Test Suites

We began by scanning the public domain for well-known test suites that had insecure C applications. We filtered out datasets that contained non-compilable code because OpenCRS analyzes full executables using static and dynamic analysis techniques. Some articles in Academia [dataset.1], for example, use just the code modifications introduced by a commit, a property that is incompatible with the executable-level, black-box analysis done by our CRS. On the other side, we chose to use programs that had been precisely tagged with vulnerability tags, such as by constructing synthetic, vulnerable-by-design programs. This contradicts the strategy of utilizing a static code analyzer (such as SonarCloud [dataset.8]) to find and label potential vulnerabilities in code.

C Test Suite for Source Code Analyzer v2 - Vulnerable [dataset.2], developed in partnership between the National Institute of Standards and Technology and Alexander Hoole of the University of Victoria in Canada, is one of the test suites integrated into the dataset modules. It is made up of 54 C source codes plus a manifest file. The latter adheres to the XML standard and includes a `<testcase>` entry for each program in the test suite, as well as the required child tags listed below. Because each program has a vulnerability, the latter is stated in each tag on the XPath `/testcase/file[]/flaw` and described in `line` (i.e. the line where the security issue occurs) and `name` (which is, in fact, the Common Weakness Enumeration item).

Despite the fact that the build command was attached in `/testcase/@instruction`, we chose to relocate the build logic into a separate `Makefile`. This allows the executable to be created without first parsing the XML file. We also published this revision to a separate repository [dataset.4] on our GitHub organization, in addition to the test suite. This allowed us to add it to the dataset repository as a submodule.

The second test suite is Juliet C/C++ 1.3 [dataset.5], which is already available on GitHub [dataset.6]. The same American institution built it, but in partnership with the National Security Agency's Center for Assured Software. It consists of 64123 susceptible test cases and an XML manifest file with minor variations from the previous one (for example, not having the build command in `/testcase/@instruction`).

The necessity for custom binaries became more apparent during the testing of other components. We ended up building an additional custom test suite in the dataset module called `toy_test_suite` with a simple folder structure: a source containing vulnerable code and a `cwe.txt` file with the associated CWE ID. Small executables such as stack buffer overflows, `NULL` pointer dereferences, and tainted format strings were included here.

### Test Suites Building. Results Retrieval

These three datasets were linked as submodules in the `dataset` repository's `raw_testsuites` folder [dataset.7]. In the `dataset.parsers` module, a distinct class (called parser) is created for each of them. The class should implement the following abstract methods because it should inherit the `BaseParser` class:

- `_get_all_sources`: Parses the folder structure of the dataset in `raw_testsuites/<testsuite_id>` and returns a list of `Source` objects. The latter contains information such as the full path of the source file and the embedded CWEs.
- `preprocess`: Having the `Source` objects already created, the method deals with preprocessing the sources with `gcc`, eventually by considering custom preprocessing flags. The resulting files are placed in the `sources` folder.
- `_generate_gcc_command`: Based on the preprocessed sources in `sources` and additional compilation flags, uses `gcc` to compile the executables in `executables`. Further details are placed in a CSV file, `vulnerables.csv`, which has columns indicating the executable ID, the test suite it comes from, its CWEs, and a boolean indicating if it is built or not.

All `gcc`-related operations (both preprocessing and building) are carried out in a Docker container running Ubuntu. This allows for the replication of the building process as well as separation from the host operating system. We like to communicate with the container using volumes (for file sharing) and Docker API calls. This was preferable to using a gRPC service because the module's code resides on the same host as the build container, and communication delays are reduced by avoiding transferring large files (e.g. the sources and executables) via (a virtualized) network.

A parsers manager is enabled when the build functionality is started from the command-line interface or by invoking the specified method from the Python module. It chooses the necessary parsers and invokes the preprocessing and construction techniques. The process's output, specifically `Executable` objects, can be queried using other procedures, in which `vulnerable.csv` is parsed once again.
