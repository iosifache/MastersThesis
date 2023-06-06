# Attack Surface Approximation Module

Besides the processes that are automated internally, the program should have ways of interfacing with the exterior, either the environment or the users. The most used communication streams are:

- Standard input (`stdin`);
- Arguments;
- Files;
- Network packets;
- System and library calls;
- Graphic user interface;
- Interrupts;
- Signals;
- Shared memory;
- Devices; and
- Environment variables.

If data is sent with bad intents, then these **input streams** are referred to as **attack vectors**. They form the **attack surface**: the full interface of the executables with the exterior, which can be attacked by a malicious actor.

OpenCRS should know the attack vectors before starting to find vulnerabilities in them. This observation consists of the basis for the next module in OpenCRS's pipeline, namely the **attack surface approximation**. Having a binary as an input (eventually piped from the **dataset module**), it discovers the attack vectors that can be leveraged by the next modules in the CRS.

As seen before, the diversity in input streams is large. We considered only three of the most used ones in our proof of concept: **`stdin`**, **files**, and **arguments**.

> **TODO**: Insert diagram here

## Theoretical Considerations

The paper [2] initially introduces the concept of **front line functions**, namely functions that are close to a source of input. [1] builds on this idea, and defines the concept of **direct entry point**: a method that contains a call to one of the specific input methods (e.g. `read` from `unistd.h`). Despite the papers presenting theoretical concepts equivalent to the input streams mentioned in this thesis, they don't provide any tool or method for detecting these input entrances in processes. Besides this, it should be mentioned that the papers approach this problem by analyzing the source code.

On the other hand, this CRS module implies generating valid arguments that can be used in the later modules (i.e. in vulnerability detection via fuzzing). Papers such as [3][4][5] are proposing argument detection, but by having deep knowledge about the analyzed software. They require either a grammar, a `getopt` option parsing call to intercept, or a man page to parse.

The single implementations that could be found were in AFL [6], AFL++ [7], and ManFuzzer [8]. The first two have the `argv_fuzzing` mode, which plugs the generated random bytes into the `argv` of the main function. The third-mentioned software generates fuzzing sequences by inspecting the man page and the output of the program ran with `-h`, `-H`, and `--help`. The disadvantage here is that the AFL-based approach does not consider the common format of an argument (e.g. `-<letter_or_word>` or `--<letter_or_word>`). For the last one, there is an inherent risk that the program does not have a help page implemented or has hidden arguments (e.g. experimental ones).

## Architecture

The module's goal of detecting the input streams was divided into multiple tasks, which implies different technologies.

### Indicators Discovery

The first task is to **statically analyze** the binary for evidence (named **indicators**) that a specific input stream is used. It should be noted that the presence of an indicator is sufficient, but not mandatory. Despite the fact the program does not call `getenv` internally, we can't guarantee that the environment variables are not used altogether as the program may implement an exotic approach of parsing its stack to retrieve that information.

This analysis helps activate features in the next modules of OpenCRS's pipeline. For instance, if the executable does not use its CLI arguments, then a fuzzer modifying some (inexistent) arguments is useless and will not lead to meaningful results.

We achieve this by using the API of a popular reverse engineering tool, namely NSA's **Ghidra**, and verifying some necessary conditions for an executable to use a specific input stream.

- For arguments: The `main` function of the executable is decompiled, and the resulting code is parsed with AST. The tree is traversed to detect interactions of `main`'s source code with the program's arguments, `argc` and `argv`.
- For standard input and files: All function calls are extracted into a set, and compared repeatedly with multiple sets of function names, specific to each input stream (e.g. `("read", "fgetc", "fread")` for `stdin`). If there is an intersection between the two sets, then an input stream is used by the executable.

### Generating Arguments Dictionaries

Before detecting if some arguments are used by the binary, a list of possible arguments should be generated. We implemented three generation heuristics:

- `generation`: Generates all the arguments respecting the Regex format `-[a-zA-Z0-9]`.
- `binary_pattern_matching`: Searches for bytes sequences in binary's content that respects the Regex `\s-{1,2}[a-zA-Z0-9][a-zA-Z0-9_-]*` (matching, for example, ` -f` and ` --file`).
- `man_parsing`: Firstly, it reads the `man` configuration files to detect folders where manuals reside. In this location, finds all gzip file, unarchives them, and match their content against the same Regex pattern as above. Optionally, the returned arguments list can be trimmed to a fixed number of elements, based on their occurrence.

### Arguments Fuzzing

To detect that an argument (or a set of arguments) is used by the executable, the **dynamic approach** is preferred, as the static one could lead to false results due to shallow inspection of the control flow graph. On the other hand, by analyzing the program dynamically, the input makes the program execution flows on a different path, reaching different basic blocks (predictable sequences of instructions, seen as nodes in a CFG) and, eventually, calling different system API methods.

The goal of the sub-task is essentially the extraction of coverage information that can be further used to differentiate between two executions of the same executable but with different arguments. Multiple dynamic binary analysis techniques can be used to extract information of this kind:

- **Debugging**: By placing hardware or software breakpoints on each basic block or system call, a debugger can detect the execution flow. But being a tool for humans (programmers, vulnerability researchers, etc.), it is not properly optimized for automation in which no manual intervention is required. The performance cost came either from user-kernel spaces switches (when calling the `ptrace` API from the debugger process) or by a form of dynamic binary instrumentation, by rewriting the code with debugging-related calls or interrupts.
- **Static instrumentation and execution**: This approach consists in lifting the program into a high-level abstraction language, insertion of instrumentation code, and compilation. The newly created executable is then run and can report coverage information for its execution. Being an alternative to compiler instrumentation, this approach can lead to unexpected behavior due to a lack of standardized technologies in this field of research.
- **Dynamic binary instrumentation** (abbreviated DBI): A DBI instrument tries to solve the issues of a debugger. Taking Quarkslab's engine, **QBDI**, as an example (that is used in our implementation), it allows the injection of instrumentation code inside the binary, at runtime. The optimization is that the analysis tool and the analyzed program runs under the umbrella of the same process, reducing friction.

The first approach for integrating QBDI consisted of the following steps:

- Transform the executable into a library by using the **LIEF** Python library to remove the PIE flag (that is present in executables) and mark the `main` function as exported.
- Load the executable into a program memory using `dlopen`.
- Instrument the code using the C API of QBDI.

It failed due to the QBDI's execution transfer: it considers non-reentrant all the calls to dynamically linked libraries, and it switches the execution from instrumented to native (with no instrumentation at all). After the function return, it turns back the instrumentation.

This limitation made us retry with a different API of QBDI, `QBDIPreload`. It is a utility library in which all the callbacks exported by the QBDI API are overwritten and called at runtime by the DBI engine. The instrumentation library is injected into the process memory by using the Linux loader's `LD_PRELOAD`. This second approach used a series of different steps:

- Create a callback function for basic block calls. It applies an address normalization technique to discard the variances introduced by Address Space Layout Randomization. This relativization of the start address of the basic block is stored inside a list.
- Create a callback function for execution transfer. It checks if \mintinline{text}{close()} is called on a canary file, provided as an argument to the program.
- On exiting, create a non-cryptographic DJB2 hash over the first 1000 addresses in the CFG and dump it inside a file.

By leveraging the described coverage extraction mechanism, the fuzzing process starts by generating some baseline hashes by running the program with no arguments and a series of random, 10-character long arguments. Following this, a sequence of execution is performed into an Ubuntu-based Docker container, with a complete QBDI setup and gRPC communication:

- Only a filename as an argument;
- Each argument in the dictionary, in turn;
- `-` as an argument; and
- Each argument in the dictionary, in turn, plus a canary string.

Each execution generates a DJB2 hash that is checked to be in the baseline set. If the hash wasn't seen until that moment, then the algorithm can say that the argument produced a different execution of the binary (i.e. different sequence of nodes in the control flow graph) and is registered as a valid argument. Besides this, each argument has a **role** attached that describes its effects:

- **Flag** (e.g. `--force`): The argument alone produces a different hash.
- **File enabler** (e.g. `--config production.yaml`): The program is run with the current argument accompanied by the name of a canary file generated by OpenCRS, having the `.opencrs` extension. On each execution transfer, the file descriptors of the process are checked to see if the canary file is opened.
- **`stdin` enabled** (e.g. `--stdin`): If an execution ends with a timeout, the program is rerun with input provided at `stdin`. If these two executions produce different hashes, then the flag is considered as enabling `stdin`-readin capabilities and marked accordingly.
- **String enabler** (e.g. `--action deploy`): The execution of the program with the arguments alone is compared with one of the arguments accompanied by a string. If the first is different from the baseline ones and the latter is different from the first, then the argument will receive this role.

## Testing

As for the dataset module, we implemented a command-line interface for calling the functions exposed by the Python module.

Below are given as examples of the generation of an argument dictionary with the most used 6 arguments, the input stream detection from a binary from our toy test suite, and the arguments fuzzing for a Linux binary, `uname`.

```bash
➜ opencrs-surface generate --heuristic man --output args.txt --top 6
Successfully generated dictionary with 6 arguments
➜ cat args.txt
--and
--get
--get-feedbacks
--no-progress-meter
--print-name
-input
➜ opencrs-surface detect --elf stdin_buffer_overflow.elf
┏━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━┓
┃ Stream                ┃ Present ┃
┡━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━┩
│ files                 │   No    │
│ arguments             │   No    │
│ stdin                 │   Yes   │
│ networking            │   No    │
│ environment_variables │   No    │
└───────────────────────┴─────────┘
➜ opencrs-surface fuzz --elf /bin/uname --dictionary args.txt
Several arguments were detected for the given program:

┏━━━━━━━━━━━┳━━━━━━━━━━━━━━━━┓
┃ Argument  ┃      Role      ┃
┡━━━━━━━━━━━╇━━━━━━━━━━━━━━━━┩
│ -         │      FLAG      │
│ -a        │      FLAG      │
│ -i        │      FLAG      │
│ -m        │      FLAG      │
│ -n        │      FLAG      │
│ -o        │      FLAG      │
│ -p        │      FLAG      │
│ -r        │      FLAG      │
│ -s        │      FLAG      │
│ -v        │      FLAG      │
└───────────┴────────────────┘
```

> TODO: Replace `key-manager.elf` with an actual binary from toy
