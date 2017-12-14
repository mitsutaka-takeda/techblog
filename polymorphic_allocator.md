Title: Polymorphic Allocator
Tags: C++, C++17
Author: Mitsutaka Takeda

こんにちは、この記事は[Altplus Advent Calnendar 2017](https://qiita.com/advent-calendar/2017/altplus)の17日目のエントリです。

仕事では[Akka Stream](https://doc.akka.io/docs/akka/2.5.4/scala/stream/index.html)の綺麗さに感動しつつ、プライベートのプロジェクトではC++17でコーディング中の竹田です。

# Polymorphic Allocator

先日発行されたC++17ではPolymorphic Allocatorと呼ばれるAllocatorが導入されました。この導入を受けてか、Cppcon 2017ではAllocator関連の発表が多くありました。

# Allocatorの問題点

そこにあるのに誰も気付かず、ないと生きていられない、まるで空気のような存在のAllocatorですが、みんな大好き`std::vector`テンプレートの第2型パラメータの彼です。

```cpp
template<
    class T,
    class Allocator = std::allocator<T> // 今回の主役。
> class vector;
```

Allocatoは型に対してメモリを提供することが役割で、STL(Standard Template Library)をコンピュータのメモリ・モデルから切り離すことを目的として導入されました。[Allocator (C++)@Wikipedia](https://en.wikipedia.org/wiki/Allocator_(C%2B%2B))従来のAllocatorは幾つかの問題点を抱えており、C++11以降、少しづつ改良されてきました。

問題点の1つは、実装の詳細であるはずのAllocatorが型の一部としてあることです。STLではテンプレートのパラメータとしてAllocatorが渡されています。これはAllocatorが型の一部になるということです。STLのコンテナはValue Semanticsを念頭に設計されており、数学のコンセプトを表現していますが、Allocatorという実装の詳細が型に表われると、この対応がくずれてしまいます。例えば、`std::vector`は、列(sequence)を表現しており、数列1, 2, 3は、`std::vector<int>{1, 2, 3}`で表現できます。Allocatorの異なる、`std::vector<int, MyAllocator>{1, 2, 3}`も、std::vector<int, YourAllocator>{1, 2, 3}`も同じ数列1, 2, 3の表現ですが、型が違うため比較することはできません。

その他にも、Allocatorが型の一部であるためカスタムAllocatorを使用したいクライアントのコードまでテンプレート化する必要があるなどの問題もあります。

これらの問題点を解決するために導入されたのが`std::pmr::polymorphic_allocator`と`std::pmr`名前空間したにあるコンテナです。

# std::pmr

まずは、`std::pmr::vector`テンプレートの定義を見てみます。[std::vector](http://en.cppreference.com/w/cpp/container/vector)単純に、`std::vector`のAllocatorに`std::prm::polymporphic_alloctor`を指定したalias templateです。

```cpp
namespace pmr {
    template <class T>
    using vector = std::vector<T, std::pmr::polymorphic_allocator<T>>;
}
```

書きかけ。

# CppCon 2017 アロケータ関連の発表

- [Allocators, the Good Pars by Pablo Halpern](https://www.youtube.com/watch?v=v3dz-AKOVL8)
- [From Security to Performance to GPU Programming - Exploring Modern Allocators by Sergey Zubkov](https://www.youtube.com/watch?v=HdQ4aOZyuHw)
- [How to Write a Custom Allocator by Bob Steagall](https://www.youtube.com/watch?v=kSWfushlvB8)
- [Local ('Arena') Memory Allocators by John Lakos](https://www.youtube.com/watch?v=nZNd5FjSquk)
