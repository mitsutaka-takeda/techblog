# エラー・コードと型消去（Type Erasure)

こんにちは、id:mitsutaka-takadaです。

C++でエラー通知というとエラー・コードや例外が通常の手段かと思います。エラー・コードは戻り値でenumを返すことでエラーを通知します。
例外と比較してenumオブジェクトを返すのみで、とても軽量な通知手段です。

今回の記事では複数のエラー・コードを型消去によってまとめて扱う方法を書きたいと思います。

# ゴール

例えば、ネットワーク・リクエストとデータベース・アクセスをしている関数があるとします。それぞれ自身のドメインに関するエラーをエラー・コード(`NetworkError`&`DatabaseError`)で通知してくるとき、この関数はエラー・コード(`SomeError`)として何を返せばよいでしょうか？

```cpp
#include <string>
#include <experimental/string_view>

// ネットワーク関連のエラー・コード。
enum class NetworkError{
    NoError,
    SomeNetworkError
};

// データベース関連のエラー・コード。
enum class DatabaseError{
    NoError,
    SomeDatabaseError
};

std::pair<std::string, DatabaseError>
getAccountIdFromDatabase(std::string_view userId){
    return {"", DatabaseError::SomeDatabaseError};
}

std::pair<int, NetworkError>
getAccountBalanceFromNetwork(std::string_view accountId){
    return {-1, NetworkError::SomeNetworkError};
}

// エラー情報。ネットワーク＆データベースの両方のエラー情報を返したい。
struct SomeError{};

std::pair<int, SomeError>
getUserBalance(std::string_view userId){

    if(auto [accountId, databaseError] = getAccountIdFromDatabase(userId);
       databaseError == DatabaseError::SomeDatabaseError){
        // データベース関連のエラー情報を返したい
        return {};
    }
    else if(auto [balance, networkError] = getAccountBalanceFromNetwork(accountId);               networkError == NetworkError::SomeNetworkError){
        // ネットワーク関連のエラー情報を返したい
        return {};
    }
    else {
        return {balance, {}};
    }
}
```

# <system_error>

エラーコードの和集合、Variantを利用する（下記、おまけ参照）など、いくつか考えられると思いますが、
今回はC++11で導入された`<system_error>`を利用した方式を紹介したいと思います。

`SomeError`に`NetworkError`や`DatabaseError`が代入でき（型消去）、`SomeError`からその情報を取得するのが目標です。

`<system_error>`では、複数ドメインのエラーコードを１つのenumとして統一的に扱うため`std::error_code`を用意しています。`std::error_code`はエラーコード（整数値）とカテゴリ(ドメインを表す`std::error_category`オブジェクト)の対です。エラーコードは整数値で複数ドメインをまたいで一意性が保証されていません。例えば、`NetworkError::SomeError`と`DatabaseError::SomeError`に同じ１という値が割り当てられているかもしれません。そのため単純に整数値を比較するだけでは`NetworkError::SomeError`と`DatabaseError::SomeError`を識別できなくなります。これを防ぐためにカテゴリを使用します。

`std::error_code`を使用するには、以下の３ステップが必要です。

1. エラーコードを`std::error_code`で使用できるように登録する。(`std::is_error_code_enum`の特殊化）
2. ドメイン用のカテゴリを定義する。(`std::error_category`の派生クラスの定義)
3. エラーコードとカテゴリの紐づけを行う。(`make_error_code`オーバーロードの定義)

実際に`std::error_code`を使用したコードを見てみましょう。

```cpp
#include <iostream>
#include <string>
#include <experimental/string_view>
#include <system_error>

enum class NetworkError{
    NoError,
    SomeNetworkError
};

enum class DatabaseError{
    NoError,
    SomeDatabaseError
};

// 1. エラーコードをstd::error_codeで利用できるように登録。
namespace std{
    template<>
    struct std::is_error_code_enum<NetworkError> : std::true_type{};

    template<>
    struct std::is_error_code_enum<DatabaseError> : std::true_type{};
}

// 2. ドメイン用カテゴリの定義。
struct NetworkErrorCategory : std::error_category{
    const char* name() const noexcept override{
        return "NetworkErrorCategory";
    }

    std::string message(int ev) const override{
        switch(static_cast<NetworkError>(ev)){
            case NetworkError::SomeNetworkError:
                return "some network error occured";
            case NetworkError::NoError:
                return "no error";
        }
    }
};

struct DatabaseErrorCategory : std::error_category{
    const char* name() const noexcept override{
        return "DatabaseErrorCategory";
    }

    std::string message(int ev) const override{
        switch(static_cast<NetworkError>(ev)){
            case NetworkError::SomeNetworkError:
                return "some database error occured";
            case NetworkError::NoError:
                return "no error";
        }
    }
};

// カテゴリオブジェクトの比較にアドレス比較を用いるため、
// カテゴリオブジェクトはシングルトンでなければいけません！
NetworkErrorCategory const networkErrorCategoryInstance;
DatabaseErrorCategory const databaseErrorCategoryInstance;

// 3. エラーコードとカテゴリの紐づけ。
inline
std::error_code make_error_code(NetworkError ne){
    return {static_cast<std::underlying_type_t<NetworkError>>(ne), networkErrorCategoryInstance};
}

inline
std::error_code make_error_code(DatabaseError de){
    return {static_cast<std::underlying_type_t<DatabaseError>>(de), databaseErrorCategoryInstance};
}

std::pair<std::string, DatabaseError>
getAccountIdFromDatabase(std::string_view userId){
    return {"", DatabaseError::SomeDatabaseError};
}

std::pair<int, NetworkError>
getAccountBalanceFromNetwork(std::string_view accountId){
    return {-1, NetworkError::SomeNetworkError};
}

std::pair<int, std::error_code>
getUserBalance(std::string_view userId){
    if(auto [accountId, databaseError] = getAccountIdFromDatabase(userId);
        databaseError == DatabaseError::SomeDatabaseError){
        return {-1, databaseError};
    }
    else if(auto [balance, networkError] = getAccountBalanceFromNetwork(accountId);
            networkError == NetworkError::SomeNetworkError) {
        return {-1, networkError};
    }
    else {
        // Successを表現するにはデフォルト・コンストラクタを使用する。
        return {balance, std::error_code{}};
    }
}

int main(){

    if(auto const [balance, error] = getUserBalance("mitsutaka-takeda");
       !error // エラーがあるときは、std::error_codeオブジェクトがtrueになる。
       ){
        // 成功！
        std::cout << "my balance is " << balance << std::endl;
    }
    else{
        // 失敗！
        if(error == NetworkError::SomeNetworkError){
           // handle network error!
        }
        else if(error == DatabaseError::SomeDatabaseError){
           // handle database error!
        }
    }
}
```

まず、getUserBalanceで複数ドメインのエラーを`std::error_code`として統一できていることに注目してください。各ドメインのエラーコード`NetworkError`や`DatabaseError`から`std::error_code`への変換は暗黙的に行われます。

またmain関数のエラーハンドリングで、`std::error_code`からエラー情報を取得する際、各ドメインのエラーコード(`NetworkError::SomeNetworkError`や`DatabaseError::SomeDatabaseError`)と直接比較しています。

`std::error_code`自体はポリモーフィズムも利用せず整数値とオブジェクトへの参照の対で軽量な構造体であり、既存のエラーコードに非侵入的に使用できるため色々な場面で活躍できます。またC++の標準ライブラリでも使用されており統一されたエラーハンドリングを行うための基礎になります。

他にも複数のエラーコードをグルーピングする`std::error_condition`など応用もあるので興味がある方は参考のリンクを見てください。

# 参考

- [Your own error code@Andrzej's C++ blog](https://akrzemi1.wordpress.com/2017/07/12/your-own-error-code/)
- [Your own error condition@Andrzej's C++ blog](https://akrzemi1.wordpress.com/2017/08/12/your-own-error-condition/)
- [Using error codes effectively@Andrzej's C++ blog](https://akrzemi1.wordpress.com/2017/09/04/using-error-codes-effectively/)
- [C++11's Quiet Little Gen@C++Now 2017 by Charles Bay](https://www.google.co.jp/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&cad=rja&uact=8&ved=0ahUKEwil3v-z4o_WAhVGKlAKHSRrDjsQtwIILDAB&url=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3Dw7ZVbw2X-tE&usg=AFQjCNFhqDd3rnPXwTAqvYiBlbnYe2kJ5g)

# おまけ
## エラーコードの和集合

`SomeError`を`NetworkError`と`DatabaseError`の値の和として定義することで、２つのエラー情報を持つエラーコードを返すことができます。`SomeError`に`NetworkError`と`DatabaseError`のコードに対応するコードをすべて追加して、`NetworkError`/`DatabaseError`から`SomeError`への変換処理`fromNetworkError`/`fromDatabaseError`を書きます。

`std::error_code`と比較すると、各ドメインのエラーコードへの修正が`SomeError`など他の箇所にも影響を与えます。

```cpp
#include <string>
#include <experimental/string_view>

enum class NetworkError{
    NoError,
    SomeNetworkError
};

enum class DatabaseError{
    NoError,
    SomeDatabaseError
};

// ネットワークとデータベースのエラーコードの和集合。
enum class SomeError{
    NoError,
    SomeNetworkError,
    SomeDatabaseError
};

SomeError
fromNetworkError(NetworkError networkError){
    // NetworkErrorからSomeErrorへの変換。
    return networkError == NetworkError::NoError ? SomeError::NoError : SomeError::SomeNetworkError;
}

SomeError
fromDatabaseError(DatabaseError databaseError){
    // DatabaseErrorからSomeErrorへの変換。
    return databaseError == DatabaseError::NoError ? SomeError::NoError : SomeError::SomeDatabaseError;
}

std::pair<std::string, DatabaseError>
getAccountIdFromDatabase(std::string_view userId){
    return {"", DatabaseError::SomeDatabaseError};
}

std::pair<int, NetworkError>
getAccountBalanceFromNetwork(std::string_view accountId){
    return {-1, NetworkError::SomeNetworkError};
}

std::pair<int, SomeError>
getUserBalance(std::string_view userId){
    if(auto [accountId, databaseError] = getAccountIdFromDatabase(userId);
       databaseError == DatabaseError::SomeDatabaseError){
        // DatabaseErrorをSomeErrorに変換して返す。
        return {-1, fromDatabaseError(databaseError)};
    }
    else if(auto [balance, networkError] = getAccountBalanceFromNetwork(accountId);
            networkError == NetworkError::SomeNetworkError) {
        // NetworkErrorをSomeErrorに変換して返す。
        return {-1, fromNetworkError(networkError)};
    }
    else {
        return {balance, SomeError::NoError};
    }
}
```

## C++17 `std::variant`

`SomeError`を`NetworkError`/`DatabaseError`のC++17で導入されるvariantとして定義する方法です。和集合と比べて、対応する値の定義や変換処理は不要になります。

`std::error_code`を使用した方法と比較すると、SomeErrorの型を事前に決定しておかなければいけません。
例えば、getUserBalanceが他ドメイン（ファイルシステム）のエラーを追加で扱わなければいけないとき、
`SomeError`の定義を`std::variant<NoError, NetworkError, DatabaseError, FileSystemError>`に変更しなければいけません。`std::error_code`では、型情報は消去されているので、そのような修正入りません。

```cpp
#include <string>
#include <experimental/string_view>
#include <variant>

enum class NetworkError{
    NoError,
    SomeNetworkError
};

enum class DatabaseError{
    NoError,
    SomeDatabaseError
};

// ネットワークとデータベースのエラーコードのVariant。
struct NoError{};
using SomeError = std::variant<NoError, NetworkError, DatabaseError>;

std::pair<std::string, DatabaseError>
getAccountIdFromDatabase(std::string_view userId){
    return {"", DatabaseError::SomeDatabaseError};
}

std::pair<int, NetworkError>
getAccountBalanceFromNetwork(std::string_view accountId){
    return {-1, NetworkError::SomeNetworkError};
}

std::pair<int, SomeError>
getUserBalance(std::string_view userId){
    if(auto [accountId, databaseError] = getAccountIdFromDatabase(userId);
        databaseError == DatabaseError::SomeDatabaseError){
        // DatabaseErrorをSomeErrorに変換して返す。
        return {-1, databaseError};
    }
    else if(auto [balance, networkError] = getAccountBalanceFromNetwork(accountId);
            networkError == NetworkError::SomeNetworkError) {
        // NetworkErrorをSomeErrorに変換して返す。
        return {-1, networkError};
    }
    else {
        return {balance, NoError{}};
    }
}
```
