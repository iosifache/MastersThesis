# Introduction

Despite the computers appeared in their analogic form yet from the 19th century, the explosive growth of them started at the middle of the next century. In 1941, Konrad Zuse created Z3, the first functional digital computer. What made this machinery special is its property to be programmable with punch cards. These could store into an analogic form source code, namely instructions that the computer could understand and execute.

Eighty years later, these computers surround us. Although their utility increased, the model is the same: a set of programs (software) dictates the physical circuits (hardware) how to behave. With the utility increase, the number of users increased too. Firstly, we have the benign users, which want to use the technology for the intended purpose. On the other hand, we have the malicious ones, which intend to obtain materials or immaterial (reputation, recognition, etc.) benefits by attacking the technology, including the software part.

These attacks are possible because the software is created by humans, which are error-prone by nature. The fault should not be borne by the latter, but methods of verifying could be set in-place to defend the code that was written. These methods consist either in a manual or automatic process. An example for the first is a security code review, in which an experienced engineer checks the security posture of a piece of code. For the latter, tools like security linters, scanners, and fuzzers can be integrated in developers' IDEs, CI/CD pipelines or staging environments.

In 2016, the Defense Advanced Research Projects Agency (DARPA), a research and development agency of the United States Department of Defense, organized the **Cyber Grand Challenge**. It was a contest between automated systems that fought each other to demonstrate their accuracy and efficiency. Named **cyber reasoning systems**, their goal was to find vulnerabilities in the binaries that they and the adversaries run simultaneously. They could earn points in the Capture The Flag competition by automatically creating exploits which could be run against their adversaries and patching their binaries to protect again adversaries' attacks.

The top three finalists are considered state of the art at the moment. Mayhem, the winner, became a commercial solution [2]. Despite Shellphish was published as an open-source project [1], the open-source landscape remains the same. This issue represents the starting point for two theses, namely the current one and another from Claudiu Ghenea [3]. They introduce an open-source cyber reasoning system, **OpenCRS**, which discovers, exploits, and patches vulnerabilities in C-based ELF executables for `i386`.

## Objectives. Structure

The main objective of this document is to **describe and assess four modules** of the cyber reasoning system, which were implemented by the author:

1. The dataset module, which manages test suites of vulnerable programs;
2. The attack surface approximation module, which detects the way in which an executable interacts with the environment;
3. The vulnerability detection module, which generates input that is triggering a vulnerability in an executable; and
4. The automatic exploit generation module, which generates a working exploit for each discovered vulnerability.

In addition to this, the document will shortly **describe how OpenCRS works**, aspect which will be covered in greater detail by the other thesis that we mentioned.

## Structure

The thesis is split in **nine chapter**. This one described the current context for software security, cyber reasoning systems, and the purpose of this document. The next one will describe OpenCRS in more details, including its overall architecture and the limitations we established for this system. The next four chapters will describe each module mentioned in the numbered list above. Each of them presents theoretical aspects (including the state of the art of each binary analysis subfield that is implemented by the module), the module's architecture and implementation, and used technologies. The seventh chapter continues the module's testing and evaluation, and presents exact data revealing the accuracy of the results. The penultimate one propose changes and improvements that can be achieved in the future for OpenCRS and the presented modules, while the last one summarize what was presented in the thesis and concludes it.
