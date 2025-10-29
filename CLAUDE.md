# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Nextflow workflow (`wf-basecalling`) for basecalling Oxford Nanopore Technologies (ONT) sequencing data. It uses Dorado for GPU-accelerated basecalling and supports various modes including simplex, duplex, modified base calling, and real-time analysis.

## Key Technologies

- **Nextflow DSL2**: Workflow orchestration (requires Nextflow >=23.04.2)
- **Dorado**: GPU-based basecaller from ONT
- **Docker/Singularity**: Containerized execution
- **Python**: Helper scripts in [bin/](bin/)
- **Groovy**: Nextflow processes and utility classes in [lib/](lib/)
- **Samtools/Minimap2**: Alignment and BAM/CRAM processing

## Project Structure

- [main.nf](main.nf): Main workflow entry point - defines high-level workflow logic, processes, and orchestration
- [nextflow.config](nextflow.config): Main configuration file - defines workflow parameters and manifest
- [base.config](base.config): Container and execution profile configuration (Docker, Singularity, AWS Batch, local, discrete_gpus)
- [lib/signal/ingress.nf](lib/signal/ingress.nf): Core basecalling workflow (`wf_dorado`) - handles signal processing, chunking, and basecalling
- [lib/signal/merge.nf](lib/signal/merge.nf): Merging and alignment logic for basecalled data
- [lib/common.nf](lib/common.nf): Common utility functions and processes
- [lib/reference.nf](lib/reference.nf): Reference genome preparation (indexing, caching for CRAM)
- [bin/](bin/): Python helper scripts for statistics, reporting, and data processing
  - [report.py](bin/report.py): Generate HTML reports
  - [fastcat_histogram.py](bin/fastcat_histogram.py): Per-read statistics
  - [duplex_stats.py](bin/duplex_stats.py): Duplex pairing statistics
  - [add_jsons.py](bin/add_jsons.py): Accumulate JSON stats in streaming mode

## Common Development Commands

### Run the workflow locally

Download demo data first:
```bash
wget https://ont-exd-int-s3-euwst1-epi2me-labs.s3.amazonaws.com/wf-basecalling/wf-basecalling-demo.tar.gz
tar -xzvf wf-basecalling-demo.tar.gz
```

Basic simplex basecalling with alignment:
```bash
nextflow run main.nf \
  --basecaller_cfg 'dna_r10.4.1_e8.2_400bps_hac@v5.0.0' \
  --dorado_ext 'pod5' \
  --input 'wf-basecalling-demo/input' \
  --ref 'wf-basecalling-demo/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta' \
  -profile standard
```

### Testing

The workflow is tested via GitLab CI ([.gitlab-ci.yml](.gitlab-ci.yml)) with a matrix of test scenarios including:
- Simplex and duplex basecalling
- Modified base calling
- Different output formats (FASTQ, BAM, CRAM)
- Real-time watch path mode
- Barcode demultiplexing

To run a specific test scenario locally, replicate the `NF_WORKFLOW_OPTS` from the CI configuration.

### Update model schema

When basecaller models change:
```bash
bash util/update_models_schema.sh . docker
```

This updates [nextflow_schema.json](nextflow_schema.json) with available Dorado models.

## Architecture Overview

### Workflow Execution Flow

1. **Input validation and setup** ([main.nf](main.nf):258-305): Validate parameters, check for conflicting options
2. **Reference preparation** ([lib/reference.nf](lib/reference.nf)): If `--ref` provided, create `.fai`, `.mmi` (minimap2 index), and ref_cache for CRAM
3. **Signal ingress** ([lib/signal/ingress.nf](lib/signal/ingress.nf)):
   - Chunk input POD5/FAST5 files
   - Run `dorado` basecaller on each chunk (GPU process)
   - Handle FAST5→POD5 conversion for duplex mode
   - Optionally demultiplex barcodes
4. **Alignment** ([lib/signal/merge.nf](lib/signal/merge.nf)): If reference provided, align with minimap2
5. **Quality filtering**: Separate pass/fail reads based on `--qscore_filter` (default: Q10)
6. **Statistics and reporting**: Progressive stats accumulation ([main.nf](main.nf):354-355, 378-379)
   - `progressive_stats`: Accumulate per-read stats using scan pattern
   - `progressive_pairings`: Accumulate duplex pairing stats (if duplex mode)
7. **Output**: Publish BAM/CRAM/FASTQ files and HTML report

### Key Nextflow Patterns

- **Scan operator** ([main.nf](main.nf):355, 379): Used for streaming/progressive statistics in watch_path mode
- **Recursion** ([main.nf](main.nf):12): `nextflow.preview.recursion=true` enables workflow recursion
- **Module aliasing** ([lib/signal/ingress.nf](lib/signal/ingress.nf):6-11): Same process used for pass/fail data
- **Publishing pattern** ([main.nf](main.nf):214-253): Separate `output_stream` and `output_last` processes decouple publishing from data processing

### GPU Handling

- GPU processes labeled with `label:gpu` and `accelerator 1`
- Default: Serial execution (`maxForks = 1`) to prevent GPU OOM on local execution
- Use `-profile discrete_gpus` on cluster/cloud to enable parallel GPU task execution
- CUDA device selection via `--cuda_device` parameter (default: `cuda:all`)

### Watch Path (Real-time) Mode

When `--watch_path` is enabled:
- Nextflow monitors input directory for new POD5/FAST5 files
- Progressive stats accumulation via scan operator
- Optional stop condition via `--read_limit` creates STOP file in input directory
- Note: Duplex + watch_path may yield lower duplex rates than batch mode

## Important Constraints and Considerations

- **GPU requirement**: Dorado requires NVIDIA GPU with Pascal architecture or newer, minimum 8GB vRAM
- **Duplex mode**: Only supports POD5 input (workflow auto-converts FAST5 if needed)
- **Duplex + demux**: Not supported (will throw error)
- **Duplex + watch_path**: Supported but may reduce duplex rates vs batch processing
- **Output format**: When `--ref` is provided with `--output_fmt fastq`, workflow will warn and output BAM instead
- **Container toolkit**: Linux users need `nvidia-container-toolkit` for GPU support in Docker

## Container and Profile System

Containers defined in [base.config](base.config):
- `wf_basecalling`: Dorado basecaller container
- `wf_common`: Common utilities (samtools, minimap2, bamstats, etc.)
- `wf_bonito`: Experimental bonito basecaller (requires `--experimental` flag)

Profiles:
- `standard`: Docker (default)
- `singularity`: Singularity/Apptainer
- `awsbatch`: AWS Batch execution
- `local`: Local executor without containers
- `discrete_gpus`: Allow parallel GPU task execution

## Parameters and Options

Major parameter categories:
- **Input**: `--input`, `--ref`, `--sample_name`
- **Basecalling**: `--basecaller_cfg`, `--remora_cfg` (modified bases), `--duplex`, `--dorado_ext` (pod5/fast5)
- **Output**: `--out_dir`, `--output_fmt` (cram/bam/fastq)
- **Advanced**: `--basecaller_args`, `--cuda_device`, `--qscore_filter`, `--basecaller_basemod_threads`
- **Demux**: `--barcode_kit`, `--demux_args`
- **Real-time**: `--watch_path`, `--read_limit`

See [nextflow_schema.json](nextflow_schema.json) for full parameter documentation.

## Modifying the Workflow

When adding new processes:
- Label appropriately: `wf_basecalling`, `wf_common`, or `gpu`
- Set CPU/memory requirements based on target instance types (e.g., g5.2xlarge, g5.12xlarge for GPU tasks)
- Use `--no-PG` flag with samtools to avoid polluting headers
- Follow the progressive stats pattern for streaming compatibility

When modifying reports:
- Update [bin/report.py](bin/report.py) for HTML report changes
- Stats are accumulated as JSON files - see [bin/fastcat_histogram.py](bin/fastcat_histogram.py)

When changing parameters:
- Update [nextflow_schema.json](nextflow_schema.json)
- Add validation in workflow block ([main.nf](main.nf):258-305)
- Update [README.md](README.md) parameter tables
