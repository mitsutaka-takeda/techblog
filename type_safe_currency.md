# 型安全な通貨型

仕事ではJava&C#を書いてるC++愛好家の竹田です。

今日は、型安全な通貨型の設計について書いてみようと思います。

## ゴール
何かと話題のビットコインですが、ビットコインには通貨の単位としてBTCとsatoshiと呼ばれる単位があります。1BTC = 100,000,000Sathoshi = 100 Million Satoshiという関係になります。日本では昔は銭という単位がありましたが、現在は円だけなのでデノミネーションという言葉は意識しないかもしれません。

今回設計する型安全な通貨では許可していない演算はコンパイル時エラーにしバグの侵入を防ぎつつ、
必要なデノミネーションの変換を自動的に行うことで使い勝手の良い型になることを目指します。

```C++
Satoshi originalPrice{100};
auto premiumPrice = price + 10; // 10の単位がわからないのでコンパイル・エラー。

Btc myWalletBalance{1};
auto recievePayment = myWalletBalance + originalPrice; // 単位の自動変換。
assert(receivePayment == Satoshi{100'000'100});
```

## 設計

通貨は、何単位あるかという量を表現する型(100や1)と、デノミネーション(SatoshiやBtc)を表現する型の組み合わせで表現できそうです。
クラス・テンプレートMonetaryAmountで通貨を表現すると以下のようになります。
型パラメータRepが量を表現する型で、Denomがデノミネーションを表現する型です。

```C++
template<typename Rep, typename Denom>
class MonetaryAmount{

};
```

量を保持できるようにRep型のメンバ変数とコンストラクタを追加します。

```C++
template <typename Rep, typename Denom>
class MonetaryAmount{
    Rep count_;
public:
    template <typename Rep2>
    constexpr MonetaryAmount(Rep2 const& count)
    : count_(count)
    {}
};
```

次に加算演算子を追加します。まずは同じデノミネーションのみの加算を考慮します。
加算演算子をフリー関数で定義するために、量にアクセスするためのCountメンバ関数を定義します。
また、動作確認のために等価演算子も定義します。
クラス・テンプレートに型引数を与えて、ここまでで動作確認をしてみます。単位がついていないプリミティブ型との
加算は意図通りコンパイル・エラーになります。

```C++
#include <cstdint>

template <typename Rep, typename Denom>
class MonetaryAmount{
    Rep count_;
public:
    template <typename Rep2>
    constexpr MonetaryAmount(Rep2 const& count)
    : count_(count)
    {}

    constexpr Rep Count() const { return count_; }
};

template <typename Rep, typename Denom>
MonetaryAmount<Rep, Denom>
constexpr operator+(
    MonetaryAmount<Rep, Denom> const& lhs,
    MonetaryAmount<Rep, Denom> const& rhs
    )
{
    return lhs.Count() + rhs.Count();
}

template <typename Rep, typename Denom>
bool
constexpr operator==(
    MonetaryAmount<Rep, Denom> const& lhs,
    MonetaryAmount<Rep, Denom> const& rhs
    )
{
    return lhs.Count() == rhs.Count();
}

// 動作確認。
using Satoshi = MonetaryAmount<std::int64_t, struct ignored>;

constexpr Satoshi a{1}, b{2};

static_assert(a + b == Satoshi{3});
static_assert(a + 10); // コンパイル・エラー。10の単位が不明。
```

次に異なるデノミネーション間(Btc <-> Satoshi)の変換をサポートしてみます。
デノミネーション間の変換情報には何が必要でしょうか？BtcとSatoshiを変換するには、
1Btcが100,000,000Satoshiという情報があれば良さそうです。Sastoshiのほうが細かい単位なので、
Satoshiを基準に考えると、1Satoshiは1/100,000,000 Btc(１億分の１)になります。
型でこれを表現するには標準ライブラリのクラステンプレートstd::ratioを使用します。

std::ratioは分数を表現する型です。例えば３分の２は`std::ratio<2, 3>`と表現できます。
分数が型で表現されていることに注意してください。`std::ratio<2, 3>`と`std::ratio<1,3>`は異なる型です。

このstd::ratioを利用すると、Btcは `MonetaryAmount<std::int64_t, std::ratio<100'000'000, 1>>` 、
Satoshiは`MonetaryAmount<std::int64_t, std::ratio<1>>`で表現できます。つまり型パラメータDenomは、
1 Countあたり何Satoshiに相当するかの情報を表しています。Satoshiの1 Countは1 Satoshiなので、`Denom = std::ratio<1>`、
Btcの1 Countは100,000,000 Satoshiに相当するので、`Denom = std::ratio<100'000'000>`となります。

異なるデノミネーションを持つ通貨の演算は、より細かいデノミネーションに合わせて量を行うことで実現します。
つまり、1 Btc + 100 Satoshiは、1 BtcをSatoshiに変換して、100,000,000 Satoshi + 100 Satoshi = 100,000,100 Satoshiになります。

BtcとSatoshiは違う型であることに注意します。クラス・テンプレートMonetaryAmountに異なる型を指定して実体化するからです。
そのため、1 Btcから100,000,000 Satoshiへの変換は型の変換が必要です。
この型変換のために標準ライブラリに用意されたクラス・テンプレートstd::common_typeと、異なる通貨型をとるコンストラクタを利用します。

```C++
#include <cstdint>

template <typename Rep, typename Denom>
class MonetaryAmount{
    Rep count_;
public:
    template <typename Rep2>
    constexpr MonetaryAmount(Rep2 const& count)
    : count_(count)
    {}

    template<typename Rep2, typename Denom2,
             // 解説１
             typename = std::enable_if_t<std::ratio_divide<Denom2, Denom>::den == 1> >
    constexpr MonetaryAmount(MonetaryAmount<Rep2, Denom2> const& m)
         : representation(m.Count() * std::ratio_divide<Denom2, Denom>::num)
    {}
};

template<typename Rep1, typename Denom1, typename Rep2, typename Denom2>
// 解説２
std::common_type_t<MonetaryAmount<Rep1, Denom1>,
                   MonetaryAmount<Rep2, Denom2> >
constexpr operator+(
    MonetaryAmount<Rep1, Denom1> const& lhs,
    MonetaryAmount<Rep2, Denom2> const& rhs
    )
{
    using common_t = std::common_type_t<MonetaryAmount<Rep1, Denom1>,
                                        MonetaryAmount<Rep2, Denom2> >;
    return static_cast<common_t>(lhs).Count() + static_cast<common_t>(rhs).Count();
}

template <typename Rep, typename Denom>
bool
constexpr operator==(
    MonetaryAmount<Rep, Denom> const& lhs,
    MonetaryAmount<Rep, Denom> const& rhs
    )
{
    return lhs.Count() == rhs.Count();
}

namespace detail {
    // gcdを実装するためのヘルパ。
    template <typename FloatingPoint,
              typename = std::enable_if_t<std::is_integral_v<FloatingPoint> > >
    constexpr FloatingPoint abs(FloatingPoint x){
        return x < 0 ? -x : x;
    }

    // 下のgcdのヘルパ。
    template <typename M, typename N>
    constexpr std::common_type_t<M, N> gcd(M m, N n){
        return n == 0 ? abs(m) : gcd(n , abs(m) % abs(n));
    }

    // 型レベルで分数の最大公約数(GCD)を計算する関数。
    template<std::intmax_t Num1, std::intmax_t Denom1, std::intmax_t Num2, std::intmax_t Denom2>
    constexpr auto gcd(std::ratio<Num1, Denom1> const& x,
                       std::ratio<Num2, Denom2> const& y){
        return std::ratio<gcd(Num1, Num2), Denom1 * Denom2>{};
    }
} // namespace detail

namespace std {
    // 解説２
    template<typename Rep1, typename Denom1, typename Rep2, typename Denom2>
    struct common_type<MonetaryAmount<Rep1, Denom1>,
                       MonetaryAmount<Rep2, Denom2> >{
        using type = MonetaryAmount<
            std::common_type_t<Rep1, Rep2>, decltype(detail::gcd(Denom1{}, Denom2{}))>;
    };

} // namespace std

// 動作確認。
using Satoshi = MonetaryAmount<std::int64_t, std::ratio<1>>;
using Btc     = MonetaryAmount<std::int64_t, std::ratio<100'000'000, 1>>;

constexpr Satoshi originalPrice{100};
// constexpr auto premiumPrice = originalPrice + 10; // 10の単位がわからないのでコンパイル・エラー。
constexpr Btc myWalletBalance{1};

// デノミネーションの自動変換。
static_assert((myWalletBalance + originalPrice) == Satoshi{100'000'100});
```

### 解説１

SFINAE(Substitution Failure Is Not A Error)を利用して、コンストラクタの利用を制限しています。

異なるデノミネーション間の変換はより大きなデノミネーションから細かなデノミネーションへの変換は誤差なしに行えますが、
逆の変換は切り捨てが発生して誤差が生まれてしまいます。例えば、1 Btcは100'000'000 Satoshiですが、
1 SatoshiはBtcの整数単位では切り捨てられて0 Btcになってしまいます。

この条件を`std::ratio_divide<Denom2, Denom>::den == 1`で表現しています。
std::ratio_divideは分数の割り算を行います。３分の１割る２分の１は`std::ratio_divide<std::ratio<1/3>, std::ratio<1, 2>> == std::ratio<2, 3>`となります。BtcからSatoshiへの変換は`std::ratio_divide<std::ratio<100'000'000, 1>, std::ratio<1>> == std::ratio<100'000'000, 1>`となり変換が許可されますが、SatoshiからBtcへの変換は`std::ratio_divice<std::ratio<1>, std::ratio<100'000'000, 1>> == std::ratio<1, 100'000'000>`となり許可されません。

### 解説２

std::common_typeは２つの型を取って、共通の型を返すクラス・テンプレートです。ユーザ定義型に対して使用する場合は、
std名前空間内で特殊化します。型から型への型レベルの関数です。

型Btc(`MonetaryAmount<std::int64_t, std::ratio<100'000'000, 1>>`)と型Satoshi(`MonetaryAmount<std::int64_t, std::ratio<1>>`)の
共通の型は何になるでしょうか。上記のように異なるデノミネーションの演算は、より細かなデノミネーションに合わせて行うことで誤差なしに行えます。
そのためBtcとSatoshiの場合は共通の型はSatoshiとします。

## 最後に

今回の記事では安全な仮想通貨型の設計を見てきました。一見複雑そうなことをしているように見えますが、ほとんどのロジックは型レベルで行われています。Btc/Satoshiオブジェクトの実態は実は64bitの整数のみです。コンパイラの最適化レベルを上げるとBtc/Satoshiを利用したコードはint64_tを利用したコードとまったく同じになります。コンパイラの最適化処理には脱帽です。記事内のすべてのコードはコンパイル可能になっているので、興味があればCompiler Explorerなどに張り付けていろいろいじってみてください。

この設計はstd::chronoライブラリを参考にしています。個人的にはC++で最も綺麗に設計されたライブラリの１つだと思います。
std::chronoについて詳しく知りたい人は以下の動画がおすすめです。

[CPPCON2016 A \<chorno> Tutorial by Howard Hinnant](https://www.youtube.com/watch?v=P32hvk8b13M)
