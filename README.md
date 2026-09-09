# CLIPseq pipeline
*Instructions on how to run the CLIPseq pipeline on UBELIX*

## Background
Popular CLIPseq pipelines options:
#### ENCODE [eCLIP pipeline](https://www.encodeproject.org/eclip/)
* Trimming: `cutadapt`
* Mapping: `STAR`
* Deduplication: `umi_tools`
* Peak calling: `CLIPper`
> [!NOTE]
> Sequencing must be paired-ended.

#### nf-core [clipseq](https://nf-co.re/clipseq/1.0.0/)
* Trimming: `cutadapt`
* Pre-mapping to contaminant sequences (e.g. rRNA and tRNA): `bowtie2`
* Mapping: `STAR`
* Deduplication: `umi_tools`
* Crosslink identification: `BEDTools`
* Bedgraph generation: `BEDTools`
* Peak calling:
    - `iCount`
    - `Paraclu`
    - `PureCLIP`
    - `Piranha`
* Motif detection: `DREME`
* Quality control:
    - Sequence: `FastQC`
    - Library complexity: `Preseq`
    - Regional distribution: `RSeQC`
    - QC summaries: `MultiQC`
> [!NOTE]
> * The outdated version 1 is the only official release. Version 2 not documented and buggy.
> * Sequencing must be single-ended.

#### Flow [CLIP-Seq](https://app.flow.bio/pipelines/960154035051242353/)
This is essentially the same as the nf-core clipseq v2 above, developed by the same contributors.
It is more stable, and it is deployed on https://app.flow.bio/.
> [!NOTE]
> Sequencing must be single-ended.

**We implemented the Flow version on UBELIX**

## Getting started
* The pipeline was fetched from the GitHub [repo](https://github.com/goodwright/clipseq/tree/1.7), and checked to branch 1.7 (this is the version currently used by Flow). It is located at `/storage/research/dbmr_luisierlab/resources/pipelines/clipseq/flow/clipseq`
* Nextflow needs Java 17 or later. Load it:
    - `module load Java` (defaults to `Java/17.0.6` on UBELIX, you can also load `Java/21.0.8` if you want)
* You can [install nextflow](https://docs.seqera.io/nextflow/install#install-nextflow) to your HOME if you want, or you can use `/storage/research/dbmr_luisierlab/resources/local/nextflow/nextflow`.

#### Workdir and inputs
* Workdir location is your choice. By default, the pipeline writes its results into `your_current_location/results`.
* The only input you need is a `samplesheet`; a CSV file containing 4 columns: `group`,`replicate`,`fastq_1`,`fastq_2.`
    - `group` is the sample name
    - `replicate` is currently unused by the pipeline, so filling with '1' is acceptable
    - `fastq_1` is your demultiplexed sample fastq
    - paired end is currently not supported, so do not add a `fastq_2`

Example `samplesheet` (remember, it should be CSV):
```
| group   | replicate | fastq_1                            | fastq_2 |
|---------|-----------|------------------------------------|---------|
| TDP43_1 | 1         | /path/to/fastq/ERR1530360.fastq.gz |         |
| TDP43_2 | 1         | /path/to/fastq/ERR1530361.fastq.gz |         |
| MATR3_1 | 1         | /path/to/fastq/ERR2210802.fastq.gz |         |
```
#### References
We prefetched all the hg38 references needed for this pipeline (`/storage/research/dbmr_luisierlab/resources/pipelines/clipseq/genome_cacheDir`). You only need to pass a custom config file when running the pipeline (see below) to avoid automatic download and generation of indexes and other reference files.

#### Singularity images
We configured the pipeline to use singularity, and we prefetched all the containers (`/storage/research/dbmr_luisierlab/resources/pipelines/singularity_cacheDir`).

#### Custom configs
These configs need to be passed when running the pipeline

* General nextflow config for UBELIX
    - `storage/research/dbmr_luisierlab/resources/pipelines/ubelix_nextflow.config`
    - This config instructs nextflow to use singularity, sets a fixed `cacheDir` location (`/storage/research/dbmr_luisierlab/resources/pipelines/singularity_cacheDir`), instructs singularity to bind `/scratch/local`, and sets some SLURM options (QOS, wckey, etc.)

* Genome config
    - `/storage/research/dbmr_luisierlab/resources/pipelines/clipseq/genome.config`
    - Defines paths to reference files needed for the pipeline

## Running the pipeline
Once you can run `nextflow` and have your `samplesheet`, you can execute the pipeline directly from the login node in a `screen` session, or as an SBATCH script.
```
NXF_VER=24.10.8 NXF_SINGULARITY_HOME_MOUNT=true NXF_SYNTAX_PARSER=v1 /storage/research/dbmr_luisierlab/resources/local/nextflow/nextflow run \
 /storage/research/dbmr_luisierlab/resources/pipelines/clipseq/flow/clipseq \
 -c /storage/research/dbmr_luisierlab/resources/pipelines/ubelix_nextflow.config \
 -c /storage/research/dbmr_luisierlab/resources/pipelines/clipseq/genome.config \
 -profile singularity,ubelix \
 --samplesheet samplesheet.csv \
 --run_move_umi_to_header true \
 --umi_header_format 'NNNNNNNNNN' \
 --umi_separator '_'
 ```

> [!NOTE]
> * Single-dash arguments are for `nextflow`. Double-dash arguments are passed to the pipeline.
> * `NXF_VER=24.10.8` is to use the same version as by flow.bio. It's not critical, just a precaution to avoid unexpected errors.
> * `NXF_SINGULARITY_HOME_MOUNT=true` is to deal with some tools like Matplotlib or Numba that write cache to HOME. Since Nextflow 23.10, the user's HOME is no longer mounted automatically.
> * `NXF_SYNTAX_PARSER=v1`: Nextflow's newest syntax is more rigid, and some older code does not respect it yet.
> * `--run_move_umi_to_header` instructs `umitools`to move UMI from fastq reads to the read header. Use it if the UMI is in the 5' end of the fastq reads. In this case, ensure you provide the UMI format to `umi_header_format`.
> * `umi_separator` is used to separate the UMI sequence in the read header. If you have processed FASTQ files, check the separator character.
> * If something goes wrong, you can resume the pipeline with the option `-resume`: completed jobs won't be repeated.
> * A failed job will be automatically resubmitted by the pipeline with double the resources. If it fails again, the pipeline will stop with an error. This usually indicates a bug. Once it's fixed, re-run the pipeline with `-resume`.
> * If you are executing the pipeline as a SLURM job, the sbatch resource requests (eg. `--cpus-per-task=2`, `--mem=8G`) only apply to the Nextflow master controller process. Nextflow will automatically use SLURM commands (`srun`/`sbatch`) behind the scenes to launch individual pipeline tasks as separate, independent cluster jobs.


## Results
Nextflow pipelines write to two folders by default: `./work` and `./results`.

`work` contains all the staged files and the outputs from every job. It also enables automatic resumption of the pipeline when something goes wrong.

`results` is the main output location.
> [!WARNING]
> * The pipeline will populate `results` with **symlinks** to `work`. Do not delete `work`.

Results will be written to `./results/` folder, containing:
* `00_genome` contains all reference files produced when the prepare_clipseq subworkflow is run.
* `01_prealign` contains pre-trimmed FastQC reports, trimmed read files and post-trimming FastQC reports.
* `02_alignment` contains two folders, one for the pre-mapping "smrna" and one for the genomic mapping "target", each contain alignment files and the "target" folder also contains useful samtools assessment of the bam file.
* `03_filt_dedup` contains de-duplicated genome mapped bams along with statistics and also two transcriptome mapped de-duplicated bams. Here those marked "filt" have been filtered to only contain alignments to the longest transcript for each gene.
* `04_crosslinks` contains genomic crosslink bed, bedgraph and normalised bedgraph (crosslinks divided by total number of crosslinks in the sample and multiplied by a million resulting in a crosslinks per million (CPM) value); also the same files for transcriptome mapping filtered by longest transcript.
* `05_peak_calling` contains Clippy, iCount and Paraclu peaks. The iCount folder also contains gene, subtype and type level summaries of crosslink information and metagene plots around transcript landmarks of interest in the rnamaps folder. Also included is PEKA output.
* `06_reports` contains various CLIP-specific QC metrics in tabular format in the clipqc folder. These are plotted, alongside other QC metrics in the html provided in the multiqc folder.
* `pipeline_info` contains run summary of the jobs run by the pipeline. The most useful is `execution_report_*.html`

## Logs
The main log is `./.nextflow.log`

`./.nextflow/history` logs the main execution commands. 
