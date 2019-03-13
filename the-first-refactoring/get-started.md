---
description: この章では、ユーザーリストを表示する簡単なアプリケーションを例に、Angularアプリケーションの設計について考えていきます。
---

# アプリケーションの作成

## ユーザーリストの取得

この章のサンプルアプリケーションでは、[JSONPlaceholder](https://jsonplaceholder.typicode.com/) のユーザーAPIで取得したユーザーを一覧して表示します。

[https://jsonplaceholder.typicode.com/users](https://jsonplaceholder.typicode.com/users)

### AppComponent

最初の状態では、すべての処理を `AppComponent` だけでおこないます。サービスへの分割も、コンポーネントの分割もまだ何もおこなっていません。

{% code-tabs %}
{% code-tabs-item title="app/user.ts" %}
```typescript
export interface User {
  id: string;
  name: string;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

{% code-tabs %}
{% code-tabs-item title="app/app.component.ts" %}
```typescript
import { Component } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { User } from './user';

@Component({
  selector: 'my-app',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {

  users: User[] = [];

  constructor(private http: HttpClient) { }

  ngOnInit() {
    this.http.get<User[]>('https://jsonplaceholder.typicode.com/users').subscribe(data => {
      this.users = data;
    });
  }
}

```
{% endcode-tabs-item %}

{% code-tabs-item title="app/app.component.html" %}
```markup
<ul>
	<li *ngFor="let user of users">
		#{{ user.id }} {{ user.name }}
	</li>
</ul>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![&#x30E6;&#x30FC;&#x30B6;&#x30FC;&#x30EA;&#x30B9;&#x30C8;](../.gitbook/assets/image%20%288%29.png)

{% embed url="https://stackblitz.com/edit/angular-i8x98w?file=src%2Fapp%2Fapp.component.html" %}

次のページからは、このアプリケーションを段階的にリファクタリングしていきます。

