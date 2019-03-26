# bamcount

BigWig and BAM related utilities.

## Build dependencies

* [htslib](http://www.htslib.org)
    * See `get_htslib.sh` for a script that gets a recent version and compiles it with minimal dependencies
* [libBigWig](https://github.com/dpryan79/libBigWig)
    * See `get_libBigWig.sh` for a script that gets a recent version anf compile it

## Compiling

From root directory:

```
mkdir -p build && cd build && cmake .. && make
```

## Usage

```
bamcount --coverage /path/to/bamfile --threads <num_threads> --no-head --coverage --bigwig <sample_name> --auc --min-unique-qual <min_qual> --annotation <annotated_intervals.bed> <sample_name> --frag-dist <sample_name> --alts <sample_name>
```

Concrete example command for sample `SRR1258218` (NA12878 Illumina RNA-seq):

```
bamcount ./SRR1258218.sorted.bam --threads 4 --no-head --coverage --bigwig SRR1258218 --auc SRR1258218 --min-unique-qual 10 --annotation ./exons.bed SRR1258218 --frag-dist SRR1258218 --alts SRR1258218
```

## Subcommands

For any and all subcommands below, if run together, `bamcount` will do only one pass through the BAM file.
While any given subcommand may not be particularly fast on its own, doing them all together can save time.

Subcommand `--coverage` is the only subcommand that will output a BigWig file (currently).
However, if along with `--coverage`, `--min-unique-qual` and `--bigwig` are specified the "unique" coverage will also be written as a BigWig file.

### `bamcount --auc <output_file_prefix>`

Reports area-under-coverage across all bases (one large sum of overlapping reads, per-base).
This will also report additional counts for:
 * `min-unique-qual` only for reads with MAPQ >= to this setting
 * `--annotation`: only for bases in the annotated regions
 
This computes the coverage (same as `--coverage`) under the hood, but won't output it unless `--coverage` is also passed in.

### `bamcount --coverage`

Generates per-base counts of overlapping reads across all of the genome.  
Typically this is used to produce a BigWig, but can be used w/o the `--bigwig` option to just output TSVs

### `bamcount --coverage --min-unique-qual <qual_value>`

In addition to producing coverage output for all reads, will also produce coverage output only for reads which have a mapping quality (MAPQ) >= <qual_value> (typically set to `10`.  

### `bamcount --coverage --annotation <annotated_file.bed> <output_file_prefix>`

In addition to reporting per-base coverage, this will also sum the per-base coverage within annotated regions submitted as a BED file.
If `--min-unique-qual` is submitted, this will produce a second set of sums for the "unique" reads that pass this filter.

The annotation BED file should be sorted by chromosome (at least), as coverage is calculated once per chromosome.
However, coordinate sorting is not necessary (though it doesn't hurt).
 
### `bamcount --coverage --double-count`

By default, `bamcount --coverage` will not double count coverage where paired-end reads overlap (same as `mosdepth`'s default).
However, double counting can be allowed with this option, which may result in faster running times.

### `bamcount --coverage --bigwig <output_file_prefix>`

Outputs coverage vectors as BigWig file(s) (including for `--min-unique-qual` option).

### `bamcount --frag-dist <output_file_prefix>`

Outputs fragment length distribution adjusting for intron lengths.

Mean, mode statistics are reported at the end of the output with string tag `STATS`.

This uses the absolute value of the `TLEN` field but uses additional filters similar to [csaw](https://github.com/LTLA/csaw)'s fragment length calculation.

The following alignments are filtered out:

 * secondary
 * supplementary
 * not paired
 * unmapped
 * mate unmapped
 * discordant (mates not on same chromosome/reference)

Further, read mates must be on forward/reverse strands and the forward mate must not be downstream of the reverse mate.

Intron length(s) in the paired alignments are also subtracted from the `TLEN` field except where the `TLEN` field is smaller than the combined length of the introns, in which case the `TLEN` is reported as is.

These numbers should be taken as an estimation of the fragment length distribtion.

### `bamcount --alts <output_file_prefix>`

Outputs information about non-reference-matching portions of reads.
Output is comma separated with 4 fields:

| Pos   |                                            Descrtiption |
|-------|---------------------------------------------------------|
| 1     | Reference record ID                                     |
| 2     | POS field (0-based offset of leftmost aligned ref base) |
| 3     | Operation label (see table below)                       |
| 4     | Extra info (see table below)                            |


These could be of a few types, summarized in this table.  All of
these are available when the `MD:Z` extra flag is present.  If not
present, only the ones with "Yes" in the "No `MD:Z`" column are
reported.

| Label | Type       |          Extra info  | No `MD:Z`  |
|-------|------------|----------------------|------------|
| `X`   | Mismatch   |            Read base |         No |
| `D`   | Deletion   |      # deleted bases |        Yes |
| `I`   | Insertion  |  Inserted read bases |        Yes |
| `S`   | Soft clip  |   Soft ckipped bases |        Yes |
| `H`   | Hard clip  |            (nothing) |        Yes |
| `P`   | Padding    |            (nothing) |        Yes |

See the usage message for options, which can selectively disable some
of the outputs listed above.  E.g. the soft-clipping outputs can be
very large, so they're not printed unless `--include-softclip` is
specified.

### `bamcount --alts --include-softclip <output_file_prefix>`

In addition to the alternate base output, this reports the bases
that were softclipped at the ends (start/end) of the read.
These are bases which are left in the sequence but don't align.

The softclipped bases themselves are printed to the file named with the
prefix passed into the `--alts` option.  The total number of sofclipped
bases and the total number of bases from the query sequences of alignments
that that aren't unmapped or secondary are reported to the file named
with the prefix passed to `--include-softclip`.

Warning: using this option w/o modifiers (e.g. `--only-polya`) 
could blow up the `--alts` output size as the full softclipped
sequence is printed in the 4th column in the table above ("Extra info").

### `bamcount --alts --include-softclip <output_file_prefix> --only-polya`

If reporting softclipped bases, this option will limit the report to only
those bases that have the following:

* Count of bases in the sofclip (column 4 below) has to be >= 3
* % of base (A/T) of softclipped bases for an alignment >= 80%

No other sofclipped bases are reported.

Output is comma separated with 7 fields:

| Pos   |                                                                      Description|
|-------|----------------------------------------------------------------------------------|
| 1     | Reference record ID                                                              |
| 2     | POS field (0-based ref offset of either leftmost or rightmost aligned base)      |
| 3     | Operation label (always "S")                                                     |
| 4     | Number of bases in the softclip (run length)                                     |
| 5     | Direction to move from POS ('+' for end of alignment, '-' for start of alignment)|
| 6     | Base (A/T)                                                                       |
| 7     | Count of the base in column 6                                                    |

### `bamcount --junctions <output_file_prefix>`

Extract locally co-occurring junctions from BAM.

This does not extract all potential junctions, only those for which a read (or read pair) had >= 2 junctions.

In a paired context, there must be at least 2 junctions across the 2 read mates to be output.

Output is tab separated with 6-12 fields (the last 6 fields are for a 2nd mate if applicable):

| Pos    |                                                                            Description|
|--------|---------------------------------------------------------------------------------------|
| 1      | Reference record ID                                                                   |
| 2      | POS field (1-based ref offset of either leftmost base)                                |
| 3      | Mapping strand (0 forward, 1 reverse)                                                 |
| 4      | Insert length (0 if not paired)                                                       |
| 5      | Cigar string (useful for determining anchor lengths)                                  |
| 6      | List of junction coordinates (comma-delimited)                                        |
| 7*     | Mate reference record ID                                                              |
| 8*     | Mate POS field (1-based ref offset of either leftmost base)                           |
| 9*     | Mate mapping strand (0 forward, 1 reverse)                                            |
| 10*    | Mate insert length (0 if not paired)                                                  |
| 11*    | Mate cigar string (useful for determining anchor lengths)                             |
| 12*    | Mate list of junction coordinates (comma-delimited)                                   |

\*optional, output if a 2nd mate is present and has the required number of junctions.

### `bamcount --bam2fastq <output_file_prefix>`

Entirely separate mode from the other subcommands, this takes a BAM and outputs one or more FASTQ files (uncompressed).

Additional options:

 * `--filter-out <int>` same as `samtools -F <int>`, skips alignments with that bitflag set
 * `--filter-in <int>`  same as `samtools -f <int>`, only includes alignments with that bitflag set 
 * `--re-reverse` same as `Picard SamToFastq -RE_REVERSE` will reverse complement sequences which were aligned to the reverse strand
 * `--one-file` force output to one FASTQ file, even if paired
 
 If `--one-file` is not set, `bamcount` will attempt to output all paired sequences to 2 separate FASTQ files and any non-paired sequences to a 3rd file.
