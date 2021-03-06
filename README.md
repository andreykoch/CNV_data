# CNV_data

Datafiles and scripts for training and running of the CNV caller.

## Data for training of CNV caller

Each file is a single comressed numpy array with the following dtype in the only 'values' key:
```python 
dtype=[('chromosome', 'i1'), ('bins_loess', '<f2')] 
```
and each array has length of 154,794, since the used bin size is 20,000 (hg19 reference). The "columns" are:
1. chromosome - chromosome number (0-based, 22 for X, 23 for Y)
2. bins_loess - loess normalized bin counts (bin size is 20,000) - used for training of PCA normalization. Loess normalized bin counts are computed using standard methods as reported in the article. 

Thus, the loess corrected read count for the whole chromosome 1 can be obtained as:
```python
import numpy as np
sample_npy = np.load('/path/to/sample/values.npz')['values']
read_count_chr1 = sum(sample_npy[sample_npy['chromosome']==0]['bins_loess'])
```

## Training

You need to have installed the following python packages to perform training (tested under python 3.5.2):
```
pip install numpy
pip install pandas
pip install dill
pip install sklearn
```

Use following steps to do the training:
1. PCA components are trained from a subset of the data (samples with higher read count (>10M)). These samples reside in the "pca" directory.
```python 
python create_pca.py
```
This command will create a file `pca_c20.pck` in the `data` directory, where the first 20 principal components are stored. 
2. New numpy files are created with the PCA attribute added.
```python 
python add_pca.py
```
This command will create a file `XXXX_pca.npz` for every `XXXX.npz` data file. These new files contain one new column `('bins_PCA', '<f2')`, which contain PCA normalized bin counts similarly as `bins_loess`.
3. Means of bin counts are trained from PCA normalized bincounts from all samples (from those in the "pca" dir, too). 
```python 
python train_means.py
```
This command will create `means.npy` and `means.tsv` files, which contain means and variances of bin counts (both loess and PCA normalized). Both files contain the same information. 

More info about the training is in the article. 

## CNV calling

You need to have `Rscript` installed (tested with R version 3.2.3):
```bash
sudo apt-get install r-base
```
and you also need the `DNAcopy` R package (installation in the R shell):
```R
source("http://bioconductor.org/biocLite.R")
biocLite("DNAcopy")
```

Furthermore, you need to have installed the following packages to run `cnv_caller.py`:
```
pip install numpy
pip install pandas
pip install rpy2
```
Python version 3.7 is recommended, 3.6 also works well and for version 3.5 you will need rpy2 package of version 2.8.6 (`pip install rpy2==2.8.6`).

Then you can run the CNV caller with the following:
```python 
python cnv_caller.py -m {trained_means} -b {bins_with_PCA} -o {output_dir} -k -f {fetal_fraction}
```
where `-m` points to the trained means (.npy), `-b` to the bins with PCA columns created with the `add_pca.py` script, `-o` points to the directory where output should be stored, `-f` specifies the fetal fraction of the sample and `-k` switches on filtering of bins. You can find more options by running `python cnv_caller.py --help`.

The CNV calling command will create two files in the `{output_dir}` directory:
1. `{name}_cbs.txt` - cnv calling result - TSV table with columns: 
 - serial number of called segment
 - start coordinate (in base pairs)
 - length in bins (bin is 20000 base pairs)
 - end coordinate (in base pairs)
 - chromosome (labeled from 0)
 - level - PCA normalized bincount average of the called segment (normalized around 0 by subtracting of the trained means)
 - effective_length - length in bins without filtered bins (bin is 20000 base pairs)
 - significant - 0 for non-significant (healthy), 1 for warning, 2 for reported fetus aberration, 3 for reported maternal aberration
 - color - color coding for significance (green, yellow, red, magenta)
 - expected_ff - expected fetal fraction from the called segment (useful only for aberrations)
An example of first few lines of the cbs file:
```
	start	length	end	chromosome	level	effective_length	significant	color	expected_ff
0	    820000	12419	 249200000	0	 0.0017	10713	0	green	 0.0004
1	     20000	12149	 243000000	1	-0.0097	11442	0	green	 0.0025
2	    100000	 9886	 197820000	2	 0.0047	 9596	0	green	 0.0012
3	     80000	 9535	 190780000	3	 0.0047	 9158	0	green	 0.0012
4	     40000	 9032	 180680000	4	-0.0006	 8611	0	green	 0.0002
...
```
2. `{name}_gaps.txt` - filtered bins, the format follows a similar pattern as the cbs file, an example of first few lines:
```
chromosome	start	length	end	full_start
0	0	41	820000	0
0	1400000	1	1420000	1400000
0	1440000	1	1460000	1440000
0	1560000	2	1600000	1560000
...
```
