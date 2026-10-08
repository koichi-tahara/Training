# dpMIG-Seq データ処理

## 0. 準備
スパコンにログイン
```sh
ssh a001
```
```sh
srun --mpi=none --pty --mem=64G bash
```
## 1. トリミング

```sh
mkdir fastqc_out
fastqc sample01_R1.fastq.gz -o output_dir
```
```sh
fastp -i sample01_R1.fastq.gz   -o sample01_R1.clean.fastq.gz \
      -I sample01_R2.fastq.gz   -O sampl01_R2.clean.fastq.gz \
      -h sampleX_fastp.html \
      --trim_front1 20 --trim_front2 20 \
```
> [!NOTE]
> `--trim_front` でdpMIG-Seqプライマー部分の配列を除去

## 2. マッピング
[https://github.com/Zoshoku-GH/NIG-SuperComputer/tree/main/Variant_Calling/BWA_Mappin]

## 3. GATKによるSNPコール
[https://github.com/Zoshoku-GH/NIG-SuperComputer/tree/main/Variant_Calling/GATK_Calling]\
[https://github.com/mmatsunami/bioarch-2023/blob/main/Osada/tutorial.md]
