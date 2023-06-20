# OpenCRS

This thesis, together with one of my colleagues Claudiu Ghenea [1], is introducing an open-source cyber reasoning system. By leveraging the latest advances in the binary analysis field from both Academia and industry, OpenCRS is meant to implement the **entire security assessment process for executables**, from discovering the ways it takes the input to create patches for the discovered vulnerabilities.

As the variations in architectures, languages, and binaries' behavior are high, the goal was to develop a **proof of concept** that deals with executables having the following characteristics:

- Run on a 32-bit Intel architecture, namely the **`i386` ISA**.
- Run on Linux, and therefore have the **ELF format**.
- Are compiled from **C source code**.
- Have **arguments**, **standard input**, and **files** as input streams.

OpenCRS's **overall architecture** and the communication happening between the modules can be seen in the above diagram:

> Insert diagram

Considering an analyst requires the analysis of an executable, the modules and their internal workflows are managed as follow by the orchestration module:

1. **Dataset module**: Integrating multiple public test suites with C source code, the module compiles it to obtain a series of executables. The main advantage of this approach is that OpenCRS can have a validation loop, considering the vulnerabilities included in the source code (as depicted by the CWE labels) and the ones discovered, exploited, and patched by the system. The dataset module is optional as the executables analyzed by OpenCRS can came already been built from other sources. For example, open (e.g. HiColor [2]) and closed (e.g. Dropbox [3]) source software provide their users the ability to download prebuilt binaries for diverse architecture.
2. **Attack surface approximation module**: With an executable (and no further information about it) as input, this module will find how it can be attacked: either input streams or a specific format for them (for example, the arguments that are expected in `argv` by the program).
3. **Vulnerability discovery module**: After the attack surface is discovered by the previous module, this module will use vulnerability discovery techniques like fuzzing and symbolic execution to find proof of vulnerabilities, namely inputs that make the program misbehave.
4. **Vulnerability analytics module**: Once a proof of vulnerability, this module analyzes it to offer further details about the vulnerability it uncovers, for example, the class of vulnerabilities (e.g. buffer overflow).
5. **Automatic exploit generation module**: The proof of vulnerability, enriched with details by the previous module, is used to generate an exploit that triggers it and achieves the highest impact possible.
6. **Signature generation module**: Compared to the offensive nature of the previous one, the signature generation module has a defensive purpose. Having the same input as the automatic exploit generation module, it creates a signature that can be used to detect (and block) exploitation attempts. The module is useful in the context of critical or obsoleted systems, where an update to another program is not possible (due to availability loss or incompatibilities).
7. **Healing module**: The goal of this module is the same as for the signature generation one, namely to protect the binary against exploitation approach. The difference is that, compared to the previous module, this one modified the binary to remove the vulnerable code or to build the proper sanitization, but by keeping the initial functionality intact.

The following chapters will describe in details the inner functioning of several modules. It should be noted that the remaining modules are tackled in other theses: the vulnerability analytics and healing modules in "" [1], and the signature generation one in "" [3].
