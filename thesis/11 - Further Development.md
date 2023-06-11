# Further Development

Due to the goal of creating a proof of concept for an open-source cyber reasoning system, we imposed the limitations presented in the chapter "OpenCRS" such that our implementation could reach the desired maturity and wholeness.

The improvements that can be made in the future firstly target the fundamental way in which OpenCRS works:

- New executable formats: PE, Mach-O;
- New CPU architectures: `x86-64`, `amd64`;
- New input streams: network packets;
- New programming languages.

Besides this, the improvements target each implemented module.

## Dataset Module

- Static code analysis over the source files included in the test suites, such that the input streams are detected and used as labels in the dataset
- More test suits, such as `cb-multios` [1], a Linux port of the executables used in DARPA's Cyber Grand Challenge

## Attack Surface Approximation Module

- New technique to detect the usage of an input stream, such as symbolically running the binary and intercepting the input-related (library or system) calls
- Pair-wise testing of arguments, for scenarios of dependency relations

## Vulnerability Discovery Module

- Timeout or coverage convergence heuristic for stopping the fuzzing session, for example by leveraging Fuzz Introspector [2]
- Binary rewriting for adding Address Sanitizer, for example with RetroWrite [3]

## Automatic Exploit Generation

- Exploitation via format string attacks, a technique that is implemented by the already-integrated `zeratool_lib`
- New exploitation techniques, eventually for other input streams too
