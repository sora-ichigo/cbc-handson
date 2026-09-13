# cbc-handson

『ふつうのコンパイラをつくろう』を Apple Silicon Mac から読み進めるための環境。
本は Linux / IA-32 (32bit x86) 前提なので、GitHub Codespaces の x86_64 Linux 上に 32bit ツールチェインを載せて動かす。

## 構成

```
.devcontainer/   Codespaces / Dev Container 定義 (Ubuntu 22.04 + gcc-multilib + JDK 17 + ant + javacc)
cbc/             本家 cbc (https://github.com/aamine/cbc) を取り込み、64bit Ubuntu 向けに修正したもの
examples/        動作確認用の Cb ソース
```

## 使い方

```sh
gh codespace create -R sora-ichigo/cbc-handson -m basicLinux32gb
gh codespace ssh
```

コンテナ作成時に `make -C cbc` が走り、`cbc/bin` に PATH が通る。

```sh
uname -m                 # x86_64
cd examples
cbc hello.cb             # hello を生成
file hello               # ELF 32-bit LSB executable, Intel 80386
./hello                  # Hello, World!
cd ..
make -C cbc test         # cbc 本体のテスト
```

`make -C cbc test` では `vardecl.cb` の 2 件が失敗する。テストが参照する `sys_errlist` が glibc 2.32 で削除されたためで、環境の問題ではない。

## ローカルで動かす場合

Rosetta ベースの amd64 コンテナ (Docker Desktop, OrbStack, colima `--vz-rosetta`) は 32bit バイナリを実行できない。
colima で x86_64 VM を立てれば QEMU エミュレーションになるが、32bit ELF まで通しで動く。

```sh
colima start x86 --arch x86_64 --cpu 4 --memory 4
export DOCKER_HOST=unix://$HOME/.colima/x86/docker.sock
docker build -t cbc-handson .devcontainer
docker run --rm -it -v "$PWD:/w" -w /w -e PATH=/w/cbc/bin:/usr/bin:/bin cbc-handson bash
```

## cbc への修正点

本家 cbc は 2009 年の 32bit Linux 前提で書かれているため、以下を変更している。

- `sysdep/GNUAssembler.java`: `as` に `--32` を付ける
- `sysdep/GNULinker.java`: `ld` に `-m elf_i386 -L/usr/lib32` を付け、crt ファイルを `/usr/lib32/` から取る
- `type/CompositeType.java`, `parser/Parser.jj`: `import java.lang.reflect.*` が Java 8 以降の `java.lang.reflect.Parameter` と `entity.Parameter` で衝突するため、必要なクラスだけ import する
