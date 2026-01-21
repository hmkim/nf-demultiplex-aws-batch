# nf-core/demultiplex AWS Batch Execution Results

## Test Data Summary

| Dataset | Instrument | Run ID | Samples | Tile Parameter |
|---------|------------|--------|---------|----------------|
| MiSeq | MiSeq | 220422_M11111_0222_000000000-K9H97 | 1 | `--tiles s_1_1101` |
| NovaSeq6000 | NovaSeq 6000 | 200624_A00834_0183_BHMTFYDRXX | 4 | `--tiles s_1_2101` |

---

## MiSeq Execution Results

### Configuration
- **Demultiplexer**: bcl2fastq
- **Skip Tools**: checkqc, samshee
- **S3 Bucket**: `s3://macrogen-demultiplex-test/`

### Pipeline Output
```
[nf-core/demultiplex] Pipeline completed successfully
Duration    : ~6m
Succeeded   : 9
```

### Generated Files
| File Type | Path |
|-----------|------|
| FASTQ (trimmed) | `results/220422_M11111_0222_000000000-K9H97/L001/Sample1_S1_L001.fastp.fastq.gz` |
| FASTQ (raw) | `results/220422_M11111_0222_000000000-K9H97/L001/Sample1_S1_L001_R1_001.fastq.gz` |
| MultiQC Report | `results/multiqc/multiqc_report.html` |
| BCL2FASTQ Stats | `results/220422_M11111_0222_000000000-K9H97/L001/Stats/` |

---

## NovaSeq6000 Execution Results

### Configuration
- **Demultiplexer**: bcl2fastq
- **Skip Tools**: checkqc, samshee, falco
- **S3 Bucket**: `s3://macrogen-demultiplex-test/`
- **Config File**: `demultiplex/conf/aws_batch_novaseq6000.config`

### Pipeline Output
```
[nf-core/demultiplex] Pipeline completed successfully
Duration    : 2m 45s
CPU hours   : (a few seconds)
Succeeded   : 9
Cached      : 6
```

### Sample Information
| Sample ID | Index | Index2 |
|-----------|-------|--------|
| Sample1 | GAACTGAGCG | TCGTGGAGCG |
| SampleA | AGGTCAGATA | CTACAAGATA |
| Sample23 | CGTCTCATAT | TATAGTAGCT |
| sampletest | ATTCCATAAG | TGCCTGGTGG |

### Generated FASTQ Files
| Sample | Raw FASTQ | Trimmed FASTQ |
|--------|-----------|---------------|
| Sample1_S1_L001 | 253 KB | 226 KB |
| SampleA_S2_L001 | 210 KB | 182 KB |
| Sample23_S3_L001 | 256 KB | 227 KB |
| sampletest_S4_L001 | 227 KB | 201 KB |

### Output Directory Structure
```
results-novaseq6000/
├── 200624_A00834_0183_BHMTFYDRXX/
│   ├── InterOp/
│   │   ├── BasecallingMetricsOut.bin
│   │   ├── CorrectedIntMetricsOut.bin
│   │   ├── QMetricsOut.bin
│   │   └── ...
│   └── L001/
│       ├── Sample1_S1_L001.fastp.fastq.gz
│       ├── Sample1_S1_L001.fastp.html
│       ├── Sample1_S1_L001.fastp.json
│       ├── Sample1_S1_L001_R1_001.fastq.gz
│       ├── SampleA_S2_L001.fastp.fastq.gz
│       ├── Sample23_S3_L001.fastp.fastq.gz
│       ├── sampletest_S4_L001.fastp.fastq.gz
│       ├── Undetermined_S0_L001_R1_001.fastq.gz
│       ├── Reports/
│       │   └── html/
│       └── Stats/
│           ├── Stats.json
│           ├── ConversionStats.xml
│           └── DemultiplexingStats.xml
├── multiqc/
│   ├── multiqc_report.html
│   ├── multiqc_data/
│   └── multiqc_plots/
├── pipeline_info/
│   ├── execution_report_*.html
│   ├── execution_timeline_*.html
│   └── execution_trace_*.txt
└── samplesheet/
    ├── rnaseq_samplesheet.csv
    ├── atacseq_samplesheet.csv
    ├── sarek_samplesheet.csv
    ├── methylseq_samplesheet.csv
    └── taxprofiler_samplesheet.csv
```

---

## Notes

### Tile Parameters for Test Data
- **MiSeq**: Use `--tiles s_1_1101` (test data trimmed to first tile)
- **NovaSeq6000**: Use `--tiles s_1_2101` (test data trimmed to first tile)

### Skipped Tools
- **checkqc**: Requires additional configuration
- **samshee**: Samplesheet validation not needed for test
- **falco**: AWS CLI volume mount compatibility issue on Batch

### AWS Batch Configuration
- Executor: AWS Batch with EC2
- Job Queue: `demultiplex-ec2-queue`
- Region: `ap-northeast-2`
