---
title: "Vibe coding a Perl interface to a foreign library - Part 3"
date: 2025-09-02
---

After a very long hiatus due to the triplet of work-vacation-work, we return to Part 3 of my AI assisted coding of a Perl interface to a foreign library.
In the last couple of months, many things have happened on the AI front, including the release of additional models, and some much needed injection of reality into the hype, so my 
final part will have a different tone than the previous ones and try to focus less on details. For those who don't wont to read the github pages, feel free to browse the MetaCPAN package [Bit::Set](https://metacpan.org/pod/Bit::Set) which features the "vibecoding" section.

In these explorations, agentic LLMs were found particularly problematic, often 
stalling to generate a solution, focusing on the wrong thing when tests were
failing and often giving up. I therefore ended up not using them, and relied 
on the "Ask" mode of Github Copilot.

To build this module, I first created the distribution structure with (what else?)
[Dist::Zilla](https://metacpan.org/pod/Dist::Zilla), and then opened the folder
using VS Code. Subsequently, I provided as context the "bit.h" header file from
the [Bit library](https://github.com/chrisarg/Bit) and the associated README
markdown file. The prompt used was the following:

```text
    Forget everything I have said and your responses so far. Look at the description 
    of the project in README.md and the Abstract Data Type interface in bit.h. 
    Put your self in the place of a senior Perl engineer with extensive understanding
    of the C language. Your goal is to create a procedural Perl API to all the 
    functions in C using the Foreign Function Interface. Assume that we have already 
    implemented an Alien module (Alien::Bit) to install the foreign (C) dependency 
    bit, so make sure you use it! Look at the checked runtime exceptions in the 
    documentation of the C interface. Your goal is to incorporate them in the Perl 
    interface too, as long as the user has set the DEBUG environmental variable. 
    If the DEBUG variable has not been set, these runtime checks should be stripped 
    during the compile phase of the PERL program. To do so, please ensure that the 
    relevant check involves ONLY DEBUG, otherwise the code may not be stripped.
    Things to adhere to during the implementation:

    The functions for the Bit_T, should end up in the module Bit::Set, and those for 
    Bit_DB to Bit::Set::DB .
    1. Ensure that you implement the Perl interface to all the functions in the C 
    interface, i.e. don't implement some functions and then tell me the others are 
    implemented similarly! Reflect that you have implemented all the functions by 
    comparing the functions that are exported by the Perl module against the 
    functions declared in the bit.h interface (excluding of course the functions 
    defined as macros).
    2. The names of the methods in the Perl interface should match those of the C 
    interface exactly, without exceptions. However, you should not implement the 
    map function(s).
    3. When implementing the wrapper, combine a table driven approach with the 
    FFI's attach to maximize conciseness and reduct repetition of the code. 
    For example, you may want to use a hash with keys the function names that 
    the module will export. CAUTION: As a senior engineer you are probably aware 
    of the DRY principle (Don't Repeat Yourself). When you generate code please 
    balance DRY with the performance penalty of function evaluations (e.g. for checks).
    4. When implementing a function, do provide the POD documentation for it. 
    However, generate the POD after you have implemented the functions.
    5. After you have implemented the modules, generate a simple test that will 
    generate a Bit::Set of capacity of 1024 bits, set the first, second and 5th one 
    and see if the popcount is equal to 3.
```
Claude did get *most* things right:

* it generated 3 chunks of code corresponding to `Bit::Set`,  `Bit::Set::DB` and the single test file
* the table driven approach was implemented effectively reducing the number of lines of code that had to be written
* The checked runtime exceptions in the C interface were incorporated in the Perl using a wrapper function that was provided to `FFI::Platypus` attach.
* The `FFI::Platypus::Record` was correctly selected into the implementation for the C structure that passes options for the CPU/GPU enhanced container functions.
* the POD documentation was skeleton but at least it followed the grouping in the  [Bit](https://github.com/chrisarg/Bit) README file. 

However, the code itself would not work, requiring a few minor tweaks that are
summarized below:
* Incorporating runtime exceptions

The relevant section is shown below and exhibits numerous problems. 
```perl
    for my $name ( sort keys %functions ) {
        my $spec        = $functions{$name};
        my @attach_args = ( $name, $spec->{args}, $spec->{ret} );
        $ffi->attach(@attach_args);
        if ( DEBUG && exists $spec->{check} ) {
            my $checker = $spec->{check};
            push @attach_args, wrapper => sub {    
                my $orig = shift;
                $checker->(@_);
                return $orig->(@_);
            };
        }
    }
```
When the DEBUG variable is not set, it is unclear whether the check for DEBUG
will strip the code that adds the runtime exception wrapper at compile time.
The pattern discussed in the Perl documentation states that a simple test
of the form `if (DEBUG) { ... }` will strip everything within the block, but
will a test of the form `if ( DEBUG && exists $spec->{check} ) { ... }` do the
same?
Secondly, the attachment of the wrapper function to the FFI call is also a
concern: it takes place early in the process, before the DEBUG check is made.
Thirdly, the snippet `push @attach_args, wrapper => sub { ... } ` as it pushes
*two* arguments into the function call for C< attach >.
If one looks into the documentation for [FFI::Platypus::attach](https://metacpan.org/pod/FFI::Platypus#attach),
```perl
    $ffi->attach($name => \@argument_types => $return_type);
    $ffi->attach([$c_name => $perl_name] => \@argument_types => $return_type);
    $ffi->attach([$address => $perl_name] => \@argument_types => $return_type);
    $ffi->attach($name => \@argument_types => $return_type, \&wrapper);
    $ffi->attach([$c_name => $perl_name] => \@argument_types => $return_type, \&wrapper);
    $ffi->attach([$address => $perl_name] => \@argument_types => $return_type, \&wrapper);
```
it becomes clear that the maintainer is using the fat comma instead of the
regular comma to pass consecutive arguments into the C<attach> function. 
However, the chatbot is confusing the syntax and adding a hashlike key-value 
pair when pushing the arguments of C<attach>.

All these problems are reasonably easy to fix, by breaking the test involving 
DEBUG into two nested ifs, moving the attach invocation at the end of the loop,
and pushing the code reference without the `wrapper => ` part into the arguments
of the attach function.

* The fat comma strikes again

The container module (`Bit::Set::DB`) uses a C structure to pass options to the 
CPU/hardware accelerator device . This C structure is passed by value and thus 
should be passed as a `FFI::Platypus::Record`, created either as a separate 
module file, or nested in the `Bit::Set::DB` module. The code that was actually
generated by Claude looked like this:
```perl
    {
        package Bit::Set::DB::SETOP_COUNT_OPTS;
        use FFI::Platypus::Record;
        record_layout_1(
        'num_cpu_threads' => 'int',
        'device_id' => 'int',
        'upd_1st_operand' => 'bool',
        'upd_2nd_operand' => 'bool',
        'release_1st_operand' => 'bool',
        'release_2nd_operand' => 'bool',
        'release_counts' => 'bool',
            );
    }
```
In the documentation for [FFI::Platypus::Record](https://metacpan.org/pod/FFI::Platypus::Record#record_layout_1), one can clearly see that the function record_layout_1 
receives arguments as `record_layout_1($type => $name, ... );` , i.e. the fat comma 
is used to separate consecutive arguments to the function, and not as part of the 
definition of a hash. However Claude must "think" that it is dealing with a hash,
as it reverses the order of the arguments to make the "keys" unique.
The fix is rather simple, i.e. one simply reverses the order of the arguments.

* Forgetting the proper way to register records with FFI

Interestingly enough, the chatbot failed to properly register the type of the 
record with FFI. In the original output, it included the line:
```perl
    $ffi->type( 'Bit::Set::DB::SETOP_COUNT_OPTS' => 'SETOP_COUNT_OPTS_t' ) 
```
rather than the correct
```perl
    $ffi->type( 'record(Bit::Set::DB::SETOP_COUNT_OPTS)' => 'SETOP_COUNT_OPTS_t' )
```
* Naive interpretation of returned pointers in the FFI interface

A subtle LLM mistake concerns the handling of returned pointers from FFI calls. 
A function in C that is declared as `int* foo(...);` may use the returned
pointer to provide a single value, or an array of values. Consider the proposal
for `BitDB_count` in the table driven interface:
```perl
    BitDB_count => {
        args  => ['Bit_DB_T'],
        ret   => 'int*',
        check => sub {
            my ($set) = @_;
            die "BitDB_count: set cannot be NULL" if !defined $set;
        }
    }
```
When `FFI::Platypus` encounters this return type, it will interpret the type
as a hash reference to a Perl scalar as stated explicitly in the [documentation](https://metacpan.org/pod/FFI::Platypus#Pointers).
The correct way to handle this is to declare the function as returning an [opaque pointer](https://metacpan.org/pod/FFI::Platypus#Opaque-Pointers-(buffers-and-strings)).
In particular, one would rewrite the last snippet as:
```perl
    BitDB_count => {
        args  => ['Bit_DB_T'],
        ret   => 'opaque',
        check => sub {
            my ($set) = @_;
            die "BitDB_count: set cannot be NULL" if !defined $set;
        }
    }
```
By doing so, one ends up receiving the memory address of the buffer as a Perl 
scalar value, rather than a reference to Perl scalar, which must be dereferenced
to yield the first (and only the first!) element of the array that is accessible
through the pointer. Please refer to the documentation of [FFI::Platypus](https://metacpan.org/pod/FFI::Platypus) for 
more information on working with opaque pointers.
The documentation of [Bit::Set::DB](https://metacpan.org/pod/Bit::Set::DB) contains examples and usage patterns for 
working with arrays returned from the Perl interface to C<Bit>.

Having fixed these errors, I proceeded to generate a Perl version of the C
test suite, by providing as context the (fixed) modules : `Bit::Set` and 
`Bit::Set::DB` as well as the C source code for ["test_bit.c"](https://github.com/chrisarg/Bit/blob/main/tests/test_bit.c). The actual
prompt was the single liner:
```text
    Convert this test file written in C to Perl, using the Bit::Set and Bit::Set::DB modules. 
```
The major problem with this conversion was the failure of the chatbot to
generate correct code when testing the functions that are loading/extracting
information from bitsets or containers into raw buffers. For example the 
following code is supposed to test the extraction of bits from a bitset:
```perl
    my $bitset = Bit_new(SIZE_OF_TEST_BIT);
    Bit_bset( $bitset, 2 );
    Bit_bset( $bitset, 0 );

    my $buffer_size = Bit_buffer_size(SIZE_OF_TEST_BIT);
    my $buffer = "\0" x $buffer_size;
    my $bytes = Bit_extract( $bitset, $buffer ); 

    my $first_byte = unpack('C', substr($buffer, 0, 1))
    is( $first_byte, 0b00000101, 'Bit_extract produces correct buffer' );
    Bit_free( \$bitset );
```
However, the code is utterly wrong (and segfaults!) as one has to provide
the memory address of the buffer, not the Perl scalar value. The fix is to
generate the buffer as a Perl string and then use `FFI::Platypus::Buffer` 
to extract the memory address of the storage buffer used by the Perl scalar:
```perl
    my $scalar = "\0" x $buffer_size;    
    my ( $buffer, $size ) = scalar_to_buffer $scalar;   
    my $bytes =  Bit_extract( $bitset, $buffer );  
    my $first_byte = unpack( 'C', substr( $scalar, 0, 1 ) ) ; 
```
**I only had to edit about 6 lines out of ~ 400 to port the C test suite to Perl.**

I had no luck with more complex "assignments" such as generating the benchmark of the `Bit::Set::DB` GPU and multi-threaded CPU accelerated interface.
Claude realized that this task, which necessitates the understanding of memory ownership and management across the interfaces of two languages and 
between DRAM and GPU RAM is way out of its league and declined by putting out the note in the snippet below:

```perl

sub database_match_container_cpu {
    my ($db1, $db2, $num_threads) = @_;
    ...
    my $results = BitDB_inter_count_cpu($db1, $db2, $opts);
    my $nelem = BitDB_nelem($db1) * BitDB_nelem($db2);
    
    my $max = 0;
    # Note: In a real implementation, we'd need to properly handle the results array
    # This is a simplified version since we can't directly access C array elements in Perl
    # without additional FFI buffer handling
    
    return $max;
}
```

**So what are MY takehome points after this exercise?** I'd say the messages are mixed. I found the agentic bots to not be very helpful, as they entered these long repetitive and useless reflections
without being able to fix the problems they identified when the build of `Bit::Set` failed. The "Ask" mode chatbots could generate lots of code, but with subtle mistakes. 
Success with porting test suites from one language to the other was highly variable, ranging from near perfect to outright refusal to execute a difficult task. 
On the other hand, the chatbot was excellent as an auto-complete, often helping me finish the structure of the POD and putting together the scaffold to fill things in. 
_Are the chatbots worth the investment?_ I'd say that as an amateur programmer the chatbot was helpful to get me out of the writer's block, but did not really save much time for me. 
I can not imagine that a professional who knows what they are doing would be assisted much by the AI chatbots, which is what the [METR study](https://arxiv.org/abs/2507.09089) said:
**overall, experienced developers experienced a 19% drop in productivity with AI assistance**.
