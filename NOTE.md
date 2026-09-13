- ELF (executable and linking format) 形式
  - プログラム本体とそのプログラムをどのように実行すべきかに関するメタデータの両方が含まれている
  - プログラムは機械語の列

  ```
  root ➜ /w/examples $ cbc hello.cb
  root ➜ /w/examples $ file hello
  hello: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 3.2.0, not stripped
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
- 構文解析
  - 字句解析 → 構文解析
  - 字句解析(スキャン)：ソースコードを解析してトークンの列を生成
    - トークン：単語 + 単語の種類 + 意味値
    - 54, 整数, "54"

    ```
    root ➜ /w/examples $ cbc --dump-tokens hello.cb
    "import"                "import"
    <SPACES>                " "
    <IDENTIFIER>            "stdio"
    ";"                     ";"
    <SPACES>                "\n\n"
    "int"                   "int"
    <SPACES>                "\n"
    <IDENTIFIER>            "main"
    "("                     "("
    "int"                   "int"
    <SPACES>                " "
    <IDENTIFIER>            "argc"
    ","                     ","
    <SPACES>                " "
    "char"                  "char"
    <SPACES>                " "
    "*"                     "*"
    "*"                     "*"
    <IDENTIFIER>            "argv"
    ")"                     ")"
    <SPACES>                "\n"
    "{"                     "{"
    <SPACES>                "\n    "
    <IDENTIFIER>            "printf"
    "("                     "("
    "\""                    "\"Hello, World!\n\""
    ")"                     ")"
    ";"                     ";"
    <SPACES>                "\n    "
    "return"                "return"
    <SPACES>                " "
    <INTEGER>               "0"
    ";"                     ";"
    <SPACES>                "\n"
    "}"                     "}"
    <SPACES>                "\n"
    <EOF>                   ""
    ```

  - 構文解析(パース)：スキャナが生成したトークンの列を解析して構文木を生成
    - 実際にはセミコロンや括弧は不要としてこの時点で削除してしまうことも多い（意味解析の仕事を先に実施して、ASTにする）

  - スキャナジェネレーター / パーサジェネレーター
    - cbc では [JavaCC](https://javacc.github.io/javacc/) を使っている
    - 人間は .jj に [EBNF (Extended Backus-Naur Form) 記法](https://ja.wikipedia.org/wiki/EBNF)で文法定義ファイルを記述
    - パーサージェネレータには種類がある LR / LALR / LL
      - 扱える記法の広さと生成速度・シンプルさのトレードオフ
      - OpenCC は LL パーサジェネレーター

    ```
    root ➜ /w/examples $ javacc Adder.jj
    Java Compiler Compiler Version 5.0 (Parser Generator)
    (type "javacc" with no arguments for help)
    Reading from file Adder.jj . . .
    File "TokenMgrError.java" is being rebuilt.
    File "ParseException.java" is being rebuilt.
    File "Token.java" is being rebuilt.
    File "SimpleCharStream.java" is being rebuilt.
    Parser generated successfully.
    root ➜ /w/examples $ ls Adder.*
    Adder.java  Adder.jj
    root ➜ /w/examples $ javac Adder.java
    root ➜ /w/examples $ ls Adder.*
    Adder.class  Adder.java  Adder.jj
    root ➜ /w/examples $ java Adder '1+1'
    2
    root ➜ /w/examples $ java Adder '1 + 1'
    2
    root ➜ /w/examples $ java Adder '(1 + 1)'
    Exception in thread "main" TokenMgrError: Lexical error at line 1, column 1.  Encountered: "(" (40), after : ""
      at AdderTokenManager.getNextToken(AdderTokenManager.java:270)
      at Adder.jj_consume_token(Adder.java:117)
      at Adder.expr(Adder.java:22)
      at Adder.evalute(Adder.java:17)
      at Adder.main(Adder.java:8)
    root ➜ /w/examples $
    ```

- 字句解析 (JavaCC)
  - 事前定義した ditective を用いて、ソースコードから正規表現で一致した ditective を TOKEN の列として出力する
  - 複数 ditective に一致する場合は最長一致の原則で処理する
    - 現在のスキャン位置から始まるトークンとして、マッチしうる最も長い文字列を1つのトークンとして切り出す
  - ditective
    - TOKEN 命令
    - SKIP 命令 / SPECIAL_TOKEN 命令
    - MORE 命令 (この規則にマッチしただけではスキャンはまだ終わっていないとスキャナに伝えることができる)
  - ブロックコメントの例:

  ```
  MORE: { <"/*"> : IN_BLOCK_COMMENT }
  <IN_BLOCK_COMMENT> MORE: { <~[]> }
  <IN_BLOCK_COMMENT> SPECIAL_TOKEN: { <BLOCK_COMMENT: "*/"> : DEFAULT }
  ```

  - `SKIP: { <"/*" (~[])* "*/"> }` とは書かない
    - `x = /* one */ a + b /* two */;` みたいなケースで最長一致の原則が働いて `a + b` までマッチしてしまう
    - 対策として `/*` にマッチした時専用の状態 `IN_BLOCK_COMMENT` を導入し、この状態専用の字句解析規則のみを動作するようにしている
    - 状態に対応する ditective が複数ある場合は、に一致する場合は最長一致の原則で処理する
    - `(~[])*` を `~[]` を書き直し、この規則が表現できる長さを1文字に固定しているので、`*/` がマッチした時は `*/` (2文字) の方のパターンとして処理できる
  - MORE でブロックコメントを閉じ忘れたままファイル末尾に到達するのを防いでいる
  - ブロックコメントや文字列のように、開始と終了があるトークンは複雑な正規表現を書くより、MORE と状態遷移で書いた方がシンプルになる
  - スキャナは状態を持てる `IN_BLOCK_COMMENT`
  - パターンマッチした時に、対応する状態名に遷移することができる
  - `DEFAULT` で最初の状態に戻ることができる
