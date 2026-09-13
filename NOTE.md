- ELF (executable and linking format) 形式
  - プログラム本体とそのプログラムをどのように実行すべきかに関するメタデータの両方が含まれている
  - プログラムは機械語の列

  ```
  $ file /bin/ls
  /bin/ls: Mach-O universal binary with 2 architectures: [x86_64:Mach-O 64-bit executable x86_64] [arm64e:Mach-O 64-bit executable arm64e]
  /bin/ls (for architecture x86_64): Mach-O 64-bit executable x86_64
  /bin/ls (for architecture arm64e): Mach-O 64-bit executable arm64e
  ```

- 実行可能ファイルへの変換 (ビルド) プロセス
  - プリプロセス → (狭義の) コンパイル → アセンブル → リンク

  ```
  gcc -E hello.c -o hello.i
  gcc -S hello.i -o hello.s
  gcc -c hello.s -o hello.o
  gcc hello.o -o hello
  ```

- (狭義の) コンパイル
  - 構文解析 (パース、構文木の生成) → 意味解析 (抽象構文木ASTの生成) → 中間表現の生成 → コード生成 (アセンブリ言語の生成)
  - 横断的に最適化ステップが入ることもある
- c♭ コンパイラの仕様メモ
  - プリプロセッサがない #include や #define が使えない
  - 浮動小数点数関係の機能がない
  - #include の代わりに Java に似せた `import` 宣言を導入
