---
title: Running change detection - Autorun with zones
date: 2024-12-31 11:22:14
tags:
---

原文：<https://angular.love/running-change-detection-autorun-with-zones>

# 執行 Change Detection – 使用 Zones 自動執行

## 使用 zones 自動執行

Angular 中的變更偵測（渲染）通常是因瀏覽器中的非同步事件而完全自動觸發的。
這是透過使用 [zone.js](https://github.com/angular/angular/blob/297d865e9e1ca0ffa183988c16429f16bc2ccbd7/packages/zone.js) 函式庫實作的 zones 來實現的。
一般來說，zones 提供了一種機制來攔截非同步操作的排程和調用。
攔截器邏輯可以在任務之前或之後執行額外的程式碼，並通知相關方有關該事件。
這些規則在建立每個 zone 時會個別定義。

<!-- more -->

Zones 以階層式的父子關係組成。
一開始，瀏覽器會在一個特殊的根 zone 中運行，該根 zone 被配置為表現得與平台完全相同，
使任何現有的非 zone 感知的程式碼都能如預期般運行。
在任何給定時間，只有一個 zone 可以處於活動狀態，並且可以透過 `Zone.current` 屬性檢索此 zone：

![圖片 15：圖片替代文字](https://wp.angular.love/wp-content/uploads/2024/08/1-11.png)

如果你有興趣學習如何直接使用 zone.js，請查看[這篇文章](https://angular.love/en/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/)。

**與普遍的看法相反，zones 不是 Angular 中變更偵測機制的一部分。**
事實上，Angular 可以使用 [變更偵測服務] 在沒有 zones 的情況下工作。
為了啟用自動變更偵測，Angular 實作了
[NgZone](https://github.com/angular/angular/blob/30d5a2ca83c9cf44f602462597a58547b05b75dd/packages/core/src/zone/ng_zone.ts#L86)
服務，該服務會 fork 一個子 zone 並訂閱來自該 zone 的通知。

這個 zone 被稱為 Angular zone，並且所有應用程式特定的程式碼都應該在這個 zone 內運行。
這是因為 `NgZone` 只會收到在 Angular zone 內發生的事件通知，而不會收到
任何其他 zone 中的事件通知：

![圖片 16：圖片替代文字](https://wp.angular.love/wp-content/uploads/2024/08/2-13.png)

如果你探索 `NgZone`，你會看到對 fork 的 Angular zone 的引用
[儲存](https://github.com/angular/angular/blob/86372538ab017058cb9b2a7910e761d823bd3c16/packages/core/src/zone/ng_zone.ts#L400)
在 `_inner` 屬性中：

```typescript
export class NgZone {
  constructor(...) {
    forkInnerZoneWithAngularBehavior(self);
  }
}

function forkInnerZoneWithAngularBehavior(zone: NgZonePrivate) {
  zone._inner = zone._inner.fork({ ... });
}
```

這是當你執行 `NgZone.run()` 時，用來執行回呼的 zone：

```typescript
export class NgZone {
  run(fn, applyThis, applyArgs) {
    return this._inner.run(fn, applyThis, applyArgs);
  }
}
```

fork Angular zone 時的目前 zone 會保留在 `_outer` 屬性中，並且當你執行 `NgZone.runOutsideAngular()` 時，會用來執行回呼：

```typescript
export class NgZone {
  runOutsideAngular(fn) {
    return (this as any as NgZonePrivate)._outer.run(fn);
  }
}
```

通常，這個外部 zone 是最頂層的「根」zone。

當 Angular 完成其引導程序時，你會得到以下 zones 的階層結構：

```
'root';
'angular';
```

你可以透過簡單地記錄對應的屬性來輕鬆地親自查看：

```typescript
export class AppComponent {
  constructor(zone: NgZone) {
    console.log((zone as any)._inner.name); // angular
    console.log((zone as any)._outer.name); // root
  }
}
```

然而，在開發模式下，還有一個 `AsyncStackTaggingZone` 位於 `root` 和 `angular` zone 之間，如下所示：

```
'root';
'AsyncStackTaggingZone';
'angular';
```

在這種情況下，`NgZone` 實例會在其 `_inner` 屬性中保留對 `AsyncStackTaggingZone` 的引用。
`AsyncStackTaggingZone` 提供連結的堆疊追蹤，以顯示非同步操作的排程位置。
有關更多詳細資訊，請參閱文章
[使用 DevTools 進行更好的 Angular 除錯](https://developer.chrome.com/blog/devtools-better-angular-debugging/)。

我們需要知道的最後一點是，Angular 在引導階段會實例化 `NgZone`：

```typescript
export class PlatformRef {
		bootstrapModuleFactory<M>(moduleFactory, options?) {
        const ngZone = getNgZone(options?.ngZone, getNgZoneOptions(options));
        const providers: StaticProvider[] = [{provide: NgZone, useValue: ngZone}];
        // 所有初始化邏輯都在 Angular zone 內運行
        return ngZone.run(() => {...});
		}
}
```

Angular 使用 `ApplicationRef` 內的
[onMicrotaskEmpty](https://github.com/angular/angular/blob/ff84c7360334792a9a422bd0fa0353e0cf670525/packages/core/src/application_ref.ts#L766)
事件，來自動觸發整個應用程式的變更偵測：

```typescript
@Injectable({ providedIn: 'root' })
export class ApplicationRef {
  constructor(
    private _zone: NgZone,
    private _injector: EnvironmentInjector,
    private _exceptionHandler: ErrorHandler
  ) {
    this._onMicrotaskEmptySubscription = this._zone.onMicrotaskEmpty.subscribe({
      next: () => {
        this._zone.run(() => {
          this.tick();
        });
      },
    });
  }
}
```

現在讓我們看看 Angular 如何在沒有 zones 的情況下工作。

## 無 zone 應用程式

若要在沒有 `zone.js` 的情況下執行 Angular 應用程式，我們需要將 `noop` 值傳遞到 `bootstrapModule` 函式的 `ngZone` 參數中，如下所示：

```typescript
platformBrowserDynamic().bootstrapModule(AppModule, {
  ngZone: 'noop',
});
```

如果我們現在執行這個簡單的應用程式：

```typescript
@Component({
  selector: 'app-root',
  template: `{{ time }}`,
})
export class AppComponent {
  time = Date.now();
}
```

我們會看到變更偵測已完全運作，並在 DOM 中呈現 `time` 值。

然而，如果我們在 `setTimeout` 的回呼中更新 `name` 屬性：

```typescript
@Component({
  selector: 'app-root',
  template: `{{ time }}`,
})
export class AppComponent {
  time = Date.now();

  constructor() {
    setTimeout(() => {
      this.time = Date.now();
    }, 1000);
  }
}
```

我們會看到變更未更新。這是預期的行為，因為
沒有 Angular zone 可以通知 Angular 關於 timeout 事件的發生。
有趣的是，我們仍然可以將 `NgZone` 注入到建構函式中：

```typescript
import { ɵNoopNgZone } from '@angular/core';

export class AppComponent {
  constructor(zone: NgZone) {
    console.log(zone instanceof ɵNoopNgZone); // true
  }
}
```

但它是 `NgZone` 的空實作，不執行任何操作。我們可以使用變更偵測器服務來手動執行變更偵測：

```typescript
@Component({
  selector: 'app-root',
  template: `{{ time }}`,
})
export class AppComponent {
  time = Date.now();

  constructor(cdRef: ChangeDetectorRef) {
    setTimeout(() => {
      this.time = Date.now();
      cdRef.detectChanges();
    }, 1000);
  }
}
```

我們將在
[手動控制](https://angular.love/en/running-change-detection-manual-control/)章節中詳細介紹這項服務。

有趣的是，如果你放錯了 `cdRef.detectChanges()` 並將其直接放在元件的 `constructor` 中
在 `setTimeout` 回呼之外，如下所示：

```typescript
@Component({...})
export class AppComponent {
  constructor(cdRef: ChangeDetectorRef) {
    setTimeout(() => {
      this.time = Date.now();
    }, 1000);
    cdRef.detectChanges();
  }
}
```

Angular 將會拋出「ASSERTION ERROR」錯誤：

![圖片 17：圖片替代文字](https://wp.angular.love/wp-content/uploads/2024/08/3-11.png)

此錯誤來自[此行程式碼](https://github.com/angular/angular/blob/5f9c7ceb907be47dff3e203dd837fd6ee9133fcb/packages/core/src/render3/instructions/shared.ts#L356)，
以防止在整個元件樹實例化之前執行變更偵測。

## 在 Angular zone 內執行程式碼

你可能會發現自己處於某種情況，其中一個函式以某種方式在 Angular 的 zone 之外運行，並且你無法獲得自動變更偵測的好處。這是一個常見的情況，第三方函式庫在不知道 Angular 上下文的情況下執行其操作。

這裡有一個涉及
[Google API Client Library](https://developers.google.com/api-client-library/) (gapi) 的[此類問題的範例](https://stackoverflow.com/a/46286400/2545680)。
常見的罪魁禍首是使用 JSONP 等技術，這些技術不使用常見的 AJAX API，如
[XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) 或
[Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)，這些 API 會由 Zones 修補和追蹤。
相反，它會建立一個具有來源 URL 的 `script` 標籤，並定義一個全域回呼，
以便在從伺服器擷取具有資料的請求指令碼時觸發。
Zones 無法修補或偵測到這一點，因此框架仍然不知道使用此技術執行的請求。

解決此類問題的常見方法是簡單地在 Angular zone 內執行回呼，如下所示。例如，對於
[gapi](https://github.com/google/google-api-javascript-client/blob/master/docs/start.md#option-1-load-the-api-discovery-document-then-assemble-the-request)，
我們應該這樣做：

```typescript
// 載入 JavaScript 客户端函式庫。
gapi.load('client', ()=> {
    // 在 Angular zone 內執行初始化程式碼
    NgZone.run(()=>{
        // 初始化 JavaScript 客户端函式庫
        gapi.client.init({...}).then(function() { ... });
    });
});
```

另一個範例是，元件會從在 Angular zone 外部運行的回呼發出通知：

```typescript
@Component({
  selector: 'n-cmp',
  template: '{{title}} <div><n1-cmp></n1-cmp></div>',
})
export class N {
  title = 'N component';
  emitter = new Subject();

  constructor(zone: NgZone) {
    zone.runOutsideAngular(() => {
      setTimeout(() => {
        this.emitter.next(3);
      }, 1000);
    });
  }
}
```

如果我們簡單地訂閱子元件 `N1` 中的變更，如下所示：

```typescript
@Component({
  selector: 'n1-cmp',
  template: '{{title}}, 發射的值：{{value}}',
})
export class N1 {
  title = 'N 的子項';
  value = '還沒有東西';

  constructor(parent: N, zone: NgZone) {
    parent.emitter.subscribe((v: any) => {
      this.value = v;
    });
  }
}
```

我們不會在螢幕上看到任何更新，即使在父項發出後，`this.value` 會更新為 `3`。
若要修正此問題，就像在 `gapi` 範例中一樣，我們可以在 Angular zone 中執行回呼：

```typescript
@Component({...})
export class N1 {
  title = 'N 的子項';
  value = '還沒有東西';

  constructor(parent: N, zone: NgZone) {
    parent.emitter.subscribe((v: any) => {
      zone.run(() => {
        this.value = v;
      });
    });
  }
}
```

這會修正此問題。
