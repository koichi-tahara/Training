# dpMIG-Seq データ処理

## 0. 準備
スパコンにログイン
```sh
ssh a001
```
```sh
srun --mpi=none --pty --mem=64G bash
```
### 1.

```sh
fastqc input_R1.fastq.gz -o output_dir
```
```sh
fastp -i sampleX_R1.fastq.gz   -o sampleX_R1.clean.fastq.gz \
      -I sampleX_R2.fastq.gz   -O sampleX_R2.clean.fastq.gz \
      -h sampleX_fastp.html \
      --length_required 50 \
      --trim_front1 20 --trim_front2 20 \
```

