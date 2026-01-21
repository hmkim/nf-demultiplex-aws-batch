# nf-core/demultiplex on AWS Batch 실행 가이드

## 개요

이 문서는 nf-core/demultiplex 파이프라인을 AWS Batch에서 실행하기 위한 설정 및 운영 가이드입니다.

### 파이프라인 정보

| 항목 | 값 |
|------|-----|
| 파이프라인 | nf-core/demultiplex |
| 버전 | 1.7.0 |
| Nextflow 버전 | 25.10.2 |
| 실행 환경 | AWS Batch (EC2 SPOT) |
| 리전 | ap-northeast-2 (Seoul) |

---

## 아키텍처

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

## AWS 리소스 구성

### 1. Compute Environment

| 설정 | 값 |
|------|-----|
| 이름 | `demultiplex-ec2-ce` |
| 타입 | MANAGED |
| 인스턴스 타입 | SPOT |
| 인스턴스 유형 | c5.xlarge, c5.2xlarge, m5.xlarge, m5.2xlarge, r5.xlarge, r5.2xlarge |
| 최소 vCPU | 0 |
| 최대 vCPU | 128 |
| Launch Template | demultiplex-nextflow-lt |

### 2. Job Queue

| 설정 | 값 |
|------|-----|
| 이름 | `demultiplex-ec2-queue` |
| 우선순위 | 1 |
| Compute Environment | demultiplex-ec2-ce |

### 3. Launch Template

AWS CLI를 컨테이너에서 사용하기 위한 Launch Template:

```bash
#!/bin/bash
# AWS CLI v2 설치
yum install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip -o awscliv2.zip
./aws/install --update

# 컨테이너 마운트용 디렉토리 생성
mkdir -p /opt/aws-cli
cp -r /usr/local/aws-cli /opt/
```

### 4. IAM Role

| Role | ARN |
|------|-----|
| Job Role | `arn:aws:iam::<AWS_ACCOUNT_ID>:role/ecsTaskExecutionRole` |
| Execution Role | `arn:aws:iam::<AWS_ACCOUNT_ID>:role/ecsTaskExecutionRole` |

---

## 스토리지 구성

### S3 버킷 구조

```
s3://macrogen-demultiplex-test/
├── input_s3.csv                    # 입력 샘플시트
├── samplesheets/
│   └── SampleSheet.csv             # Illumina 샘플시트
├── testdata/
│   └── MiSeq/
│       └── *.tar.gz                # BCL 데이터 (압축)
├── work/                           # Nextflow 작업 디렉토리
│   └── [hash]/                     # 각 태스크별 작업 공간
└── results/                        # 최종 결과물
    ├── [flowcell]/                 # Demultiplex 결과
    ├── multiqc/                    # QC 리포트
    ├── samplesheet/                # 다운스트림 샘플시트
    └── pipeline_info/              # 실행 정보
```

---

## 실행 결과

### 테스트 데이터

| 항목 | 값 |
|------|-----|
| Flowcell ID | 220422_M11111_0222_000000000-K9H97 |
| 플랫폼 | MiSeq |
| 입력 크기 | 40.7 MB (tar.gz) |
| 출력 크기 | 105.8 MB |
| 샘플 수 | 1 |
| 레인 | 1 |
| 타일 | s_1_1101 (테스트용 단일 타일) |

### 단계별 실행 시간

| 단계 | 실행 시간 | 실제 처리 시간 | CPU 사용률 | 메모리 |
|------|-----------|----------------|------------|--------|
| BCL2FASTQ | 3m 39s | 2.4s | 314.5% | 269.6 MB |
| FASTP | 1m 26s | 35s | 110.0% | 1.3 GB |
| MD5SUM | 39.3s | 78ms | 98.5% | 6.8 MB |
| MULTIQC | 5m 4s | 7.5s | 54.1% | 737.8 MB |
| **총 소요 시간** | **5m 25s** | - | - | - |

> **참고**: 실행 시간에는 EC2 인스턴스 프로비저닝 및 S3 데이터 스테이징 시간이 포함됩니다.

### 출력 파일

| 카테고리 | 파일 | 크기 |
|----------|------|------|
| **Raw FASTQ** | Sample1_S1_L001_R1_001.fastq.gz | 37.4 MB |
| **Trimmed FASTQ** | Sample1_S1_L001.fastp.fastq.gz | 34.5 MB |
| **QC Report** | multiqc_report.html | 4.7 MB |
| **Downstream Samplesheets** | rnaseq, atacseq, sarek, methylseq, taxprofiler | 각 ~150 B |

---

## 비용 분석

### EC2 SPOT 인스턴스 비용 (예상)

| 인스턴스 | On-Demand 가격 | SPOT 가격 (예상) | 절감률 |
|----------|----------------|------------------|--------|
| c5.xlarge | $0.17/hr | ~$0.05/hr | ~70% |
| m5.xlarge | $0.192/hr | ~$0.06/hr | ~70% |
| r5.xlarge | $0.252/hr | ~$0.08/hr | ~70% |

### 이번 실행 비용 (예상)

| 항목 | 계산 | 비용 |
|------|------|------|
| EC2 SPOT (c5.xlarge) | ~10분 × $0.05/hr | ~$0.008 |
| S3 스토리지 | 150 MB × $0.025/GB | ~$0.004 |
| S3 요청 | ~1000 requests | ~$0.005 |
| **총 비용** | | **~$0.02** |

> **참고**: 실제 비용은 SPOT 가격 변동, 인스턴스 시작 시간 등에 따라 달라질 수 있습니다.

---

## 설정 파일

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

## 실행 방법

### 기본 실행

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

### Resume 실행 (실패 시 재시작)

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

## 문제 해결

### 1. "aws: command not found" 오류

**원인**: 컨테이너에 AWS CLI가 설치되어 있지 않음

**해결**: Launch Template을 통해 AWS CLI 설치 및 볼륨 마운트
```groovy
aws.batch.cliPath = '/opt/aws-cli/v2/current/bin/aws'
aws.batch.volumes = ['/opt/aws-cli:/opt/aws-cli:ro']
```

### 2. BCL 파일 누락 오류

**원인**: 테스트 데이터에 일부 타일만 포함

**해결**: ext.args로 특정 타일 지정
```groovy
process {
    withName: 'BCL2FASTQ' {
        ext.args = "--tiles s_1_1101"
    }
}
```

### 3. FALCO 라이브러리 오류

**원인**: AWS CLI 공유 라이브러리가 falco 컨테이너에 없음

**해결**: skip_tools에 falco 추가
```bash
--skip_tools "checkqc,samshee,falco"
```

### 4. Fargate 실행 실패

**원인**: Fargate 모드는 Seqera Platform 토큰(Fusion) 필요

**해결**: EC2 기반 Compute Environment 사용

---

## 주의사항

1. **SPOT 인스턴스**: 비용 절감이 크지만 중단될 수 있음. 중요 워크로드는 On-Demand 고려
2. **S3 버킷 리전**: Compute Environment와 동일한 리전에 배치하여 데이터 전송 비용 절감
3. **작업 디렉토리 정리**: S3 work 디렉토리는 자동 삭제되지 않으므로 주기적 정리 필요
4. **IAM 권한**: Job Role에 S3, ECR 접근 권한 필요

---

## 라이선스

이 프로젝트는 MIT 라이선스를 따릅니다.

## 참고 자료

- [nf-core/demultiplex](https://nf-co.re/demultiplex)
- [Nextflow AWS Batch 문서](https://www.nextflow.io/docs/latest/aws.html)
- [AWS Batch 사용자 가이드](https://docs.aws.amazon.com/batch/)
