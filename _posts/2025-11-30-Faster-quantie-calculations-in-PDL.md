---
title: "Faster quantile calculations in the Perl Data Language(PDL)"
date: 2025-11-30
---

I was writing a data intensive code in `Perl` relying heavily on `PDL` for some statistical calculations (estimation of percentile points in some very BIG vectors, e.g. 100k to 1B elements), when 
I noticed that PDL was taking a very (and unusually long!) time to produce results compared to my experience in Python.
This happened irrespective of whether one used the `pct` or `oddpct` functions in [PDL::Ufunc](https://metacpan.org/pod/PDL::Ufunc). 

The performance degradation had a very interesting quantitative aspect: if one asked `PDL` to return a single percentile it did so very fast; 
but if one were to ask for more than one percentiles, the time-to-solution increased linearly with the number of percentiles specified. 
Looking at the source code of the `pct` function, it seems that it is implemented by calling the function `pctover`, which according to the PDL documentation "_Broadcasts over its inputs._"

But what is exactly **broadcasting**? According to [PDL::Broadcasting](https://metacpan.org/dist/PDL/view/lib/PDL/Broadcasting.pod) : "[broadcasting] can produce very compact and very fast PDL code by avoiding multiple nested for loops that C and BASIC users may be familiar with. 
The trouble is that it can take some getting used to, and new users may not appreciate the benefits of broadcasting." 
Reading the relevant PDL examples and revisiting the [NumPy documentation](https://numpy.org/doc/stable/user/basics.broadcasting.html) (which also uses this technique), 
broadcasting : _treats arrays with different shapes during arithmetic operations. Subject to certain constraints, the smaller array is “broadcast” across the larger array so that they have compatible shapes. 
Broadcasting provides a means of vectorizing array operations so that looping occurs in C instead of Python._.

It seems that when one does something like:
```perl
use PDL::Lite;
my $very_big_ndarray = ... ; # code that constructs a HUGE PDL ndarrat
my $pct              = sequence(100)/100;   # all percentiles from 0 to 100%
my $pct_values       = pct( $very_big_ndarray, $pct);
```
the broadcasting effectively executes sequentially the code for calculating a single percentile and concatenates the results. 

The problem with broadcasting for this operation is that the percentile calculation includes a VERY expensive operation, namely the 
sorting of the `$very_big_darray` before the (trivial) calculation of the percentile from the sorted values as detailed in 
[Wikipedia](https://en.wikipedia.org/wiki/Percentile). So when the percentile operation is broadcast by `PDL`, the sorting is repeated for each 
percentile value in `$pct`, leading to catastrophic loss of performance!

How can we fix this? It turns out to be reasonably trivial : we need to reimplement the percentile function so that it does not broadcast. 
One of the simplest quantile functions to implement, is the one based on the empirical cumulative distribution function (this corresponds to the Type 3 quantile in the 
classification by [Hyndman and Fan](https://www.tandfonline.com/doi/abs/10.1080/00031305.1996.10473566)).
This one can be trivially implemented in Perl using `PDL` as:

```perl
sub quantile_type_3 {
    my ( $data, $pct ) = @_;
    my $sorted_data = $data->qsort;
    my $nelem       = $data->nelem;
    my $cum_ranks   = floor( $pct * $nelem );
    $data->index($cum_ranks);
}
```
(The other quantiles can be implemented equally trivially using affine operations as explained in `R`'s documentation of the [quantile](https://www.rdocumentation.org/packages/stats/versions/3.6.2/topics/quantile) function).

To see how well this works, I wrote a [`Perl benchmark script`](https://github.com/chrisarg/Killing-It-with-PERL/blob/main/_includes/quantile_pdl.md) that benchmarks 
the builtin function `pct`,  the `quantile_type_3` function on synthetic data and then calls the companion 
[`R script`](https://github.com/chrisarg/Killing-It-with-PERL/blob/main/_includes/quantile_speed.md) to profile the 9 quantile functions and the 3 sort 
functions in R for the same dataset.

I obtained the following performance figures in my old Xeon: the "de-broadcasted" version of the quantile function achieves the same performance as the R implementations, whereas the 
`PDL` broadcasting version is 100 times slower.

| Test                      | Iterations      | Elements   | Quantiles  | Elapsed Time (s)     |
|---------------------------|-----------------|------------|------------|----------------------|
| pct                       | 10              | 1000000    | 100        | 132.430000           |
| quantile_type_3           | 10              | 1000000    | 100        | 1.320000             |
| pct_R_1                   | 10              | 1000000    | 100        | 1.290000             |
| pct_R_2                   | 10              | 1000000    | 100        | 1.281000             |
| pct_R_3                   | 10              | 1000000    | 100        | 1.274000             |
| pct_R_4                   | 10              | 1000000    | 100        | 1.283000             |
| pct_R_5                   | 10              | 1000000    | 100        | 1.290000             |
| pct_R_6                   | 10              | 1000000    | 100        | 1.286000             |
| pct_R_7                   | 10              | 1000000    | 100        | 1.233000             |
| pct_R_8                   | 10              | 1000000    | 100        | 1.309000             |
| pct_R_9                   | 10              | 1000000    | 100        | 1.291000             |
| sort_quick                | 10              | 1000000    | 100        | 1.220000             |
| sort_shell                | 10              | 1000000    | 100        | 1.758000             |
| sort_radix                | 10              | 1000000    | 100        | 0.924000             |

As can be seen from the table, the sorting operations account mostly for the bulk of the execution time of the quantile functions.

Two major takehome points:
1) don't be afraid to look under the hood/inside the blackbox when performance is surprisingly disappointing!
2) be careful of broadcasting operations in `PDL`, `NumPy`, or `Matlab`.
