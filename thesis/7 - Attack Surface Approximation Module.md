# Attack Surface Approximation Module

## Plan

- As a system
  - Input data structures
  - Output data structures
- Detection of arguments usage with `argc` and `argv` tracing
- Detection of `stdin` and files using calls to functions
- Detection of arguments
  - Creating arguments fuzzing dictionaries with `man` processing and pattern
  - Arguments fuzzing with QBDI
- Technologies
  - Ghidra
  - Docker
  - QBDI
- CLI

## Existent Content

A submodule of the \textbf{Vulnerability Detection Module}, \textbf{Attack Surface Approximation} \cite{surface_repo}: This submodule deals with the static analysis of the input binaries for discovering their attack surface, namely the input mechanism they use. It is helpful for activating the next modules in the pipeline of the CRS. We achieve this by using the API of a popular reverse engineering tool, namely NSA's Ghidra, and verifying some necessary condition for an executable to use a specific input stream. For instance, if the executable does not interact with the program's parameters (the \mintinline{text}{main()}'s arguments, \mintinline{text}{argc} and \mintinline{text}{argv}), then a fuzzer modifying some (inexistent) parameters is useless.

It is worth mentioning to be mentioned that the latter is required due to our goals. In comparison to the cyber reasoning systems proposed at Cyber Grand Challenge, which deal with an executable taking input only from network packets, our CRS is meant to discover vulnerabilities in all major input streams, \mintinline{text}{stdin} and arguments included.

> Source: second report

\subsection{Fuzzing Arguments}

\subsubsection{Introduction}

Another topic that we addressed this semester is the arguments fuzzing. This is required for two different components of the cyber reasoning system architecture:
\begin{itemize}
    \item \textbf{Discovering the names and order of the arguments} in the attack surface discovery module; and
    \item \textbf{Generating random arguments} to be passed directly as argument in the vulnerability discovery module.
\end{itemize}

Due to difficulties in choosing a technology, we only implemented the first sub-task.

To detect that an argument (or a set of arguments) is used by the executable, the \textbf{dynamic approach} is recommended, as the static one could lead to false results due to shallow inspection of the control flow graph. On the other hand, by analyzing the program dynamically, the input makes the program execution flows on a different path, reaching different basic blocks (predictable sequences of instructions, seen as nodes in a CFG) and, eventually, calling different system API methods.

The goals of the first sub-task is essentially the extraction of coverage information that can be further used to differentiate between two executions of the same executable, but with different arguments. There are multiple dynamic binary analysis techniques that can be used to extract information of this kind: 
\begin{itemize}
    \item \textbf{Debugging}: By placing hardware or software breakpoints on each basic block or system call, a debugger can detect the execution flow. But being a tool for humans (programmers, vulnerability researchers, etc.), it is not properly optimized for automations in which no manual intervention is required. The performance cost came either from user - kernel spaces switches (when calling the \mintinline{text}{ptrace} API from the debugger process) or by a form of dynamic binary instrumentation, by rewriting the code with debugging-related calls or interrupts.
    \item \textbf{Static instrumentation and execution}: This approach consists in lifting the program into a high-level abstraction language, insertion of instrumentation code and compilation. The newly created executable is then ran and can report coverage information for its execution. Being an alternative for compiler instrumentation, this approach can lead to unexpected behavior due to a lack of standardized technologies in this field of research.
    \item \textbf{Dynamic binary instrumentation} (abbreviated DBI): A DBI instrument tries to solve the issues of a debugger. Taking QuarkslaB's engine, \textbf{QBDI} \cite{qbdi}, as an example (that is actually used in our implementation), it allows the injection of instrumentation code inside the binary, at runtime. The optimization is that the analysis tool and the analyzed program runs under the umbrella of the same process, reducing the friction.
\end{itemize}

From our tests, we concluded that the last technique applied in QBDI could be used, taking in consideration its \textbf{advantages}:
\begin{itemize}
    \item Efficiency: The slowdown is less than as in other techniques.
    \item Compatibility: There are multiple APIs available: C, C++, Python and even JavaScript (when used with Frida).
\end{itemize}

\subsubsection{Architecture. Implementation}

The first approach for integrating QBDI consisted in the following steps:
\begin{enumerate}
    \item Transform the executable into a library by using the LIEF \cite{lief} Python library to remove the PIE flag (that is present in executables) and mark the \mintinline{text}{main()} function as exported.
    \item  Load the executable into a program memory using \mintinline{text}{dlopen()}.
    \item  Instrument the code using the C API of QBDI.
\end{enumerate}

It failed due to the QBDI's mode of functioning: it considers non-reentrant all the calls to dynamic linked libraries, and it switches the execution from instrumented to native (with no instrumentation at all). After the function return, it turns back the instrumentation. This whole mechanism is called execution transfer.

This limitation made us retry with a different API of QBDI, \textbf{QBDIPreload}. It is a utility library in which all the callbacks exported by the QBDI API are overwritten and called at runtime by the DBI engine. The instrumentation library is injected in the process memory by using the Linux loader's \mintinline{text}{LD_PRELOAD}.

This second approach used a series of different steps:
\begin{enumerate}
    \item Create a callback function for basic block call. It applies an address normalization technique to discard the variances introduced by Address Space Layout Randomization. This relativization of the start address of the basic block is stored inside a list.
    \item  Create a callback function for execution transfer. It checks if \mintinline{text}{close()} is called on a canary file, provided as an argument to the program.
    \item  On exiting, create a non-cryptographic DJB2 hash over the first 1000 addresses in the CFG and dump it inside a file.
\end{enumerate}

In this manner, it can be said that if two arguments produces two different DJB2 hashes, then they produced a different sequence of nodes in the control flow graph.

The last step here is the \textbf{generation of a dictionary with arguments}, which is created by parsing all the manuals present in a Linux operating system:
\begin{enumerate}
    \item Read the \mintinline{text}{man} configuration files to detect folders where manuals reside.
    \item  Get each gzip archive from this kind of folder.
    \item  Unarchive its content and search with Regex for common patters.
\end{enumerate}

Combining the coverage information with the arguments from the dictionary, we can apply a bruteforce strategy to discover the arguments that are used by the provided executables.

> About arguments fuzzing
> Source: second report
