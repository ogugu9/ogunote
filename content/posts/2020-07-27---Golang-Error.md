---
title: Go言語のエラー処理
date: "2020-07-27T20:00:00+0900"
template: "post"
draft: false
slug: "golang-error"
category: "Go言語"
tags:
  - "Go言語"
description: "チームの試みとして小さなAppをGo言語で書いたものがあり、そのレビューをすることになった。その際、A Tour Of Go の中にはなかったエラー周りの記述をまとめてみた。"
socialImage: "/media/42-line-bible.jpg"
---

- [はじめに](#はじめに)
- [error interface](#error interface)
- [エラーの生成](#エラーの生成)
- [エラーハンドリング](#エラーハンドリング)
- [Wrap(err)](#Wrap(err))
- [Panic](#Panic)
- [Recover](#Panic)
- [まとめ](#まとめ)

# はじめに
チームの試みとして小さなAppをGo言語で書いたものがあり、そのレビューをすることになった。  
その際、[A Tour Of Go](https://go-tour-jp.appspot.com/list) の中にはなかったエラー周りの記述をまとめてみた。

今回は、そこで自分が調べた記事の総集編という形式になる。

# error interface
* Go言語ではエラーを処理するために `error` インタフェースがある

```go
type error interface {
    Error() string
}
```

* `func Error() string` (Errorという名前で引数がなくstringを返却する関数) を実装した型はすべてerrorインタフェースが実装されているものとみなす

# エラーの生成

## errors.New
```go
package main

import (
    "errors"
    "fmt"
    "os"
)

func main() {
    if err := doError(); err != nil {
        fmt.Println("err", err)
        os.Exit(1)
    }
    fmt.Println("(o・∇・o)終わりだよ～") // ここにはこない
}
func doError() error {
    return errors.New("(*>△<)<ナーンナーンっっ")
}
```

## fmt.Errorf
```go
package main

import (
    "fmt"
    "os"
)

func main() {
    if err := doError(); err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    fmt.Println("(o・∇・o)終わりだよ～") // ここにはこない
}
func doError() error {
    msg := "(*>△<)<ナーンナーンっっ"
    return fmt.Errorf("err %s", msg)
}
```

## error interfaceを実装する
```go
package main

import (
    "fmt"
    "os"
)

// エラー処理用の構造体
type MyError struct {
    Msg string
    Code int
}
// MyError構造体にerrorインタフェースのError関数を実装
func (err *MyError) Error() string {
    return fmt.Sprintf("err %s [code=%d]", err.Msg, err.Code)
}

func main() {
    if err := doError(); err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    fmt.Println("(o・∇・o)終わりだよ～") // ここにはこない
}
func doError() error {
    return &MyError{Msg: "(*>△<)<ナーンナーンっっ", Code: 19}
}
```

# エラーハンドリング
Go言語では `try～catch～finally` の例外処理は存在しない。  
では、どうやるのか…？

例) `os.Open`

```go
func Open(name string) (*File, error)
```

```go
f, err := os.Open("/tmp/hogehoge.txt")
if err != nil {
    // エラー時の処理
    log.Fatal(err)
}
```

## 型ごとのゼロ値
ここで補足として、型ごとのゼロ値についておさらいしておく。  
なぜなら、前述の通り、 Go言語のエラーハンドリングは「それがゼロ値＝`nil`であるか」によって行うためである。

| type      | zero value       |
| ---       | ---              |
| boolean   | false            |
| int       | 0                |
| float	    | 0.0
| string	| ""(空文字)
| pointer   | nil(他言語でいうnullなど)
| function  | nil(他言語でいうnullなど)
| interface	| nil(他言語でいうnullなど)
| slice     | nil(他言語でいうnullなど)
| channel	| nil(他言語でいうnullなど)
| map	    | nil(他言語でいうnullなど)

補足
* nilは型を持つ
* interfaceの場合のみ、型もnilでないとxxx == nilはfalse


# Wrap(err)
* Goのエラーはスタックトレースを保持しない
* なので、どうにかしてエラー発生箇所のコンテキスト・情報（stacktrace, etc.）をちゃんと残したい
* `errors.Wrap(err, "...")` もしくは `errors.WithStack(err)` しておくことで，`pkg/errors.StackTrace` をみれば「アプリケーションコード内で最初にエラーが起きたのはどこか」を調べることができる

ref. [`Wrap(err)` in our production #golang](https://www.wantedly.com/companies/wantedly/post_articles/139554)

# Panic
* 関数内で発生したエラーが致命的なため、あえてエラーハンドリングを行う必要がないケースがある
* その場合、`panic()` という組み込み関数を呼び出して「パニック状態」にする

```go
package main

func func1() {
	panic("Occured panic!")
}

func main() {
	func1()
}
```

実行すると…
1. その関数の実行は中断され、呼び出し元に復帰
2. 呼び出し元に復帰後も、順次関数の呼び出し元をさかのぼる
3. 最終的にプログラム自体が終了
4. 「panic」を呼び出した箇所とともに、引数として渡したパラメータの値が標準エラー出力

ref. [http://cuto.unirita.co.jp/gostudy/post/panic/](http://cuto.unirita.co.jp/gostudy/post/panic/)

# Recover

* defer 文を使って panic() したときの後始末が記述できる
* あらかじめ関数の呼び出しで panic() 関数が起こる可能性が予見できる場合、 `defer` を使ってその復旧方法を書いておく
* panic() 関数が呼ばれた場合には recover() 関数を使ってその内容を取得できる

```go
package main

import (
    "fmt"
)

func helloworld() {
    defer func() {
        fmt.Println("End")
        err := recover()
        if err != nil {
            fmt.Println("Recover!:", err)
        }
    }()

    fmt.Println("Start")
    panic("Panic!")
}


func main() {
    helloworld()
}
```

ref. [Golang の defer 文と panic/recover 機構について](https://blog.amedama.jp/entry/2015/10/11/123535)

# まとめ

- Go言語ではエラーを処理するために `error` インタフェースがある
- エラーハンドリングさせたい場合、関数側で正常値とエラー値を複数戻り値としてセットで返す
- エラーハンドリングする場合、エラー値が `nil` かどうかで行う
- 致命的なエラーが起きた場合、 `panic()` という組み込み関数を呼び出し、「パニック状態」になる
- 「パニック状態」をハンドリングする場合、 `defer` 文を使って `panic()` したときの後始末をする
