# Testing and Assesment

This chapter will outline the functional testing and assessment of the OpenCRS modules described earlier.

A virtual machine running **Ubuntu 22.04** was utilized for the purposes of testing and assessment. The setup guide for each module was adhered to, thereby ensuring that all prerequisites, such as Python libraries, Docker, and built images, were satisfied.

Although each module has the capability to function as a Python library, we have opted to utilize the **command line interface** of each module due to its superior appropriateness for manual testing. Upon integration of the modules into the OpenCRS, a Python library will be utilized under an orchestration module.

## Dataset Generation

For the dataset module, we have built all three integrated test suites, which are identified in the OpenCRS context by a unique ID: `nist_juliet`, `nist_c_test_suite`, and `toy_test_suite`. For this specific task, the `built` command was used in conjunction with the `get` one, which queries the index of built executables.

```bash
➜ opencrs-dataset build --testsuite NIST_C_TEST_SUITE
Successfully built 50 executables.
```

```bash
➜ opencrs-dataset get
The available executables are:

┏━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ ID                    ┃ CWEs                                       ┃ Parent Database    ┃ Full Path                             ┃
┡━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ nist_c_test_suite_113 │ Use of Externally-Controlled Format String │ nist_c_test_suite  │ executables/nist_c_test_suite_113.elf │
│ nist_c_test_suite_133 │ Use of Externally-Controlled Format String │ nist_c_test_suite  │ executables/nist_c_test_suite_133.elf │
[...]
```

This process resulted in a collection of 54591 vulnerable ELF executables, compiled from C sources and targetting the 32-bit i386 architecture. They were made available to the open source community by creating a setarate repository, `opencrs_dataset` [1].

The distribution of the executables per test suite (`54531` for `nist_juliet`, `50` for `nist_c_test_suite` and `10` for `toy_test_suite`) shows that compilation erorrs occured during the creation of the executables. We have stored all build logs and concluded that the main causes are either missing headers (e.g. `mysql/mysql.h`, `security/pam_appl.h`) and incompatibilties between Windows and Linux (e.g. usage of types like `wchar_t`).

The details of all executables were stored in a CVE file named `index.csv`, with the following labels:

- `name`: Unique identifier of a vulnerable program. It is used to determine the executable file path, namely by using the format `executables/<name>.elf`;
- `cwes`: One or more CWEs that are present in the executable; and
- `parent_dataset`: Parent dataset's identifier.

Lastly, it should be mentioned that the used test suites does not adhere to an uniform executables distribution over each vulnerability type, identified by the CWE ID.

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

## Arguments Dictionaries Generation

The next tested and evaluated module was the attack surface discovery one.

By leveraging the `generate` command and all three implemented heuristics, we could create dictionaries of arguments, which could be used for arguments fuzzing:

- `generation`: `62` entries with arguments like `-a`, `-D`, and `-9`
- `man_parsing`: `6701` entries with arguments like `--ascii`, `--cert-status`, and `--message-id`
- `binary_pattern_matching` for `/usr/bin/uname`: `37` enties with arguments like `--operating-system`, `--processor` and `-linux-x86-64`. As it can be observed, the precision is non-unitary as not all returned strings are valid arguments. This happens because some string in the binaries respects the Regex for arguments. But this is not a blocker, as the arguments will be used in two more filtering stages afterward, namely the arguments fuzzing and in the vulnerability detection module.

```bash
➜ opencrs-surface generate --heuristic binary_pattern_matching --elf /usr/bin/uname --output uname.dict
Successfully generated dictionary with 37 arguments
➜ head -3 uname.dict
--operating-system
--processor
--version
```

## Arguments Fuzzing

In the same module, the `fuzz` command was tested and evaluated against a Linux utility for retrieving information about the current host and operating system, namely `uname`. As a dictionary, the previously generated one was used.

```bash
➜ opencrs-surface fuzz --elf /usr/bin/uname --dictionary uname.dict
Several arguments were detected for the given program:

┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━┓
┃ Argument                   ┃      Role      ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━┩
│ -                          │      FLAG      │
│ --all                      │      FLAG      │
│ --all string               │ STRING_ENABLER │
│ --hardware-platform        │      FLAG      │
│ --hardware-platform string │ STRING_ENABLER │
[...]
```

`-`, `--all`, `--hardware-platform`, `--kernel-name`, `--kernel-release`, `--kernel-version`, `--machine`, `--nodename`, `--operating-system`, and `--processor` were detected as valid arguments of the programs with two roles: `FLAG` (valid, as they are activating functionality), and `STRING_ENABLER` (invalid, as they are not requiring the user to provide any string afterward). By inspecting the results returned by QBDI, it can be concluded why this happens: the generated DJB2 hash for `-a` (i.e. `-939574273`) and for `-a <string>` (i.e. `657648750`) are different.

On the other hand, `-` is not a valid argument, and the agument `--version`, which was initially included in the generated dictionary, was not detected as valid. This may happen due to `uname`'s implementation, as the code flow specific to this option is not detected by QBDI as a basic block transfer.

A last observation for this functionality of the attack surface approximation module is aliases detection. `uname` accepts for all its arguments (the ones listed before) a short form. By taking `--all` as an example and continuing the QBDI results' inspection, it can be seen that its hash when tested for the flag role is `-939574273`, which is equal to the DJB2 hash of the `-a` argument. As the OpenCRS's module detects this colision, it will include only the first argument from the dictionary detected with this specific hash (in this case, `--all`).

## Input Streams Detections

We have used four executables from the toy test suite to assess the functionality of the input streams detection. All of them had at least one input stream from the set composed of `stdin`, files, and arguments.

```bash
➜ opencrs-surface detect --elf null_pointer_deref_args.elf
Several input mechanisms were detected for the given program:

┏━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━┓
┃ Stream               ┃ Present ┃
┡━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━┩
│ STDIN                │   No    │
│ ARGUMENTS            │   Yes   │
│ FILES                │   No    │
│ ENVIRONMENT_VARIABLE │   No    │
│ NETWORKING           │   No    │
└──────────────────────┴─────────┘
```

The below table summarizes the results.

| Executable                    | Arguments Stream | Files Stream  | stdin Stream  |
|-------------------------------|------------------|---------------|---------------|
| null_pointer_deref_args.elf   | Detected (TP)    | N/A           | N/A           |
| null_pointer_deref_files.elf  | Detected (TP)    | Detected (TP) | Detected (FP) |
| null_pointer_deref_stdin.elf  | Detected (TP)    | Detected (FP) | N/A           |
| multiple_inputs_streams.elf   | Detected (TP)    | Detected (TP) | Detected (TP) |

In case of two programs, the file and `stdin` streams were incorrectly detected as present. This happens because the programs are using reading library calls, which are not specific to one or another. They are able in fact to read from a file pointer, so they are able to handle from both `stdin` (the `0` descriptor) and files (the descriptor returned by calls like `open` and `fopen`). From this reason, the modules preffer to favor false positives (streams that are not in fact real, which can be validated further in the vulnerability detection module) and not false negatives (streams that are use, but not reported).

## Vulnerability Detection

For detecting vulnerability with the three implemented fuzzers, we have choosed three programs crashing (by different causes) when a specific input was provided on an supported input streams. The vulnerabilities were:

- Tainted format string: If `%s` was provided as an (input) format string, then a `NULL` pointer dereferencing occured. Another attack was providing `%n`, which tries to write to the same `NULL` pointer.
- Stack buffer overflow: If the input has a sufficient length, then the saved EBP and EIP registers from the stack were overwritten. On return, the program will crash because the values are invalid.
- NULL pointer dereferencing.

```bash
➜ opencrs-vulnerability fuzz --fuzzer FILES_AFLPLUSPLUS --stream FILES --elf tainted_format_string_files.elf
New proof of vulnerability was generated with the following payloads:
- For FILES:

00000000: 6E 25 6E 25                                       n%n%                                           .
```

In this example, the NULL pointer dereferencing and the further crash were triggered by the input generated by afl++, namely the `%n`. In the other two cases, the fuzzer successfully generated the crashing inputs namely a long input for the executable vulnerable of buffer overflow and another having `y` on the fifth position for the one vulenrable of NULL pointer dereferencing.

## Automatic Exploit Generation

For the last functionality, the automatic exploit generation module was tested with three binary provided by Zeratool, all with `stdin` as an input stream and vulnerable by buffer overflow, but with different mitigations:

1. NX, canaries, and PIE disabled; and
2. NX enabled + canaries and PIE disabled.

In addition, one executable had a sensitive function that could be called in the exploit (as an alternative to other symbols, for example). The module successfully generated exploits in all cases.

```bash
➜ opencrs-aeg recommend --elf bof_nx_32 --stream=STDIN --mitigation=NX --weakness=STACK_OUT_OF_BOUND_WRITE
Exploiters that can be used considering the context are:
- ZERATOOL
```

```bash
➜ opencrs-aeg exploit --exploiter ZERATOOL --elf bof_nx_32 --stream STDIN --mitigation NX --weakness STACK_OUT_OF_BOUND_WRITE
The exploiter could generate an exploit with the outcome of SHELL and the following payloads:

- For STDIN:
[...]
\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x90\x90\x04\x08baaa\x1e\xa0\x04\x08\x00\x00\x00\x00\x00
[...]
```
