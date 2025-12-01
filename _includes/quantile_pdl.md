```perl
#!/home/chrisarg/perl5/perlbrew/perls/current/bin/perl

use v5.38;
use Benchmark qw(:hireswallclock :all);
use Cwd;
use File::Basename;      # for extracting the filename from a path
use File::Find;
use File::Spec;          # for creating cross platform paths
use FindBin qw($Bin);    # for finding the script location
use PDL;
use PDL::IO::CSV ':all';
use PDL::NiceSlice;
use PDL::Ufunc;
use Text::CSV qw(csv);
my $random_sample_file = 'quantile_sample.csv';
my $R_results_file     = 'quantile_summary.csv';

sub read_csv_as_columns( $filepath, $sep_char = ',' ) {
    my $csv =
      Text::CSV->new( { binary => 1, auto_diag => 1, sep_char => $sep_char } );
    open my $fh, "<:encoding(utf8)", $filepath
      or die "$filepath: $!";
    my $header = $csv->getline($fh);
    my %csv_as_hash;
    my @aoa            = map { [] } @$header;
    my $num_of_columns = scalar @$header;
    while ( my $row = $csv->getline($fh) ) {
        for ( my $i = 0 ; $i < $num_of_columns ; $i++ ) {
            push @{ $aoa[$i] }, $row->[$i];
        }
    }
    close $fh;
    for ( my $i = 0 ; $i < $num_of_columns ; $i++ ) {
        $csv_as_hash{ $header->[$i] } = $aoa[$i];
    }
    return \%csv_as_hash;
}

sub quantile_type_3 {
    my ( $data, $pct ) = @_;
    my $sorted_data = $data->qsort;
    my $nelem       = $data->nelem;
    my $cum_ranks   = floor( $pct * $nelem );
    $data->index($cum_ranks);
}

my $NELEMENTS   = 1_000_000;
my $NQUANTILES  = 100;
my $NREPLICATES = 10;
my $data        = random($NELEMENTS) * 10 - 20;
my $pct         = sequence($NQUANTILES) / $NQUANTILES;

my $perl_results = timethese(
    $NREPLICATES,
    {
        pct             => sub { $data->pct($pct) },
        quantile_type_3 => sub {
            quantile_type_3( $data, $pct );
        },

    }
);

wcsv1D( $data, $random_sample_file );

# call the companion R script to benchmark R's quantile function
system(
"Rscript $Bin/quantile_speed.R $random_sample_file $R_results_file $NQUANTILES $NREPLICATES"
);    #or die "Failed to execute Rscript: $!";

my $csv = Text::CSV->new( { binary => 1, auto_diag => 1, sep_char => ',' } );
my %r_summary = %{ read_csv_as_columns( $R_results_file, ',' ) };

## now put the results together

my %combined_results;
my @perl_keys =  sort keys %$perl_results;
$combined_results{'Test'}       = [ @perl_keys, $r_summary{'test'}->@* ];
$combined_results{'Iterations'} = [
    ( map { $_->iters } @{$perl_results}{@perl_keys} ),
    $r_summary{'replications'}->@*
];
$combined_results{'Elapsed Time (s)'} = [
    ( map { sprintf( "%.6f", $_->cpu_a ) } @{$perl_results}{@perl_keys} ),
    ( map { sprintf( "%.6f", $_ ) } $r_summary{'elapsed'}->@* )
];

say "\nCombined Benchmark Results:\n";
printf "| %-25s | %-15s | %-10s | %-10s | %-20s |\n",
  "Test", "Iterations", "Elements", "Quantiles", "Elapsed Time (s)";
printf "|%s|%s|%s|%s|%s|\n", "-" x 27, "-" x 17, "-" x 12, "-" x 12, "-" x 22;
for my $i ( 0 .. $combined_results{'Test'}->@* - 1 ) {
    printf "| %-25s | %-15s | %-10s | %-10s | %-20s |\n",
      $combined_results{'Test'}->[$i],
      $combined_results{'Iterations'}->[$i],
      $NELEMENTS,
      $NQUANTILES,
      $combined_results{'Elapsed Time (s)'}->[$i];
}

```
