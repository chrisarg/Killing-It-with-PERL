title: "Vibe coding a Perl interface to a foreign library - Part 2"
date: 2025-07-04
---

# Of makefiles and alienfiles

In the second part of this series, we turn to the chatbot's output for bringing in the foreign (C) [Bit library dependency](https://github.com/chrisarg/Bit) in Perl using `Alien`. 
The C library uses Make as a build system (here is the [makefile](https://github.com/chrisarg/Killing-It-with-PERL/blob/main/_includes/Makefile_bit.md) that was provided as context to Claude if you are interested), to create static and dynamic versions of the Bit library using one of three compilers (`gcc`, Intel's `icx` and `clang`), targetting NVIDIA and AMD GPUs[^1]. The makefile makes the two versions of the library, the CPU benchmarks, the GPU benchmarks and the tests, but does not include a `make install` step in order to stress the chatbot's quality of representation of the `Alien` process and its ability to generate good code for less travelled build paths.

[^1]: At the time of this writing, the makefile has been tested only for NVIDIA GPU compiled with gcc

