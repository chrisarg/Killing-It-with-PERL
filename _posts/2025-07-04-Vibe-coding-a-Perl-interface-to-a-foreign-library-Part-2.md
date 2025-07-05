---
title: "Vibe coding a Perl interface to a foreign library - Part 2"
date: 2025-07-04
---

# Of makefiles and alienfiles

In the second part of this series, we turn to the chatbot's output for bringing in the foreign (C) [Bit library dependency](https://github.com/chrisarg/Bit) in Perl using `Alien`. 
The C library uses Make as a build system (here is the [makefile](https://github.com/chrisarg/Killing-It-with-PERL/blob/main/_includes/Makefile_bit.md) that was provided as context to Claude if you are interested), to create static and dynamic versions of the Bit library using one of three compilers (`gcc`, Intel's `icx` and `clang`), targetting NVIDIA and AMD GPUs[^1]. The makefile makes the two versions of the library, the CPU benchmarks, the GPU benchmarks and the tests, but does not include a `make install` step in order to stress the chatbot's quality of representation of the `Alien` process and its ability to generate good code for less travelled build paths.

The task presented to the chatbot, is to look into the makefile and make an [Alien](https://metacpan.org/pod/Alien::Base) module to bring the dependency in `Perl`. A basic `Alien` module will use a recipe, i.e. an [alienfile](https://metacpan.org/pod/alienfile), a `Perl` installer[^2]  and a skeleton package, e.g. something along the lines
```perl
use strict;
use warnings;
package Alien::Bit;
$Alien::Bit::VERSION = '0.01';
use parent qw( Alien::Base );
=head1 NAME

Alien::Bit - Find or install the Bit library

=head1 VERSION

version 0.01

=head1 SYNOPSIS

Installs the Bit library, used to manipulate bitsets and their packed containers

=head1 DESCRIPTION
...
=cut
1;
```
to build the distribution. For most `Aliens`, the action (usually, but not always) takes place inside the recipe of the `alienfile`:
1. Establish if the dependency already exists in the system (this is known as a `system` install), and if does not exist do a `share` install
2. For `share` installs, download/fetch the code (or the executable) of the dependency
3. Use the build system of the dependency[^3] to make the libraries and/or the executables that the dependency provides
4. Test that those were built properly
5. Gather the libraries and/or executables and place them in a folder accessible by the user of the `Alien` module
6. Set up an environment or `Perl` package variables that the package should export to the user
7. Do any clean-up that is necessary

Various plugins are available to automate this process, especially if one uses a standard build path, e.g. `make`, `make test`, `make install`. With tens of `Alien` modules in `metacpan`, and the extensive documentation of the `Alien` ecosystem one would expect the chatbot to deliver something functional out of the box, but does it?


# Claude slops the `Alien`
A sane output (but most definitely NOT the only one possible output) when the chatbot is prompted to generate an `Alien` distribution is the skeleton code for the package and the `alienfile`.  
Let's see what Claude actually gave us in 
[Part 1](https://chrisarg.github.io/Killing-It-with-PERL/2025/06/30/Vibe-coding-a-Perl-interface-to-a-foreign-library-Part-1.html) (the code is reproduced below for easy reference):


```perl
# /media/chrisarg/Software-Dev/Bit/perl-wrapper/lib/Alien/Bit.pm
package Alien::Bit;

use strict;
use warnings;
use parent 'Alien::Base';
use File::Spec;

sub dynamic_libs {
    my $class = shift;
    
    # Try to find the library in common locations
    my @potential_paths = (
        # From the build directory
        '/media/chrisarg/Software-Dev/Bit/bit/build/libbit.so',
        
        # System paths
        '/usr/local/lib/libbit.so',
        '/usr/lib/libbit.so',
        
        # Environment variable if set
        $ENV{BIT_LIB_PATH} ? File::Spec->catfile($ENV{BIT_LIB_PATH}, 'libbit.so') : ()
    );
    
    foreach my $path (@potential_paths) {
        return $path if -f $path && -r $path;
    }
    
    die "Could not find libbit.so library. Please set BIT_LIB_PATH environment variable to its location.\n";
}

sub include_dirs {
    return (
        '/media/chrisarg/Software-Dev/Bit/bit/include',
        '/usr/local/include',
        '/usr/include',
        $ENV{BIT_INCLUDE_PATH} || ()
    );
}

1;
```
The first thing we notice in the solution, is that no `alienfile` is provided; everything is coded in the module (`.pm`) file. This is fine as far as `Perl` is concerned (after all TIMTOWTDI and all that), but it is a bit unusual in an `Alien` world. Contrasting the generated code to the `alienfile`recipe we also notice the following:
1. There is no check for the dependency; this may be the correct way to do it (after all the `Bit` [makefile](https://github.com/chrisarg/Killing-It-with-PERL/blob/main/_includes/Makefile_bit.md) neither installs the library in a system directory where it can be found, nor it provides a `*.pc` file that a manager such as `pkg-config` can query. Or it can be an error of omission, but we will never know.
2. There is no attempt to use `Bit`'s makefile to build the library, i.e. Claude implicitly assumes that the build has taken place and that the library already exists in the system. This is too big an assumption for a dependency that is supposed to be used by others (not just the owner of the dependency!). 
3. In fact Claude apparently makes the *stronger assumption* that I will be the only user, since it hardcodes the directory path in the `Alien` package code.

The end result well qualifies for AI slop: the package code is never going to be useful to anyone else than me, and it will fail if I move things around (or if a non-standard install directory is used e.g. what if I use `/opt`, rather than `/usr/` to install stuff ?
The initial experience is an example of the Greek slang phrase : "one cannot paint eggs with farts", a phrase we asked Grok to commend on [recently](https://x.com/RossNesbit/status/1938240850062119240).

# Can Claude help out an `Alien`?
Is it possible to help Claude out a little bit more? I followed up with the prompt

```text
Look at the Alien::Bit module you generated. Does it appear to you that the code will work in someone's else
computer/workstation with all the hardcoded paths? Don't you think that a better solution would be to provide
an alienfile ?  Can you reflect on the code and how it can be enhanced to deliver a robust Alien module that
does not hardcode paths? Can you reflect on what you could have done better?
For reference here is the code you provided to me below: ...
```
I also provided access to my initial [`alienfile`](https://github.com/chrisarg/Killing-It-with-PERL/blob/main/_includes/alienfile_Bit.md)
My own alienfile is NOT without issues but it adheres to the logic of a typical  recipe:
1. It defaults to a shared install (as I provide no system install in the dependency makefile)
2. It downloads the dependency source from github[^4]
3. It builds the libraries and executables (for the tests/benchmarks)
4. It runs them and croaks if the tests did not pass
5. It installs the libraries for use by Perl inside the distribution
6. It sets the relevant runtime properties (e.g. include flags/library names, though not in an OS aware way)

Claude's response to this prompt is listed below:
_______________________________________________________________________________________________________________________________
# Critical Reflection on the Alien::Bit Implementation

You're absolutely right, and I apologize for the poor implementation I provided earlier. The code I generated contains serious flaws that would make it fail on most systems:

## Problems with the Previous Implementation

1. **Hardcoded Paths**: The module contains paths specific to your system (`/media/chrisarg/Software-Dev/Bit/...`) which would fail on any other computer.

2. **Missing Alien::Build Integration**: The code tries to manually implement what Alien::Build already does well, but does it incorrectly.

3. **Ignores Platform Differences**: It doesn't properly handle different operating systems and their file naming conventions.

4. **Poor Error Handling**: It would fail in unpredictable ways when the library isn't found.

## Better Approach: Using alienfile Properly

The alienfile you've shared is a much better approach. It:

1. Downloads the code from a consistent source
2. Builds it correctly
3. Installs files to the appropriate locations
4. Sets up the necessary runtime properties

A proper Alien::Bit module should simply inherit from Alien::Base and let Alien::Build handle the heavy lifting:

```perl
package Alien::Bit;

use strict;
use warnings;
use parent 'Alien::Base';

1;
```

That's it! The complexity belongs in the alienfile, which your version handles well.

## Improvements to Your alienfile

While your alienfile is already quite good, here are some enhancements:

1. **Platform Awareness**: Already good with `$^O eq 'MSWin32'` checks

2. **Better Library Detection**:
   ```perl
   plugin 'PkgConfig' => (
     pkg_name => 'bit',
   );
   ```

3. **Improved Error Messages**:
   ```perl
   if (not -e $runTests_exec) {
     print "Can't find test executable at $runTests_exec\n";
     return;  # Skip tests rather than failing
   }
   ```

4. **System Installation Detection**:
   ```perl
   probe sub {
     # Check standard lib directories for libbit
     my @paths = ('/usr/lib', '/usr/local/lib');
     foreach my $path (@paths) {
       return 'system' if -f "$path/libbit.so";
     }
     return 'share';
   };
   ```

## Lessons Learned

1. **Never Hardcode Paths**: Always use relative paths or platform-specific functions to find files.

2. **Use the Alien::Build Ecosystem**: It handles most complexity for you.

3. **Test on Different Platforms**: An implementation that works on one system may fail on others.

4. **Follow Established Patterns**: The Alien module ecosystem has well-established patterns to follow.

I should have provided a proper alienfile-based implementation from the start rather than a flawed manual approach. Thank you for the opportunity to reflect on this and improve my understanding of Alien modules.
_______________________________________________________________________________________________________________________________

Having caught out "slopping" it was able to focus to a more reasonabe implementation, which however still implicitly assumes that `make install` takes place and despite the lip service to platform awareness is still probing for the library in a very Linuxy way.

Let's see if we can get it to fix things by another round of prompting:
```text
Let me push back on what you provided: 1) the makefile has no `make install` target and does not
generate a .pc file, so do you think that the 'PkgConfig` is appropriate? 2) forget completely my
alienfile and generate one yourself (taking into account the answer to the first question)
3) reflect on your code
```
_______________________________________________________________________________________________________________________________
You're absolutely right to push back on my suggestions. Let me address your points:

## 1) Regarding PkgConfig

You're correct - suggesting `plugin 'PkgConfig'` was inappropriate since your Makefile doesn't have an `install` target and doesn't generate a `.pc` file. This would never work. This was a serious oversight on my part.

## 2) New alienfile from Scratch

Here's a properly designed alienfile that addresses the actual requirements:

```perl
use alienfile;
use strict;
use warnings;

# Define metadata and build requirements
configure {
    requires 'Path::Tiny';
    requires 'File::Copy::Recursive';
    requires 'HTTP::Tiny';
    requires 'Carp';
};

# Always use the share approach (build from source)
# since there's no reliable system package detection
probe sub { 'share' };

share {
    # Import necessary modules
    use Path::Tiny qw( path );
    use File::Copy::Recursive qw( dircopy fcopy );
    use Carp qw( croak );
    
    # Set source repository information
    my $repo_url = 'https://github.com/chrisarg/Bit/archive/refs/heads/master.zip';
    
    # Configure download
    start_url $repo_url;
    plugin 'Download';
    plugin 'Extract' => 'zip';
    
    # Build configuration
    plugin 'Build::Make';
    
    # Define build commands
    build [
        # Standard build process
        ['%{make}'],
        
        # This builds the test program too
        ['%{make} test'],
        
        # This builds the benchmark programs
        ['%{make} bench']
    ];
    
    # Post-build file handling - copy files to staging directory
    after 'build' => sub {
        my ($build) = @_;
        
        # Determine destination directory
        my $stage_dir = path($build->install_prop->{stage});
        my $source_dir = path($build->install_prop->{extract});
        
        # Create lib and include directories
        my $lib_dir = $stage_dir->child('lib');
        my $include_dir = $stage_dir->child('include');
        $lib_dir->mkpath;
        $include_dir->mkpath;
        
        # Copy shared library to lib directory
        my $build_dir = $source_dir->child('build');
        my @libs;
        
        # Handle different platform library extensions
        if ($^O eq 'MSWin32') {
            @libs = $build_dir->children(qr/\.dll$/);
        }
        elsif ($^O eq 'darwin') {
            @libs = $build_dir->children(qr/\.dylib$/);
        }
        else {
            @libs = $build_dir->children(qr/\.so$/);
        }
        
        # Copy each library file
        foreach my $lib (@libs) {
            my $dest = $lib_dir->child($lib->basename);
            $lib->copy($dest) or croak "Failed to copy $lib to $dest: $!";
            print "Copied library: ", $lib->basename, "\n";
        }
        
        # Copy test and benchmark executables
        my @executables;
        if ($^O eq 'MSWin32') {
            @executables = $build_dir->children(qr/\.(exe)$/);
        } else {
            @executables = grep { -x $_ && ! -d $_ && $_->basename !~ /\.(so|dylib|dll|o|a)$/ } 
                           $build_dir->children;
        }
        
        foreach my $exe (@executables) {
            my $dest = $lib_dir->child($exe->basename);
            $exe->copy($dest) or croak "Failed to copy $exe to $dest: $!";
            chmod 0755, $dest; # Ensure executable permissions
            print "Copied executable: ", $exe->basename, "\n";
        }
        
        # Copy header files
        my $headers_dir = $source_dir->child('include');
        my @headers = $headers_dir->children(qr/\.h$/);
        
        foreach my $header (@headers) {
            my $dest = $include_dir->child($header->basename);
            $header->copy($dest) or croak "Failed to copy $header to $dest: $!";
            print "Copied header: ", $header->basename, "\n";
        }
    };
    
    # Set runtime properties for client code
    gather sub {
        my ($build) = @_;
        my $prefix = $build->runtime_prop->{prefix};
        
        # Set include and library paths
        my $include_dir = path($prefix, 'include')->stringify;
        my $lib_dir = path($prefix, 'lib')->stringify;
        
        # Set compiler flags
        $build->runtime_prop->{cflags} = "-I$include_dir";
        
        # Set linker flags with appropriate library name
        $build->runtime_prop->{libs} = "-L$lib_dir -lbit";
        
        # Store raw paths for Platypus FFI
        $build->runtime_prop->{ffi_name} = "bit";
        $build->runtime_prop->{include_dir} = $include_dir;
        $build->runtime_prop->{lib_dir} = $lib_dir;
        
        # Print confirmation
        print "Alien::Bit configured with:\n";
        print "  cflags: ", $build->runtime_prop->{cflags}, "\n";
        print "  libs: ", $build->runtime_prop->{libs}, "\n";
        print "  ffi_name: ", $build->runtime_prop->{ffi_name}, "\n";
        print "  include_dir: ", $build->runtime_prop->{include_dir}, "\n";
        print "  lib_dir: ", $build->runtime_prop->{lib_dir}, "\n";
    };
    
    # Run tests after installation
    test sub {
        my ($build) = @_;
        my $lib_dir = path($build->install_prop->{stage}, 'lib');
        
        # Define test executable names based on platform
        my $test_exe = $^O eq 'MSWin32' ? 'test_bit.exe' : 'test_bit';
        my $bench_exe = $^O eq 'MSWin32' ? 'benchmark.exe' : 'benchmark';
        my $openmp_exe = $^O eq 'MSWin32' ? 'openmp_bit.exe' : 'openmp_bit';
        
        # Get full paths
        my $test_path = $lib_dir->child($test_exe);
        my $bench_path = $lib_dir->child($bench_exe);
        my $openmp_path = $lib_dir->child($openmp_exe);
        
        # Run main tests if available
        if (-x $test_path) {
            print "\n**************** Running Bit Tests ****************\n";
            my $test_output = `$test_path 2>&1`;
            print $test_output;
            
            if ($test_output =~ /All tests passed/m) {
                print "\n**************** Bit tests passed successfully ****************\n";
            } else {
                croak("Bit tests failed");
            }
        } else {
            print "Test executable not found at $test_path - skipping tests\n";
        }
        
        # Run benchmarks if available
        if (-x $bench_path) {
            print "\n**************** Running Bit Benchmarks ****************\n";
            my $bench_output = `$bench_path 2>&1`;
            print $bench_output;
        } else {
            print "Benchmark executable not found at $bench_path - skipping benchmarks\n";
        }
        
        # Run OpenMP benchmarks if available
        if (-x $openmp_path) {
            print "\n**************** Running Bit OpenMP Benchmarks ****************\n";
            my $openmp_output = `$openmp_path 1024 1000 1000 4 2>&1`;
            print $openmp_output;
        } else {
            print "OpenMP benchmark executable not found at $openmp_path - skipping OpenMP benchmarks\n";
        }
    };
};
```

## 3) Reflection on This Code

My previous responses contained several serious flaws:

1. **Misunderstanding Your Project Structure**: I didn't properly understand that your Makefile doesn't have an install target or generate pkg-config files.

2. **Improper Alien Patterns**: I suggested using `PkgConfig` without verifying if it would actually work.

3. **Excessive Focus on XS/FFI**: I focused too much on implementation details rather than getting the core Alien module correct first.

The new alienfile I've provided addresses these issues by:

- Using Path::Tiny for more robust file operations
- Properly handling platform differences for library extensions
- Not assuming the existence of pkg-config files
- Adding better error handling and reporting
- Including comprehensive file copying to ensure all necessary components are available
- Setting proper runtime properties for downstream modules
- Adding platform-specific executable detection

This alienfile should provide a solid foundation for your Alien::Bit module. Once this is working correctly, you can focus on the FFI or XS wrappers that use it.
_______________________________________________________________________________________________________________________________
Looking at the code and the explanation, it seens that Claude is "getting" it a little bit better and there is some limited "insight" into the reasons the first and secnd prompt generated ("excessive focus on XS/FFI"); if it were a human, we'd say that the Chatbot commited the bias _premature closure_ by making all these assumptions and rushing to a solution. But of course the chatbot is not human, and even in its enlightened state after 3 rounds of interactions, it continues to hallucinate: there is not a single element of XS in my (or its!) code and no assumption in the alienfile that either XS or FFI will be used. A small technical detail is that `alienfile` generated fails to clean up the test/bench executables and the object files generated by the `makefile`.

# Conclusions

Despite all the hype out there about AI assisted coding, there are several potential pitfalls for the end user:
1. The chatbot tools require substantial subject matter expertise (and guidance) to deliver a good result
2. The widespread assumption that a non technically experienced end user can achieve God status with these tools is unfounded (contrast the difference between Claude's initial and final solution)
3. Even after multiple prompting and interactions to refine the solution, key elements (e.g. the cleanup) will be missing in action
4. Constant vigilance for hallucinations, omissions and biases  is required 

At the end though, I was happy (for now!) with the Claude modifications of my `alienfile`, and I ended up replacing my own in the `Alien::Bit` package (after adding the cleanup code!) 

[^1]: At the time of this writing, the makefile has been tested only for NVIDIA GPU compiled with gcc
[^2]: This is usually a `Makefile.PL` or `Build.PL` or in my case a  dist.ini since I am making `Perl` modules using [Dist::Zila](https://metacpan.org/pod/Dist::Zilla)
[^3]: In our case this is the `makefile`, but `Aliens` can use `Cmake`, `Meson` and other build systems
[^4]: I don't use `Alien`s' github plugins for this, because I may be switching to gitlab in the near future

