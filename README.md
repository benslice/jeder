JEDER
=====

Joint Estimate of Data and Error Rates
--------------------------------------

The goal of this software is to jointly estimate the most likely underlying
data and error rates of an experimental screen using technical or biological
replicates. Alternatives to this approach include relying on an external
gold-standard, or applying a "rule of thumb" on replicate data to generate a
gold standard. Often, the former is unavailable, and the latter is circular, as
such a "rule of thumb" assumes error rates are within a given range, and the
error estimates will depend heavily on the rule. Such a rule might take the
following form: "Any experiment in our screen will be considered a
true-positive "hit" if it qualifies as a "hit" in two or more out of our six
technical replicates." JEDER estimates the most likely False Positive Rate
(FPR), False Negative Rate (FNR), and the underlying "true" data profile
simultaneously, using a Markov-Chain Monte-Carlo approach.

JEDER estimates rates for Boolean data only. It therefore takes in quantitative
data, and a user defined rule/threshold for what constitutes a "hit" in each
individual replicate; a so called `hit_spec`. So if, for example, you want to
evaluate the data at several thresholds, or wish to consider positive and
negative values separately, these constitute individual and separate runs, each
with a different `hit_spec`.

Installation
------------
JEDER is written in python, and depends on several external libraries. These
are listed in the `requirements.txt` file and can be installed via pip in the
normal fashion `pip install -r requirements.txt`.


Usage
-----
`jeder.py` is written to be invoked from the command line. However, it contains
a relatively small number of functions, and a simple `__main__` routine, and so
should be relatively simple to use as a python module should you choose.

```
Usage:
   jeder.py [options] run <hit_spec> <input> <output>
   jeder.py [options] view <output>
```
`jeder.py` has one mandatory subcommand (`run` or `view`) 
  * `run` to evaluate the input data and save the results
  * `view` to view the results of a previous run

`jeder.py` has several mandatory parameters depending on the subcommand:
  * `hit_spec` a rule that defines a "hit" using one or more columns from the input file
  * `input` your input file.
  * `output` where to save results (as an hdf5 file)


To evaluate a dataset, use the `run` subcommand, along with a `hit_spec` to
define hits, along with the path to your dataset (`input`), and a path to save
the results (`output`).

Input File Format
-----------------

`input` will be read in using `pandas.read_table()`, which expects
tab-delimited, long-form data. Columns must be named, but can be named anything
you like, as you will tell JEDER which columns to look at for what information.
The file may be compressed with a standard file extension. You must supply the
following information in your input file:
  * `expid` some unique (within each replicate) identifier for this observation in the screen.
  * `repid` some identifier describing to which replicate does this observation belong.
  * one or more data columns, referenced in the `hit_spec` passed on the command line.

Hit Specification
-----------------
```
hit_spec: A "hit" is defined using a combination of threholds on one or more
         columns. Use the column names from the input file as variables.
         
         Currently supported syntax
         BASIC AND "(score < -0.08) & (pvalue < 0.05)"
         RELATIVE  "(col_A > col_B) & (col_C > 0)"

         Future work: 

         NO SPACES "(score<-0.08)&(pvalue<0.05)"
         ORs       "(score < -0.08) | (score  > 0.08)"

         Nested expressions are not allowed, and evaluated left to right.
         e.g. ( A | B & C) is evaluated as ((A | B) & C)
         If you need more complex logic, you will have to preprocess
         your data, and include the result in a column 
```


Example
-------
Suppose we have an input file, `input.txt`, with four columns: `GENE`,
`REPLICATE`, `SCORE`, and `PVALUE`; and we wish to evaluate the profiles with
the following rule: "a hit is defined as a `SCORE` > 0.08 **AND** a `PVALUE` <
0.05." We can then run JEDER thus:

```
jeder.py run --expid=GENE --repid=REPLICATE "(SCORE > 0.08) & (PVALUE < 0.05)" input.txt results.hdf5
```

Once it has completed, and saved the results to `results.hdf5` can view the results with:

```
jeder.py view results.hdf5
```

This would use the default options for number of iterations, and ranges to
search for FPR / FNR rates. You will very likely have to run JEDER multiple
times, successively narrowing the search ranges until you find a range that
outputs "well-behaved" posterior distributions. The ranges to search can be
defined on the command line, for example to limit the prior for FPR from 2%-3%,
run:

```
jeder.py run --expid=GENE --repid=REPLICATE "(SCORE > 0.08) & (PVALUE < 0.05)" input.txt results.hdf5 --fpr=0.02,0.03
```

Options
-------
JEDER uses `docopt` to parse the command line, and therefore is pretty
permissive about where you put options, and should throw an intelligible error
when it can't figure out what you give it. 

See also, `jeder.py --help`
```
Options:
   -b --burn=<int>         [default: 100] how many initial iterations
   -i --iterations=<int>   [default: 1000] how many subsequent iterations
   -e --expid=<colname>    [default: expid] which col contains the experiment id
   -r --repid=<colname>    [default: repid] which col contains the replicate id
   -p --fpr=<pspec>        [default: 0,0.05]  the prior distrubution for the FPR
   -n --fnr=<pspec>        [default: 0.2,0.8] the prior distrubution for the FNR
   -s --standard=<runfile> take "truth" vector from previous run instead of estimating it
   -t --trace              [default: False] save the profile trace (see NOTE) 
   -c --clobber            overwrite output file
   -q --quiet              do not print out messages, and disable the progress bar
   -h --help               show this help
```

