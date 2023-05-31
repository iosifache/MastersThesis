# Introduction

## Plan

- **Context**
  - Cyber reasoning systems
  - Short introduction in OpenCRS
- **Objectives** (with this thesis)
  - Description of 4 modules of OpenCRS
  - Description of the techniques used for each module
  - Description of the available techology for each technique
  - Implementation for each module
  - Testing
- **Structure**
  - Short description of each chapter

## Existent Content

As the number of devices has increased steadily over the last few decades, so has the number of executables, which are the set of actions that the device's processors must perform. This increase reflected in the attention paid by malicious people (to find security-related issues) and made infeasible the manual vulnerabilities assessments of all the programs.

Automated tools have been developed, and a novel approach has proven its worth in recent years: cyber reasoning systems. These are specialized systems capable of dealing with all aspects of binary program vulnerabilities, including discovery, exploitation, and patching. Following DARPA's Cyber Grand Challenge, in which multiple cyber reasoning systems were opposed against each other to demonstrate their accuracy and efficiency, the majority of them became closed source, unmaintained, or commercial. As a result, the open-source community does not reap the benefits of this novel approach.

> For context about about CRS
> Source: First report

A \textbf{cyber reasoning system} is an automated system for managing vulnerabilities inside executable programs, starting from discovery and continuing with exploiting, signature creation and self-healing.

The first viable prototypes were shown at a competition organized by DARPA back in 2016, Cyber Grand Challenge. Then, multiple solutions were installed on physical systems that were able to communicate with each, with a single purpose: to automatically attack other systems and to defend from them, earning points in a Capture The Flag competition.

From the top 3 finalists, that are still considered \textbf{state of the art} at the moment, Mayhem (the winner) became a commercial solution \footnote{https://forallsecure.com/mayhem-for-code} and Shellphish was published on GitHub as an open source project \footnote{https://github.com/shellphish} and became unmaintained. Since then, some others commercial software appeared, but the open source landscape remained the same.

> For context about CRS
> Source: Second and third report

We propose a new open source cyber reasoning system in this thesis. Due to the issue's complexity, we are currently targeting only ELF executables built from a C codebase and running on an x86 instruction set architecture.

Given a vulnerable program and no additional information about it, our system will be able to detect flaws using techniques like fuzzing and symbolic execution. The resulting proof of vulnerability will then be used, along with additional information about it, to create exploits with the greatest possible impact. Switching to a defender's perspective, the system will generate a signature for detecting an exploit attempt and a patched binary that can successfully replace the old one, but without the discovered security flaw.

> For context about OpenCRS
> Source: First report

In our thesis, we propose the creation of an \textbf{open source cyber reasoning system} using the latest advances in the binary analysis field. As the variety regarding the executables is high, we will niche our solution for ELF CLI executables, built from a C codebase on an x86 architecture.

> For context about OpenCRS
> Source: Second and third report

We have shown above that cyber reasoning systems are holistic approaches to manage all the aspects related to vulnerabilities in executable files. As the solutions considered, at the moment, state of the art are either unmaintained or closed source (even commercial, as the case of Mayhem, the winner of DARPA's CGC), we propose with our thesis a \textbf{novel open-source CRS} that uses the latest advances in the fields of binary analysis, from both the academia and industry.

The \textbf{variety in the binary analysis} is high, due to various aspects:
\begin{itemize}
    \item Processor's architecture: Bit count, instruction set architecture (abbreviated ISA);
    \item Operating system: System calls;
    \item Binary file format: The effective way of representing the opcodes; and
    \item Programming language: Vulnerabilities classes. 
\end{itemize}

Because we want to implement a \textbf{proof of concept} for a CRS that integrates all the latest improvement, we do not target all the variation in the aspects above. Instead, we want at first to achieve the holistic vulnerability assessment only for \textbf{ELF} files (namely a \textbf{Linux} operating system), build from a \textbf{C} codebase and run on a \textbf{x86} ISA.

> For context about OpenCRS
> Source: First report

The goals of this system is to \textbf{find}, \textbf{patch}, \textbf{exploit} and \textbf{protect binaries}. Seeing the CRS as a blackbox, without detailing in greater details its functioning (a thing that we will do in the following section), its input are vulnerable programs. Relying on different binary analysis techniques, the CRS then detects its vulnerability, producing \textbf{proofs of vulnerabilities} (abbreviated PoV), each of which triggering a certain vulnerability.

However, the vulnerabilities are limited by constraints accidentally imposed by the executable, namely its shape (for example, the allocated segments and their permissions) and functioning (for instance, some transformations that are applied to the user's input). These constraints are modeled to \textbf{detect which is the greatest impact} that an exploit can have. With this model of the space in which the exploit resides, our CRS will generate a \textbf{payload} which will have a high impact considering the constraints that are present.

For a defender point of view, even if it knows about a given vulnerability (via the PoV), he could not act accordingly. Here, the CRS will help by generating a new version of the program, with the vulnerable parts patched. This can be achieved by analyzing the PoV, determining the root cause of the vulnerability, and rewriting that weak functionality.

In addition, the proof of vulnerability can be further analyzed to detect what are its fundamental characteristics, without which an exploit could not function properly. This information helps to protect a binary program that could not be easily replaced with a patched executable, by creating \textbf{signatures}. These can be concrete, if we talk about a byte sequence in a packet, that can be detected (and blocked) by a network intrusion detection and prevention systems (abbreviated IDPS), or behavioral, as in the case of the host IDPS, which looks, for example, at the parameters of the system calls made by active processes.

> For context about OpenCRS
> Source: First report