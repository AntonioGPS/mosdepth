fast BAM/CRAM depth calculation for **WGS**, **exome**, or **targetted sequencing**.

![logo](https://user-images.githubusercontent.com/1739/29678184-da1f384c-88ba-11e7-9d98-df4fe3a59924.png "logo")

[![Build Status](https://travis-ci.org/brentp/mosdepth.svg?branch=travis)](https://travis-ci.org/brentp/mosdepth)

`mosdepth` can output per-base depth about twice as fast `samtools depth`--about 25 minutes of CPU time for a 30X
genome.

it can output mean per-window depth given a window size--as would be used for CNV calling.

it can output the mean per-region given a BED file of regions.

it can create a distribution of proportion of bases covered at or above a given threshhold.

## usage

```
mosdepth

  Usage: mosdepth [options] <BAM-or-CRAM>
  
  -t --threads <threads>     number of BAM decompression threads [default: 0]
  -F --flag <FLAG>           exclude reads with any of the bits in FLAG set [default: 1796] (default excludes unmapped, qcfailed, duplicates, secondary)
  -c --chrom <chrom>         chromosome to restrict depth calculation.
  -Q --mapq <mapq>           mapping quality threshold [default: 0]
  -b --by <bed|window>       BED file of regions or an (integer) window-size.
  -f --fasta <fasta>         fasta file for use with CRAM files.
  -d --distribution <file>   a cumulative distribution file (coverage, proportion).
  -h --help                  show help
```

See the section below for more info on distribution.

If `--by` is a BED file with 4 or more columns, it is assumed the the 4th column is the name.
That name will be propagated to the `mosdepth` output in the 4th column with the depth in the 5th column.
If you don't want this behavior, simply send a bed file with 3 columns.

### exome example

To calculate the coverage in each exome capture region:
```
mosdepth --by capture.bed sample.exome.bam > sample.exome.coverage.bed
```
For a 5.5GB exome file and all 1,195,764 ensembl exons as the regions,
this completes in 1 minute 38 seconds with a single CPU.

To calculate per-base coverage:

```
mosdepth sample.exome.bam > sample.coverage.txt
```

### WGS example

For per-base whole-genome coverage:

```
mosdepth $sample.wgs.bam > $sample.txt
```

For 500-base windows (and a coverage distribution):

```
mosdepth -d $sample.dist --by 500 $sample.wgs.bam > $sample.500.bed
```

## Installation

Unless you want to install [nim](https://nim-lang.org), simply download the
[binary from the releases](https://github.com/brentp/mosdepth/releases).

If you get an error about "`libhts.so` not found", set `LD_LIBRARY_PATH`
to the directory that contains `libhts.so`. e.g.

```LD_LIBRARY_PATH=~/src/htslib/ mosdepth -h```

If you do want to install from source, see the [travis.yml](https://github.com/brentp/mosdepth/blob/master/.travis.yml)
and the [install.sh](https://github.com/brentp/mosdepth/blob/master/scripts/install.sh).

## distribution output

This is **useful for QC**.

The `--distribution` option outputs a cumulative distribution indicating then
proportion of genome bases (or the proportion of the `--by`) that were covered
for at least a given coverage value.

This will write a file to the requested path with values indicating the coverage
threshold and the proportion of bases covered at that threshold.

A python plotting script is provided in `scripts/plot-dist.py` that will make 
plots like below. Use is `python scripts/plot-dist.py *.dist` and the output
is `dist.png`.

Using something like that, we can plot the distribution from the entire genome.
Below we show this for samples with ~60X coverage:

![WGS Example](https://user-images.githubusercontent.com/1739/29646192-2a2a6126-883f-11e7-91ab-049295eb3531.png "WGS Example")

We can also run `mosdepth` on just the Y chromosome (--chrom Y) to verify that males and females
track separately. Below, we that see female samples cluster along the axes while male samples have
close to 30X coverage for almost 40% of the genome.

![Y Example](https://user-images.githubusercontent.com/1739/29646191-2a246564-883f-11e7-951a-aa68d7a1a6ed.png "Y Example")

See [this blog post](http://www.gettinggeneticsdone.com/2014/03/visualize-coverage-exome-targeted-ngs-bedtools.html) for
more details.

## how it works

As it encounters each chromosome, `mosdepth` creates an array the length of the chromosome.
For every start it encounters, it increments the value in that position of the array. For every
stop, it decrements that position. From this, the depth at a particular position is the
cumulative sum of all array positions preceding it (a similar algorithm is used in BEDTools
where starts and stops are tracked separately). `mosdepth` **avoids double-counting
overlapping mate-pairs** and it **tracks every aligned part of every read using the CIGAR
operations**. Because of this data structure, the the coverage `distribution` calculation
can be done without a noticeable increase in run-time. The image below conveys the concept:

![alg](https://user-images.githubusercontent.com/1739/29647913-d79ab028-8848-11e7-86cf-60d4b087bc3b.png "algorithm")

This array accounting is very fast. There are no extra allocations or objects to track and
it is also conceptually simple. For these reasons, it is faster than `samtools depth` which
works by using the [pileup](http://samtools.sourceforge.net/pileup.shtml) machinery that
tracks each read, each base. 

The `mosdepth` method has some limitations. Because a large array is allocated and it is
required (in general) to take the cumulative sum of all preceding positions to know the depth
at any position, it is slower for small, 1-time regional queries. It is, however fast for
window-based or BED-based regions, because it first calculates the full chromosome coverage
and then reports the coverage for each region in that chromosome. Another downside is it uses
more memory than samtools. The amount of memory is approximately equal to 32-bits * longest chrom
length, so for the 249MB chromosome 1, it will require 500MB of memory.

`mosdepth` is written in [nim](https://nim-lang.org/) and it uses our [htslib](https://github.com/samtools/htslib)
via our nim wrapper [hts-nim](https://github.com/brentp/hts-nim/)

## output

When the `--by` argument is *not* specified, the output of `mosdepth`. The output looks like:

```
chr1	216	0
chr1	232	1
chr1	236	2
chr1	255	1
chr1	9991	0
chr1	9992	1
chr1	9993	2
chr1	9994	6
chr1	9995	5
chr1	9996	6
```

Each line indicates the end of the previous coverage level and the start of the next.

So the first line indicates that:

+ the values on chr1 from 0 to 216 (in 0-based, half-open (BED) coordinates) have a depth of 0. 
+ the values on chr1 from 216 to 232 have a depth of 1 
+ 232..236 == 2
+ 236..255 == 1
+ 255..9991 == 0
+ and so on ...

## speed and memory comparison

`mosdepth`, `samtools`, `bedtools`, and `sambamba` were run on a 30X genome.
relative times are relative to mosdepth per-base mode with a single thread.

`mosdepth` can report the mean depth in 500-base windows genome-wide info
under 9 minutes of user time with 3 threads.

| format |    tool    | threads  | mode   | relative time | run-time | memory |
| ------ | ---------- | -------- | ------ | ------------- | -------  | -------|
|  BAM   |  mosdepth  |    1     | base   |     1         |  25:23   |  1196  |
|  BAM   |  mosdepth  |    3     | base   |    0.57       |  14:27   |  1197  |
|  CRAM  |  mosdepth  |    1     | base   |    1.17       |  29:47   |  1205  |
|  CRAM  |  mosdepth  |    3     | base   |    0.56       |  14:08   |  1225  |
|  BAM   |  mosdepth  |    3     | window |    0.34       |  8:44    |  1277  |
|  BAM   |  mosdepth  |    1     | window |    0.80       |  20:26   |  1212  |
|  CRAM  |  mosdepth  |    3     | window |    0.35       |  8:47    |  1233  |
|  CRAM  |  mosdepth  |    1     | window |    0.88       |  22:23   |  1209  |
|  BAM   |  sambamba  |    1     | base   |    5.71       | 2:24:53  |  166   |
|  BAM   |  samtools  |    1     | base   |    1.98       | 50:12    |  27    |
|  CRAM  |  samtools  |    1     | base   |    1.79       | 45:21    |  451   |
|  BAM   |  bedtools  |    1     | base   |    5.31       | 2:14:44  |  1908  |

### Accuracy

We compared `samtools depth` with default arguments to `mosdepth` without overlap detection and discovered **no
differences across the entire chromosome**.

