# Secondary MS/MS feature identification with GNPS

After processing our metabolomics LCMS data using the MS-DIAL pipeline, we can get secondary annotations for our MS/MS data using the feature-based
molecular networking (FBMN) workflow available from the Global Natural Products Social (GNPS) Molecular Networking team.

To use this tool, you will first have to register for a new account if you do not already have one.
You can do this [here](https://gnps.ucsd.edu/ProteoSAFe/user/register.jsp).

GNPS have a tutorial on the use of MS-DIAL data [here](https://ccms-ucsd.github.io/GNPSDocumentation/featurebasedmolecularnetworking-with-ms-dial/),
but we will cover the most important steps here. 

We will look at how to export your MS-DIAL output for use with GNPS, running your data through the FBMN workflow, and then
incorporating the annotations into the `SummarizedExperiment` objects created during pre-processing steps with the `pmp` package.

### Overview

- Export files for GNPS from MS-DIAL.
- Upload and process you data on the GNPS website.
- Download your processed data.
- Import into R, clean the data, and append the names to your `SummarizedExperiment` objects.

## Method

### Data export for GNPS from MS-DIAL

Similar to exporting raw height matrices from MS-DIAL, you can export data for GNPS by selecting the `GNPS export` checkbox (deselect
the `Raw data matrix (Height)` checkbox).

To get the correct output, you will also need to set the `Target file` to one of the QC or sample files. For simplicity, select the
same QC file as you used during the MS-DIAL data processing parameters.

Then you can click export, and the data will be in the selected directory.

<img src="https://github.com/respiratory-immunology-lab/metabolome-lipidome-MSDIAL/blob/main/gnps_processing/assets/msdial_gnps_export.PNG">

### Running your data on GNPS

Navigate to the [Superquick Start page](http://dorresteinappshub.ucsd.edu:5050/featurebasednetworking), enter an email address to be notified
of completion, and enter your GNPS login credentials. Then set the `Feature Generation Tool` option to "MS-Dial".

You need to provide files to each of the top two boxes:

- `Drop file here to upload feature quantification table`
- `Drop file here to upload feature MS2 MGF/MSP/mzML file(s)`

In the first box, drop in your "GnpsTable_xyz.txt" file (this will typically have the largest file size). In the second box, drop
in your "GnpsMgf_xyz.mgf" file.

Then press the `Analyze Uploaded Files with GNPS Feature Based Molecular Networking` button to run the workflow.

<img src="https://github.com/respiratory-immunology-lab/metabolome-lipidome-MSDIAL/blob/main/gnps_processing/assets/fbmn_submit_page.PNG">

### Retrieving the data output

Once your job has completed, you will see a green window with multiple download/viewing options.

To download your data for integration of names with your existing `SummarizedExperiment` objects, you will need to click the link beneath
`Export/Download Network Files` (it will say `Download Cytoscape Data`.

This will take you to a new screen where you can download your data (leave all parameters as they are).

<img src="https://github.com/respiratory-immunology-lab/metabolome-lipidome-MSDIAL/blob/main/gnps_processing/assets/gnps_job_complete.PNG">
<img src="https://github.com/respiratory-immunology-lab/metabolome-lipidome-MSDIAL/blob/main/gnps_processing/assets/gnps_download_output.PNG">

### Rename results file and convert to .csv

Unzip your downloaded file and navigate to the `DB_result` folder.
This should contain a single .tsv file with a long name which can be renamed to something you will recognise.

Optionally, open this file with excel and save it as a .csv file, then copy the file to somewhere within the scope of your R project,
e.g. `/data/gnps/gnps_stool_positive.csv`.

<img src="https://github.com/respiratory-immunology-lab/metabolome-lipidome-MSDIAL/blob/main/gnps_processing/assets/gnps_output_folders.PNG">

### Import data into R

You should have GNPS output files for both your positive and negative ionisation modes.
What we want to do now, is to produce a new column of values (most will be `NA`) for the `rowData` element of our `SummarizedExperiment` objects.

The naming of the metabolite features from GNPS is very messy however (given that there are many community-provided features), with different
styles. Some of these may come from MoNA, Massbank, ReSpect etc., and there are extra tags on the names that we can remove for clarity.

#### Cleaning up the naming

To clean up the names, and produce a new column that looks nice, we can use the function `gnps_format_names()` provided above.

It takes two input files (`gnps_pos_df` and `gnps_neg_df`), and returns a single combined data.frame with a new column named
`compound_name_gnps` with our cleaned feature names.

Also of note, it generates another new column called `alignment_ionisation` which contains the alignment IDs and ionisation modes that match
with our outputs from `pmp_preprocess()`; this will be necessary for the next step.

Below is an example with stool metabolomics data output from GNPS:

```r
# Import GNPS data
gnps_stool_pos <- read_csv(here::here('data', 'gnps', 'gnps_data', 'gnps_20210727_stool_positive.csv'))
gnps_stool_neg <- read_csv(here::here('data', 'gnps', 'gnps_data', 'gnps_20210727_stool_negative.csv'))

# Format GNPS compound names and return a joined pos and neg data.frame
gnps_stool <- gnps_format_names(gnps_pos_df = gnps_stool_pos, gnps_neg_df = gnps_stool_neg)
```

The `gnps_stool` data.frame now contains our cleaned names and alignment IDs/ionisation modes.

### Appending GNPS names to SummarizedExperiment objects

The next step is to incorporate our new GNPS names with our existing data. 
For this, we will need a vector the same length as the number of features remaining after pre-processing with `pmp`,
with GNPS names matched by their alignment IDs and ionisation mode.

We can achieve this using the `gnps_SE_names()` feature provided above. 
It takes two inputs: 
- `gnps_df`: i.e. the output from `gnps_format_names()` (must contain columns named `'alignment_ionisation'` and `'compound_name_gnps'`).
- `metab_SE`: one of the two `SummarizedExperiment` objects within the `pmp_preprocess()` output list; these can be accessed using standard list notation, e.g. `pmp_output$glog_results`.

```r
# Add the GNPS compound names to both the glog-transformed and imputed SE objects
rowData(metab_stool_pmp$glog_results)$compound_name_gnps <- gnps_SE_names(gnps_df = gnps_stool, 
                                                                          metab_SE = metab_stool_pmp$glog_results)
rowData(metab_stool_pmp$imputed_results)$compound_name_gnps <- gnps_SE_names(gnps_df = gnps_stool,
                                                                             metab_SE = metab_stool_pmp$imputed_results)
```

Now these new names will be available within the `rowData` element of each of our `SummarizedExperiment` objects, and can be
accessed using `rowData(SE_object)$compound_name_gnps`.
