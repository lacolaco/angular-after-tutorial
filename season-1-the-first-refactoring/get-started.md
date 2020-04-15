---
description: この章では、ユーザーリストを表示する簡単なアプリケーションを例に、Angularアプリケーションの設計について考えていきます。
---

# アプリケーションの作成

## ユーザーリストの取得

この章のサンプルアプリケーションでは、[https://reqres.in/](https://reqres.in/) のユーザーAPIで取得したユーザーを一覧して表示します。

[https://reqres.in/api/users](https://reqres.in/api/users) 

### AppComponent

最初の状態では、すべての処理を `AppComponent` だけでおこないます。サービスへの分割も、コンポーネントの分割もまだ何もおこなっていません。

{% code title="app/user.ts" %}
```typescript
export interface User {
  id: string;
  first_name: string;
  last_name: string;
}
```
{% endcode %}

{% tabs %}
{% tab title="app/app.component.ts" %}
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
    this.http.get<{ data: User[] }>('https://reqres.in/api/users').subscribe(resp => {
      this.users = resp.data;
    });
  }
}

```
{% endtab %}

{% tab title="app/app.component.html" %}
```markup
<ul>
	<li *ngFor="let user of users">
		#{{ user.id }} {{ user.first_name }} {{ user.last_name }}
	</li>
</ul>
```
{% endtab %}
{% endtabs %}

![&#x30E6;&#x30FC;&#x30B6;&#x30FC;&#x30EA;&#x30B9;&#x30C8;](../.gitbook/assets/image%20%288%29.png)

{% embed url="https://stackblitz.com/edit/angular-i8x98w?file=src%2Fapp%2Fapp.component.html" %}

次のページからは、このアプリケーションを段階的にリファクタリングしていきます。

