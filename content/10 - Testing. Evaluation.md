# Testing and Assesment

The forthcoming chapter will delineate the process of functional testing and evaluation of the OpenCRS modules that were previously explicated.

For testing and evaluation, a virtual machine was employed, which was configured to run Ubuntu 22.04. The guidelines for configuring each module were followed, thus guaranteeing the fulfillment of all requirements, including Python libraries, Docker, and built images.

Even though every module possesses the ability to operate as a Python library, we have chosen to employ the command line interface of each module owing to its superior suitability for manual testing. The integration of the modules into the OpenCRS will involve the utilization of a Python library within an orchestration module.

# Dataset Generation

For the dataset module, we have constructed all three integrated test suites, which are identifiable in the OpenCRS context by a unique ID: `nist_juliet`, `nist_c_test_suite`, and `toy_test_suite`. The present task involved the utilization of the `built` command in combination with the `get` command, which serves to retrieve the index of the built executables.

```bash
$ opencrs-dataset build --testsuite NIST_C_TEST_SUITE
Successfully built 50 executables.
```

```bash
$ opencrs-dataset get
The available executables are:

-----------------------------------------------------------------------------------------------------------------------------------
| ID                    | CWEs                                       | Parent Database    | Full Path                             |
|-----------------------|--------------------------------------------|--------------------|---------------------------------------|
│ nist_c_test_suite_113 │ Use of Externally-Controlled Format String │ nist_c_test_suite  │ executables/nist_c_test_suite_113.elf │
│ nist_c_test_suite_133 │ Use of Externally-Controlled Format String │ nist_c_test_suite  │ executables/nist_c_test_suite_133.elf │
[...]
```

The outcome of this procedure yielded a set of 54591 vulnerable ELF executables, which were generated from C source code and designed for the 32-bit i386 architecture. The open-source community was granted access to the dataset through the establishment of a distinct repository, namely `opencrs_dataset`.

The observed distribution of executables across the test suites, namely `54531` for `nist_juliet`, `50` for `nist_c_test_suite`, and `10` for `toy_test_suite`, suggests the presence of compilation errors during the executables' generation process. Upon analyzing the build logs, it has been determined that the primary factors contributing to the issue are the absence of essential headers such as `mysql/mysql.h` and `security/pam_appl.h`, as well as inconsistencies between the Windows and Linux operating systems, such as the utilization of data types such as `wchar_t`.

The executable details were recorded in a CVE file denominated as `index.csv`, featuring the subsequent labels:

- `name`: Unique identifier of a vulnerable program. The format `executables/<name>.elf` is utilized to determine the path of the executable file.
- `cwes`: One or more CWEs that are present in the executable; and
- `parent_dataset`: Parent dataset's identifier.

Finally, it is noteworthy that the employed test suites do not conform to a consistent distribution of executables across each vulnerability type, as identified by the Common Weakness Enumeration (CWE) ID.

| Weakness                                   | Count   |
| ------------------------------------------ | ------- |
| Stack-based Buffer Overflow                | `13836` |
| Heap-based Buffer Overflow                 | `11088` |
| Integer Overflow or Wraparound             | `3960`  |
| Mismatched Memory Management Routines      | `3564`  |
| Integer Underflow                          | `2952`  |
| Free of Memory not on the Heap             | `2680`  |
| Use of Externally-Controlled Format String | `2410`  |
| Buffer Underflow                           | `2048`  |
| Buffer Under-read                          | `2048`  |
| OS Command Injection                       | `1921`  |

# Arguments Dictionaries Generation

Subsequently, the module subjected to testing and evaluation was the attack surface discovery module.

Through utilization of the `generate` command and the three implemented heuristics, it is possible to generate dictionaries of arguments that may be employed for the purpose of argument fuzzing:

- `generation`: `62` entries with arguments like `-a`, `-D`, and `-9`
- `man_parsing`: `6701` entries with arguments like `--ascii`, `--cert-status`, and `--message-id`
- `binary_pattern_matching` for `/usr/bin/uname`: `37` enties with arguments like `--operating-system`, `--processor` and `-linux-x86-64`. As evident from the observation, the precision is not unitary since not all the strings returned are valid arguments. This phenomenon occurs due to the adherence of certain character sequences within the binary files to the regular expression pattern for arguments. However, this does not present an impediment, as the aforementioned arguments will undergo two additional filtering stages, specifically the argument fuzzing and the vulnerability detection module.

```bash
$ opencrs-surface generate --heuristic binary_pattern_matching --elf /usr/bin/uname --output uname.dict
Successfully generated dictionary with 37 arguments
$ head -3 uname.dict
--operating-system
--processor
--version
```

# Arguments Fuzzing

The `fuzz` command was subjected to testing and evaluation in the identical module, in comparison to the `uname` Linux utility, which is utilized for obtaining details regarding the present host and operating system. The previously generated dictionary was utilized for reference purposes.

```bash
$ opencrs-surface fuzz --elf /usr/bin/uname --dictionary uname.dict
Several arguments were detected for the given program:

-----------------------------------------------
| Argument                   |      Role      |
|----------------------------|----------------|
│ -                          │      FLAG      │
│ --all                      │      FLAG      │
│ --all string               │ STRING_ENABLER │
│ --hardware-platform        │      FLAG      │
│ --hardware-platform string │ STRING_ENABLER │
[...]
```

`-`, `--all`, `--hardware-platform`, `--kernel-name`, `--kernel-release`, `--kernel-version`, `--machine`, `--nodename`, `--operating-system`, and `--processor` were identified as valid program arguments with two roles: `FLAG` (valid since they activate functionality) and `STRING_ENABLER` (invalid because they do not require the user to give any string after that). Upon examining the outcomes produced by QBDI, it can be inferred that the reason for this occurrence is due to the fact that the DJB2 hash generated for `-a` (i.e. `-939574273`) and for `-a <string>` (i.e. `657648750`) are different.

Conversely, the hyphen symbol (`-`) does not constitute a legitimate argument, and the argument `--version`, which was originally incorporated in the produced dictionary, was not identified as a valid one. The observed occurrence could be attributed to the implementation of `uname`, as QBDI fails to recognize the code flow related to this option as a basic block transfer.

The final aspect to note regarding the functionality of the attack surface approximation module pertains to the detection of aliases. The `uname` command has the capability to accept abbreviated forms for all of its preceding arguments. By way of illustration using the `--all` option and further scrutinizing the QBDI outcomes, it is observable that the hash value of `-939574273` is obtained when the flag role is tested, corresponding to the DJB2 hash of the `-a` parameter. Upon collision detection by the OpenCRS module, it will selectively incorporate solely the initial argument from the dictionary that has been detected with the corresponding hash (namely, `--all`).

# Input Streams Detections

Four executables from the toy test suite were utilized to evaluate the efficacy of input stream detection. Each of the aforementioned programs possessed a minimum of one input stream originating from a collection consisting of `stdin`, files, and arguments.

```bash
$ opencrs-surface detect --elf null_pointer_deref_args.elf
Several input mechanisms were detected for the given program:

----------------------------------
| Stream               | Present |
|----------------------|---------|
│ STDIN                │   No    │
│ ARGUMENTS            │   Yes   │
│ FILES                │   No    │
│ ENVIRONMENT_VARIABLE │   No    │
│ NETWORKING           │   No    │
└──────────────────────┴─────────┘
```

The following table presents a summary of the outcomes.

| Executable                    | Arguments Stream | Files Stream  | stdin Stream  |
|-------------------------------|------------------|---------------|---------------|
| null_pointer_deref_args.elf   | Detected (TP)    | N/A           | N/A           |
| null_pointer_deref_files.elf  | Detected (TP)    | Detected (TP) | Detected (FP) |
| null_pointer_deref_stdin.elf  | Detected (TP)    | Detected (FP) | N/A           |
| multiple_inputs_streams.elf   | Detected (TP)    | Detected (TP) | Detected (TP) |

In two programs, the file and `stdin` streams were wrongly recognized as present. This phenomenon occurs due to the utilization of reading library calls by the programs, which lack specificity towards any particular one. It is noteworthy that these entities possess the capability to read from a file pointer, thereby enabling them to manage input from both `stdin` (the `0` descriptor) and files (the descriptor obtained through invocations such as `open` and `fopen`). For this reason, the modules tend to prioritize the promotion of false positives (streams that are not actually genuine but can be confirmed through the vulnerability detection module) over false negatives (streams that are authentic but remain unreported).

# Vulnerability Detection

To identify vulnerabilities using the three implemented fuzzers, we selected three programs that experienced crashes due to distinct causes when provided with a particular input on supported input streams. The identified weaknesses or susceptibilities were:

- Tainted format string: If `%s` was used as an input format string, a `NULL` pointer dereferencing occurred. Another attack used `%n` to try to write to the same `NULL` pointer.
- Stack buffer overflow: If the input is long enough, the saved EBP and EIP registers on the stack are overwritten. The program will crash when it returns because the values are invalid.
- NULL pointer dereferencing.

```bash
$ opencrs-vulnerability fuzz --fuzzer FILES_AFLPLUSPLUS --stream FILES --elf tainted_format_string_files.elf
New proof of vulnerability was generated with the following payloads:
- For FILES:

00000000: 6E 25 6E 25                                       n%n%                                           .
```

The input supplied by afl++, namely the `%n`, prompted the NULL pointer dereferencing and subsequent crash in this example. The fuzzer successfully created the crashing inputs in the other two cases, notably a long input for the executable prone to buffer overflow and another with `y` on the fifth place for the one vulnerable to NULL pointer dereferencing.

# Automatic Exploit Generation

The automatic exploit generating module was tested for the final functionality with three binary given by Zeratool, all having `stdin` as an input stream and vulnerable to buffer overflow, but with various mitigations:

1. Two with NX, canaries, and PIE disabled; and
2. One with NX enabled + canaries and PIE disabled.

Furthermore, one executable contained a sensitive function that could be used in the exploit (as a substitute to, say, other symbols). In every example, the module effectively developed exploits.

```bash
$ opencrs-aeg recommend --elf bof_nx_32 --stream=STDIN --mitigation=NX --weakness=STACK_OUT_OF_BOUND_WRITE
Exploiters that can be used considering the context are:
- ZERATOOL
```

```bash
$ opencrs-aeg exploit --exploiter ZERATOOL --elf bof_nx_32 --stream STDIN --mitigation NX --weakness STACK_OUT_OF_BOUND_WRITE
The exploiter could generate an exploit with the outcome of SHELL and the following payloads:

- For STDIN:
[...]
\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x90\x90\x04\x08baaa\x1e\xa0\x04\x08\x00\x00\x00\x00\x00
[...]
```
