
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
- _*Bridgeの導入により、ファクトリ関数から別のファクトリ関数への派生を構造化できる。
- 必要なインターフェースのみを採用することによりコード量が複雑さに相関する。

## 注意

- このパターンは通常のクラスインスタンスに比べて何か新しいことができるようになるわけではありません。  
- また、このパターンでやろうとしていることは通常のクラスインスタンスを用いても実現可能です。
- ただクロージャの一般的な特性として、外部向けインターフェースが参照している内部状態へのアクセスが容易ではなくなる点は挙げられます。これにより、不意な内部状態へのアクセスの危険性は通常のクラスインスタンスに比べて低くなります。が、これはこのパターンに限ったことではなく、ファクトリ関数を用いたインターフェースの提供全般言えることです。
- このパターンが重要視しているのは、冗長な記述になったとしても、実装の構造化を優先するという点です。

## 実装の方法の説明

ファクトリ関数を2階層に分けて実装しています。一階層目の_create_*_allはロールとして構造化された複数のインターフェース(_*Constantと_*Stateは定数、内部状態のゲッター)を返します。  
テストなどはこの各インターフェースに対して行うことができます。

### 実装の構造(疑似コード):
```python
from __future__ import annotations

from abc import ABC, abstractmethod
from enum import Enum, auto
from typing import Protocol
from dataclasses import dataclass

# * = インターフェース名

class *(ABC):
    __slots__ = ()
    # 外部公開用のインターフェース。型チェックを行えるようにABCで定義。
    @abstractmethod
    def greet(self) -> None:
        ...

class _*Constant(Protocol):
    # ファクトリ関数内で使用する定数群を定義したインターフェース
    GREET_IN_MORNING: str
    GREET_IN_AFTERNOON: str


class _*State(Protocol):
    # 可変内部状態にアクセスするためのインターフェース
    init_state: str
    meridiem: Meridiem


class _*Core(Protocol):
    # 内部でのみ使用する内部関数群を定義したインターフェース。
    # ただし、initialize()は初期化を行うため外側から呼び出される
    def init_process_a(self) -> None:
        ...
    def init_process_b(self) -> None:
        ...
    def init_process_c(self) -> None:
        ...
    def initialize(self) -> None:
        ...

# _*+Bridge(Protocol): # + = 委譲内容に応じたインターフェース名
# 関連のある実体へ機能の一部を渡すためのインターフェース
# 内容(権限)によって複数に分けることで派生する実体の分類を行う。

class _*ReaderBridge(Protocol):
    # Readerは初期化状態と現在のMeridiemを参照する
    def is_initialized(self) -> bool:
        ...
    def current_meridiem(self) -> Meridiem:
        ...

class _*UpdaterBridge(_*ReaderBridge, Protocol):
    # Updaterは午前と午後を切り替える
    # Updaterは同時に_*ReaderBrdigeの機能も使用できる(_*ReaderBridgeの継承)
    def toggle_meridiem(self) -> None:
        ...

class Meridiem(Enum):
    AM = auto()
    NOON = auto()
    PM = auto()
    MIDNIGHT = auto()

def _create_*_all() -> tuple[*, _*Constant, _*State, _*Core, _*Bridge]: # 1
    # この疑似コード中では_create_*_all()はすべてのインターフェースを返していますが、
    # _*Constant、_*State、_*Core, _*Bridgeは存在しない場合があります。*は必ず存在します。
    # 特に_*Coreが存在しないのはファクトリ関数への引数のみで初期化の完了と同義になる場合です。

    @dataclass(slots = True) # 2, 3
    class _Constant(_*Constant):
        GREET_IN_MORNING: str = "good morning!!"
        GREET_IN_AFTERNOON: str = "good afternoon!!"
    constant = _Constant()
    
    @dataclass(slots = True)
    class _State(_*State):
        init_state: str = "not initialized"
        meridiem: Meridiem = Meridiem.NOON
    state = _State() # 4

    class _Core(_*Core):
        __slots__ = ()
        def init_process_a(self) -> None:
            state.init_state = "start initialization"
        def init_process_b(self) -> None:
            state.init_state = "initialization is still in progress"
        def init_process_c(self) -> None:
            state.init_state = "initialized"
            state.meridiem = Meridiem.AM
        def initialize(self) -> None:
            self.init_process_a()
            self.init_process_b()
            self.init_process_c()
    core = _Core()

    class _ReaderBridge(_*ReaderBridge):
        __slots__ = ()
        def is_initialized(self) -> bool:
            return state.init_state == "initialized"
        def current_meridiem(self) -> Meridiem:
            return state.meridiem
    reader_bridge = _ReaderBridge()

    class _UpdaterBridge(_*UpdaterBridge):
        __slots__ = ()
        def is_initialized(self) -> bool: # 5
            return reader_bridge.is_initialized()
        def current_meridiem(self) -> Meridiem:
            return reader_bridge.current_meridiem()
        def toggle_meridiem(self) -> None:
            meridiem = state.meridiem
            if meridiem is Meridiem.AM:
                state.meridiem = Meridiem.PM
            elif meridiem is Meridiem.PM:
                state.meridiem = Meridiem.AM
            else:
                raise RuntimeError("Bug")
    updater_bridge = _UpdaterBridge()

    class _Interface(*):
        __slots__ = ()
        def greet(self):
            meridiem = state.meridiem
            if meridiem is Meridiem.AM:
                print(constant.GREET_IN_MORNING)
            elif meridiem is Meridiem.PM:
                print(constant.GREET_IN_AFTERNOON)
            else:
                print("... (If you see this message, please contact support.)")
    interface = _Interface()

    return (interface, constant, state, core, reader_bridge, updater_bridge)

def create_*() -> *:
    roles = _create_*_all()
    core = roles[3]
    core.initialize()
    interface = roles[0]
    return interface

```

1. ファクトリ関数名の*(インターフェース名)はスネークケースで記されます。
2.  _Constantは実装中の定数へのアクセスを他のインターフェースと一貫性を保つためインスタンス化していますが必ず必要なことではありません。
3. また、_Constantはfrozenを設定していないdataclassとして定義しています。これは定数群をテスト時に変更する必要がある場合に有効です。その必要がない場合、frozenを設定することで実装における不意の書き換えが防止され安全性は向上します。
4. stateは自由に書き換えられるため、テスト時に任意の状態を作ることができます。
5. _*+Bridige(Protocol)は簡単のため継承を用いてインターフェースを定義ています。しかし、実装ではコンポジションを用いて行うこともできます。これも任意であり、継承とコンポジションどちらも選択することができます。

### 補足
- 疑似コード中でインターフェースの実装のすべてに__slots__を定義していますが、これは任意です。(インターフェースに不意な状態が注入されるのを防ぐためにしていますが、これは好みで、完全な任意です。)
- 上記に関連することですが、すべてのインターフェースはクラスでよく、インスタンス化する必要は厳密に言ってありません。ただ、クラスをそのまま用いると、__slots__の設定にメタクラスが必要だったり、@classmethodを用いなければならなかったりするので、簡単のためにすべてインスタンス化しています。これは実装の仕方の問題で、インターフェースを表現できるならどちらでも選択することができると思います。冒頭でも書いた通り想定はしていませんが、軽量、多数が見込まれる実体にこのパターンを採用する際はすべてのインターフェースをクラスにしてインスタンスの生成の必要を無くすのも一考の価値があるかもしれません。
- 全体的に冗長な記述になっていますが。段階的なテストと、静的な型チェックを失わないためにこのような構造になっています。
- すべてのインターフェースはABCを用いてもいいのですが、デコレーションが面倒なため内部向けインターフェースにはProtocolを使用しています。これらの選択は開発者の好みや目的で任意に行うことができます。


