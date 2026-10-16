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
fastqc raw_fastq/sample01_R1.fastq.gz
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
fastqc clean_fastq/${ID}_R1.fastq.gz
```
> [!tip]
> **多数サンプルの処理**
> 
> 1行に1つのサンプル名を書いたテキストファイル `samples.txt` を用意する.
> ```sh
> while read ID; do
>   # FastQC (処理前)
>   fastqc raw_fastq/${ID}_R1.fastq.gz
>
>   # fastp
>   fastp -i raw_fastq/${ID}_R1.fastq.gz   -o clean_fastq/${ID}_R1.clean.fastq.gz \
>         -I raw_fastq/${ID}_R2.fastq.gz   -O clean_fastq/${ID}_R2.clean.fastq.gz \
>         -h clean_fastq/${ID}_fastp.html \
>         --trim_front1 20 --trim_front2 20
>
>   # FastQC (処理後)
>   fastqc clean_fastq/${ID}_R1.fastq.gz
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
    ../fastp/clean_fastq/sample01_R2.clean.fastq.gz \
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

[https://github.com/mmatsunami/bioarch-2023/blob/main/Osada/tutorial.md]

### 4. フィルタリング
#### 全体の流れ

[**1. 解析範囲を常染色体に限定**](#my-custom-anchor-point1)

$`\downarrow`$

[**2. GQ, DPに対するリード単位でのフィルタリング**](#my-custom-anchor-point2)

$`\downarrow`$

[**3. 欠損率/欠損数でのフィルタリング**](#my-custom-anchor-point3)

$`\downarrow`$

[**4. MAF/MAC (マイナーアリル頻度/マイナーアリルカウント) でのフィルタリング**](#my-custom-anchor-point4)

$`\downarrow`$

[**5. HWE逸脱座位のフィルタリング**](#my-custom-anchor-point5)

$`\downarrow`$

[**6. LD-Pruning**](#my-custom-anchor-point6)



### 準備
ディレクトリの作成
```bash
DIR=your_directory
mkdir $DIR
```
フィルタリング前の生VCFファイルを指定
```bash
RAW_VCF=
```
高速化のため block gzip し, index をつけておく
```bash
# 保存したい prefix を指定(適宜変更)
OUT=$DIR/raw

# CHROM, POSでソートし, bgzip
bcftools sort $RAW_VCF -Oz -o $OUT.vcf.gz

# index作成
bcftools index $OUT.vcf.gz
```


<a name="my-custom-anchor-point1"></a>
### 4.1. 解析範囲を常染色体に限定
>[!caution]
> 下記スクリプトでは, 入力VCFの常染色体の`CROM`フィールドが `chr1`, `chr2`, ... のようになっていることを想定している.
>
> VCFの CROM フィールドの変更には `bcftools annotate --rename-chrs` などが有効.

常染色体数を指定
```bash
chr=20
```

```bash
# インプット・アウトプットファイル名 (Prefix) の指定. 適宜変更.
IN=$OUT
OUT=$DIR/chr1-$chr

# 目的の染色体だけ抜き出し
bcftools view $IN.vcf.gz -Oz -o $OUT.vcf.gz \
              -r $(seq $chr | xargs -I{} echo "chr{}" | xargs echo | tr " " ",")
bcftools index $OUT.vcf.gz
```

<a name="my-custom-anchor-point2"></a>
### 4.2. GQ (ジェノタイプクオリティ) と DP (デプス) によるフィルタリング
フィルターに引っかかった場合は`GT`を`./.` (欠損) に置換.
```bash
# ジェノタイプクオリティ (ジェノタイプの信頼度) の下限値
GQ=15

# サイトあたりのデプスの下限値と上限値
minDP=5
maxDP=500
```
```bash
# ファイル名にフィルタリング条件を追記
IN=$OUT; OUT=${IN}_GQ${GQ}_DP${minDP}-${maxDP}

# 実行
bcftools +setGT $IN.vcf.gz \
         -- -t q -n . \
            -i "FMT/GQ<$GQ | FMT/DP<$minDP | FMT/DP>$maxDP" \
| bcftools annotate --remove "INFO" \
| bcftools +fill-tags -- --tags NS \
| bcftools view -Oz -o $OUT.vcf.gz \
                -e "NS=0"
bcftools index $OUT.vcf.gz
```
>[!note]
> 数分かかる.
>
> やっていることは, 低クオリティGTを `./.` に置換 → INFO フィールドをリセット → 非欠損サンプル数を計算 → 全欠損座位の除去.


<a name="my-custom-anchor-point3"></a>
### 4.3. サンプル欠損率/欠損数でのフィルタリング
欠損率が閾値を超えるSNPを除去する.
```bash
# 許容する欠損率の上限値
miss=0.05
```
```bash
# ファイル名にフィルタリング条件を追記
IN=$OUT; OUT=${IN}_miss${miss}

# INFOフィールドに欠損率/数を書き込み, その値に基づいてフィルタリング
bcftools +fill-tags $IN.vcf.gz \
         -- --tags NS,F_MISSING \
| bcftools view -e "F_MISSING > $miss" -Oz -o $OUT.vcf.gz
bcftools index $OUT.vcf.gz
```
>[!note]
> 非欠損数でフィルタリングする場合は `-e "NS < 5"` のようにする

>[!note]
> ### 集団内ごとの非欠損個体数でフィルタリングする場合
> ```bash
> # 集団分けを書いた2列のタブ区切りテキストファイルを指定 (全個体が含まれていなくてもOK)
> POPMAP=popmap.txt
> # 許容する非欠損個体数の最小値
> min_inds=5
> ```
> ```bash
> # ファイル名にフィルタリング条件を追記
> IN=$OUT; OUT=${IN}_min${min_inds}perpop
>
> # 集団ごとに非欠損サンプル数を計算し, INFO列に書き込む (例: NS_pop1=10;NS_pop2=8;...)
> bcftools +fill-tags $IN.vcf.gz -Oz -o ${IN}_NStagged.vcf.gz \
>          -- --samples-file $POPMAP --tags NS,F_MISSING
> bcftools index ${IN}_NStagged.vcf.gz
> 
> # どれか1集団でも閾値に満たないSNPがあれば除去
> # サブシェル内の実行結果は「 NS_pop1<5 || NS_pop2<5 || ... 」のような感じになる
> bcftools view ${IN}_NStagged.vcf.gz -Oz -o $OUT.vcf.gz \
>               -e "$(bcftools view -h ${IN}_NStagged.vcf.gz \
>                     | grep -o -E 'NS_[A-Za-z0-9_]+' | sort -u \
>                     | while read line; do echo "${line}<${min_inds}"; done \
>                     | xargs echo | sed -s 's/ / || /g')"
> bcftools index $OUT.vcf.gz
> ```

<a name="my-custom-anchor-point4"></a>
### 4.4. MAF (マイナーアリル頻度) によるフィルタリング
```bash
# 許容するマイナーアリル頻度の下限値
maf=0.1
```
```bash
# ファイル名にフィルタリング条件を追記
IN=$OUT; OUT=${IN}_maf${maf}

# 実行
bcftools view $IN.vcf.gz -Oz -o $OUT.vcf.gz \
              -e "MAF < $maf"
bcftools index $OUT.vcf.gz
```
>[!note]
> ### マイナーアリルのカウントでフィルタリングする場合
> ```bash
> # 許容するマイナーアリルカウントの下限値
> mac=3
> ```
> ```bash
> # ファイル名にフィルタリング条件を追記
> IN=$OUT; OUT=${IN}_mac${mac}
>
> # 実行
> bcftools view $IN.vcf.gz -Oz -o $OUT.vcf.gz \
>               -e "MAC < $mac"
> bcftools index $OUT.vcf.gz
> ```


<a name="my-custom-anchor-point5"></a>
### 4.5. ハーディワインベルグ平衡検定によるフィルタリング
[ハーディーワインベルグ平衡検定](https://www.cell.com/ajhg/fulltext/S0002-9297(07)60735-6?_returnURL=https%3A%2F%2Flinkinghub.elsevier.com%2Fretrieve%2Fpii%2FS0002929707607356%3Fshowall%3Dtrue#fig1)を行い, ハーディワインベルグ平衡からの有意な逸脱 (=低い*p*値) を示した座位を除去する.

```bash
# 許容する HWE p の下限値
hwe=0.001
```
```bash
# ファイル名にフィルタリング条件を追記
IN=$OUT; OUT=${IN}_hwe${hwe}

# HWE p を計算し, INFO列に書き込む (例: HWE=0.59)
# 閾値に満たないSNPがあればそのSNPを除去
bcftools +fill-tags $IN.vcf.gz \
         -- --tags HWE \
| bcftools view -Oz -o $OUT.vcf.gz \
                -e "HWE < $hwe"
bcftools index $OUT.vcf.gz
```

>[!note]
> ### データセット内に集団構造がある場合の HWE フィルタリング
> 集団ごとに個別に HWE p を計算する必要がある ([詳細](https://onlinelibrary.wiley.com/doi/10.1111/1755-0998.13646))
>
> ```bash
> # 許容する HWE p の下限値
> hwe=0.001
> 
> # Popmap (集団分けを書いたタブ区切りテキスト)
> POPMAP=popmap.txt
> ```
> ```bash
> # ファイル名にフィルタリング条件を追記
> IN=$OUT; OUT=${IN}_hwe${hwe}
>
> # 集団ごとに HWE p を計算し, INFO列に書き込む (例: HWE_pop1=0.59;HWE_pop2=1;...)
> bcftools +fill-tags $IN.vcf.gz -Oz -o ${IN}_HWEtagged.vcf.gz \
>          -- --samples-file $POPMAP --tags HWE
> bcftools index ${IN}_HWEtagged.vcf.gz
> 
> # どれか1集団でも閾値に満たないSNPがあればそのSNPを除去
> # サブシェル内の実行結果は「 HWE_pop1<0.001 || HWE_pop2<0.001 || ... 」のような感じになる
> bcftools view ${IN}_HWEtagged.vcf.gz -Oz -o $OUT.vcf.gz \
>               -e "$(bcftools view -h ${IN}_HWEtagged.vcf.gz \
>                     | grep -o -E 'HWE_[A-Za-z0-9_]+' | sort -u \
>                     | while read line; do echo "${line}<${hwe}"; done \
>                     | xargs echo | sed -s 's/ / || /g')"
> bcftools index $OUT.vcf.gz
> ```

<a name="my-custom-anchor-point6"></a>
### 4.6. 連鎖不平衡にある SNP の除去 (LD pruning)
>[!caution]
> ID フィールドで抜き出しを行うので, 各 SNP に固有の ID が与えられているかを確認しておく.
>
> ID を `<染色体>:<位置bp>` のようにする場合は, `bcftools annotate -x ID -I +'%CHROM:%POS'` が有効.


`plink`の`--indep-pairwise`オプションを使用する

```bash
# ウインドウサイズ. 末尾にkb をつけるとリファレンスゲノム上での塩基数になり, つけないとSNP数になる
win=100kb

# ウィンドウの移動幅
stp=5

# 許容する r^2 値の最大値
rsq=0.2
```

```bash
# ファイル名
IN=$OUT
OUT=${IN}_ld${win}-${stp}-${rsq}

# bed 形式への変換
tmpdir=$(mktemp -d ./bed_XXXXXX)
plink --allow-extra-chr --const-fid --keep-allele-order --make-bed \
      --vcf $IN.vcf.gz \
      --out $tmpdir/$(basename $IN)

# plink で残すSNPのIDを書いたリスト (*.prune.in) を作成
plink --allow-extra-chr --const-fid --keep-allele-order \
      --bfile $tmpdir/$(basename $IN) \
      --indep-pairwise $win $stp $rsq \
      --out $OUT

# リストされたSNPだけを抜き出す
bcftools view $IN.vcf.gz -Oz -o $OUT.vcf.gz \
              -i ID=@$OUT.prune.in
```

>[!tip]
> ### `plink` の LD Pruning の原理
>
> 指定したウィンドウ内の2つのSNPペア $`X, Y`$ について, 各個体のアリルカウント $` x_i, y_i \in \{ 0, 1, 2 \} `$ に基づいて得られたジェノタイプベクトル $`\{ x_1, x_2, \cdots , x_n \}, \{ y_1, y_2, \cdots , y_n \}`$  の間の相関係数の二乗値 $`r^2`$ を計算する.
> これをウィンドウ内の全 SNP ペアに対して行い, ウィンドウ内に $`r^2`$ が閾値を超えるペアがなくなるまで SNP を除去し続ける.
> 最終的に, 残った SNP の ID は `*.prune.in` に, 除去された SNP の ID は `*.prune.out` に保存される.


>[!note]
> ### `PCAone` による集団構造を考慮した LD Pruning
> データセット内に強い集団構造があると, `plink` に実装されているような通常のアルゴリズムの場合, 実際には連鎖していないSNPペア間てあっても, 集団間のアリル頻度差によって $`r^2`$ が上昇する傾向がある (たとえば, 2 つの SNP がそれぞれ 2 集団間で異なるアリルに固定していた場合, 実際には連鎖していなくても $`r^2 = 1`$ になってしまう). `PCAone` はこれを考慮し, 集団構造の上位主成分に基づいて補正した $`r^2_{\text{adj}}`$ を算出する ([詳細](https://academic.oup.com/genetics/article-abstract/229/3/iyaf009/8002559?redirectedFrom=fulltext)).
>
> ```bash
> # リファレンスゲノム上でのウインドウサイズ. 数字で指定
> win=100000
>
> # 許容する r^2 値の最大値
> rsq=0.2
> ```
> ```bash
> # ファイル名
> IN=$OUT; OUT=${IN}_ld-adj-${win}-${rsq}
> 
> # PLINK バイナリ形式 (.bed + .bim + .fam) への変換
> tmpdir=$(mktemp -d ./bed_XXXXXX)
> plink --allow-extra-chr --const-fid --keep-allele-order --make-bed \
>       --vcf $IN.vcf.gz \
>       --out $tmpdir/$(basename $IN)
>
> # Ancestry adjusted LD matrix を計算
> # --pc で考慮する主成分の数を, --emu で欠損データに対する扱いを指定
> PCAone --bfile $tmpdir/$(basename $IN) \
>        --pc 3 --emu --svd 0 --ld --out ${IN}_adj
>
> # ウィンドウ内の r^2 閾値によって LD pruning
> PCAone --binary ${IN}_adj.residuals \
>        --match-bim ${IN}_adj.mbim \
>        --ld-bp $win --ld-r2 $rsq --out ${IN}_adj-${win}-${rsq}
>
> # リストされたSNPを抜き出し
> bcftools view $IN.vcf.gz -Oz -o $OUT.vcf.gz \
>               -i ID=@<(cut -f2 ${IN}_adj-${win}-${rsq}.ld.prune.in)
> bcftools index  $OUT.vcf.gz
>
> # 中間生成ファイルの削除
> rm -r $tmpdir
> ```



