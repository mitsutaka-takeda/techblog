この記事は[Altplus Advent Calendar 2017](https://qiita.com/advent-calendar/2017/altplus)の6日目のエントリです。

こんにちは、職業Scalalian、C++愛好家の竹田です。

識別子は整数型で表現すること多いですが、整数型などをそのまま使用してしまうと、”順序付け可能である”や”配列のインデックスで使用できる”などの整数型の性質を前提にコーディングしてしまい危険です。

例えば、”順番通り”に並んでいて、ID番の要素にそのIDと関連付けられているデータを使用してコーディングをしてしまうと、様々なバグの温床になります。

```cpp
#include <vector>
#include <iostream>

using ID = int;
using Probability = double;
using Image = int;

static std::vector<Probability> probabilityForEachID = {0.1, 0.2, 0.3, 0.4};
static std::vector<Image> imageForEachID = {10, 20, 30, 40};

Probability badProbability(ID id){
    return probabilityForEachID[id];
}

Image badImage(ID id) {
    return imageForEachID[id];
}

void displayImageAndProbability(Probability const& p, Image const& i){
    std::cout << "probability = " << p << " , image = " << i << std::endl;
}

int main(){
    displayImageAndProbability(badProbability(0), badImage(0));
}
```

このコードでは、`probabilityForEachID`と`imageForEachID`の、i番目の要素がID iに関連するデータでなければいけないという暗黙のルールを守っている間は、想定通りに動作します。しかし、データの追加やコード中の操作で`probabilityForEachID`と`imageForEachID`の要素の並び順が変わってしまうと、インデックスとIDが対応しているという暗黙のルールが破られてしまい、IDに関連するデータが正確に引けなくなってしまいます。

# 識別子

問題の原点は、識別子に必要以上の性質を与えてしまったことにあります。Wikipediaで識別子の定義を引くと「識別子（しきべつし、英: identifier）とは、ある実体の集合の中で、特定の元を他の元から曖昧さ無く区別することを可能とする、その実体に関連する属性の集合のこと」([By Wikipedia](https://ja.wikipedia.org/wiki/%E8%AD%98%E5%88%A5%E5%AD%90))とあります。

識別子とは、集まりの中である個を他の個と区別するための情報です。"区別"をC++で表現するには、等価演算子が相応しいので、等価演算子をサポートして識別子を表現するクラスを導入して、どのようにコードが変わるか見てみます。

```cpp
#include <unordered_map>
#include <iostream>

struct ID{
    int id;
};

bool operator==(ID lhs, ID rhs) {
    return lhs.id == rhs.id;
}

bool operator!=(ID lhs, ID rhs) {
    return !(lhs == rhs);
}

namespace std {
    // IDをunordered_mapで使用するための特殊化。
    template<>
    struct hash<ID>{
        size_t operator()(ID const& x) const
        {
            return hash<int>{}(x.id);
        }
    };
}

using Probability = double;
using Image = int;

static std::unordered_map<ID, Probability> probabilityForEachID 
= {{ID{0}, 0.1}, {ID{2}, 0.3}, {ID{1}, 0.2}, {ID{3},0.4}};
static std::unordered_map<ID, Image> imageForEachID 
= {{ID{1}, 20}, {ID{2}, 30}, {ID{3}, 40}, {ID{0}, 10}};

Probability betterProbability(ID id){
    return probabilityForEachID.at(id);
}

Image betterImage(ID id) {
    return imageForEachID.at(id);
}

void displayImageAndProbability(Probability const& p, Image const& i){
    std::cout << "probability = " << p << " , image = " << i << std::endl;
}

int main(){
    displayImageAndProbability(betterProbability(ID{0}), betterImage(ID{0}));
}
```

このコードでは、IDの同値関係(`operator==`)とHashを利用して、`unordered_map`のキーにIDを利用しています。`probabilityForEachID`と`imageForEachID`のデータの順番をシャッフルしていますが、意図通りに動作します。

`unordered_map`は、Hashマップで実装されています。メモリ使用量やメモリ・アクセス速度が気になる場合は、以下のように`std::vector<std::pair<ID, Data>>`を使用することができます。ただしこの方法では、識別子の一意性などが保証できないため、同じIDに対するデータが複数ある状況になりえるので、データ用のコンテナを操作するところでは、事後条件のチェックやユニットテストを通じて、一意性を保証するようにしましょう。

またBoostが使える状況では、`std::vector<std::pair>`のような性質で`map`のIFを提供している`flat_map`もあるので、`flat_map`の使用も検討しましょう。

```cpp
#include <vector>
#include <iostream>
#include <algorithm>

struct ID{
    int id;
};

bool operator==(ID lhs, ID rhs) {
    return lhs.id == rhs.id;
}

bool operator!=(ID lhs, ID rhs) {
    return !(lhs == rhs);
}

using Probability = double;
using Image = int;

static std::vector<std::pair<ID, Probability>> probabilityForEachID 
= {{ID{0}, 0.1}, {ID{2}, 0.3}, {ID{1}, 0.2}, {ID{3},0.4}};
static std::vector<std::pair<ID, Image>> imageForEachID 
= {{ID{1}, 20}, {ID{2}, 30}, {ID{3}, 40}, {ID{0}, 10}};

Probability betterProbability(ID id){
    auto const found = std::find_if(probabilityForEachID.begin(), probabilityForEachID.end(),
                [&](auto const& p){ return p.first == id; });
    if(found == probabilityForEachID.end()){
        throw std::out_of_range("Not found.");
    }
    return found->second;
}

Image betterImage(ID id) {
    auto const found = std::find_if(imageForEachID.begin(), imageForEachID.end(),
                [&](auto const& p){ return p.first == id; });
    if(found == imageForEachID.end()){
        throw std::out_of_range("Not found.");
    }
    return found->second;
}

void displayImageAndProbability(Probability const& p, Image const& i){
    std::cout << "probability = " << p << " , image = " << i << std::endl;
}

int main(){
    displayImageAndProbability(betterProbability(ID{0}), betterImage(ID{0}));
}
```

# 最後に

あるコンセプト（識別子）の表現に必要以上の性質（int型で表現して、順序付けやインデックスとしての使用できる性質）を与えてしまうと、バグの温床になることを見てきました。

自分で型を定義して適切な性質を与えることで、安全で見通し良いコードを書くことができます。
