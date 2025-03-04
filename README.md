# (expertly named) CPPCxN
This version of PCxN is optimized for speed by adjusting code structure and using C++(Rcpp package). The changes do not alter the idea of the original PCxN method found [here](http://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1006042). 

## Rcpp (C++)
The Rcpp package is widely used to speed-up R scripts by using lower level C++ calculations, majorly decreasing looping overheads (among other things). A large number of R functions are supported by the package (e.g. `colMeans()`) but inevitably a few have to be implemented manually, or worst case (very slow) imported from R (e.g. `cor.shrink()`).

## R code
The original version is based on two nested loops (through tissues and process pair elements). The latter one is distributed between cores. Turning these 2 loops into Rcpp would be hard as we would need to implement all fucntions included to C++. The main problem would be the `mclapply()` function, which would require us to find a C++ that provides the same functionality.

What we can do is restructure the code in the following ways:
1. Precalculate the Summary matrices (joint and disjoint)
2. Subset the pair elements. That means only calculate interesting relationships (e.g. pathways to drugs)
3. Enable the concatenationg of result matrices that come from running PCxN with different arguments

## Code changes

### R code adjustments/additions
A more detailed report of the runtime comparisons can be found [here](https://docs.google.com/spreadsheets/d/1359vW0Rua5wTmuHGkCloJ8ft-8A-TltPQdUCIocctBE/edit?usp=sharing) In short, the steps to be implemented:

1. Pre-calculate the matrix that holds the disjoint summaries(estimated represent 40% of the geneset pairs).
2. Pre-calculate the matrix that holds the joint summaries (multi-core)
3. Select desired relationships feature (e.g. pathway - pathway, CMAP - pathway etc)
4. Replace/remove Shrinkage, if decided so.
5. Concatenate result matrices 

|                              R code                                   |    Status     |      Note      |
| ----------------------------------------------------------------------|:-------------:|:--------------:|
| Pre-calculate the matrix that holds the disjoint summaries            | Done          |                |
| Pre-calculate the matrix that holds the joint summaries (multi-core)  |    On hold    | Need multicore |
| Desired relationships feature                                         | Done          | disjoint + sub | 
| Replace/remove Shrinkage                                              |  Not decided  |                |
| Concatenate matrices                                                  |      Done     |                |

#### Concatenate matrices
We have to test whether we can add up matrices which are results of different PCxN runs. Towards that direction we will run a mini experiment of concatenating different sized matrices with different argumetns and see how the actual numbers in the concatenated matrix are affected.

##### Experiments setup
The geneset file DPD.Hs.gs.mini.PDN.RDS is used. We build 3 versions of PCxN (main_01_disjoint.R):
1. Base: using the first 50 gene sets from the above file to produce a baseline output matrix with correlations, p-value and adjusted p-values
2. Plus_10: This PCxN version uses the 50 base gene sets and 10 new ones.
3. Plus_20: This version uses the 50 base gene sets and 20 new ones (not the same as the 10 new ones above)

Once we have the 3 matrices we will cross-check the correlations, p-values and adjusted p-values for any changes. 

###### Results
The results show that only the p.Adjust column changes (as expected), which means we can concatenate matrices like that if we just recalculate the adjusted p-value in estimates02. The resulting matrices can be found here: */shared/hidelab2/shared/Sokratis/PCxN_Plos/Concatenate experiment*

### New C++ code
Implementing current PCxN functions in Rcpp. Four functions have been translated to C++ but at the moment they don't offer a speed advantage (yet). More effort will go towards implementing them as effieciently as possible. A new function has been created (`precalculate_matrices.cpp`) that pre-calculates both joint and disjoint matrices. To gain speed with this function, C++ and multicores have to be used.

|         C++ code          |    Status     |      Note      |
| --------------------------|:-------------:|:--------------:|
|       GetSummary.cpp      |  Implemented  |  Optimization  |
|    OverlapCoefficient.cpp |     Done      |                |
|        ShrinkCor.cpp      |     Done      |  On dis + sub  | 
|       ShrinkPCor.cpp      |     Done      |                |
| precalculate_matrices.cpp |    On hold    |                |

## Memory Issues
The original PCxN code was built and run for 1,330 gene sets/pathways. When we increased the pathways to 5000, memory issues occcured during execution on sharc. Using rmem=48G, mem=48G and 14 cores scripts that handled tissues with more than one GSEs got stopped shortly after the first completed GSE. One thing we can do is adjust the scripts to clean-up after each loop but still the memory needed at any time will definitely be considerably more than before. Only considering a subset of pairs would do wonders for that problem. 

## Comparisons
1. [R/C++](https://docs.google.com/spreadsheets/d/18Z3dXQc22dZ0K_BdF5zopYjVm5YCLSJ_TPR49G47ap0/edit?usp=sharing)
2. [250gene sets,500 gene sets, subpairs](https://docs.google.com/spreadsheets/d/1359vW0Rua5wTmuHGkCloJ8ft-8A-TltPQdUCIocctBE/edit?usp=sharing)

## Suggestions
1. [C++] Try and avoid ALL imports from R. Especially in repeated parts of the code it results in way slower execution. Specifically important case: `cor.shrink()` in *ShrinkCor.cpp* and *ShrinkPCor.cpp* scripts. This is the current reason we can't gain speed from these two scripts.
2. The code is structured to run per tissue. By splitting the batches(each one having a few tissues) we can cut down the total runtime.  

## Notes about the code
1. In Shrink(P)Cor : currently I am importing the cor.shrink function from R and run in C++. Altought the results are identical, this is not speed optimized. The optimal solution would be using a pure C++ function 

---
                                                        Software
---

## Improved PCxN
This PCxN version is working and has the following characteristics:
1. All arguments that may change (e.g. genesets file or relationships desired) can be handled through the batch script. No R code changes required.
2. *Pick relationships* feature
3. All outputs found in */shared/hidelab2/shared/Sokratis/PCxN_Plos/output_improved_PCxN*
4. Enable concatenate relationship matrices

### How to run
Run batch script `improved_PCxN_sharc` by first adjusting:
1. Batch memory used
2. Batch number of cores (suggested 14)
3. Batch max time to let running (suggested ?)
4. rels (gene sets relationships desired). Use: 1,2,3,4,5,6 to request all relationships. If you need only some of the relationships use any subset of the following numbers:

|        Desired Pairs        | No |
| --------------------------- |:--:|
|        pathway-CMAP         |  1 |
|        pathway-CTD          |  2 |
|      pathway-PharmGKB       |  3 |
|         CMAP-CTD            |  4 |
|       CMAP-PharmGKB         |  5 |
|     CMAP.up-CMAP.down       |  6 |
|       pathway-L1000CDS2     |  7 |
| L1000CDS2.up-L1000CDS2.down |  8 |
|  L1000CDS2.up-L1000CDS2.up  |  9 |
|L1000CDS2.down-L1000CDS2.down| 10 |
|       pathway-pathway       | 11 |

### How to run `powerful_PCxN`, step-by-step
1. make gene set file (see `genesets`) and put into `data`
2. make output file of correct structure (see `example_output_folder`)
3. modify `src/PCxN_sharc_1` and `src/PCxN_sharc_2` with correct output file name, relationships (rels, see above) and gene set file name
4. run `PCxN_sharc_1` with 1-134 jobs (approx ram 5gb * 4 cores, approx 1h per tissue)
5. run `PCxN_sharc_2` with 1:300 jobs, if it fails, check how many parts and adjust (see details in `powerful_PCxN`) - also adjust the memory (currently 1 part successfully finishes with 8 cores * 20gb, part 1 took about 1h for approx 8000 gene sets)

More info in `powerful_PCxN` on github


## Additional tools
Additional tools that provide flexibility to PCxN.

### P-combiner
This tool allows the combination of **2+** PCxN output matrices into one. The combination requires both output matrices to be calculated beforehand. Adjusted p-values are re-calculated and duplicates are removed (check **Concatenate matrices** section above). 

Due to the simplicity and low resource requirements, this script can be run both locally(`P-combiner_local.R`) and on sharc (`P-combiner_sharc.R`). The latter should only be used in extreme cases where a large number of matrices needs to be combined or the matrices themselves are very large in size.

#### How to use
Specify the path to parts folder with first part of the parts name eg. `output1/mean_pcor2_barcode/res/pcxn_mean_pcor2_barcode` (output of `improved_PCxN_estimates02.R`) of all matrices without `_parts11.RDS`. In both local/sharc:
1. The first argument is composed of the matrix names (the part before "_part1.RDS") seperated by commas.
2. The second argument is composed of number of parts for each run separated by commas.
3. The third argument is the entire name of the combined matrix without the .RDS filename extension (e.g. matri1_matrix2).

Example
1. Local(load the function first): `p_combiner(inputs = c("pcxn_conc_base","pcxn_conc_plus_10","pcxn_conc_plus_20"), nparts = c(2,2,2), output_name = "pcxn_conc_base_plus_10_plus_20")`
2. Sharc: `Rscript P-combiner_sharc.R pcxn_conc_base,pcxn_conc_plus_10,pcxn_conc_plus_20 2,2,2 pcxn_conc_base_plus_10_plus_20`

### P-adder
This tool takes a PcxN output matrix as a base and runs/adds the results of a different set of genesets (much like running a new PCxN without the pairs that have already been calculated). Checks that same named genesets have an identical gene member list. The tools requires only the base matrix to be pre-calculated and calculates only the necessary pairs (the ones that don't already exist in the base matrix).

#### How to use
Place the base matrix, base geneset and new geneset files in the data folder. you will handle the batch script `P-adder_PCxN_sharc`. In sharc fill in the following variables:
1. base_matrix: The name of the base(already calculated) matrix you added earlier
2. gs_base: The geneset used for the base matrix above
3. gs_new: The geneset to be used for the new data you want to compute
4. old_folder: The path(including a backlash at the end of it) where the part_1, part_2.RDS (etc.) files for the base matirx are kept
5. rels: Which relationships you desire for the new computation (works identically to improved PCxN)

The output files (combined matrix in PCxN and PDN style) can be found in the output_adder_PCxN folder.

### P-converter
This tool converts the original PCxN output (matrix where each row represents a pathway pair with p-value/adj. p-value/cor/overlap) to the PDN-style input(square matrix, where rows and columns are the same pathways and the cells only hold correlation between the different pathway pairs). The converted file is named "square_" + original name.

#### How to use
Place the result matrix (from estiamtes03.R) in the same folder as the converter script.In both local/sharc:
1. The only argument is the original result's name as a string 

Example
1. Local(load the function first): `convert_pcxn_result("original_PCxN_result.RDS")`
2. Sharc: 

## Improved PCxN additional packages needed
1. svd (cran)
2. corpcor (cran)
3. parallel (in R core packages)
4. metap (cran)

## Improved PCxN diagram
![Improved PCxN diagram](https://github.com/hidelab/CPPCxN/blob/master/improved_PCxN.png "PCxN diagram")
