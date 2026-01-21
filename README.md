# nf-core/demultiplex on AWS Batch Deployment Guide

## Overview

This document provides a comprehensive guide for running the nf-core/demultiplex pipeline on AWS Batch.

### Pipeline Information

| Item | Value |
|------|-------|
| Pipeline | nf-core/demultiplex |
| Version | 1.7.0 |
| Nextflow Version | 25.10.2 |
| Execution Environment | AWS Batch (EC2 SPOT) |
| Region | ap-northeast-2 (Seoul) |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS Cloud (ap-northeast-2)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐     ┌─────────────────┐     ┌─────────────────────────┐  │
│  │   EC2        │     │   AWS Batch     │     │      Amazon S3          │  │
│  │  (Launcher)  │────▶│                 │────▶│                         │  │
│  │              │     │  Job Queue      │     │  - Input (BCL/tar.gz)   │  │
│  │  Nextflow    │     │  ┌───────────┐  │     │  - Work directory       │  │
│  │  Head Job    │     │  │  EC2 SPOT │  │     │  - Results              │  │
│  └──────────────┘     │  │  Compute  │  │     └─────────────────────────┘  │
│                       │  │  Env      │  │                                   │
│                       │  └───────────┘  │                                   │
│                       └─────────────────┘                                   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        Container Images (Quay.io)                     │  │
│  │  - quay.io/nf-core/bcl2fastq:2.20.0.422                              │  │
│  │  - quay.io/biocontainers/fastp:0.23.4                                │  │
│  │  - quay.io/biocontainers/multiqc:1.21                                │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## AWS Resource Configuration

### 1. Compute Environment

| Setting | Value |
|---------|-------|
| Name | `demultiplex-ec2-ce` |
| Type | MANAGED |
| Instance Type | SPOT |
| Instance Classes | c5.xlarge, c5.2xlarge, m5.xlarge, m5.2xlarge, r5.xlarge, r5.2xlarge |
| Min vCPU | 0 |
| Max vCPU | 128 |
| Launch Template | demultiplex-nextflow-lt |

### 2. Job Queue

| Setting | Value |
|---------|-------|
| Name | `demultiplex-ec2-queue` |
| Priority | 1 |
| Compute Environment | demultiplex-ec2-ce |

### 3. Launch Template

Launch Template for AWS CLI access within containers:

```bash
#!/bin/bash
# Install AWS CLI v2
yum install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip -o awscliv2.zip
./aws/install --update

# Create directory for container mount
mkdir -p /opt/aws-cli
cp -r /usr/local/aws-cli /opt/
```

### 4. IAM Roles

| Role | ARN |
|------|-----|
| Job Role | `arn:aws:iam::<AWS_ACCOUNT_ID>:role/ecsTaskExecutionRole` |
| Execution Role | `arn:aws:iam::<AWS_ACCOUNT_ID>:role/ecsTaskExecutionRole` |

---

## Storage Configuration

### S3 Bucket Structure

```
s3://macrogen-demultiplex-test/
├── input_s3.csv                    # Input samplesheet
├── samplesheets/
│   └── SampleSheet.csv             # Illumina samplesheet
├── testdata/
│   └── MiSeq/
│       └── *.tar.gz                # BCL data (compressed)
├── work/                           # Nextflow work directory
│   └── [hash]/                     # Per-task workspace
└── results/                        # Final outputs
    ├── [flowcell]/                 # Demultiplex results
    ├── multiqc/                    # QC reports
    ├── samplesheet/                # Downstream samplesheets
    └── pipeline_info/              # Execution info
```

---

## Execution Results

### Test Data

| Item | Value |
|------|-------|
| Flowcell ID | 220422_M11111_0222_000000000-K9H97 |
| Platform | MiSeq |
| Input Size | 40.7 MB (tar.gz) |
| Output Size | 105.8 MB |
| Samples | 1 |
| Lane | 1 |
| Tile | s_1_1101 (single tile for testing) |

### Step-by-Step Execution Time

| Step | Duration | Real Time | CPU Usage | Memory |
|------|----------|-----------|-----------|--------|
| BCL2FASTQ | 3m 39s | 2.4s | 314.5% | 269.6 MB |
| FASTP | 1m 26s | 35s | 110.0% | 1.3 GB |
| MD5SUM | 39.3s | 78ms | 98.5% | 6.8 MB |
| MULTIQC | 5m 4s | 7.5s | 54.1% | 737.8 MB |
| **Total Time** | **5m 25s** | - | - | - |

> **Note**: Duration includes EC2 instance provisioning and S3 data staging time.

### Output Files

| Category | File | Size |
|----------|------|------|
| **Raw FASTQ** | Sample1_S1_L001_R1_001.fastq.gz | 37.4 MB |
| **Trimmed FASTQ** | Sample1_S1_L001.fastp.fastq.gz | 34.5 MB |
| **QC Report** | multiqc_report.html | 4.7 MB |
| **Downstream Samplesheets** | rnaseq, atacseq, sarek, methylseq, taxprofiler | ~150 B each |

---

## Cost Analysis

### EC2 SPOT Instance Pricing (Estimated)

| Instance | On-Demand Price | SPOT Price (Est.) | Savings |
|----------|-----------------|-------------------|---------|
| c5.xlarge | $0.17/hr | ~$0.05/hr | ~70% |
| m5.xlarge | $0.192/hr | ~$0.06/hr | ~70% |
| r5.xlarge | $0.252/hr | ~$0.08/hr | ~70% |

### This Execution Cost (Estimated)

| Item | Calculation | Cost |
|------|-------------|------|
| EC2 SPOT (c5.xlarge) | ~10min × $0.05/hr | ~$0.008 |
| S3 Storage | 150 MB × $0.025/GB | ~$0.004 |
| S3 Requests | ~1000 requests | ~$0.005 |
| **Total Cost** | | **~$0.02** |

> **Note**: Actual costs may vary based on SPOT price fluctuations, instance startup time, etc.

---

## Configuration Files

### aws_batch.config

```groovy
process {
    executor = 'awsbatch'
    queue = 'demultiplex-ec2-queue'
}

aws {
    region = 'ap-northeast-2'
    batch {
        maxParallelTransfers = 5
        maxTransferAttempts = 3
        jobRole = 'arn:aws:iam::<AWS_ACCOUNT_ID>:role/ecsTaskExecutionRole'
        cliPath = '/opt/aws-cli/v2/current/bin/aws'
        volumes = ['/opt/aws-cli:/opt/aws-cli:ro']
    }
}

docker {
    enabled = true
}

workDir = 's3://macrogen-demultiplex-test/work'

process {
    withLabel:process_single { cpus = 1; memory = 4.GB }
    withLabel:process_low { cpus = 2; memory = 8.GB }
    withLabel:process_medium { cpus = 4; memory = 16.GB }
    withLabel:process_high { cpus = 8; memory = 30.GB }
    withLabel:process_high_memory { cpus = 8; memory = 60.GB }
}
```

### input_s3.csv

```csv
id,samplesheet,lane,flowcell
220422_M11111_0222_000000000-K9H97,s3://bucket/SampleSheet.csv,1,s3://bucket/flowcell.tar.gz
```

---

## Usage

### Basic Execution

```bash
nextflow run demultiplex/main.nf \
    -profile docker \
    -c demultiplex/conf/aws_batch.config \
    --input s3://your-bucket/input.csv \
    --outdir s3://your-bucket/results \
    --demultiplexer bcl2fastq \
    --skip_tools "checkqc,samshee,falco" \
    -work-dir s3://your-bucket/work
```

### Resume Execution (Restart on Failure)

```bash
nextflow run demultiplex/main.nf \
    -profile docker \
    -c demultiplex/conf/aws_batch.config \
    --input s3://your-bucket/input.csv \
    --outdir s3://your-bucket/results \
    --demultiplexer bcl2fastq \
    --skip_tools "checkqc,samshee,falco" \
    -work-dir s3://your-bucket/work \
    -resume
```

---

## Troubleshooting

### 1. "aws: command not found" Error

**Cause**: AWS CLI not installed in container

**Solution**: Install AWS CLI via Launch Template and mount volume
```groovy
aws.batch.cliPath = '/opt/aws-cli/v2/current/bin/aws'
aws.batch.volumes = ['/opt/aws-cli:/opt/aws-cli:ro']
```

### 2. Missing BCL Files Error

**Cause**: Test data contains only partial tiles

**Solution**: Specify tiles via ext.args
```groovy
process {
    withName: 'BCL2FASTQ' {
        ext.args = "--tiles s_1_1101"
    }
}
```

### 3. FALCO Library Error

**Cause**: AWS CLI shared libraries not present in falco container

**Solution**: Add falco to skip_tools
```bash
--skip_tools "checkqc,samshee,falco"
```

### 4. Fargate Execution Failure

**Cause**: Fargate mode requires Seqera Platform token (Fusion)

**Solution**: Use EC2-based Compute Environment

---

## Important Notes

1. **SPOT Instances**: Significant cost savings but may be interrupted. Consider On-Demand for critical workloads
2. **S3 Bucket Region**: Place in same region as Compute Environment to minimize data transfer costs
3. **Work Directory Cleanup**: S3 work directory is not auto-deleted; periodic cleanup required
4. **IAM Permissions**: Job Role requires S3 and ECR access permissions

---

## License

This project is licensed under the MIT License.

## References

- [nf-core/demultiplex](https://nf-co.re/demultiplex)
- [Nextflow AWS Batch Documentation](https://www.nextflow.io/docs/latest/aws.html)
- [AWS Batch User Guide](https://docs.aws.amazon.com/batch/)
