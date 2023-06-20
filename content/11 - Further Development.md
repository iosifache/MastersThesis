# Further Development

In order to achieve the objective of developing a proof of concept for an open-source cyber reasoning system, we have incorporated the constraints outlined in the "OpenCRS" chapter. This approach has enabled us to attain the desired level of maturity and comprehensiveness in our implementation.

Initially, potential enhancements could be directed towards the fundamental operational framework of OpenCRS.

- New executable formats: PE, Mach-O;
- New CPU architectures: `x86-64`, `amd64`;
- New input streams: network packets;
- New programming languages; and
- The merging of configuration settings for all modules into a single file.

Moreover, the aforementioned modifications have the potential to enhance the functionality of the implemented module.

# Dataset Module

- Static code analysis of the source files in the test suites, such that the input streams are detected and used as labels in the dataset
- More tests in the suite built by us by importing Zeratool binaries and constructing compact and handcrafted executables (which can be used in testing phases because of their execution speed).
- More test suits, such as `cb-multios` [1], a Linux port of DARPA's Cyber Grand Challenge executables

# Attack Surface Approximation Module

- Improvements in the binary matching heuristic's accuracy by searching just in certain regions of the executable (for example, `.text`, `.data`, and `.bss`)
- Improvements in the accuracy of argument fuzzing by examining other QBDI execution events
- New method for detecting input stream usage, such as symbolically running the binary and intercepting input-related (library or system) calls.
- Argument testing in pairs for cases with dependency relations
- Migration of QBDI and Ghidra containers from mounted Docker volumes to gRPC

# Vulnerability Discovery Module

- Stopping the fuzzing session using a timeout or coverage convergence strategy, such as Fuzz Introspector [2]
- Binary rewriting for adding Address Sanitizer, for example with RetroWrite [3]
- Migration of afl++ containers from mounted Docker volumes to gRPC

# Automatic Exploit Generation

- In a sandboxed environment, reverification of the produced exploit
- Exploitation through format string attacks, a technique supported by the already-integrated `zeratool_lib`
- New exploitation techniques, possibly for other input streams as well
