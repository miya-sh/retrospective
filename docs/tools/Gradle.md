# はじめに
Gradleをお勉強（入門）したのでまとめる。

内容的には、公式Documentをざっと読み、内容をざっくり意訳ないし、自分の解釈を入れた感じ。

気になる依存解決周りは比重多めで。読んだVersionは、7.5.1。

ドキュメントはこちらから。
- https://docs.gradle.org/7.5.1/userguide/userguide.html

差し込まれている画像は全て、docs.gradle.org から参照しました。

# Gradleとは
OSSのビルド自動化ツール。Groovy、Kotlin DSLを使ってビルドスクリプトを書き、色々（ビルド、テスト、デプロイ etc.）できる。

AndroidではKotlinDSLで書くことがデフォになるらしい（[2023 Roadmap](https://github.com/gradle/build-tool-roadmap/issues/39)より）。覚えておいて損はないかも。
書き方とかについては、[ここら辺](https://docs.gradle.org/current/userguide/kotlin_dsl.html)を。

Gradle = Java, Kotlin のビルドツール、主にパッケージ管理システムというイメージだったが、もっと広義の存在だった。
言語に関わらずビルドを扱えるし、Automation toolなだけあり、Build、PackaingとDeliveryまでに必要なタスクを自動化できるだけのパワーがある。

# コンセプトや強み
[gradle.org](https://gradle.org/) や、[What is Gradle?](https://docs.gradle.org/7.5.1/userguide/what_is_gradle.html) から。

- Build Anything
  - どんな言語で書かれていても、それをビルド、パッケージングして、デプロイするツール。
  - 実際、C++, Swift は公式に提供されている。
    - https://docs.gradle.org/current/userguide/building_cpp_projects.html
      - pre-build binary をDLし、依存解決もできる。  
        現状は、GradleによってMaven Repo.にパブリッシュされたものに限り、[Conan](https://conan.io/), Nuget のものは未サポートらしい。
    - https://docs.gradle.org/current/userguide/building_swift_projects.html
- Automate Everything
  - Gradleの機構（API,　Plugin）によって、ビルド〜デリバリーまでを自動化する。
- Deliver Faster
  - 高速なビルド（その為の機構など）を提供して、CDをサポートする。

色々強みはあるみたいだが、主にはこんなところか。
- JVMで走るので、Java APIを利用できる。
- Mavenを参考に、規約を導入している（CoC (Convention over configuration) という概念だそう）。これによってビルドが易しくなる。
- Pluginを用意することで、いかようにも拡張できる。Androidでは公式ビルドツールとして採用されている。

## Convention over configuration ってなに
邦訳では、「設定より規約」とのこと。Wikipediaに説明はあるので、そこらへんは省きつつ。

調べてみると、ツールが規約（いい感じのデフォルト）を持っているので、意識をしていなくとも規約によって、いい感じの設定を提供してくれるものらしい。
つまり、最初っからよしなにやってくれるツール、という感じだろうか。
Ruby on Railsが有名とのことで、確かに学生の時くらいに触った時にscaffoldで色々と整備してくれたなぁと。
規約より「慣習」のほうが収まりがいい感があるとか見かけた。確かに。

## Gradleを知るための5つのこと [ref.](https://docs.gradle.org/7.5.1/userguide/what_is_gradle.html#five_things)

1. Gradle is a general-purpose build tool
    - ただし、依存管理について現状は Maven と Ivy 互換のリポジトリしかサポートしていない

1. The core model is based on tasks
    - ビルドのタスクが有向非巡回グラフ（DAG）によってモデル化されている。
      - DAGってなに...
        - その名の通り、閉路の無い有向グラフを指す言葉で、トポロジカルソートできる特徴がある。
        - 因果関係や物事の依存関係をモデル化するのに有効とのこと。
        - トポロジカルソートは、Makefileのコンパイル順序決定、論理合成など色々と利用されている（Wikipediaより）。
    - 公式ドキュメントにある例（左が一般的、右がJava特有）。
      このようにタスクには依存関係があり、GradleはこのGraphをPluginやスクリプトで定義できる。
      ![Two examples of Gradle task graphs](https://docs.gradle.org/7.5.1/userguide/img/task-dag-examples.png)

1. Gradle has several fixed build phases
    - 3フェーズでビルドが実行される
      1. Initialization
          - 単一/複数プロジェクトのビルドができるので、ここでどのプロジェクトがビルドに参加するかを決定し、それぞれインスタンスを作成する。
      1. Configuration
          - ビルドに参加しているProjectのビルドスクリプトが実行される。ここでタスクの依存関係が構築される。
          - `doLast {}`や`doFirst {}`といったビルドアクションは実行時に評価される。
      1. Execution
          - コマンド引数によって選択されたタスク（そのサブセット）を実行する。

1. Gradle is extensible in more ways than one
    - Custom task types, Extra propertiesなどGradleが柔軟であると主張する機能
    - ビルドタスクのカスタマイズ関連なので一旦割愛。

1. Build scripts operate against an API
    - Gradleのベストプラクティスでは、スクリプトとはいえ命令的なロジックをあまり組まないほうがとのこと。
      APIやPluginで宣言的にやる。

## vs. Maven
https://gradle.org/maven-vs-gradle/

- Flexibility
  - タスクのカスタマイズなどGradleの基礎部分で差があるとのこと
- Performance
  以下の要素でGradleの方が速度が良いと言っている
  - Incremental Build Support
    - 変更ファイルだけをなるべく処理
    - タスクの入出力をトラッキングして、変化がなければ処理しない
  - Build Cache
  - Gradle Daemon
    - JVM起動コストや、ビルド関連の情報をキャッシュするので高速化できる
- User Experience
  - IDEサポートや、WebUIなど
- Dependency Management
  - 依存解決においてもMavenより柔軟である
    - Mavenはバージョンで依存の書き換えができるが、Gradleは代替ルールによって反映できる。
    - Mavenは依存解決のスコープは少ないが、Gradleはカスタムできる。
    - Mavenはコンフリクトの解決に宣言の順序が影響するが、GradleはGraphを完全解決し最新を利用したり厳密に宣言もできる。

## （その他）Gradle Wrapper
https://docs.gradle.org/7.5.1/userguide/gradle_wrapper.html

Gradle wrapperは推奨された方法で、特定バージョンのGradleでのビルドを実行できるスクリプトのこと。

![The Wrapper workflow](https://docs.gradle.org/7.5.1/userguide/img/wrapper-workflow.png)

以下のレイアウトで配備される。
```
.
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
└── gradlew.bat
```
- gradlew, gradlew.bat
  - Wrapperでビルド実行するためのスクリプト
- gradle-wrapper.properties
  - Gradleのバージョンなどを指定するプロパティのファイル。ここを見れば、どのバージョンでビルドしたいのかわかる。
- gradle-wrapper.jar
  - Gradle distributionをDLしたりするコード込みJar

# Dependency Management

参考
- https://docs.gradle.org/7.5.1/userguide/core_dependency_management.html
- https://gradle.org/features/#dependency-management

## dependency configuration
特定の目的のためグループ化された、依存関係のネームドセット。

多くのPluginはConfigurationはPre-definedされている。
[Java plugin](https://docs.gradle.org/7.5.1/userguide/java_plugin.html)の場合だと、
- compileOnly
- runtimeOnly
- runtimeClasspath
- testRuntimeClasspath

など。全てはページ内の[Configurationリスト](https://docs.gradle.org/7.5.1/userguide/java_plugin.html#tab:configurations)にて。
それぞれのタスクで、決まったConfigurationが参照される。
![Configurations use declared dependencies for specific purposes](https://docs.gradle.org/7.5.1/userguide/img/dependency-management-configurations.png)

継承による階層を持たせることもできる。実際、testRuntimeClasspath は runtimeClasspath から拡張されている。
[公式の例](https://docs.gradle.org/7.5.1/userguide/declaring_dependencies.html#sub:config-inheritance-composition)では、Smoke testを追加するためにpre-definedのtest用Configurationを拡張（依存の追加）する方法を提示している。

```build.gradle.kts
val smokeTest by configurations.creating {
    extendsFrom(configurations.testImplementation.get())
}

dependencies {
    testImplementation("junit:junit:4.13")
    smokeTest("org.apache.httpcomponents:httpclient:4.5.5")
}
```

## こんな感じに書く、宣言していく
### Repository
Maven, Ivy 互換のリポジトリである必要がある。MavenCentral, Google Maven, それ以外のRepo.を指定するやり方。

```build.gradle.kts
repositories {
    mavenCentral()
    google()
    maven {
        url = uri("https://repo.spring.io/release")
    }
}
```

## Javaプロジェクトのビルドを読んでみる
https://docs.gradle.org/7.5.1/userguide/dependency_management_for_java_projects.html
