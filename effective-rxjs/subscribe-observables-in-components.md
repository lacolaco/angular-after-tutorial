---
description: >-
  この章では
  AngularコンポーネントにおいてどのようにObservableを購読すべきなのかについて学びます。単純なsubscribeからはじめ、段階的にAsyncパイプを使ったリアクティブなパターンの使い方へステップアップしていきます。
---

# コンポーネントにおけるObservableの購読

Angularのコンポーネントの基本とRxJSの基本を習得したら、次に学ぶのは実際にどうやってObservableとコンポーネントを組み合わせるかというプラクティスです。うまくObservableを利用することで、ユーザーインタラクションとリアクティブに協調するコンポーネントを簡単に実装できるようになるでしょう。

## 明示的な購読

最初に解説する方法は**明示的な購読**です。これはObservableの使い方の中でもっとも基本的で単純な方法です。これは公式チュートリアルの中でも、HeroServiceから値を取得するために学んだはずです。

コンポーネントを実装する前に、まずはサンプル用の `DataService` を用意します。これは[BehaviorSubject](http://reactivex.io/rxjs/manual/overview.html#behaviorsubject)を持つ単純なAngularのサービスです。`setValue` メソッドが `valueSubject` に値を流し、 `valueChanges` getterが `valueSubject` をObservableとして外部に公開します。  


{% code-tabs %}
{% code-tabs-item title="data.service.ts" %}
```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class DataService {

  private valueSubject = new BehaviorSubject<any>('initial');

  get valueChanges() {
    return this.valueSubject.asObservable();
  }

  setValue(value: any) {
    this.valueSubject.next(value);
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

そして AppComponentには新たな値を流すためのボタンを用意します。ここをスタートとして改めて学んでいきましょう。

```typescript
import { Component } from '@angular/core';
import { DataService } from './data.service';

@Component({
  selector: 'my-app',
  template: `
  <button (click)="updateValue()">Update Value</button>
  `,
})
export class AppComponent {

  constructor(private dataService: DataService) { }

  updateValue() {
    const value = new Date().toISOString();
    this.dataService.setValue(value);
  }
}

```

まずは DataServiceの値を表示するためのコンポーネントを作成します。  明示的なsubscribeをおこなう `ExplicitSubscribeComponent` クラスは、次のように記述します。`value` フィールドを持っていて、テンプレート内で補間構文 `{{ value }}` を使って表示しています。`value` フィールドの更新は、 `valueChanges.subscribe` メソッドに渡されたコールバック関数で行われています。

{% hint style="warning" %}
`subscribe` メソッドはコンストラクタ内で呼び出してはいけません。Angularのコンポーネントインスタンスが作成されるタイミングと、実際にコンポーネントがビューに配置されるタイミングは一致しません。予期しない挙動やエラーを防ぐために、`ngOnInit` よりも後のタイミングで呼び出しましょう。
{% endhint %}

{% code-tabs %}
{% code-tabs-item title="explicit-subscribe.component.ts" %}
```typescript
import { Component, OnInit } from '@angular/core';
import { DataService } from '../data.service';

@Component({
  selector: 'app-explicit-subscribe',
  template: `<div>{{ value }}</div>`,
})
export class ExplicitSubscribeComponent implements OnInit {
  value: any;

  constructor(private dataService: DataService) { }

  ngOnInit() {
    this.dataService.valueChanges.subscribe(value => {
      this.value = value;
    });
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

`subscribe` メソッドを使う方法はシンプルに見えますが注意点があります。[Observableのライフサイクル](observable-lifecycle.md#noobservable-1) で述べたように、購読解除をしなければコンポーネントが破棄されたあとにメモリリークが発生します。おそらく、`ngOnDestroy` ライフサイクルフックメソッドを使って次のように `unsubscribe` メソッドを呼ぶ方法をまず最初に思いつくでしょう。

{% code-tabs %}
{% code-tabs-item title="explicit-subscribe.component.ts" %}
```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subscription } from 'rxjs';
import { DataService } from '../data.service';

@Component({ ... })
export class ExplicitSubscribeComponent implements OnInit, OnDestroy {
  value: any;
  private subscription: Subscription;

  constructor(private dataService: DataService) { }

  ngOnInit() {
    this.subscription = this.dataService.valueChanges.subscribe(value => {
      this.value = value;
    });
  }

  ngOnDestroy() {
    this.subscription.unsubscribe();
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

この方法はクラスフィールドとして `Subscription` オブジェクトを保持する必要があります。`subscribe` メソッドの戻り値を使うことになりますが、コンポーネントが購読する Observableが複数になると、煩雑なコードになってしまうのが欠点であり、購読解除忘れも発生しがちです。

ところで、[Observableのライフサイクル](observable-lifecycle.md#noobservable) で説明したとおり、Observableが完了するとその時点ですべての購読が自動的に解除されます。それを利用して、購読を開始する前の段階で、自動的に完了するように仕込んでおくことで購読解除忘れを防ぐことができます。

次の例では、`ngOnDestroy` ライフサイクルフックのタイミングで発行される `onDestroy$` Subjectを作成し、 `valueChanges` には`takeUntil` オペレーターを追加します。`takeUntil` オペレータは、引数に指定したObservableに値が流れたときに、オペレーターが適用されたObservableを強制的に完了します。このように実装すると、複数のObservableになったときもクラスフィールドを増やすことなくそれぞれに `takeUntil` オペレーターを追加するだけです。

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { DataService } from '../data.service';

@Component({
  selector: 'app-explicit-subscribe',
  template: `<div>{{ value }}</div>`,
})
export class ExplicitSubscribeComponent implements OnInit, OnDestroy {
  value: any;
  private onDestroy$ = new Subject();

  constructor(private dataService: DataService) { }

  ngOnInit() {
    this.dataService.valueChanges
      .pipe(takeUntil(this.onDestroy$))
      .subscribe(value => {
        this.value = value;
      });
  }

  ngOnDestroy() {
    this.onDestroy$.next();
  }
}
```

## Asyncパイプを使う

ここまで明示的な購読をどのようにうまく書くかについて解説しましたが、そもそも明示的に購読しなければ、購読解除について考える必要もありません。ここからは Observableで提供されるデータをテンプレートにバインディングするときに使える **Async パイプ** を紹介します。基本的な方針として、Asyncパイプで解決できるケースでは常にAsyncパイプを利用し、 **明示的な購読は最低限に留めましょう。**Observableをコアにしたリアクティブなアプリケーション設計を進める上で Asyncパイプは必需品です。

先ほどの `ExplicitSubscribeComponent` と同じことを Asyncパイプで実装するコンポーネントを、 `AsyncPipeComponent` という名前で次のように作成します。

{% code-tabs %}
{% code-tabs-item title="async-pipe.component.ts" %}
```typescript
import { Component } from '@angular/core';
import { Observable } from 'rxjs';
import { DataService } from '../data.service';

@Component({
  selector: 'app-async-pipe',
  template: `<div>{{ value$ | async }}</div>`,
})
export class AsyncPipeComponent {
  value$: Observable<any>;

  constructor(private dataService: DataService) {
    this.value$ = this.dataService.valueChanges;
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

びっくりするほどコードがスッキリしました。購読や購読解除に関するコードはなくなりました。同期的な`value` フィールドを廃止して、Observable型の `value$`  フィールドに変更しています。`value$` フィールドには`valueChanges` を代入し、テンプレートで `{{ value$ | async }}` のように Asyncパイプを適用しています。

なぜこれほどコンポーネントのコードが少なくなったかというと、明示的な購読の例で記述していた `subscribe` メソッドの呼び出しや `unsubscribe` メソッドの呼び出しを Asyncパイプが肩代わりしているからです。 Asyncパイプが適用されたObservableは、そのコンポーネントのライフサイクルに合わせて自動的に購読・購読解除がおこなわれます。これにより、**開発者は面倒なRxJSのお作法から解放され、ビジネスロジックだけに集中できます**。たとえば、`valueChanges` で流れてくる値を操作したいと思ったときには、`this.value$` へ代入する際にオペレーターを追加するだけでいいのです。

```typescript
this.value$ = this.dataService.valueChanges.pipe(
  map(value => `Value: ${value}`)
);
```

もうひとつの良いところは、**`this.value$` の初期化をコンストラクタで完結できることです。**購読を開始しているわけではないため、クラスフィールドへの代入は `ngOnInit` まで待たなくてもよいのです。

{% hint style="warning" %}
Asyncパイプは、コンポーネントの初期化時にフィールドとして存在する、無限なObservableにのみ使われるべきです。つまりユーザーインタラクションなどによって後から発生するHTTPリクエストのような有限のObservableは、Asyncパイプを使うのに適していません。
{% endhint %}

## ngIfによるテンプレートのブロック化

Asyncパイプを使うときに注意するのは、同じObservableに複数回 Asyncパイプを適用してしまうことです。たとえば次のようなテンプレートを書いてしまうと、同じObservableを2度購読することになります。少しなら影響はありませんが、購読の数が増えるのはメモリ使用量が上昇し、本来不要な変更検知処理も実行されるので、パフォーマンスに悪影響があります。さらに、Asyncパイプの処理タイミングが別なので、2つのデータバインディングの解決が常に完全に同時であることは保証されません。

```markup
<div>1. {{ value$ | async }}</div>
<div>2. {{ value$ | async }}</div>
```

これを解決するのが、`*ngIf` ディレクティブと Asyncパイプを使ったテンプレートのブロック化です。次のように、Asyncパイプと `*ngIf` の `as` 構文を使うことで、 `*ngIf` ディレクティブの内側で非同期的に更新される `value` を同期的に扱えるようになります。

```markup
<ng-container *ngIf="{ value: value$ | async } as snapshot" >
　　<div>1. {{ snapshot.value }}</div>
　　<div>2. {{ snapshot.value }}</div>
</ng-container>
```

はじめは難しいテンプレートに感じるかもしれませんが、構成要素を順番に紐解いていくことで理解できるはずです。

### ng-containerタグ

`<ng-container>` タグはAngularのテンプレート内でだけ使える特殊なタグです。このタグはテンプレート内で任意の範囲をブロック化するために使いますが、**実際にDOMへレンダリングされるときには除去されます**。

`*ngIf` や `*ngFor` のような[構造ディレクティブ](https://angular.jp/guide/structural-directives#ng-container-to-the-rescue)を使うときに、テンプレート内の階層構造をうまく表現するためのタグとしてよく用いられる擬似的なタグです。

### ngIf-as 構文

`*ngIf` の `as` 構文は、`*ngIf` に渡された式の評価結果を、テンプレート内で新しい変数に代入する機能です。同期的な `*ngIf` では使い所はありませんが、 Asyncパイプを併用すれば、Asyncパイプで得られた値が `*ngIf` に渡されるタイミングで、値を変数化できるのです。

```markup
<ng-container *ngIf="user$ | async as user">
  {{ user.name }}
</ng-container>
```

しかしこのテンプレートには問題があります。`user$` がnullを返したとき、 `*ngIf` の評価はfalseとなるため、内側のブロック全体が表示されなくなります。これを解決するには、次のようにfalseだったときのテンプレートを指定できる`*ngIf` の `else` 構文を使う必要があります。

```markup
<ng-container *ngIf="user$ | async as user; else nullUser">
  {{ user.name }}
</ng-container>
<ng-template #nullUser>
  User is null
</ng-template>
```

### as-snapshotパターン

値がnullだったときの表示は `else` 構文でカバーすることもできますが、テンプレートが肥大化しがちです。そもそもの原因は Observableがnullを流したときに `*ngIf` の評価結果が falseになってしまうことです。つまり、**常にtrueになるような評価式**にしておくことで `else` 構文を使わなくても常に同じテンプレートで Observableのデータを描画できます。

その解決法が、 先ほどの **`{ value: value$ | async } as snapshot`** というテンプレート式です。この式でまず最初に評価されるのはパイプ部分なので、 `value$ | async` が評価されます。  
次に、 `as` の左辺が評価されます。左辺はJavaScriptのオブジェクトリテラルの宣言なので、Asyncパイプの計算結果（ `value$` の最新の値）が、オブジェクトの `value` プロパティに代入されます。  
最後に、`as snapshot` が評価され、左辺で宣言されたオブジェクトを  `snapshot` 変数に代入します。

結果として、 `value$ | async` という式で得られていた値が、 `snapshot.value` という形でアクセスできるようになります。 `snapshot` は常にオブジェクトですから、 `*ngIf` の評価がfalseになることはありません。

```markup
<ng-container *ngIf="{ value: value$ | async } as snapshot">
　　<div>1. {{ snapshot.value }}</div>
　　<div>2. {{ snapshot.value }}</div>
</ng-container>
```

このようにAsyncパイプを内包したオブジェクトを `*ngIf` の `as` 構文で変数化するパターンを、本書では **as-snapshot パターン**と呼ぶことにします。nullやundefined、`0` や `""` のような `*ngIf` の評価結果がfalseになってしまう値を Asyncパイプで扱うときには、as-snapshotパターンを使うのが便利です。そうでない場合にも一貫性を持ってas-snapshotパターンに統一するとよいでしょう。

### as-snapshotパターンの注意点

as-snapshotパターンにも注意点はあります。それは、`snapshot` に**複数の無関係なAsyncパイプを内包させてはいけない**ということです。次の例を見てください。as-snapshotパターンの左辺で、 `foo` と `bar` の2つのプロパティにそれぞれ別のAsyncパイプが適用されています。

```markup
<ng-container *ngIf="{ foo: foo$ | async, bar: bar$ | async } as snapshot">
　　<div>Foo: {{ snapshot.foo }}</div>
　　<div>Bar: {{ snapshot.bar }}</div>
</ng-container>
```

複数のObservableを1つのsnapshotで扱うのは合理的のように見えますが、実は問題点がありあます。このテンプレートでは `foo$` の更新によっても、`bar$` の更新によっても、どちらの場合も `*ngIf` の内部全体が再評価されることです。

本来、 `foo$` の更新は `bar$` には影響していないはずなので、 `{{ snapshot.bar }}` は再評価する必要がないはずですが、ひとつのブロックとして `*ngIf` で囲ってしまうと、その内部は同じタイミングで評価されてしまいます。

このような場合には、冗長ではありますがそれぞれのブロックで別々に Asyncパイプを使うべきです。 `<ng-container>` タグは描画後には消えるので、次のように書いたとしてもDOMとしては先ほどのテンプレートと同じように兄弟要素として描画されます。

```markup
<ng-container *ngIf="{ foo: foo$ | async } as snapshot">
　　<div>Foo: {{ snapshot.foo }}</div>
</ng-container>
<ng-container *ngIf="{ bar: bar$ | async } as snapshot">
　　<div>Bar: {{ snapshot.bar }}</div>
</ng-container>
```

{% hint style="info" %}
もし `foo$` と `bar$` が両方必要なテンプレートだった場合には 同じ `snapshot` にまとめることが合理的になるでしょう。しかしテンプレート式が複雑になるとメンテナンス性が下がるため、階層を作って対応することが好ましいです。

```markup
<ng-container *ngIf="{ foo: foo$ | async } as fooSnapshot">
  <ng-container *ngIf="{ bar: bar$ | async } as barSnapshot">
    <foo-bar-component 
      [foo]="fooSnapshot.foo" 
      [bar]="barSnapshot.bar"
    ></foo-bar-component>
  </ng-container>
</ng-container>

```
{% endhint %}

