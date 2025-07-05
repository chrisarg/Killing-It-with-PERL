```perl
use strict;
use warnings;
use alienfile;

configure {
    requires 'Carp';
    requires 'File::Copy::Recursive';
    requires 'HTTP::Tiny';
    requires 'Path::Tiny';
};

## not doing system install, so must deal with the package name
## being not obvious

probe sub {
    return 'share';
};

share {
    use Carp;
    use File::Copy::Recursive qw(dircopy fcopy);
    use HTTP::Tiny;
    use File::Spec::Functions;

    my $install_root;
    my $path_to_static_lib;
    my $repo_url;
    my $repo_response;

    $repo_url = 'https://github.com/chrisarg/Bit/archive/refs/heads/master.zip';
    $repo_response = HTTP::Tiny->new->head($repo_url);
    croak
"Failed to download Bit repository: $repo_response->{status} $repo_response->{reason}"
      unless ( $repo_response->{success} );

    start_url $repo_url;

    plugin 'Download';
    plugin 'Extract' => 'zip';
    plugin 'Build::Make';



    ## build both the dynamic and the dynamic libs in a single step
    build [ ['%{make} all'], ['%{make} test'], ['%{make} bench'] ];

    ## various postbuild activities to facilitate the gathering of files
    after 'build' => sub {
        my ($build) = @_;
        ## move the bin directories into the final location
        ## this includes the suite of the edlib library
        if ( $build->meta_prop->{destdir} ) {
            my $destdir = $ENV{DESTDIR};
            $install_root =
              catfile( $ENV{DESTDIR}, $build->install_prop->{prefix} );
        }
        else {
            $install_root = catfile( $build->install_prop->{stage} );
        }
        my $source_directory   = $build->install_prop->{extract};
        my $lib_dest_directory = catfile( $install_root, 'lib' );
        dircopy( catfile( $source_directory, 'build' ), $lib_dest_directory );
        ## get the include files
        my $source_header = catfile( $source_directory, 'include', 'bit.h' );
        my $dest_header   = catfile( $install_root,     'include', 'bit.h' );
        print "\n**************** Copying header file ****************\n";
        print "Copying header file from $source_header to $dest_header\n";
        print "\n","==" x 50, "\n";
        mkdir( catfile( $install_root, 'include' ) )
          unless -d catfile( $install_root, 'include' );
        fcopy( $source_header, $dest_header );

    };

    gather sub {
        my ($build) = @_;
        my $prefix = $build->runtime_prop->{prefix};
        my $incdir = catfile( $prefix, 'include' );
        my $libdir = catfile( $prefix, 'lib' );

        # Set the compiler flags
        $build->runtime_prop->{cflags} = "-I$incdir";

        # Set the linker flags
        $build->runtime_prop->{libs}     = "-L$libdir -lbit";
        $build->runtime_prop->{ffi_name} = "bit";

        # store the actual paths if you ever have to debug
        $build->runtime_prop->{include_dir} = $incdir;
        $build->runtime_prop->{lib_dir}     = $libdir;

    };

    test sub {
        my ($build) = @_;
        my $binary_dest_directory =
          catfile( $build->install_prop->{stage}, 'lib' );
        my $runTests_exec = $^O eq 'MSWin32' ? 'test_bit.exe' : 'test_bit';
        $runTests_exec = catfile( $binary_dest_directory, $runTests_exec );
        print("Can't find test executable") if not -e $runTests_exec;
        print("\n**************** Running Bit Tests ****************\n");
        my $test_output = `$runTests_exec`;
        print $test_output;

        if ( $test_output =~ /All tests passed/m ) {
            print(
"\n**************** Bit tests passed successfully ****************\n"
            );
        }
        else {
            croak("Bit tests failed");
        }
        ## execute benchmarks
        my $runBench_exec = $^O eq 'MSWin32' ? 'benchmark.exe' : 'benchmark';
        $runBench_exec = catfile( $binary_dest_directory, $runBench_exec );
        print("Can't find benchmark executable") if not -e $runBench_exec;
        print("\n**************** Running Bit Benchmarks ****************\n");
        my $bench_output = `$runBench_exec`;
        print $bench_output;

        # execute openmp + gpu benchmarks
        my $runBenchOpenMP_exec =
          $^O eq 'MSWin32' ? 'openmp_bit.exe' : 'openmp_bit';
        $runBenchOpenMP_exec =
          catfile( $binary_dest_directory, $runBenchOpenMP_exec );
        print("Can't find openmp benchmark executable")
          if not -e $runBenchOpenMP_exec;
        print(
"\n**************** Running Bit OpenMP Benchmarks ****************\n"
        );
        my $bench_openmp_output = qx{$runBenchOpenMP_exec 1024 1000 1000 4};
        print $bench_openmp_output;

        unlink $runTests_exec;
        unlink $runBench_exec;
        unlink $runBenchOpenMP_exec;

        # delete object files that end in .o
        my @object_files = glob( catfile( $binary_dest_directory, '*.o' ) );
        unlink $_ for @object_files;
    };
};


```
