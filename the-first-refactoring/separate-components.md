# コンポーネントの分割

### コンポーネントの分割

まずはじめに、ユーザーリストの表示において、リストを反復する役割の **リストコンポーネント** と、ユーザー1件ごとのビューを担当する **リストアイテムコンポーネント** を分割します。

#### 単一責任の原則

{% embed url="https://xn--97-273ae6a4irb6e2hsoiozc2g4b8082p.com/%E3%82%A8%E3%83%83%E3%82%BB%E3%82%A4/%E5%8D%98%E4%B8%80%E8%B2%AC%E4%BB%BB%E5%8E%9F%E5%89%87/" %}

今のところ、AppComponentは次のようなたくさんの責任を負っています。

* アプリケーションブートストラップ のエントリポイントとなること 
* ユーザーの配列をAPIから取得すること
* ユーザーの配列をリストとして表示すること
* ユーザーの情報を表示すること

このままでは、AppComponentはいくつもの理由で変更されることになります。これは単一責任の原則に反しています。

まずはリストで反復される単位要素を分割することで、ひとつの責任を別のモジュールに切り出すことにしましょう。

### UserListItemComponent

ユーザーリストの繰り返し単位のコンポーネントは、 `UserListItemComponent` と定義します。このコンポーネントの責任は、渡された1件のユーザーをリストアイテムとして表示することです。それ以外の理由でこのコンポーネントが変更されてはいけません。

{% code-tabs %}
{% code-tabs-item title="user-list-item.component.ts" %}
```typescript
import { Component, ChangeDetectionStrategy, Input } from '@angular/core';
import { User } from '../../user';

@Component({
  selector: 'user-list-item',
  templateUrl: './user-list-item.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserListItemComponent {

  @Input()
  user!: User;
}
```
{% endcode-tabs-item %}

{% code-tabs-item title="user-list-item.component.html" %}
```markup
#{{user.id}} {{user.name}}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

結果として、AppComponentのテンプレートは次のようになります。AppComponentが持っていたユーザーリストをどのように表示するか、ユーザーの情報をどうレイアウトするかという責任から開放されました。今後、ユーザーリストにメールアドレスを表示したいとなっても、AppComponentは変更されません。

{% code-tabs %}
{% code-tabs-item title="app.component.html" %}
```markup
<ul>
	<li *ngFor="let user of users">
		<user-list-item [user]="user"></user-list-item>
	</li>
</ul>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### UserListComponent

続いてもうひとつの責任、ユーザーの配列をどのようにリストとして表示するかという関心事をAppComponentから切り出すことにしましょう。新しく `UserListComponent` を作成します。

{% code-tabs %}
{% code-tabs-item title="user-list.component.ts" %}
```typescript
import { Component, ChangeDetectionStrategy, Input } from '@angular/core';
import { User } from '../../user';

@Component({
  selector: 'user-list',
  templateUrl: './user-list.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListComponent {

  @Input()
  users!: User[];
}
```
{% endcode-tabs-item %}

{% code-tabs-item title="user-list.component.html" %}
```markup
<ul>
	<li *ngFor="let user of users">
		<user-list-item [user]="user"></user-list-item>
	</li>
</ul>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

結果として、AppComponentのテンプレートは次のようになります。AppComponentの責任は、アプリケーションのエントリポイントであることと、ユーザーの配列を取得することになりました。ユーザーの配列をどのようにリストとして表示するかというUIの関心事はUserListComponentに分離できました。

{% code-tabs %}
{% code-tabs-item title="app.component.html" %}
```markup
<user-list [users]="users"></user-list>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

{% embed url="https://stackblitz.com/edit/angular-brjyhu?file=src%2Fapp%2Fapp.component.html" %}



UIの責任はほとんどAppComponentから切り出すことができたので、次のページではユーザーの配列を取得する責任をサービスに切り出します。





