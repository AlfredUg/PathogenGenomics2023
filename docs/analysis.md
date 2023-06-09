## Basic analysis commands

### Analysis tools

+ Quality control: `fastqc` and `multiqc`
+ Read-based taxonomic identification: `kraken2` 
+ Processing kraken results: `KrakenTools`
+ Visualising Kraken results: `krona`
+ Mapping reads onto reference genomes: `tanoti`
+ Visualising alignments: `tablet`
+ De novo assembly: `spades`

### Creating the analysis directory

We create a directory `analysis`, which will be our working space for this session. We use the famous `mkdir` command to achive this. Later, we shall be creating subdirectories inside the `analysis` directory for the different steps, including; qc, mapping, e.t.c.

```
mkdir analysis
cd analysis
```

### The practice dataset

In interest of time, the data was downloaded from ENA ahead of time and stored on the server. Use the command below to create a copy of the data in your working directory.

```
cp -r /opt/metagenome/data .
```

### Quality control

First, let us look at the quality of the sequence data that we have using `fastqc` and `multiqc` tools.

```
mkdir qc
fastqc data/SRR19400451_*.fastq -o qc
multiqc qc/* -o qc 
```

At this point, we use `scp` to download the MultiQC report and have a look at it, according to our assessement of the report, we can choose whether or not to do some adaptor/quality trimming. To do this, open a new tab on your terminal/mobaxterm shell, navigate to where you would want to keep the MultQC report and run the command below.

```
scp acountname@xxx.xx.xxx.xx:/home/accountname/analysis/qc/multiqc_report.html .
```

For participants using Mobaxterm, we could simply download, by navigating to the `qc` directory in the left panel of mobaxterm window and downloading the MultiQC report to a desired folder on own computer. 

We use `trim_galore` for adaptor and quality trimming. Below is an example, the cut-offs on quality scores, length, e.t.c indicated below are arbitrary. In practice, the choice of these parameters is guided by the assessment made on the QC plots generated above.

```
mkdir trimmed
trim_galore -q 30 --paired data/sample1_R1.fq data/sample1_R2.fq -o trimmed
```

In this case, we may not have to quality trim our data, but just in case we did, we would used the trimmed data from this point onwards.

### Read-based taxonomic identification using Kraken2

To get an idea of the viral pathogens that could be in this sample, we do screen the read data using `kraken2`. Basically, the taxonomic identification is done in comparison/reference to a kraken-compatible pre-built database. Since we are focusing on viral pathogens for this session, we use the kraken2 viral database. We have a copy of this databse on the server but you can download one [here](https://genome-idx.s3.amazonaws.com/kraken/k2_viral_20230605.tar.gz).

First, get a copy of the database and keep it in the `analysis` directory. 

```
cp -r /opt/metagenome/kraken krakenDB
```

Make a directory where we shall keep the kraken-results, call it `kraken-output` and run `kraken2` keeping the outputs in the directory created.

```
mkdir kraken-output
kraken2 --db krakenDB --paired data/SRR19400485_1.fastq data/SRR19400485_2.fastq --report kraken-output/SRR19400485.report > kraken-output/SRR19400485.txt
```

To visualise `kraken2` results, we use `KronaTools`. But before that happens, we need to convert the `kraken2` report into a format compartible to `Kronatools`. We do that using `KrakenTools`. So, we start by getting a copy of the KrakenTools into the analysis directory and then convert the `kraken2` report into an equivalent report that is compartible to `KronaTools`.

```
cp -r /opt/metagenome/KrakenTools .
python KrakenTools/kreport2krona.py -r kraken-output/SRR19400485.report -o kraken-output/SRR19400485.krona 
ktImportText kraken-output/SRR19400485.krona  -o kraken-output/SRR19400485_krona.html
```

As we did for the `multiqc` report earlier on, let us use the `scp` command to download the `KronaTools` report and inspect it. At this point, we should be able to answer the question, **which known  viral pathogens could be in this sample?** 

```
scp acountname@xxx.xx.xxx.xx:/home/accountname/analysis/kraken-output/SRR19400485_krona.html .
```

### Mapping mNGS onto reference genomes

Map the short reads onto the reference genome, and obtain mapping statistics

```
tanoti -r data/sars-cov-2.fasta -i data/SRR19400485_1.fastq data/SRR19400485_2.fastq -p 1
SAM_STATS FinalAssembly.sam
```

To keep order in our working directory, let us create a mapping directory add the alignment map to it and change it name to reflect the sample ID. 

```
mkdir mapping
mv FinalAssembly.sam mapping/SRR19400485.sam
```

Generate a consensus sequence from the alignment map

```
SAM2CONSENSUS -i mapping/SRR19400485.sam -o mapping/SRR19400485-SARS-CoV-2.fasta
```

### De novo assembly

This section comprises of three main stages; removing host reads from the read data, assembling short reads into longer contiguous sequences using a metagenome assembler, spades in this case and blasting the contigs against a protein database using `diamond`.

#### Removal of host reads 

Prior to performing de novo assembly, we remove the host reads from the data. Here we use `bowtie2` to map reads to the host genome and retain the reads that do not map to the host genome for de novo assembly. The Human reference genome can be downloaded from Ensembl, we have added a bowtie2-index of the Human genome to the server, on the path `/opt/metagenome/GRCh38_noalt_as/GRCh38_noalt_as`.

```
mkdir hosttfree
bowtie2 -x /opt/metagenome/GRCh38_noalt_as/GRCh38_noalt_as -1 data/SRR19400485_1.fastq -2 data/SRR19400485_2.fastq --un-conc-gz hosttfree/SRR19400485_clean > hosttfree/SRR19400485_host.sam &

mv hosttfree/SRR19400485_clean.1 hosttfree/SRR19400485_clean_1.gz
mv hosttfree/SRR19400485_clean.2 hosttfree/SRR19400485_clean_2.gz
gunzip hosttfree/*.gz
```

Assemble the short reads into longer contigs using `spades`.

```
mkdir spades_output
metaspades.py -1 hosttfree/SRR19400485_clean_2.fastq  -2 hosttfree/SRR19400485_clean_2.fastq  -o spades_output
```


