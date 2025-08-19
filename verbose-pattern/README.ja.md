
# Verbose Pattern in Python

## 初めに
`Verbose Pattern`という用語は、この構造を説明するために本稿で便宜的に用いている非公式な名称です。これは、正式な既存のデザインパターンには該当しません。

また、以下のデザインパターン・考え方に関連があります。
- Factory Pattern: オブジェクトの生成を専用の関数に委ねる
- Interface Segregation Principle: 役割ごとにインターフェースを分離する
- Facade Pattern: 複雑な内部構造を簡潔な外部インターフェースで隠蔽する
- Strategy Pattern: 役割ベースでの機能分離

ファクトリ関数を2階層に分けて定義し、処理単位別の段階的な動作確認やテストをしやすくすることを目的として定義したパターンです。
長寿命、低数量のファンダメンタルな部分を担う実体への採用を想定しています。

## 特徴

- 外部向けインターフェースと開発、テスト段階での内部向けインターフェースを明確に分けることができる。
- 状態変数の定義は終わっているが、`意味のある初期化がされていない状態`を表現することができるため、初期化に関する段階的なテストを実施しやすい。
- ロール別にインターフェースが定義され、それぞれを検証できるため、テストの粒度が設定しやすい。
- _Kernelの導入により、同じ方法(または、類似した方法)を用いた外部ファクトリ関数の使用時にテストと内部関数が同条件でそれらを使用できる。
- _DelegatingBridgeの導入により、ファクトリ関数から別のファクトリ関数への派生を構造化できる。
- 必要なインターフェースのみを採用することによりコード量が複雑さに相関する。


## 実装の方法の説明

ファクトリ関数を2階層に分けて実装しています。一階層目の_create_interface_roleはロールとして構造化された複数のインターフェース(_Constantと_Stateは定数、内部状態のゲッター)を返します。  
テストなどはこの各インターフェースに対して行うことができます。

### 実装の構造(疑似コード):

```python
from abc import ABC, abstractmethod
from typing import Protocol

# TOC = 'Table of content'

class Interface(ABC):
    # 外部公開用のインターフェース。型チェックを行えるようにABCで定義。
    __slots__ = ()

class _ConstantTOC(Protocol):
    # ファクトリ関数内で使用する定数群を定義したインターフェース。
    ...

class _StateTOC(Protocol):
    # 可変内部状態にアクセスするためのインターフェース。
    ...

class _KernelTOC(Protocol):
    # 外部のファクトリ関数へのアクセスを行うインターフェース。
    # Coreとテストが同条件で外部ファクトリ関数を呼び出せるように抽象化を行う。
    # これがあることによりテストが外部ファクトリ関数のロールにアクセスすることができる。
    ...

class _CoreTOC(Protocol):
    # 内部でのみ使用する内部関数群を定義したインターフェース。
    ...

class _DelegatingBridgeTOC(Protocol):
    # このファクトリ関数の機能・状態に関連のある実体を定義するために
    # それらを委譲するためのインターフェース
    # 委譲内容の分類によって複数定義することができる。
    ...

class _RoleTOC(Protocol):
    # インターフェースの内部構造へのインデックス
    constant: _ConstantTOC
    state: _StateTOC
    kernel: _KernelTOC
    core: _CoreTOC
    bridge: _DelegatingBridge
    interface: Interface

def _create_interface_role() -> _RoleTOC:
    # テスト、拡張その他、開発時において開発者が使用するための内部構造のファクトリ関数
    class _Constant(_ConstantTOC):
        __slots__ = ()
    constant = _Constant()

    class _State(_StateTOC):
        __slots__ = ()
    state = _State()

    class _Kernel(_KernelTOC):
        __slots__ = ()
    kernel = _Kernel()

    class _Core(_CoreTOC):
        __slots__ = ()
    core = _Core()

    class _DelegatingBridge(_DelegatingBridgeTOC):
        __slots__ = ()
    bridge = _DelegatingBridge()

    class _Interface(Interface):
        __slots__ = ()
    interface = _Interface()

    class _Role(_RoleTOC):
        __slots__ = ()
        constant: _ConstantTOC
        state: _StateTOC
        kernel: _KernelTOC
        core: _CoreTOC
        bridge: _DelegatingBridgeTOC
        interface: Interface

    return _Role(
        constant = constant,
        state = state,
        kernel = kernel,
        core = core,
        bridge = bridge
        interface = interface
    )

def create_interface() -> Interface:
    # 外部公開用インターフェースを作成するファクトリ関数
    return _create_interface_role().interface

```

### 補足
- 疑似コード中でインターフェースの実装のすべてに__slots__を定義していますが、これは任意です。(インターフェースに不意な状態が注入されるのを防ぐためにしていますが、これは好みで、完全な任意です。)
- 上記に関連することですが、すべてのインターフェースはクラスでよく、インスタンス化する必要は厳密に言ってありません。ただ、クラスをそのまま用いると、__slots__の設定にメタクラスが必要だったり、@classmethodを用いなければならなかったりするので、簡単のためにすべてインスタンス化しています。これは実装の仕方の問題で、インターフェースを表現できるならどちらでも選択することができると思います。冒頭でも書いた通り想定はしていませんが、軽量、多数が見込まれる実体にこのパターンを採用する際はすべてのインターフェースをクラスにしてインスタンスの生成の必要を無くすのも一考の価値があるかもしれません。
- 全体的に冗長な記述になっていますが。段階的なテストと、静的な型チェックを失わないためにこのような構造になっています。
- すべてのインターフェースはABCを用いてもいいのですが、デコレーションが面倒なため内部向けインターフェースにはProtocolを使用しています。これらの選択は開発者の好みや目的で任意に行うことができます。


