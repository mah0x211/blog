# Go バイナリに静的ファイルを埋め込む方法

夏頃から仕事で所謂 CI ツール––CircleCI とか Travis CI の様なコミット時に自動実行する様なツール––の PoC 開発をやっているのだけれど、フロントエンドの静的ファイルをどうしたものか考えていたら良い感じのモジュールを見つけたのでメモ。

背景としては、折角 Go で作ってるのだから静的ファイルも実行バイナリに埋め込めるとデプロイがすごく楽になっていいよなぁ、なんてありふれた思いつきから端を発している。

そこで GitHub 探してみると結構色々とあったのだけれど、最終的にはぱっと見でコードもドキュメントもとてもシンプルでわかりやすく、外部依存モジュールもない `statik` を利用することにした。

> ### [rakyll/statik](https://github.com/rakyll/statik)
>
> statik allows you to embed a directory of static files into your Go binary to be later served from an http.FileSystem.
>
> Is this a crazy idea? No, not necessarily. If you're building a tool that has a Web component, you typically want to serve some images, CSS and JavaScript. You like the comfort of distributing a single binary, so you don't want to mess with deploying them elsewhere. If your static files are not large in size and will be browsed by a few people, statik is a solution you are looking for.

使い方はドキュメントに書いてある通りで、`go get github.com/rakyll/statik` でコマンドをインストールして、`statik` コマンドで静的ファイルの配置されたディレクトリを指定する事で Go のコード＝パッケージが生成され、そのパッケージをインポートする事で実行バイナリに埋め込まれる様になる。

サクッと使えたら良いのだけれど、実際のプロジェクトに適用するには以下の三つの課題をクリアする必要がある；

- 「継続的な開発」がやりづらいと困るので、コマンドライン引数で埋め込んだ静的ファイルを使うか、指定されたディレクトリを使うかを選択可能である。
- 利用している URL ルータ––ちなみに [httprouter](https://github.com/julienschmidt/httprouter) を利用している––で利用可能である。
- 静的ファイルにはテンプレートファイルも含まれているが、テンプレートファイルは静的配信を行ってはいけない。

そこで、以下の様にラッパーレイヤーを設けて使う事にした。

```go
package assets

import (
	"io/ioutil"
	"net/http"
	"log"
	"os"
	"path/filepath"
	"syscall"

	_ "./statik" // register embedded data to statik/fs on init func

	"github.com/rakyll/statik/fs"
)

var gRootDir string
var gStatikFS http.FileSystem

func init() {
	var err error
	if gStatikFS, err = fs.New(); err != nil {
		log.Fatalf("failed to fs.New():", err)
	}
}

type ProxyFS string

func (p ProxyFS) Open(filename string) (http.File, error) {
	return gStatikFS.Open(string(p) + "/" + filepath.Clean(filename))
}

func FileSystem(dirname string) http.FileSystem {
	if gRootDir == "" {
		return ProxyFS(filepath.Join("/", dirname))
	}
	return http.Dir(filepath.Join(gRootDir, filepath.Clean(dirname)))
}

func SetRootDir(dirname string) error {
	if gRootDir != "" {
		log.Fatal("root directory already exists")
	}

	var err error
	dirname, err = Realpath(dirname)
	if err != nil {
		return err
	} else if info, err := os.Lstat(dirname); err != nil {
		return err
	} else if !info.IsDir() {
		return syscall.ENOTDIR
	}
	gRootDir = dirname

	return nil
}

func ReadFile(filename string) ([]byte, error) {
	filename = filepath.Clean(filename)
	if gRootDir != "" {
		return ioutil.ReadFile(gRootDir + "/" + filename)
	} else if f, err := gStatikFS.Open("/" + filename); err != nil {
		return nil, err
	} else {
		defer f.Close()
		return ioutil.ReadAll(f)
	}
}
```

`assets.SetRootDir(dirname string)` で、開発時には OS のファイルシステムを利用する様に切り替える。また、`httprouter` を利用しているので静的ファイル配信は `static` ディレクトリの配下に配置して以下の様に `assets.FileSystem` の戻り値を登録する。

```go
route := httprouter.New()
route.ServeFiles("/static/*filepath", assets.FileSystem("/static"))
```

テンプレートファイルを読み込むときは `assets.ReadFile` で読み込む。

```go
buf, err := assets.ReadFile(pathname)
```

実際のコードとはちょっと違うけど大体こんな感じで実装できた。

まだ試してないけど、サーバレスのコードを実装する時も同じ要領でテンプレートファイルを埋め込めば楽かもしれない。
