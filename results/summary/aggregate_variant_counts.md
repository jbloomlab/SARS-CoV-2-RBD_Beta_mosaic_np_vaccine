# Aggregate variant counts for all samples
Separate `Snakemake` rules count the observations of each variant in each sample from the Illumina barcode sequencing.
This Python Jupyter notebook aggregates all of this counts, and then adds them to a codon variant table.

## Set up analysis
### Import Python modules.
Use [plotnine](https://plotnine.readthedocs.io/en/stable/) for ggplot2-like plotting.

The analysis relies heavily on the Bloom lab's [dms_variants](https://jbloomlab.github.io/dms_variants) package:


```python
import glob
import itertools
import math
import os
import warnings

import Bio.SeqIO

import dms_variants.codonvarianttable
from dms_variants.constants import CBPALETTE
import dms_variants.utils
import dms_variants.plotnine_themes

from IPython.display import display, HTML

import pandas as pd

from plotnine import *

import yaml
```

Set [plotnine](https://plotnine.readthedocs.io/en/stable/) theme to the gray-grid one defined in `dms_variants`:


```python
theme_set(dms_variants.plotnine_themes.theme_graygrid())
```

Versions of key software:


```python
print(f"Using dms_variants version {dms_variants.__version__}")
```

    Using dms_variants version 0.8.10


Ignore warnings that clutter output:


```python
warnings.simplefilter('ignore')
```

Read the configuration file:


```python
with open('config.yaml') as f:
    config = yaml.safe_load(f)
```

Make output directory if needed:


```python
os.makedirs(config['counts_dir'], exist_ok=True)
```

## Initialize codon variant table
Initialize the [CodonVariantTable](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html#dms_variants.codonvarianttable.CodonVariantTable) using the wildtype gene sequence and the CSV file with the table of variants:


```python
wt_seqrecord = Bio.SeqIO.read(config['wildtype_sequence'], 'fasta')
geneseq = str(wt_seqrecord.seq)
primary_target = wt_seqrecord.name
print(f"Read sequence of {len(geneseq)} nt for {primary_target} from {config['wildtype_sequence']}")
      
print(f"Initializing CodonVariantTable from gene sequence and {config['codon_variant_table']}")
      
variants = dms_variants.codonvarianttable.CodonVariantTable(
                geneseq=geneseq,
                barcode_variant_file=config['codon_variant_table'],
                substitutions_are_codon=True,
                substitutions_col='codon_substitutions',
                primary_target=primary_target)
```

    Read sequence of 603 nt for B1351 from data/wildtype_sequence.fasta
    Initializing CodonVariantTable from gene sequence and results/variants/codon_variant_table.csv


## Read barcode counts / fates
Read data frame with list of all samples (barcode runs):


```python
print(f"Reading list of barcode runs from {config['barcode_runs']}")

barcode_runs = (pd.read_csv(config['barcode_runs'])
                .assign(sample_lib=lambda x: x['sample'] + '_' + x['library'],
                        counts_file=lambda x: config['counts_dir'] + '/' + x['sample_lib'] + '_counts.csv',
                        fates_file=lambda x: config['counts_dir'] + '/' + x['sample_lib'] + '_fates.csv',
                        )
                .drop(columns='R1')  # don't need this column, and very large
                )

assert all(map(os.path.isfile, barcode_runs['counts_file'])), 'missing some counts files'
assert all(map(os.path.isfile, barcode_runs['fates_file'])), 'missing some fates files'

display(HTML(barcode_runs.to_html(index=False)))
```

    Reading list of barcode runs from data/barcode_runs.csv



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>date</th>
      <th>experiment</th>
      <th>library</th>
      <th>antibody</th>
      <th>concentration</th>
      <th>sort_bin</th>
      <th>selection</th>
      <th>sample</th>
      <th>experiment_type</th>
      <th>number_cells</th>
      <th>frac_escape</th>
      <th>sample_lib</th>
      <th>counts_file</th>
      <th>fates_file</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>220118</td>
      <td>beta_19-26</td>
      <td>lib1</td>
      <td>none</td>
      <td>0</td>
      <td>ref</td>
      <td>reference</td>
      <td>beta_19-26-none-0-ref</td>
      <td>ab_selection</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>beta_19-26-none-0-ref_lib1</td>
      <td>results/counts/beta_19-26-none-0-ref_lib1_counts.csv</td>
      <td>results/counts/beta_19-26-none-0-ref_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_19-26</td>
      <td>lib2</td>
      <td>none</td>
      <td>0</td>
      <td>ref</td>
      <td>reference</td>
      <td>beta_19-26-none-0-ref</td>
      <td>ab_selection</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>beta_19-26-none-0-ref_lib2</td>
      <td>results/counts/beta_19-26-none-0-ref_lib2_counts.csv</td>
      <td>results/counts/beta_19-26-none-0-ref_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_19</td>
      <td>lib1</td>
      <td>mosaic_6848</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_19-mosaic_6848-3125-abneg</td>
      <td>ab_selection</td>
      <td>875492.0</td>
      <td>0.096</td>
      <td>beta_19-mosaic_6848-3125-abneg_lib1</td>
      <td>results/counts/beta_19-mosaic_6848-3125-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_19-mosaic_6848-3125-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_19</td>
      <td>lib2</td>
      <td>mosaic_6848</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_19-mosaic_6848-3125-abneg</td>
      <td>ab_selection</td>
      <td>1002826.0</td>
      <td>0.136</td>
      <td>beta_19-mosaic_6848-3125-abneg_lib2</td>
      <td>results/counts/beta_19-mosaic_6848-3125-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_19-mosaic_6848-3125-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_20</td>
      <td>lib1</td>
      <td>mosaic_6849</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_20-mosaic_6849-3125-abneg</td>
      <td>ab_selection</td>
      <td>957229.0</td>
      <td>0.105</td>
      <td>beta_20-mosaic_6849-3125-abneg_lib1</td>
      <td>results/counts/beta_20-mosaic_6849-3125-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_20-mosaic_6849-3125-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_20</td>
      <td>lib2</td>
      <td>mosaic_6849</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_20-mosaic_6849-3125-abneg</td>
      <td>ab_selection</td>
      <td>1100621.0</td>
      <td>0.132</td>
      <td>beta_20-mosaic_6849-3125-abneg_lib2</td>
      <td>results/counts/beta_20-mosaic_6849-3125-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_20-mosaic_6849-3125-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_21</td>
      <td>lib1</td>
      <td>mosaic_6850</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_21-mosaic_6850-3125-abneg</td>
      <td>ab_selection</td>
      <td>862307.0</td>
      <td>0.089</td>
      <td>beta_21-mosaic_6850-3125-abneg_lib1</td>
      <td>results/counts/beta_21-mosaic_6850-3125-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_21-mosaic_6850-3125-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_21</td>
      <td>lib2</td>
      <td>mosaic_6850</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_21-mosaic_6850-3125-abneg</td>
      <td>ab_selection</td>
      <td>938801.0</td>
      <td>0.105</td>
      <td>beta_21-mosaic_6850-3125-abneg_lib2</td>
      <td>results/counts/beta_21-mosaic_6850-3125-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_21-mosaic_6850-3125-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_22</td>
      <td>lib1</td>
      <td>mosaic_6851</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_22-mosaic_6851-3125-abneg</td>
      <td>ab_selection</td>
      <td>751160.0</td>
      <td>0.080</td>
      <td>beta_22-mosaic_6851-3125-abneg_lib1</td>
      <td>results/counts/beta_22-mosaic_6851-3125-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_22-mosaic_6851-3125-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_22</td>
      <td>lib2</td>
      <td>mosaic_6851</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_22-mosaic_6851-3125-abneg</td>
      <td>ab_selection</td>
      <td>952949.0</td>
      <td>0.103</td>
      <td>beta_22-mosaic_6851-3125-abneg_lib2</td>
      <td>results/counts/beta_22-mosaic_6851-3125-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_22-mosaic_6851-3125-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_23</td>
      <td>lib1</td>
      <td>mosaic_6852</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_23-mosaic_6852-3125-abneg</td>
      <td>ab_selection</td>
      <td>896191.0</td>
      <td>0.094</td>
      <td>beta_23-mosaic_6852-3125-abneg_lib1</td>
      <td>results/counts/beta_23-mosaic_6852-3125-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_23-mosaic_6852-3125-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_23</td>
      <td>lib2</td>
      <td>mosaic_6852</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_23-mosaic_6852-3125-abneg</td>
      <td>ab_selection</td>
      <td>988504.0</td>
      <td>0.107</td>
      <td>beta_23-mosaic_6852-3125-abneg_lib2</td>
      <td>results/counts/beta_23-mosaic_6852-3125-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_23-mosaic_6852-3125-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_24</td>
      <td>lib1</td>
      <td>mosaic_6853</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_24-mosaic_6853-3125-abneg</td>
      <td>ab_selection</td>
      <td>856793.0</td>
      <td>0.091</td>
      <td>beta_24-mosaic_6853-3125-abneg_lib1</td>
      <td>results/counts/beta_24-mosaic_6853-3125-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_24-mosaic_6853-3125-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_24</td>
      <td>lib2</td>
      <td>mosaic_6853</td>
      <td>3125</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_24-mosaic_6853-3125-abneg</td>
      <td>ab_selection</td>
      <td>952068.0</td>
      <td>0.104</td>
      <td>beta_24-mosaic_6853-3125-abneg_lib2</td>
      <td>results/counts/beta_24-mosaic_6853-3125-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_24-mosaic_6853-3125-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_25</td>
      <td>lib1</td>
      <td>homotypic_6881</td>
      <td>7812</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_25-homotypic_6881-7812-abneg</td>
      <td>ab_selection</td>
      <td>973619.0</td>
      <td>0.101</td>
      <td>beta_25-homotypic_6881-7812-abneg_lib1</td>
      <td>results/counts/beta_25-homotypic_6881-7812-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_25-homotypic_6881-7812-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_25</td>
      <td>lib2</td>
      <td>homotypic_6881</td>
      <td>7812</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_25-homotypic_6881-7812-abneg</td>
      <td>ab_selection</td>
      <td>826391.0</td>
      <td>0.086</td>
      <td>beta_25-homotypic_6881-7812-abneg_lib2</td>
      <td>results/counts/beta_25-homotypic_6881-7812-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_25-homotypic_6881-7812-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_26</td>
      <td>lib1</td>
      <td>homotypic_6883</td>
      <td>7812</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_26-homotypic_6883-7812-abneg</td>
      <td>ab_selection</td>
      <td>787699.0</td>
      <td>0.083</td>
      <td>beta_26-homotypic_6883-7812-abneg_lib1</td>
      <td>results/counts/beta_26-homotypic_6883-7812-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_26-homotypic_6883-7812-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220118</td>
      <td>beta_26</td>
      <td>lib2</td>
      <td>homotypic_6883</td>
      <td>7812</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_26-homotypic_6883-7812-abneg</td>
      <td>ab_selection</td>
      <td>853705.0</td>
      <td>0.085</td>
      <td>beta_26-homotypic_6883-7812-abneg_lib2</td>
      <td>results/counts/beta_26-homotypic_6883-7812-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_26-homotypic_6883-7812-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_27-34</td>
      <td>lib1</td>
      <td>none</td>
      <td>0</td>
      <td>ref</td>
      <td>reference</td>
      <td>beta_27-34-none-0-ref</td>
      <td>ab_selection</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>beta_27-34-none-0-ref_lib1</td>
      <td>results/counts/beta_27-34-none-0-ref_lib1_counts.csv</td>
      <td>results/counts/beta_27-34-none-0-ref_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_27-34</td>
      <td>lib2</td>
      <td>none</td>
      <td>0</td>
      <td>ref</td>
      <td>reference</td>
      <td>beta_27-34-none-0-ref</td>
      <td>ab_selection</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>beta_27-34-none-0-ref_lib2</td>
      <td>results/counts/beta_27-34-none-0-ref_lib2_counts.csv</td>
      <td>results/counts/beta_27-34-none-0-ref_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_27</td>
      <td>lib1</td>
      <td>homotypic_6880</td>
      <td>5000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_27-homotypic_6880-5000-abneg</td>
      <td>ab_selection</td>
      <td>1012281.0</td>
      <td>0.113</td>
      <td>beta_27-homotypic_6880-5000-abneg_lib1</td>
      <td>results/counts/beta_27-homotypic_6880-5000-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_27-homotypic_6880-5000-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_27</td>
      <td>lib2</td>
      <td>homotypic_6880</td>
      <td>5000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_27-homotypic_6880-5000-abneg</td>
      <td>ab_selection</td>
      <td>768288.0</td>
      <td>0.090</td>
      <td>beta_27-homotypic_6880-5000-abneg_lib2</td>
      <td>results/counts/beta_27-homotypic_6880-5000-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_27-homotypic_6880-5000-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_28</td>
      <td>lib1</td>
      <td>homotypic_6882</td>
      <td>5000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_28-homotypic_6882-5000-abneg</td>
      <td>ab_selection</td>
      <td>812393.0</td>
      <td>0.104</td>
      <td>beta_28-homotypic_6882-5000-abneg_lib1</td>
      <td>results/counts/beta_28-homotypic_6882-5000-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_28-homotypic_6882-5000-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_28</td>
      <td>lib2</td>
      <td>homotypic_6882</td>
      <td>5000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_28-homotypic_6882-5000-abneg</td>
      <td>ab_selection</td>
      <td>728487.0</td>
      <td>0.084</td>
      <td>beta_28-homotypic_6882-5000-abneg_lib2</td>
      <td>results/counts/beta_28-homotypic_6882-5000-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_28-homotypic_6882-5000-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_29</td>
      <td>lib1</td>
      <td>homotypic_6884</td>
      <td>5000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_29-homotypic_6884-5000-abneg</td>
      <td>ab_selection</td>
      <td>802387.0</td>
      <td>0.099</td>
      <td>beta_29-homotypic_6884-5000-abneg_lib1</td>
      <td>results/counts/beta_29-homotypic_6884-5000-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_29-homotypic_6884-5000-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_29</td>
      <td>lib2</td>
      <td>homotypic_6884</td>
      <td>5000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_29-homotypic_6884-5000-abneg</td>
      <td>ab_selection</td>
      <td>638644.0</td>
      <td>0.079</td>
      <td>beta_29-homotypic_6884-5000-abneg_lib2</td>
      <td>results/counts/beta_29-homotypic_6884-5000-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_29-homotypic_6884-5000-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_30</td>
      <td>lib1</td>
      <td>homotypic_6885</td>
      <td>5000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_30-homotypic_6885-5000-abneg</td>
      <td>ab_selection</td>
      <td>901855.0</td>
      <td>0.111</td>
      <td>beta_30-homotypic_6885-5000-abneg_lib1</td>
      <td>results/counts/beta_30-homotypic_6885-5000-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_30-homotypic_6885-5000-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_30</td>
      <td>lib2</td>
      <td>homotypic_6885</td>
      <td>5000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_30-homotypic_6885-5000-abneg</td>
      <td>ab_selection</td>
      <td>729795.0</td>
      <td>0.090</td>
      <td>beta_30-homotypic_6885-5000-abneg_lib2</td>
      <td>results/counts/beta_30-homotypic_6885-5000-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_30-homotypic_6885-5000-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_31</td>
      <td>lib1</td>
      <td>NHP_mosaic_MA292</td>
      <td>1000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_31-NHP_mosaic_MA292-1000-abneg</td>
      <td>ab_selection</td>
      <td>1076671.0</td>
      <td>0.094</td>
      <td>beta_31-NHP_mosaic_MA292-1000-abneg_lib1</td>
      <td>results/counts/beta_31-NHP_mosaic_MA292-1000-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_31-NHP_mosaic_MA292-1000-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_31</td>
      <td>lib2</td>
      <td>NHP_mosaic_MA292</td>
      <td>1000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_31-NHP_mosaic_MA292-1000-abneg</td>
      <td>ab_selection</td>
      <td>804993.0</td>
      <td>0.087</td>
      <td>beta_31-NHP_mosaic_MA292-1000-abneg_lib2</td>
      <td>results/counts/beta_31-NHP_mosaic_MA292-1000-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_31-NHP_mosaic_MA292-1000-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_32</td>
      <td>lib1</td>
      <td>NHP_mosaic_AO2</td>
      <td>1000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_32-NHP_mosaic_AO2-1000-abneg</td>
      <td>ab_selection</td>
      <td>873108.0</td>
      <td>0.088</td>
      <td>beta_32-NHP_mosaic_AO2-1000-abneg_lib1</td>
      <td>results/counts/beta_32-NHP_mosaic_AO2-1000-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_32-NHP_mosaic_AO2-1000-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_32</td>
      <td>lib2</td>
      <td>NHP_mosaic_AO2</td>
      <td>1000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_32-NHP_mosaic_AO2-1000-abneg</td>
      <td>ab_selection</td>
      <td>721759.0</td>
      <td>0.086</td>
      <td>beta_32-NHP_mosaic_AO2-1000-abneg_lib2</td>
      <td>results/counts/beta_32-NHP_mosaic_AO2-1000-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_32-NHP_mosaic_AO2-1000-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_33</td>
      <td>lib1</td>
      <td>NHP_mosaic_0Z5</td>
      <td>1000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_33-NHP_mosaic_0Z5-1000-abneg</td>
      <td>ab_selection</td>
      <td>809798.0</td>
      <td>0.092</td>
      <td>beta_33-NHP_mosaic_0Z5-1000-abneg_lib1</td>
      <td>results/counts/beta_33-NHP_mosaic_0Z5-1000-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_33-NHP_mosaic_0Z5-1000-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_33</td>
      <td>lib2</td>
      <td>NHP_mosaic_0Z5</td>
      <td>1000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_33-NHP_mosaic_0Z5-1000-abneg</td>
      <td>ab_selection</td>
      <td>803102.0</td>
      <td>0.083</td>
      <td>beta_33-NHP_mosaic_0Z5-1000-abneg_lib2</td>
      <td>results/counts/beta_33-NHP_mosaic_0Z5-1000-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_33-NHP_mosaic_0Z5-1000-abneg_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_34</td>
      <td>lib1</td>
      <td>NHP_mosaic_AT5</td>
      <td>1000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_34-NHP_mosaic_AT5-1000-abneg</td>
      <td>ab_selection</td>
      <td>821781.0</td>
      <td>0.085</td>
      <td>beta_34-NHP_mosaic_AT5-1000-abneg_lib1</td>
      <td>results/counts/beta_34-NHP_mosaic_AT5-1000-abneg_lib1_counts.csv</td>
      <td>results/counts/beta_34-NHP_mosaic_AT5-1000-abneg_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>220210</td>
      <td>beta_34</td>
      <td>lib2</td>
      <td>NHP_mosaic_AT5</td>
      <td>1000</td>
      <td>abneg</td>
      <td>escape</td>
      <td>beta_34-NHP_mosaic_AT5-1000-abneg</td>
      <td>ab_selection</td>
      <td>853365.0</td>
      <td>0.096</td>
      <td>beta_34-NHP_mosaic_AT5-1000-abneg_lib2</td>
      <td>results/counts/beta_34-NHP_mosaic_AT5-1000-abneg_lib2_counts.csv</td>
      <td>results/counts/beta_34-NHP_mosaic_AT5-1000-abneg_lib2_fates.csv</td>
    </tr>
  </tbody>
</table>


Confirm sample / library combinations unique:


```python
assert len(barcode_runs) == len(barcode_runs.groupby(['sample', 'library']))
```

Make sure the the libraries for which we have barcode runs are all in our variant table:


```python
unknown_libs = set(barcode_runs['library']) - set(variants.libraries)
if unknown_libs:
    raise ValueError(f"Libraries with barcode runs not in variant table: {unknown_libs}")
```

Now concatenate the barcode counts and fates for each sample:


```python
counts = pd.concat([pd.read_csv(f) for f in barcode_runs['counts_file']],
                   sort=False,
                   ignore_index=True)

print('First few lines of counts data frame:')
display(HTML(counts.head().to_html(index=False)))

fates = pd.concat([pd.read_csv(f) for f in barcode_runs['fates_file']],
                  sort=False,
                  ignore_index=True)

print('First few lines of fates data frame:')
display(HTML(fates.head().to_html(index=False)))
```

    First few lines of counts data frame:



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>barcode</th>
      <th>count</th>
      <th>library</th>
      <th>sample</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>AAAGATACTACATGGT</td>
      <td>17357</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
    </tr>
    <tr>
      <td>GCATATGCTAGTAATG</td>
      <td>16395</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
    </tr>
    <tr>
      <td>TAGCCAGCTAACCTAA</td>
      <td>14685</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
    </tr>
    <tr>
      <td>GCACATCTAGTAAGAT</td>
      <td>14104</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
    </tr>
    <tr>
      <td>ACAGATGATTACAAAA</td>
      <td>13805</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
    </tr>
  </tbody>
</table>


    First few lines of fates data frame:



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>fate</th>
      <th>count</th>
      <th>library</th>
      <th>sample</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>valid barcode</td>
      <td>50742238</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
    </tr>
    <tr>
      <td>invalid barcode</td>
      <td>9133928</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
    </tr>
    <tr>
      <td>low quality barcode</td>
      <td>3405417</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
    </tr>
    <tr>
      <td>failed chastity filter</td>
      <td>1201254</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
    </tr>
    <tr>
      <td>unparseable barcode</td>
      <td>1121594</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
    </tr>
  </tbody>
</table>


## Examine fates of parsed barcodes
First, we'll analyze the "fates" of the parsed barcodes.
These fates represent what happened to each Illumina read we parsed:
 - Did the barcode read fail the Illumina chastity filter?
 - Was the barcode *unparseable* (i.e., the read didn't appear to be a valid barcode based on flanking regions)?
 - Was the barcode sequence too *low quality* based on the Illumina quality scores?
 - Was the barcode parseable but *invalid* (i.e., not in our list of variant-associated barcodes in the codon variant table)?
 - Was the barcode *valid*, and so will be added to variant counts.
 
First, we just write a CSV file with all the barcode fates:


```python
fatesfile = os.path.join(config['counts_dir'], 'barcode_fates.csv')
print(f"Writing barcode fates to {fatesfile}")
fates.to_csv(fatesfile, index=False)
```

    Writing barcode fates to results/counts/barcode_fates.csv


Next, we tabulate the barcode fates in wide format:


```python
display(HTML(fates
             .pivot_table(columns='fate',
                          values='count',
                          index=['sample', 'library'])
             .applymap('{:.1e}'.format)  # scientific notation
             .to_html()
             ))
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>fate</th>
      <th>failed chastity filter</th>
      <th>invalid barcode</th>
      <th>low quality barcode</th>
      <th>unparseable barcode</th>
      <th>valid barcode</th>
    </tr>
    <tr>
      <th>sample</th>
      <th>library</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2" valign="top">beta_19-26-none-0-ref</th>
      <th>lib1</th>
      <td>1.2e+06</td>
      <td>9.1e+06</td>
      <td>3.4e+06</td>
      <td>1.1e+06</td>
      <td>5.1e+07</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>1.3e+06</td>
      <td>1.4e+07</td>
      <td>3.7e+06</td>
      <td>1.2e+06</td>
      <td>5.1e+07</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_19-mosaic_6848-3125-abneg</th>
      <th>lib1</th>
      <td>1.4e+05</td>
      <td>1.1e+06</td>
      <td>3.9e+05</td>
      <td>1.5e+05</td>
      <td>5.7e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>1.3e+05</td>
      <td>1.5e+06</td>
      <td>3.7e+05</td>
      <td>1.2e+05</td>
      <td>5.1e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_20-mosaic_6849-3125-abneg</th>
      <th>lib1</th>
      <td>1.4e+05</td>
      <td>1.1e+06</td>
      <td>4.2e+05</td>
      <td>1.2e+05</td>
      <td>6.2e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>1.6e+05</td>
      <td>1.8e+06</td>
      <td>4.5e+05</td>
      <td>1.5e+05</td>
      <td>6.0e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_21-mosaic_6850-3125-abneg</th>
      <th>lib1</th>
      <td>1.2e+05</td>
      <td>9.6e+05</td>
      <td>3.6e+05</td>
      <td>1.1e+05</td>
      <td>5.2e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>1.3e+05</td>
      <td>1.5e+06</td>
      <td>3.7e+05</td>
      <td>1.2e+05</td>
      <td>5.0e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_22-mosaic_6851-3125-abneg</th>
      <th>lib1</th>
      <td>1.1e+05</td>
      <td>8.5e+05</td>
      <td>3.1e+05</td>
      <td>1.1e+05</td>
      <td>4.7e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>1.3e+05</td>
      <td>1.4e+06</td>
      <td>3.6e+05</td>
      <td>1.3e+05</td>
      <td>4.8e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_23-mosaic_6852-3125-abneg</th>
      <th>lib1</th>
      <td>1.2e+05</td>
      <td>9.2e+05</td>
      <td>3.5e+05</td>
      <td>1.2e+05</td>
      <td>5.1e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>1.2e+05</td>
      <td>1.4e+06</td>
      <td>3.6e+05</td>
      <td>1.2e+05</td>
      <td>4.8e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_24-mosaic_6853-3125-abneg</th>
      <th>lib1</th>
      <td>1.1e+05</td>
      <td>8.8e+05</td>
      <td>3.3e+05</td>
      <td>1.1e+05</td>
      <td>4.9e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>1.2e+05</td>
      <td>1.4e+06</td>
      <td>3.6e+05</td>
      <td>1.1e+05</td>
      <td>4.8e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_25-homotypic_6881-7812-abneg</th>
      <th>lib1</th>
      <td>1.3e+05</td>
      <td>9.9e+05</td>
      <td>3.7e+05</td>
      <td>1.5e+05</td>
      <td>5.5e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>1.2e+05</td>
      <td>1.4e+06</td>
      <td>3.4e+05</td>
      <td>1.1e+05</td>
      <td>4.6e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_26-homotypic_6883-7812-abneg</th>
      <th>lib1</th>
      <td>1.8e+05</td>
      <td>1.3e+06</td>
      <td>4.8e+05</td>
      <td>1.7e+05</td>
      <td>7.1e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>1.2e+05</td>
      <td>1.4e+06</td>
      <td>3.5e+05</td>
      <td>1.2e+05</td>
      <td>4.7e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_27-34-none-0-ref</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>8.9e+06</td>
      <td>9.3e+06</td>
      <td>2.5e+06</td>
      <td>4.9e+07</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.3e+07</td>
      <td>1.0e+07</td>
      <td>2.7e+06</td>
      <td>5.0e+07</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_27-homotypic_6880-5000-abneg</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>1.0e+06</td>
      <td>1.1e+06</td>
      <td>2.8e+05</td>
      <td>5.7e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.3e+06</td>
      <td>9.1e+05</td>
      <td>2.3e+05</td>
      <td>4.3e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_28-homotypic_6882-5000-abneg</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>8.6e+05</td>
      <td>8.8e+05</td>
      <td>2.3e+05</td>
      <td>4.8e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.2e+06</td>
      <td>8.0e+05</td>
      <td>2.0e+05</td>
      <td>3.8e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_29-homotypic_6884-5000-abneg</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>8.7e+05</td>
      <td>9.1e+05</td>
      <td>2.4e+05</td>
      <td>4.8e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.2e+06</td>
      <td>8.2e+05</td>
      <td>2.0e+05</td>
      <td>3.8e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_30-homotypic_6885-5000-abneg</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>1.1e+06</td>
      <td>1.2e+06</td>
      <td>2.9e+05</td>
      <td>5.9e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.7e+06</td>
      <td>1.2e+06</td>
      <td>2.9e+05</td>
      <td>5.5e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_31-NHP_mosaic_MA292-1000-abneg</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>1.3e+06</td>
      <td>1.3e+06</td>
      <td>3.4e+05</td>
      <td>6.9e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.4e+06</td>
      <td>9.6e+05</td>
      <td>2.4e+05</td>
      <td>4.6e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_32-NHP_mosaic_AO2-1000-abneg</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>9.6e+05</td>
      <td>9.9e+05</td>
      <td>2.6e+05</td>
      <td>5.3e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.1e+06</td>
      <td>7.5e+05</td>
      <td>1.9e+05</td>
      <td>3.5e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_33-NHP_mosaic_0Z5-1000-abneg</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>8.4e+05</td>
      <td>1.0e+06</td>
      <td>2.3e+05</td>
      <td>4.6e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.2e+06</td>
      <td>8.2e+05</td>
      <td>2.1e+05</td>
      <td>4.0e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">beta_34-NHP_mosaic_AT5-1000-abneg</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>9.0e+05</td>
      <td>9.4e+05</td>
      <td>2.4e+05</td>
      <td>4.9e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.7e+06</td>
      <td>1.1e+06</td>
      <td>2.9e+05</td>
      <td>5.4e+06</td>
    </tr>
  </tbody>
</table>


Now we plot the barcode-read fates for each library / sample, showing the bars for valid barcodes in orange and the others in gray.
We see that the largest fraction of barcode reads correspond to valid barcodes, and most of the others are invalid barcodes (probably because the map to variants that aren't present in our variant table since we didn't associate all variants with barcodes). The exception to this is lib2 Titeseq_03_bin3; the PCR for this sample in the original sequencing run failed, so we followed it up with a single MiSeq lane. We did not filter out the PhiX reads from this data before parsing, so these PhiX reads will deflate the fraction of valid barcode reads as expected, but does not indicate any problems.


```python
ncol = 4
nfacets = len(fates.groupby(['sample', 'library']))

barcode_fate_plot = (
    ggplot(
        fates
        .assign(sample=lambda x: pd.Categorical(x['sample'],
                                                x['sample'].unique(),
                                                ordered=True),
                fate=lambda x: pd.Categorical(x['fate'],
                                              x['fate'].unique(),
                                              ordered=True),
                is_valid=lambda x: x['fate'] == 'valid barcode'
                ), 
        aes('fate', 'count', fill='is_valid')) +
    geom_bar(stat='identity') +
    facet_wrap('~ sample + library', ncol=ncol) +
    scale_fill_manual(CBPALETTE, guide=False) +
    theme(figure_size=(3.25 * ncol, 2 * math.ceil(nfacets / ncol)),
          axis_text_x=element_text(angle=90),
          panel_grid_major_x=element_blank()
          ) +
    scale_y_continuous(labels=dms_variants.utils.latex_sci_not,
                       name='number of reads')
    )

_ = barcode_fate_plot.draw()
```


    
![png](aggregate_variant_counts_files/aggregate_variant_counts_28_0.png)
    


## Add barcode counts to variant table
Now we use the [CodonVariantTable.add_sample_counts_df](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html#dms_variants.codonvarianttable.CodonVariantTable.add_sample_counts_df) method to add the barcode counts to the variant table:


```python
variants.add_sample_counts_df(counts)
```

The variant table now has a `variant_count_df` attribute that gives a data frame of all the variant counts.
Here are the first few lines:


```python
display(HTML(variants.variant_count_df.head().to_html(index=False)))
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>target</th>
      <th>library</th>
      <th>sample</th>
      <th>barcode</th>
      <th>count</th>
      <th>variant_call_support</th>
      <th>codon_substitutions</th>
      <th>aa_substitutions</th>
      <th>n_codon_substitutions</th>
      <th>n_aa_substitutions</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>B1351</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
      <td>AAAGATACTACATGGT</td>
      <td>17357</td>
      <td>36</td>
      <td>CCT7AGA GCG190TCT</td>
      <td>P7R A190S</td>
      <td>2</td>
      <td>2</td>
    </tr>
    <tr>
      <td>B1351</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
      <td>GCATATGCTAGTAATG</td>
      <td>16395</td>
      <td>30</td>
      <td>ATA104ATG</td>
      <td>I104M</td>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <td>B1351</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
      <td>TAGCCAGCTAACCTAA</td>
      <td>14685</td>
      <td>33</td>
      <td>CTT5TGT</td>
      <td>L5C</td>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <td>B1351</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
      <td>GCACATCTAGTAAGAT</td>
      <td>14104</td>
      <td>39</td>
      <td>TTT62TCT ACC148ACT</td>
      <td>F62S</td>
      <td>2</td>
      <td>1</td>
    </tr>
    <tr>
      <td>B1351</td>
      <td>lib1</td>
      <td>beta_19-26-none-0-ref</td>
      <td>ACAGATGATTACAAAA</td>
      <td>13805</td>
      <td>28</td>
      <td>TCT53GCT</td>
      <td>S53A</td>
      <td>1</td>
      <td>1</td>
    </tr>
  </tbody>
</table>


Write the variant counts data frame to a CSV file.
It can then be used to re-initialize a [CodonVariantTable](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html#dms_variants.codonvarianttable.CodonVariantTable) via its [from_variant_count_df](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html#dms_variants.codonvarianttable.CodonVariantTable.from_variant_count_df) method:


```python
print(f"Writing variant counts to {config['variant_counts']}")
variants.variant_count_df.to_csv(config['variant_counts'], index=False, compression='gzip')
```

    Writing variant counts to results/counts/variant_counts.csv.gz


The [CodonVariantTable](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html#dms_variants.codonvarianttable.CodonVariantTable) has lots of nice functions that can be used to analyze the counts it contains.
However, we do that in the next notebook so we don't have to re-run this entire (rather computationally intensive) notebook every time we want to analyze a new aspect of the counts.
