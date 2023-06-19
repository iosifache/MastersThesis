# Attack Surface Approximation Module
Aside from the internal procedures that are automated, the software should have ways of interacting with the outside world, either the environment or the users. The following are the most commonly utilized communication channels:

- `stdin` (standard input);
- Arguments;
- Files;
- Network packets;
- System and library calls;
- Graphical user interface (GUI); and
- Environment variables.

When data is sent with malicious intent, the input streams are referred to as attack vectors. They constitute the attack surface: the entire interface of the executables with the outside world, which can be exploited by a hostile actor.

Before looking for vulnerabilities in OpenCRS, it should be aware of the attack vectors. This observation serves as the foundation for the following module in OpenCRS' pipeline, the attack surface approximation. With a binary as input (eventually fed from the dataset module), it discovers attack vectors that can be used by the CRS's subsequent modules.

As previously stated, the variety of input sources is extensive. In our proof of concept, we simply used three of the most common: `stdin`, files, and arguments.

## Theoretical Considerations

The paper [2] begins by introducing the concept of front line functions, or functions adjacent to a source of input. [1] expands on this concept by introducing the concept of direct entry point: a method that includes a call to one of the specified input methods (for example, `read` from `unistd.h`). Despite the fact that the papers give theoretical notions that are analogous to the input streams discussed in this thesis, they do not provide any instrument or approach for identifying these input entrances in processes. It should also be noted that the papers approach this topic by studying the source code.

This CRS module, on the other hand, implies creating valid arguments that can be used in later modules (for example, in vulnerability identification via fuzzing). Papers such as [3][4][5] propose argument identification, but only with extensive understanding of the investigated software. To parse, they require a grammar, a `getopt` option parsing call, or a man page.

The only implementations identified were in AFL [6], afl++ [7], and ManFuzzer [8]. The first two have the `argv_fuzzing'`mode, which inserts the generated random bytes into the main function's `argv`. The third piece of software generates fuzzing sequences by scanning the man page and the output of programs run with the `-h`, `-H`, and `--help` options. The downside of this technique is that it does not take into account the typical format of an input (e.g. `-letter_or_word>` or `--letter_or_word>`). For the latter, there is an inherent risk that the software lacks a help page or has hidden arguments (e.g. experimental ones).

## Architecture

The purpose of the module in detecting the input streams was divided into numerous tasks, implying distinct technologies.

### Indicators Discovery

The first objective is to statically examine the binary for evidence (dubbed indications) that a particular input stream is being used. It should be emphasized that the presence of an indicator is sufficient but not required. Despite the fact that the program does not call `getenv` internally, we cannot guarantee that the environment variables are not used altogether because the program may employ an unusual method of parsing its stack to acquire that information.

This analysis aids in the activation of features in the pipeline's subsequent modules in OpenCRS. For example, if the executable does not use its CLI arguments, a fuzzer altering some (non-existent) arguments is pointless and will yield no significant findings.

We accomplish this by utilizing the API of a well-known reverse engineering tool, NSA's Ghidra, and confirming the required criteria for an executable to utilise a specific input stream.

- For arguments: The executable's 'main' function is decompiled, and the resulting code is processed with AST. The tree is traversed in order to find interactions between the source code of 'main' and the program's arguments, `argc` and `argv`.
- For standard input and files: All function calls are collected into a set and repeatedly compared with multiple sets of function names, one for each input stream (e.g. `("read", "fgetc", "fread")` for `stdin`). The executable uses an input stream if there is an intersection between the two sets.

### Generating Arguments Dictionaries

Before determining whether the binary uses any parameters, a list of possible arguments should be produced. We used three generation heuristics:

- `generation`: Generates all parameters while adhering to the Regex format `-[a-zA-Z0-9]`.
- `binary_pattern_matching`: Looks for bytes sequences in binary material that match the Regex `\s-{1,2}[a-zA-Z0-9][a-zA-Z0-9_-]*` (matching, for example, ` -f` and ` --file`).
- `man_parsing`: First, it scans the `man` configuration files to determine which folders contain manuals. Finds all gzip files in this location, unarchives them, and matches their content against the same Regex pattern as above. The returned arguments list can optionally be reduced to a specified amount of elements based on their occurrence.

### Arguments Fuzzing

The dynamic approach is better for detecting that an argument (or combination of arguments) is used by the executable, as the static approach may result in erroneous results due to superficial inspection of the control flow graph. By evaluating the program dynamically, on the other hand, the input causes the program execution to take a different path, reaching distinct basic blocks (predictable sequences of instructions, represented as nodes in a control flow graph) and, subsequently, invoking different system API methods.

The purpose of the subtask is to extract coverage information that can then be utilized to distinguish between two executions of the same executable but with different parameters. This information can be extracted using a variety of dynamic binary analysis approaches, including:

- Debugging: A debugger can identify the execution flow by establishing hardware or software breakpoints on each basic block or system call. However, because it is designed for people (programmers, vulnerability researchers, etc.), it is not well-suited for automation in which no manual interaction is necessary. The performance hit was caused by either user-kernel space switches (when invoking the `ptrace` API from the debugger process) or a type of dynamic binary instrumentation, which involved rewriting the code with debugging-related calls or interrupts.
- Static instrumentation and execution: This method entails translating the program into a high-level abstraction language, inserting instrumentation code, and compiling it. The newly produced executable is then run and can report execution coverage information. This strategy, as an alternative to compiler instrumentation, may result in unanticipated behavior due to a lack of standardized technology in this field of research.
- Dynamic binary instrumentation (abbreviated DBI): A DBI instrument attempts to resolve debugger difficulties. Using Quarkslab's engine, QBDI (which is utilized in our implementation), it is possible to inject instrumentation code inside the binary at runtime. The optimization is that the analysis tool and the analyzed program are both performed under the same process, which reduces friction.

Following several experiments with the first two methodologies, we found that the QBDI could be integrated by taking the following steps:

- Convert the executable to a library by removing the PIE flag (which is present in executables) and marking the `main` function as exported with the LIEF Python library.
- `dlopen` is used to load the executable into program memory.
- Instrument the code with the QBDI C API.

It failed because of the QBDI's execution transfer: it deems any calls to dynamically linked libraries to be non-reentrant and moves the execution from instrumented to native (with no instrumentation at all). It returns the instrumentation after the function return.

This limitation prompted us retry with a different API of QBDI, `QBDIPreload`. It is a utility library in which the DBI engine overwrites and calls all of the callbacks exported by the QBDI API during runtime. The instrumentation library is loaded into process memory via the Linux loader's `LD_PRELOAD` command. This second method involved a number of steps:

- Design a callback method to handle basic block calls. To eliminate the variations generated by Address Space Layout Randomization, it employs an address normalization technique. This relativization of the basic block's start address is saved in a list.

    ```c
    start_address -= segments[parent_segment].start;
    abstract_address = (parent_segment << 24) + start_address;
    ```

- Write a callback function to handle execution transfers. It checks to see if `close` is called on a canary file passed to the application as an argument.
- Make a non-cryptographic DJB2 hash over the first 1000 addresses in the CFG and dump it into a file upon exit.

The fuzzing process begins by creating some baseline hashes using the stated coverage extraction mechanism by executing the program with no arguments and a series of random, 10-character long arguments. Following that, a series of commands are executed into an Ubuntu-based Docker container, with a full QBDI setup and communication via mounted Docker volumes:

- Only a filename as an argument;
- Each argument in the dictionary, one at a time;
- `-` as an argument; and
- Each argument in the dictionary, one at a time, plus a canary string.

Each execution generates a DJB2 hash, which is compared to the baseline provided. If the hash was not observed until that point, the algorithm can conclude that the argument resulted in a different execution of the binary (i.e. a different sequence of nodes in the control flow graph) and is therefore registered as a legitimate argument. Aside from that, each argument has a role that specifies its effects:

- Flag (e.g. `--force`): The argument alone generates a unique hash.
- File enabler (e.g. `--config production.yaml`): The program is invoked with the current input followed by the name of an OpenCRS canary file with the `.opencrs` extension. On each execution transfer, the process's file descriptors are examined to see if the canary file is open.
- `stdin` enabled (e.g. `--stdin`): If an execution fails due to a timeout, the program is repeated with input from `stdin`. If the hashes produced by these two executions differ, the flag is regarded to be enabling `stdin`-reading capabilities and is tagged appropriately.
- String enabler (e.g. `--action deploy`): The program's execution with the arguments alone is compared against one of the arguments supplemented with a string. If the first differs from the baselines and the latter differs from the first, the argument will be assigned this role.
