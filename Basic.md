# ターミナルの基本
20240627作成\
20261008更新
## 1. ファイル・ディレクトリの確認
### `ls`コマンド
現在のディレクトリにあるファイルやディレクトリを表示する
```sh
ls
```
ファイルやディレクトリの詳細も同時に表示する
```sh
ls -l
```
隠しファイルも含めファイルやディレクトリを全て表示する
```sh
ls -a
```
隠しファイルを含む全てのファイルやディレクトリの詳細を表示する
```sh
ls −la
```

### `cd`コマンド
`test`というディレクトリに移動する
```sh
cd test
```
または
```sh
cd test/
```
ホームディレクトリに移動する
```sh
cd ~
```
一つ上の階層のファイルに移動する
```sh
cd ..
```

現在のディレクトリのパスを表示する
```sh
pwd
```

## 2. ファイル・ディレクトリの編集
### ファイルの中身の確認
`test.txt`というテキストファイルの中身を表示する
```sh
cat test.txt
```
#### 大きなファイルの場合
```sh
less test.txt
```
### ファイルの作成
`test.txt`という名前で中身が空のファイルを作る
```sh
touch test.txt
```
`test.txt`という名前で中身が`Hello Goodbye`のファイルを作る
```sh
echo "Hello Goodbye" > test.txt
```
### ファイルの編集
`test.txt`を直接編集
```sh
nano test.html
```
### `mkdir`コマンド
`test`というディレクトリを作る
```sh
mkdir test
```
複数階層のディレクトリを一気に作成
```sh
mkdir -p test1/test2/test3
```
### `mv`コマンド
`test.html`というファイルを相対パスで`tmp`というディレクトリに移動させる
```sh
mv test.html tmp
```
`test.html`というファイルを`test2.html`に名前変更する
```sh
mv test.html test2.html
```
### `cp`コマンド
`test.html`を相対パスで`tmp`というディレクトリの中にコピーする
```sh
cp test.html tmp/
```
`test.html`を`test2.html`という名前で現在のディレクトリにコピーする
```sh
cp test.html test2.html
```
`dir`というディレクトリとその中身を`tmp`内にそっくりコピーする
```sh
cp −r dir tmp/
```
>`scp`: ネットワークごしにファイルをコピーする (`ssh` + `cp` の意)
>
>基本的な使い方は`cp`と同じ
### `rm`コマンド
`test.txt`というファイルを削除する
```sh
rm test.html
```
`test`というディレクトリとその中身を削除する
```sh
rm -r test
```


## 3. その他 よく使うコマンド
|コマンド|機能|使用例|意味|
|-|-|-|-|
|`grep`|テキストファイル内を検索|`grep "abc" example.txt`|`example.txt`内の`abc`を含む行だけを出力|
|`wc`|テキストファイルの文字数をカウント|`wc -l example.txt`|`example.txt`の行数をカウント|
|`head`/`tail`|テキストファイルの先頭/末尾を表示|`head -5 example.txt`|`example.txt`の冒頭5行だけ表示|
|`\|`|出力を次のコマンドの入力に渡す|`grep "a" ex.txt \| wc -l`|`ex.txt`内の`a`を含む行数をカウント|
|`>`|出力をファイルに保存|`echo "Hello" > ex.txt`|"Hello" という出力を`ex.txt`というファイルに保存|
