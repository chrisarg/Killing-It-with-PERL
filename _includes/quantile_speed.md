```r
library(rbenchmark)
library(data.table)

## read the name of the CSV file and the number of quantiles to compute from command line arguments
args <- commandArgs(trailingOnly = TRUE)
csv_file <- args[1]
summary_file<-args[2]
n_quantiles <- as.integer(args[3])
n_replicates <- as.integer(args[4])

data <- as.data.frame(fread(csv_file,header=FALSE))[,1]

probvector<-seq(0,1,by=1/n_quantiles);
summary<-benchmark(
  "vec1"={quantile(data,probs=probvector,type=1)},
  "vec2"={quantile(data,probs=probvector,type=2)},
  "vec3"={quantile(data,probs=probvector,type=3)},
  "vec4"={quantile(data,probs=probvector,type=4)},
  "vec5"={quantile(data,probs=probvector,type=5)},
  "vec6"={quantile(data,probs=probvector,type=6)},
  "vec7"={quantile(data,probs=probvector,type=7)},
  "vec8"={quantile(data,probs=probvector,type=8)},
  "vec9"={quantile(data,probs=probvector,type=9)},
  "sort_quick" = {sort( data, method = "quick")},
  "sort_shell" = {sort( data, method = "shell")},
  "sort_radix" = {sort( data, method = "radix")},
replications=n_replicates,
columns = c(
      "test", "replications", "elapsed"))

rownames(summary)<-summary$test;
write.csv(summary[c(paste("vec", 1:9, sep=""),"sort_quick","sort_shell","sort_radix"),], file=summary_file, row.names=FALSE)
```
