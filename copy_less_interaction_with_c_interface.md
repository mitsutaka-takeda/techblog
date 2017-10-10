# コピーなしでC IFを呼び出す。

## 目標

CのIFでは互換性を守るためにOpaque Data Typeを使用していることがあります。

Oapque Data Typeを使用したIFでは、ライブラリとクライアントの間をやりとりする型は、サイズのみ
クライアントに提供して実際の構造は隠しています。

今回はOpaque Data Typeを使用しているC IFと効率的にデータをやり取りする方法を考えます。

## 説明
以下のコードでは、`OpaqueDataType`は4バイトという情報のみをクライアントに提供しています。ライブラリの実装では、
それを`int`として扱っています。

```c
/** In header file **/
// ヘッダ・ファイルではサイズのみ。中身の構造は分からない。
struct OpaqueDataType{
  unsigned char data[4];
};

// 初期化関数でdataを初期化。
void initializeLibrary(OpaqueDataType* p);

// 使うときは、OpaqueDataTypeを引数に取る。
void add(OpaqueDataType* p, int x);
int getResult(OpaqueDataType* p);


/** In implementation file. **/
void initializeLibrary(OpaqueDataType* p){
  // dataをintとして扱う。
  int* sum = (int*)(p->data);
  *sum = 0;
}

void add(OpaqueDataType* p, int x){
  int* sum = (int*)(p->data);
  (*sum) += x;
}

int getResult(OpaqueDataType* p){
  return *((int*)(p->data));
}

/** Client **/

int main(){
  OpaqueDataType opaqueData;
  initializeLibrary(&opaqueData);
  add(&opaqueData, 1);
  add(&opaqueData, 2);

  assert(3 == getResult(&opaqueData));
}
```

C++からこのようなIFを扱うときは、上記のように`OpaqueDataType`の変数を定義して、IFを呼び出すのが一般的かと思います。

ライブラリの使用は実装の詳細であるため、`OpaqueDataType`をクライアントのコードの色々なところで使用するのは好ましくありません。
`std::vector`などのbufferに`OpaqueDataType::data`を保持して`OpaqueDataType`の型を消すことを考えます。


```cpp
/** Client **/
#include <cassert>
#include <vector>

int main(){

  std::vector<unsigned char> buf(4);

  {// ブロック1
      OpaqueDataType opaqueData;
      initializeLibrary(&opaqueData);
      add(&opaqueData, 1);
      std::copy(std::begin(opaqueData.data), std::end(opaqueData.data), std::begin(buf));
  }

  {// ブロック2
    OpaqueDataType opaqueData;
    std::copy(std::begin(buf), std::end(buf), std::begin(opaqueData.data));
    add(&opaqueData, 2);
    // bufに必要なデータがコピーされているので、1つめのブロックで足された1も結果に反映されている。
    assert(3 == getResult(&opaqueData));
  }

}
```

意図通り、必要なデータがコピーされていますが、何度もバイト列のコピーが行なわれています。

このコピーを省いて`std::vector`を直接`OpaqueDataType`として扱うことができるでしょうか。
`OpaqueDataType`も結局は4バイトのメモリ領域であるということ以外の情報は保持していません。

C++の標準では[StandardLayoutTypeというコンセプト](http://en.cppreference.com/w/cpp/concept/StandardLayoutType)があります。
ある型TがStandardLayoutTypeであるとき、その第1メンバ変数へのポインタは安全にT*へのポインタとして扱うことができます。

今回のケースでは`OpaqueDataType`がStandardLayoutTypeで、第1メンバ変数はdataになります。つまり、`OpaqueDataType`が、
StandardLayoutTypeであれば、unsigned char[4]と同等に扱えることになります。これをふまえて、`std::vector`を直接
`OpaqueDataType`として扱ってみます。


```cpp
/** Client **/
#include <cassert>
#include <vector>
#include <type_traits>

// C++: コンパイル時に型情報を取得してassertする。
static_assert(std::is_standard_layout_v<OpaqueDataType>);

int main(){

  std::vector<unsigned char> buf(4);

  {// ブロック1
    // このreinterpret_castはOpaqueDataTypeがStandardLayoutTypeなら安全。
    auto opaqueDataPointer = reinterpret_cast<OpaqueDataType*>(buf.data());
    initializeLibrary(opaqueDataPointer);
    add(opaqueDataPointer, 1);
  }

  {// ブロック2
    auto opaqueDataPointer = reinterpret_cast<OpaqueDataType*>(buf.data());
    add(opaqueDataPointer, 2);
    assert(3 == getResult(opaqueDataPointer));
  }
}
```

無事コピーなしで`std::vector`を`OpaqueDataType`として扱うことができました。

C++では`type_traits`というコンパイル時に型情報を取得するライブラリが用意されています。
`static_assert`で`OpaqueDataType`がStandardLayoutTypeであることを確認しておきましょう。
ライブラリのアップデートで`OpaqueDataType`がStandardLayoutTypeで無くなってしまっても、
コンパイル・エラーで気付くことができます。

[std::is_standard_layoutリファレンス@cpprefjp](https://cpprefjp.github.io/reference/type_traits/is_standard_layout.html)

## 最後に

今回はOpaque Data Typeを使用しているC IFと効率的にデータを交換する方法を見てきました。

StandardLayoutTypeであれば、`reinterpret_cast`で安全にキャストができます。`reinterpret_cast`は、適切に使えないと
危険ですが強力なツールです。`type_traits`や`static_cast`を利用して安全性を高めましょう。
