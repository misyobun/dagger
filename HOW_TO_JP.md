
### Introduction

アプリケーションにおいて最良なクラス（the BarcodeDecoder, the KoopaPhysicsEngine, and the AudioStreamerといったようなクラス）は自分の専門的なことを行います。こういったクラスは、恐らくBarcodeCameraFinder, DefaultPhysicsEngine, and an HttpStreamer.といったクラスとの依存関係を持ちます。

対照的にアプリケーションにおける最悪なクラス（BarcodeDecoderFactory, the CameraServiceLoader, and the MutableContextWrapper.といったようなクラス）は、冗長な造りで大したことを行わないクラスです。こういったクラスは関心のあるものを無理矢理いっしょくたに繋ぎ合わせる扱いの難しいダクトテープのようなものでしょう。

Daggerはこういったいわゆる「FactoryFactory クラス」の造りを置き換えるものです。
実装者に対しアプリにとって一番関心のあるクラスの開発に専念することを可能にしました。
依存関係を宣言して、どうのようにしてその依存関係を満たすかを明らかにして、アプリを送り出す事ができます。

標準的なjavax.inject (JSR-330) のアノテーションによって、それぞれのクラスはテストを容易に行えます。（テスト用の）FakeCreditCardServiceのためにRpcCreditCardServiceを交換する一連のひな形も不要となります。

依存関係の注入はテストのためのものだけではありません。それは再利用可能で交換可能なモジュールの作成をもまた容易にします。同一のAuthenticationModuleを、開発した全てのアプリで共有することが可能です。
そして、開発時はDevLoggingModuleを、本番ではProdLoggingModuleをシチュエーションごとに（開発時 or 本場）使い分けて、それぞれで正しい振る舞いを行うよう実行することができます。

## Using Dagger

サンプルで用意されたCoffeeMakerアプリでのデモンストレーションです。
サンプルコードはコンパイルして実行することが可能です。
サンプルコード:[https://github.com/square/dagger/tree/master/examples/simple/src/main/java/coffee]


### DECLARING DEPENDENCIES
Daggerはアプリケーションのクラスのインスタンスを作成して、それらの依存関係を満たします。javax.inject.Inject注釈を使って関心のあるコンストラクターやメンバ変数の特定を行います。
Daggaerがクラスのインスタンスを作るために使うコンストラクターを特定するには@Inject注釈を使います。
新しいインスタンスの生成要求がある時には、Daggerは必要なパラメーターの値を
取得し、このコンストラクターを実行します。

```
class Thermosiphon implements Pump {
  private final Heater heater;

  @Inject
  Thermosiphon(Heater heater) {
    this.heater = heater;
  }

  ...
}
```

Daggerはフィールドへ直にインスタンスを注入する事ができます。
この例ではheaterフィールドにHeaterクラスのインスタンスをpump フィールドにPumpクラスのインスタンスを注入しています。

```
class CoffeeMaker {
  @Inject Heater heater;
  @Inject Pump pump;

  ...
}
```
もしフィールドに@Injectがあって、コンストラクタに@Injectが無い場合、引数無しのコンストラクタがあればDaggerはそれを利用してインスタンスを作ります。@Injectが不足しているクラスはDaggerによってそのインスタンスは作成されません。

Daggerはメソッド・インジェクションはサポートしていません。

### SATISFYING DEPENDENCIES
デフォルトでは、Daggerは要求された型のインスタンスを作成することで依存関係を
満たします。あなたがCoffeeMakerクラスを要求すれば, Dagger はnew CoffeeMaker() のコールと注入可能なフィールドの設定を行いCoffeeMaker の1つのインスタンスを作成するでしょう。

しかし@Inject はどういった局面でも機能するわけではありません。

* インターフェースは作成できません
* 外部のクラスには@Inject の注釈がつけられません
* 設定可能なオブジェクトは設定する必要があります

こういった@Injectでは不十分または扱い難いケースでは、依存関係を満たすために@Providesの注釈が指定されたメソッドを使います。

例えば、Heaterクラスが要求される時はprovideHeater()が実行されます。
```
@Provides Heater provideHeater() {
  return new ElectricHeater();
}
```
@Providesの注釈がついたメソッドに、メソッド自体が所有するクラス（上記の場合はElectricHeaterクラス）の依存関係を持たせることが可能です。下記の例では、Pumpの型クラスが要求された場合にThermosiphonのインスタンスが返却されます。
```
@Provides Pump providePump(Thermosiphon pump) {
  return pump;
}
```
@Providesなメソッドは全てmoduleに所属します。
これらのmoduleは@Moduleを持つクラスです。
```
@Module
class DripCoffeeModule {
  @Provides Heater provideHeater() {
    return new ElectricHeater();
  }

  @Provides Pump providePump(Thermosiphon pump) {
    return pump;
  }
}
```
規約により、@Providesメソッドはprovide の接頭辞が、@Moduleはmoduleという接尾辞によってそれぞれ命名されます。

### BUILDING THE GRAPH
オブジェクト・グラフ（オブジェクト及びそれらの関係）に属する@Injectや@Providesといった注釈のついたクラスは依存関係でリンクされています。
下記の要領でObjectGraph.create()をコールすることで、複数のモジュールを内部に保持しているオブジェクト・グラフを得ることができます。

```
ObjectGraph objectGraph = ObjectGraph.create(new DripCoffeeModule());
```
オブジェクトグラフを使うためには、ブートストラップインジェクションが必要です。
これは通常、コマンドラインアプリではmainクラスで、またはAndroidアプリのActivityクラスでの注入が必要とされます。
コーヒーアプリのサンプルではCoffeeAppクラスで依存関係の注入が開始されます。
オブジェクトグラフに注入されたインスタンスの提供を頼みます。

```
class CoffeeApp implements Runnable {
  @Inject CoffeeMaker coffeeMaker;

  @Override public void run() {
    coffeeMaker.brew();
  }

  public static void main(String[] args) {
    ObjectGraph objectGraph = ObjectGraph.create(new DripCoffeeModule());
    CoffeeApp coffeeApp = objectGraph.get(CoffeeApp.class);
    ...
  }
}
```

ここで不足している唯一の事は、注入可能なCoffeeAppクラスがオブジェクトグラフに知られていない事です。私たちは下記のように@Module 注釈内でCoffeeAppを注入される型のクラスとして明示的に登録する必要があります。

```
@Module(
    injects = CoffeeApp.class
)
class DripCoffeeModule {
  ...
}
```

injectsオプションはコンパイル時にオブジェクトグラフの検証を行うことを可能にします。早期の問題検知は、開発をスピードアップさせリファクタリング以外の危険を取り払うでしょう。

さあ、今や、オブジェクトグラフは作成されて、その根幹をなすオブジェクトが（CoffeeApp.classに）注入されたことで、私たちはコーヒーメーカアプリを実行することができます。
```
$ java -cp ... coffee.CoffeeApp
~ ~ ~ heating ~ ~ ~
=> => pumping => =>
 [_]P coffee! [_]P
```

### SINGLETONS

@Provides メソッドか注入できるクラスに@Singletonの注釈をつけます。
オブジェクト・グラフは１つのインスタンスを全てのクライアントに対して
使い回します。
```
@Provides @Singleton Heater provideHeater() {
  return new ElectricHeater();
}
```
注入可能クラス上でも@Singletonは同様に動きます。
そして、そのクラスは複数のThread上で共有されることについて潜在的な考慮を注意させます。
```
@Singleton
class CoffeeMaker {
  ...
}
```
### LAZY INJECTIONS

遅延して生成されるオブジェクトが時に必要です。
何かしらの型のオブジェクト Tのバインディングにたいして、Lazy<T>'s get() がコールされるまで
インスタンスの生成を先延ばしするLazy<T>型を作ることができます。
もし T がシングルトンの場合、オブジェクト・クラスの中では全て同一のインスタンスとなります。
そうでない場合は、各注入部位は独自のLazy<T>インスタンスを取得するでしょう。
ただそれでも、各Lazy<T> インスタンスに対するそれ以降の呼び出しは同一のTインスタンスが返却されます。

```
class GridingCoffeeMaker {
  @Inject Lazy<Grinder> lazyGrinder;

  public void brew() {
    while (needsGrinding()) {
      // Grinder created once on first call to .get() and cached.
      lazyGrinder.get().grind();
    }
  }
}
```
### PROVIDER INJECTIONS

単一のインスタンスを注入する代わりに、複数のインスタンスが返却される場合が必要な時があるでしょう。Factories, Builders, etc.といった幾つかの方法がありますが、一つの方法として T の代わりにProvider<T> を注入することがあげられます。
Provider<T> は.get()がコールされるたびに新しいTのインスタンスを生成します。
```
class BigCoffeeMaker {
  @Inject Provider<Filter> filterProvider;

  public void brew(int numberOfPots) {
    ...
    for (int p = 0; p < numberOfPots; p++) {
      maker.addFilter(filterProvider.get()); //new filter every time.
      maker.addCoffee(...);
      maker.percolate();
      ...
    }
  }
}
```
### QUALIFIERS

型だけでは依存関係を特定するのに困難な場合があります。
例えば、洗練されたコーヒーメーカーアプリの場合、水とホットプレート用に
異なるヒーターが欲しいかもしれません。

こういったケースでは、修飾注釈を追加します。
これはその注釈自体が@Qualifier注釈を持つ任意の注釈です。
下記は@Named注釈の宣言です。修飾注釈自体はjavax.inject:に含まれています。
```
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Named {
  String value() default "";
}
```
独自の修飾注釈を作ることもできますし、この@Namedを使うこともできます。
フィールドまたは関心のあるパラメータに注釈を付けることにより、修飾子を適用します。
型と修飾注釈を両方使うことで依存関係を特定します。

```
class ExpensiveCoffeeMaker {
  @Inject @Named("water") Heater waterHeater;
  @Inject @Named("hot plate") Heater hotPlateHeater;
  ...
}
```
@ Providesメソッドに対応した注釈にそって値を与えます
```
@Provides @Named("hot plate") Heater provideHotPlateHeater() {
  return new ElectricHeater(70);
}

@Provides @Named("water") Heater provideWaterHeater() {
  return new ElectricHeater(93);
}
```
依存関係は複数の修飾修飾しを持たないかもしれません。

### STATIC INJECTION
この機能は殆ど使われないべきである。何故ならば静的な依存関係は試験や
再利用するのが難しいからだ。
Daggerは静的なフィールドにも注入ができる。
@Inject注釈付きで宣言された静的なフィールドを宣言するクラスは
モジュールの注釈のなかでstaticInjectionsとして記載されなくてはならない。

```
@Module(
    staticInjections = LegacyCoffeeUtils.class
)
class LegacyModule {
}
```
これらの静的なフィールドに値を注入するためにObjectGraph.injectStatics() を使う。

```
ObjectGraph objectGraph = ObjectGraph.create(new LegacyModule());
objectGraph.injectStatics();
```
### COMPILE-TIME VALIDATION
Daggerにはモジュールと指定した注入を検証する annotation processor が含まれています。
このprocessorは静的で、データの結合が無効か不完全だった場合はコンパイルエラーを起こす仕様です。
例えば、このモジュールはExecutorのデータの受け渡しで欠落があります。

```
@Module
class DripCoffeeModule {
  @Provides Heater provideHeater(Executor executor) {
    return new CpuHeater(executor);
  }
}
```
コンパイルした場合、javacはこの欠落を拒絶します。
```
[ERROR] COMPILATION ERROR :
[ERROR] error: No binding for java.util.concurrent.Executor
               required by provideHeater(java.util.concurrent.Executor)
```
問題を正すにはExecutor用に@Providesの注釈付きメソッドを追加するか、このモジュールを不完全なものであると目印をつけるかです。不完全なモジュールは依存性が欠落していても許容されます。
```
@Module(complete = false)
class DripCoffeeModule {
  @Provides Heater provideHeater(Executor executor) {
    return new CpuHeater(executor);
  }
}
```
モジュール内に注入クラスとして記載されているもので使われない型があった場合もまたエラーの引き金となります。
```
@Module(injects = Example.class)
class DripCoffeeModule {
  @Provides Heater provideHeater() {
    return new ElectricHeater();
  }
  @Provides Chiller provideChiller() {
    return new ElectricChiller();
  }
}
```
上の例ではExampleクラスの注入記述がモジュール内にありますが、Heaterクラスしか使われていません。
javacはこの非利用も拒絶します。
```
[ERROR] COMPILATION ERROR:
[ERROR]: Graph validation failed: You have these unused @Provider methods:
      1. coffee.DripCoffeeModule.provideChiller()
      Set library=true in your module to disable this check.
```
もし、injectsで記載されているクラス外でつかう場合は、このmoduleにlibraryのマークをつけます。
```
@Module(
  injects = Example.class,
  library = true
)
class DripCoffeeModule {
  @Provides Heater provideHeater() {
    return new ElectricHeater();
  }
  @Provides Chiller provideChiller() {
    return new ElectricChiller();
  }
}
```
アプリケーションのモジュールをまとめたモジュールを作り、コンパイル時の検証を最大限に活かしましょう。注釈プロセッサは各モジュールにおいて問題を検知して報告するでしょう。
```
@Module(
    includes = {
        DripCoffeeModule.class,
        ExecutorModule.class
    }
)
public class CoffeeAppModule {
}
```
注釈プロセッサはDagger'sのjarをクラスパスに追加した際に自動的に有効になります。

### COMPILE-TIME CODE GENERATION
Daggerの注釈プロセッサはCoffeeMaker$InjectAdapter.java または DripCoffeeModule$ModuleAdapterのようにソースファイルを生成します。これらはDaggerの実装の詳細です。
注入でのステップ実行時に便利かもしれませんが、これらを直接使う必要はありません。

### MODULE OVERRIDES
Daggerは同一の依存関係で@Providesメソッドに複数の競合があった場合はエラーで失敗します・
しかし、時に開発やテストの代わりとして本番のコードを置き換える必要があるでしょう。
module 注釈でoverrides = true オプションを使うと、モジュールのバインディングにおいて先攻
することができます。

このJUnitのテストではDripCoffeeModule's のHeaterをMockito.を使ったモックオブジェクト
で上書きしています。
モックオブジェクトはCoffeeMakerとテストにもまたへ注入されます。

```
public class CoffeeMakerTest {
  @Inject CoffeeMaker coffeeMaker;
  @Inject Heater heater;

  @Before public void setUp() {
    ObjectGraph.create(new TestModule()).inject(this);
  }

  @Module(
      includes = DripCoffeeModule.class,
      injects = CoffeeMakerTest.class,
      overrides = true
  )
  static class TestModule {
    @Provides @Singleton Heater provideHeater() {
      return Mockito.mock(Heater.class);
    }
  }

  @Test public void testHeaterIsTurnedOnAndThenOff() {
    Mockito.when(heater.isHot()).thenReturn(true);
    coffeeMaker.brew();
    Mockito.verify(heater, Mockito.times(1)).on();
    Mockito.verify(heater, Mockito.times(1)).off();
  }
}
```

こういった上書きはアプリケーション上の小さい変更に適しています。
* 実際の実装をユニットテスト用のモックに置き換える
* LDAPでの認証を開発用の偽認証に置き換える

さらに実質的な変更では、異なるモジュールの組み合わせを使うとわかり易いでしょう。


