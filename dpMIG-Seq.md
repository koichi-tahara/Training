# 20261016 dpMIG-Seq データ処理

## 準備
ワークステーションにログイン
```sh
ssh ユーザー名@IPアドレス
```

## 1. クオリティチェック
### 準備
ディレクトリを作り、FASTQデータをコピー
```sh
mkdir -p fastp/raw_fastq fastp/clean_fastq
cd fastp
```
### 1.1. FastQC (処理前)
```sh
mkdir raw_fastq/fastqc_out
fastqc raw_fastq/sample01_R1.fastq.gz -o raw_fastq/fastqc_out/
```
### 1.2. fastp でリードペアを処理
低クオリティリードやアダプタ配列の除去。\
dpMIG-Seq の場合は、これに加えて`--trim_front` でプライマー部分の配列を除去する必要がある。
```sh
fastp -i raw_fastq/sample01_R1.fastq.gz   -o clean_fastq/sample01_R1.clean.fastq.gz \
      -I raw_fastq/sample01_R2.fastq.gz   -O clean_fastq/sample01_R2.clean.fastq.gz \
      -h sample01_fastp.html \
      --trim_front1 20 --trim_front2 20
```

### 1.3. FastQC (処理後)
```sh
mkdir clean_fastq/fastqc_out
fastqc clean_fastq/${ID}_R1.fastq.gz -o clean_fastq/fastqc_out/
```
> [!tip]
> **多数サンプルの処理**
> 
> 1行に1つのサンプル名を書いたテキストファイル `samples.txt` を用意する.
> ```sh
> while read ID; do
>   # FastQC (処理前)
>   fastqc raw_fastq/${ID}_R1.fastq.gz -o raw_fastq/fastqc_out/
>
>   # fastp
>   fastp -i raw_fastq/${ID}_R1.fastq.gz   -o clean_fastq/${ID}_R1.clean.fastq.gz \
>         -I raw_fastq/${ID}_R2.fastq.gz   -O clean_fastq/${ID}_R2.clean.fastq.gz \
>         -h clean_fastq/${ID}_fastp.html \
>         --trim_front1 20 --trim_front2 20
>
>   # FastQC (処理後)
>   fastqc clean_fastq/${ID}_R1.fastq.gz -o clean_fastq/fastqc_out/
> 
> done < samples.txt
> ```

## 2. マッピング

### 準備
作業ディレクトリの下に `bwa_mapping/bam` 等を作っておく
```sh
mkdir -p bwa_mapping/bam
```
マッピング用ディレクトリに移動し, リファレンス配列をコピー
```sh
cd bwa_mapping
cp <オリジナルのリファレンス配列へのパス> .
```

### 2.1. リファレンス配列の確認
リファレンス配列の概要を確認する。\
ここではリファレンス配列の名前を `Reference.fasta` にしているが、必要に応じて適宜変更。
```sh
seqkit stats Reference.fasta
```
リファレンス配列のヘッダー (染色体やcontig) を確認
```sh
grep "^>" Reference.fasta
```
### 2.2. リファレンスのインデックスを作成
この操作により、マッピングの際の相同性検索を高速化できる
```sh
bwa index Reference.fasta
```

### 2.3. ペアエンドリードをマッピング
BWA MEM というアルゴリズムによりペアエンドリードをマッピングする。\
`-R` オプションは後々 GATK で必要になる。
```sh
bwa mem -R "@RG\tID:SP01\tLB:LB\tSM:SP01\tPL:ILLUMINA" \
    Reference.fasta \
    ../fastp/clean_fastq/sample01_R1.clean.fastq.gz \
    ../fastp/clean_fastq/sample01_R2.clean.fastq.gz
```
-> sam フォーマットのアラインメントが標準出力に出ることを確認し, `Ctrl + C` でストップ

PCR duplicates のマーク (`samblaster`) , リファレンス配列に沿った並び替えとファイルへの保存 (`samtools`) を一気に行う
```sh
bwa mem -R "@RG\tID:sample01\tLB:LB\tSM:sample01\tPL:ILLUMINA" \
    Reference.fasta \
    ../fastp/clean_fastq/sample01_R1.clean.fastq.gz \
    ../fastp/clean_fastq/sample01_R2.clean.fastq.gz
| samblaster \
| samtools sort -O BAM -o bam/sample01.bam
```
bam ファイルの確認
```sh
samtools view bam/sample01.bam | less
```
インデックスの作成 (今後の操作で必要)
```sh
samtools index bam/sample01.bam
```
> [!tip]
> **多数サンプルの処理**
> 
> 先ほどと同様に `samples.txt` をつかう
> ```sh
> while read ID; do
>   bwa mem -R "@RG\tID:${ID}\tLB:LB\tSM:${ID}\tPL:ILLUMINA" \
>       Reference.fasta \
>       ../fastp/clean_fastq/${ID}_R1.clean.fastq.gz \
>       ../fastp/clean_fastq/${ID}_R2.clean.fastq.gz
>   | samblaster \
>   | samtools sort -O BAM -o bam/${ID}.bam
> done < samples.txt

## 3. SNPコール
### 3A. stacksの場合
stacks は RAD-seq データを想定して作られたプログラムで, リファレンスの有無によって異なるワークフローになる. \
ここではリファレンスありの場合のフローを紹介する.
### 準備
ディレクトリの作成
```sh
mkdir -p stacks/gstacks_out stacks/populations_out
cd stacks
```
集団情報をかいたテキストファイル `popmap.txt` を準備しておく (populations のフィルタリングで使う場合あり)
### 3A.1. gstacks
SNP カタログの作成とハプロタイプの推定をおこなう
```sh
gstacks -I ../bwa_mapping/bam \
        -M popmap.txt \
        --min-mapq 30 \
        -O gstacks_out
```
### 3A.2. populations
SNPカタログに基づいてVCFを書き出す
```sh
populations -P gstacks_out \
            -O populations_out \
            --vcf
```
### 3B. GATK
GATK はSTACKSよりも細かいオプションの設定などが可能だが、\
全ゲノムシーケンスを前提としたモデルに基づくものも多いので注意が必要。
[https://github.com/Zoshoku-GH/NIG-SuperComputer/tree/main/Variant_Calling/GATK_Calling]\
[https://github.com/mmatsunami/bioarch-2023/blob/main/Osada/tutorial.md]
