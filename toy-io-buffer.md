C言語でI/Oバッファリング遊び
----

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main () {
    printf("|'-') hi ichirin desu\n");
    write(1, "|'o')\n", 6);
    pid_t pid = fork();
    if (pid == 0) {
        exit(0);
    }
    return 0;
}
```

```
$ gcc fork.c
$ ./a.out
|'-') hi ichirin desu
|'o')
$ ./a.out > out.txt
$ cat out.txt
|'o')
|'-') hi ichirin desu
|'-') hi ichirin desu
```

### 一見不思議なこと

- printf(3)の文字列が2回出力される
- write(2)の文字列は1回しか出力されない
- write(2)の出力がprintf(3)の出力より先行している


### 各関数のポイント

- printf(3)はライブラリ関数でwrite(2)はシステムコール  
- printf(3), fscanf(3), fgets(3), などのstdioライブラリ関数は内部でバッファを持つ（ユーザメモリ空間）
- fork(2)はプロセスの複製でstdioバッファもその対象
- exit(3)はstdioバッファをフラッシュする


### printf(3)の文字列が2回出力される件
stdioライブラリだと出力が端末の場合はデフォルトで行バッファリングになるため、改行が含まれる文字列はprintfでも直後に表示される。改行が含まれない場合はバッファリングされる。出力がファイルの場合はデフォルトでブロックバッファリングになる。リダイレクトすることによって標準出力先がファイルになり、改行ではフラッシュされなくなる。fork(2)時点ではstdioバッファ内に溜まったままなので、子プロセスにも複製される。親/子プロセスがexit(3)を実行することにより、それぞれが持つstdioバッファがフラッシュされて結果的に同じ文字列が出力される。ちなみに標準エラー出力はデフォルトで非バッファリングなので、今回のように2回出力 & 出力が前後することもない。

### write(2)の文字列は1回しか出力されない
write(2)はシステムコールなのでstdioライブラリ関数のようにユーザメモリ空間にバッファリングしないため、fork(2)による複製が行われない。write(2)はどこに書くかと言うとカーネルバッファに直接書くようになっている。


### write(2)の出力がprintf(3)の出力より先行している
まず、出力が前後することとfork(2)は関係がなく、fork処理がなくても今回のケースでは出力の前後が発生する。端末に出力するときは行バッファリングで改行文字によりフラッシュされてwrite(2)システムコールが呼ばれていたが、リダイレクトにより出力がブロックバッファリングになった関係ですぐにフラッシュされずにwrite(2)システムコールを呼び出すタイミングが遅れた。そのため、直後のwrite(2)の文字列出力が先行することになった。

- write(2) => カーネルバッファに書き込み
- printf(3) => stdioバッファに書き込み => exit(3)でstdioバッファをフラッシュ => カーネルバッファに書き込み

![io-buffer](https://github.com/ichirin2501/doc/blob/master/images/io-buffer-image.png)
Linuxプログラミングインタフェースから拝借

### 参考
- https://linuxjm.osdn.jp/html/LDP_man-pages/man2/write.2.html
- https://linuxjm.osdn.jp/html/LDP_man-pages/man2/fork.2.html
- https://linuxjm.osdn.jp/html/LDP_man-pages/man2/open.2.html
- https://linuxjm.osdn.jp/html/LDP_man-pages/man3/setbuf.3.html
- https://linuxjm.osdn.jp/html/LDP_man-pages/man3/stdio.3.html
- https://linuxjm.osdn.jp/html/LDP_man-pages/man3/stdout.3.html
- https://linuxjm.osdn.jp/html/LDP_man-pages/man3/exit.3.html
- https://www.oreilly.co.jp/books/9784873115856/
