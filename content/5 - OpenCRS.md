# OpenCRS

This study, together with Claudiu Ghenea's, presents an open-source cyber reasoning system. OpenCRS aims to implement a comprehensive security assessment process for executables by utilizing the latest advancements in binary analysis from both academic and industrial sectors. The purpose of OpenCRS is to execute a comprehensive security evaluation of executable files, encompassing the identification of input methods and the development of patches to address any detected vulnerabilities.

Given the significant diversity in architectures, programming languages, and binary behaviors, the objective was to create a proof of concept that addresses the execution of programs exhibiting the subsequent attributes:

- The system operates on a 32-bit Intel architecture, specifically utilizing the `i386` Instruction Set Architecture.
- The software is compatible with the Linux operating system and utilizes the Executable and Linkable Format (ELF).
- These are generated from the source code written in the C programming language.
- The input streams for arguments, standard input, and files are utilized.

The diagram above depicts the comprehensive architecture of OpenCRS and the inter-module communication taking place within the system.

Given that an analyst demands an executable analysis, the orchestration module manages the modules and their internal procedures as follows:

1. Dataset module: By incorporating various public test suites into the C source code, the module can compile it and generate a sequence of executable files. The primary benefit of this methodology is that OpenCRS can implement a validation mechanism that takes into account the vulnerabilities present in the source code, as indicated by the CWE labels, as well as those that have been identified, exploited, and remediated by the system. The dataset module is not mandatory since the analyzed executables utilized by OpenCRS may have already been constructed from alternative origins. Both open-source software, such as HiColor, and closed-source software, such as Dropbox, offer their users the option to download prebuilt binaries that cater to a variety of architectures.
2. Attack surface approximation module: With an executable (and no other knowledge about it) as input, this module will determine how it might be attacked: either through input streams or a predefined format for them (for example, the arguments that the program expects in `argv`).
3. Vulnerability discovery module: Subsequent to the identification of the attack surface by the preceding module, this module will employ vulnerability discovery methodologies such as fuzzing and symbolic execution to ascertain the existence of vulnerabilities, specifically inputs that result in erroneous program behavior.
4. Vulnerability analytics module: Upon detecting a proof of vulnerability, this module conducts an analysis to provide additional information regarding the specific vulnerability that has been identified. This may include identifying the category of vulnerability, such as a buffer overflow.
5. Automatic exploit generation module: The preceding module's proof of vulnerability is utilized to develop an exploit that triggers it and has the greatest possible impact.
6. Signature generation module: In contrast to the aggressive connotation of its predecessor, the module responsible for generating signatures serves a protective function. By utilizing identical input to the automatic exploit generation module, a signature is generated with the purpose of identifying and preventing exploitation endeavors. The module is deemed valuable in scenarios involving critical or outdated systems, wherein the installation of an alternative program is not feasible due to the unavailability of the program or compatibility issues.
7. Healing module: The objective of this module is akin to that of the signature generation module, which is to safeguard the binary from exploitation techniques. In contrast to the preceding module, the current one has made alterations to the binary in order to eliminate the vulnerable code or implement appropriate sanitization measures, while preserving the original functionality.

Subsequent chapters will provide a comprehensive account of the internal operations of various modules. It is noteworthy that the outstanding modules have been addressed in separate theses: the vulnerability analytics and healing modules in TBD, and the signature generation module in TBD.
