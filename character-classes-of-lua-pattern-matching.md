**NOTE：** Qiita に書いていたものをこちらへ移動。

Lua のパターンマッチングの文字クラスが具体的に何を指してるのか、マニュアルに記載されてないおかげか、はたまた、脳に Stop-The-World を起こさせるジメジメした暑さのおかげかなのか、調べては忘れ調べてはまた忘れなんて事を毎度毎度繰り返してるので、いい加減まとめておこうと思ったのでメモ。

> [Lua 5.1 Reference Manual - 5.4.1 Patterns](https://www.lua.org/manual/5.1/manual.html#5.4.1)
>
> - %a: represents all letters.
> - %c: represents all control characters.
> - %d: represents all digits.
> - %l: represents all lowercase letters.
> - %p: represents all punctuation characters.
> - %s: represents all space characters.
> - %u: represents all uppercase letters.
> - %w: represents all alphanumeric characters.
> - %x: represents all hexadecimal digits.

マニュアルに書いてない事はググるよりソースにあたる方が良いので、ソースあさると [lstrlib.c](https://github.com/lua/lua/blob/98194db4295726069137d13b8d24fca8cbf892b6/lstrlib.c#L225) にそれっぽい箇所を発見。

```c:lstrlib.c
static int match_class (int c, int cl) {
  int res;
  switch (tolower(cl)) {
    case 'a' : res = isalpha(c); break;
    case 'c' : res = iscntrl(c); break;
    case 'd' : res = isdigit(c); break;
    case 'l' : res = islower(c); break;
    case 'p' : res = ispunct(c); break;
    case 's' : res = isspace(c); break;
    case 'u' : res = isupper(c); break;
    case 'w' : res = isalnum(c); break;
    case 'x' : res = isxdigit(c); break;
    case 'z' : res = (c == 0); break;
    default: return (cl == c);
  }
  return (islower(cl) ? res : !res);
}
```

コード内に明確に記載されているように標準Cライブラリ関数で実装されているので、具体的に指しているものは以下の文字集合になる；

- `%a`: `man isalpha` -> *A-Za-z*
- `%c`: `man iscntrl` -> *0x00-0x1F*
- `%d`: `man isdigit` -> *0-9*
- `%l`: `man islower` -> *a-z*
- `%p`: `man ispunct` -> *!"#$%&'()\*+,-./:;<=>?@[\\]^_`{|}~*
- `%s`: `man isspace` -> *\t\n\v\f\r* and *0x20:whitespace*
- `%u`: `man isupper` -> *A-Z*
- `%w`: `man isalnum` -> *A-Za-z0-9*
- `%x`: `man isxdigit` -> *0-9A-Fa-f*

ついでに、Lua 5.2 と 5.3 系を見ると文字クラス `%g` が追加されている。

> [Lua 5.2 Reference Manual - 6.4.1 – Patterns](https://www.lua.org/manual/5.2/manual.html#6.4.1)
>
> ---snip---
> - %g: represents all printable characters except space.

こちらも同様に調べて見ると [`case 'g' : res = isgraph(c); break;`](https://github.com/lua/lua/blob/78d986590060b615334ac214b6cc5b7d951b1d58/lstrlib.c#L260) てな感じで標準ライブラリ関数で実装されているので、具体的に指しているものは以下の文字集合になる。

- `%g`: `man isgraph` -> *0-9a-zA-Z* and *!"#$%&'()\*+,-./:;<=>?@[\\]^_`{|}~*


***


２年くらい前から「調べてメモっとこ。」なんて思いつつもコードから離れると瞬間的に忘れてく鳥頭なのでなかなか書けなかったのだけれど、Kobito の試用ついでに書いてみた。

やっぱりこう、メモ的な Tips をサクッと書いてアップできるツールってのは便利。でも、プレビュー画面が書いてる箇所にスクロールしてくれないので、エディタとしては VSCode の方が楽かなぁ。
