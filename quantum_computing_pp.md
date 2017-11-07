# Title: 量子コンピューティング++ (1)

# はじめに

MicrosoftやGoogleが続々と量子コンピュータのサービスやライブラリを公開しつつあります。数年後には「量子コンピュータ元年」が来るかもしれません。

C/C++でも量子コンピュータのシミュレータが多数実装されています。今回はgithubで公開されている[Quantum++](https://github.com/vsoftco/qpp)シミュレータで、量子コンピューティングの時代を少し覗いてみましょう。

# Disclaimer

この記事は、量子コンピューティングを勉強し始めた素人が書いています。理解不足から間違えた記載が多々あると思いますので、参考資料を読んで下さい。

# Qubit

現在のコンピュータの基本単位はbitで、量子コンピュータの基本単位はqubitです。

bitでは`0`か`1`かの2つの状態を取ります。一方、qubitも`|0>`と`|1>`という２つの状態を取ります。
bitでは`0`か`1`か、どちらか２つの状態しか取れません。qubitは`|0>`、`|1>`と、さらにその組み合わせの状態(superposition)を取ることができます。

```math
|\psi> = \alpha |0> + \beta |1>
```

$\alpha$と$\beta$は、$|\psi>$を観測した時に$|0>$と$|1>$になる確率を表現する係数です。確率を表現するため係数の絶対値の２乗の和が１となります（$|\alpha|^2 + |\beta|^2 = 1$）。qubitは確率を含んだ状態であることと、bitのような特定の状態(`0`または`1`)にあるのを確定するにはqubitを観測する必要があることが、現在のコンピュータとの大きな違いです。

表と裏が出る確率が$\frac{1}{2}$づつの公平なコインをqubitで表現してみましょう。`|0>`を表、`|1>`を裏で表現すると、
公平なコインは、$|\alpha|^2 = \frac{1}{2}$、$|\beta|^2 = \frac{1}{2}$。

```math
|coin> \ = \frac{|0>}{\sqrt{2}} + \frac{|1>}{\sqrt{2}}
```

bitも複数つなげることができるようにqubitも複数つなげることができます。例えば、2 qubitは、0と1を２つ並べた`|00>`、`|01>`、`|10>`、`|11>`の重ね合わせを表現することができます。確率的に４つの状態を取るものを表現できるため、例えば、公平なジャンケン・プログラムは、`|00>`をグー、`|01>`をチョキ、`|10>`をパーとすると、以下のように表現できます。

```math
| rock\ paper\ cissors >\ = \frac{|00>}{\sqrt{3}} + \frac{|01>}{\sqrt{3}} + \frac{|10>}{\sqrt{3}} + 0 \cdot |11>
```

`|11>`に対応する手がないので係数は0になります。

ここで量子コンピュータをシミュレートするQuantum++を使用して、コインとジャンケンのqubitを表現するコードを書いてみます。

```cpp
#include <iostream>
#include "qpp.h"

// Qubitsを表現するユーザ定義型リテラル。
template <char... Bits> 
qpp::ket operator "" _q(){
    constexpr char bits[sizeof...(Bits) + 1] = {Bits..., '\0'};
    qpp::ket q = qpp::ket::Zero(sizeof...(Bits)*2);
    q(std::stoi(bits, nullptr, 2)) = 1;
    return q;
}

int main()
{
    //                    (      表      ) + (      裏      )
    qpp::ket const coin = 0_q/std::sqrt(2) + 1_q/std::sqrt(2);
    std::cout << "= コイン =\n" << qpp::disp(coin) << '\n';
    //                                  (      グー     ) + (     チョキ     ) + (      パー      )
    qpp::ket const rock_paper_cissors = 00_q/std::sqrt(3) + 01_q/std::sqrt(3) + 10_q/std::sqrt(3);
    std::cout << "= ジャンケン =\n" << qpp::disp(rock_paper_cissors) << '\n';
}
```

このコードの実行結果は以下のようになります。実行結果の中で"#"以降は実行結果に追加したコメントです。標準出力にqubitを出力すると、各状態の係数が表示されます。$\frac{1}{\sqrt {2}} \approx 0.707107$で$\frac{1}{\sqrt{3}} \approx 0.57735$です。

```shell
= コイン =
0.707107      # 表|0>の係数。
0.707107      # 裏|1>の係数。
= ジャンケン =
0.57735       # グー   |00>の係数。
0.57735 　　　 # チョキ |01>の係数。
0.57735       # パー   |10>の係数。
      0       # 空き   |11>の係数。
```

# Qubitの観測

qubitは、複数の状態とその状態になりえる確率を掛けた重ね合わせの状態です。qubitを観測することで、qubitは特定のビットに収束します。

例えば、公平なコインの例では「表」`|0>`、「裏」`|1>`の出る割合が$1/2$でした。公平なコインを表現するqubitを観測すると$1/2$の確率で`|0>`、または、`|1>`に収束します。

```math
|coin>_{before} = \frac{|0>}{\sqrt{2}} + \frac{|1>}{\sqrt{2}}
```

```math
|coin>_{measured} = \begin{cases}
 |0>  \qquad (1/2\space probabilities)\\
 |1>  \qquad (1/2\space probabilities)
 \end{cases}
```

ジャンケンの場合は$1/3$づづの確率で「グー」、「チョキ」、「パー」を表現するビットに収束します。

```math
|rock\ paper\ cissors>_{before}\enspace = \frac{|00>}{\sqrt{3}} + \frac{|01>}{\sqrt{3}} + \frac{|10>}{\sqrt{3}} + 0 \cdot |11>
```

```math
|rock\ paper\ cissors>_{measured}\enspace = \begin{cases}
|00> \qquad (1/3\ probabilities)\\
|01> \qquad (1/3\ probabilities)\\
|10> \qquad (1/3\ probabilities)
\end{cases}
```

Quantum++で、コインを表現するqubitを観測してみます。

```cpp
#include <iostream>
#include "qpp.h"

// Qubitsを表現するユーザ定義型リテラル。
template <char... Bits>
qpp::ket operator "" _q(){
    constexpr char bits[sizeof...(Bits) + 1] = {Bits..., '\0'};
    qpp::ket q = qpp::ket::Zero(sizeof...(Bits)*2);
    q(std::stoi(bits, nullptr, 2)) = 1;
    return q;
}

int main()
{
    //                    (      表      ) + (      裏      )
    qpp::ket const coin = 0_q/std::sqrt(2) + 1_q/std::sqrt(2);

    // コインの表か裏か観測する。
    // qpp::gt.Id2は、2-by-2の行列で、列ベクタは観測用のベクタ。1列目が|0>で|1>。
    auto const& [result, ignore0, ignore1] = qpp::measure(coin, qpp::gt.Id2, {0});
    std::cout << "観測結果 = " << (result == 0 ? "表" : "裏") << "\n";

    // 1万回、観測して表と裏の回数を数える。
    int heads_counter = 0, tails_counter = 0;
    for(int i = 0; i < 10000; ++i){
        auto const& [result, ignore0, ignore1] = qpp::measure(coin, qpp::gt.Id2, {0});
        (result == 0? heads_counter : tails_counter)++;
    }

    std::cout << "表の回数 = " << heads_counter << ", 裏の回数 = " << tails_counter << "\n";
}
```

以下が私の手元でのある実行の結果です。この結果は毎回実行するたびに異なりますが、表の回数、裏の回数はともに約5,000回になります。qubitは公平なコインを上手く表現できてるようです。

```shell
観測結果 = 裏
表の回数 = 5047, 裏の回数 = 4953
```

# 最後に

今回はQubitが確率的に表現されることと、観測することによって特定の状態に収束することをQuantam++で見てみました。次回（？）は、qubitへの操作など見てみようと思います。

# 参考

- [Quantum Computation and Quantum Information: 10th Anniversary Edition](http://www.cambridge.org/jp/academic/subjects/physics/quantum-physics-quantum-information-and-quantum-computation/quantum-computation-and-quantum-information-10th-anniversary-edition?format=HB&isbn=9781107002173#FGK0MkTXuETpQyGd.97)
- [qpp@github](https://github.com/vsoftco/qpp)
- [Eigenプロジェクト・ページ](http://eigen.tuxfamily.org/index.php?title=Main_Page)
- [Microsoft Quantum](https://www.microsoft.com/en-us/quantum/)
- [Google OpenFermion](https://github.com/quantumlib/OpenFermion)

# ビルド・インストラクション

サンプル・コードはclang 5.0でビルドしています。qppをgithubからクローンして、Eigenをプロジェクト・ページからダウンロード。qppのディレクトリと同レベルにEigenのディレクトリを配置します。

以下のコマンドでqppがビルドできます。サンプルコードをビルドするには、qppのCMakeLists.txtを編集してサンプルコードをビルド対象に含めるか。

```shell
CC=clang CXX=clang++ cmake -DCMAKE_CXX_FLAGS="-std=c++17 -stdlib=libc++" -DWITH_OPENMP=OFF -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ~/src/cpp/qpp/ && cmake --build ./
```
