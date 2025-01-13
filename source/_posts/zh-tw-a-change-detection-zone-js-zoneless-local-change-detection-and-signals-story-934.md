---
title: A change detection, zone.js, zoneless, local change detection, and signals story 📚
tags:
---

原文：<https://itnext.io/a-change-detection-zone-js-zoneless-local-change-detection-and-signals-story-9344079c3b9d>

# 變更偵測、zone.js、無區域 (zoneless)、局部變更偵測和 Signals 的故事 📚


![Image 71](https://miro.medium.com/v2/resize:fit:700/1*N0FoEeuERFDvvkuOzhgOJA.png)

<!-- more -->


Angular 是一個元件驅動的框架。就像其他所有框架一樣，它應該向使用者顯示資料，並在資料變更時重新整理視圖。

![Image 72](https://miro.medium.com/v2/resize:fit:667/1*eVpyKXodMU9thMSDn1XY4Q.png)

一個在範本中顯示使用者資料的 UserCard 元件

隨著時間的推移，我們建立越來越多的元件並將它們組合在一起，我們最終可能會得到如下所示的元件樹狀結構。

![Image 73](https://miro.medium.com/v2/resize:fit:700/1*QbEmmWPDvOTKw60oW-RKmw.png)

元件結構

但是，Angular 如何知道何時重新整理視圖？它如何知道資料何時變更？它如何知道何時執行變更偵測？

**使用同步程式碼的變更偵測**
------------------------------------------

讓我們從一個簡單的範例開始。我們有一個帶有屬性 **name** 和方法 **changeName** 的元件。當我們點擊按鈕時，我們會呼叫 **changeName** 方法並變更 **name** 屬性。

```typescript
@Component({
  template: `
    <button (click)="changeName()">Change name</button>
    <p>{{ name }}</p>
  `
})
export class AppComponent {
  name = 'John';

  changeName() {
    this.name = 'Jane';
  }
}
```

當我們點擊按鈕時，會呼叫 **changeName** 方法，並且因為所有內容都由 Angular 包裝，我們可以安全地假設在名稱變更後，Angular 可以執行一些程式碼來更新視圖（並且所有內容都將同步）。

> ⚠️ 想像中的 Angular 底層程式碼：

```typescript
component.changeName();
// 此程式碼會執行 Angular 將對整個元件樹執行變更偵測，因為我們可能已更新其他元件中使用的服務中的某些資料
angular.runChangeDetection();
```

這運作良好！但是，大多數情況下，當我們變更資料時，我們不會同步執行。我們通常會發出 HTTP 請求，或使用一些計時器，或等待其他事件發生，然後再更新資料。這就是問題開始的地方。

**使用非同步程式碼的變更偵測**
-------------------------------------------

現在，假設我們想在 1 秒後變更名稱。我們可以使用 **setTimeout** 函式來執行此操作。

```typescript
@Component({
  template: `
    <button (click)="changeName()">Change name</button>
    <p>{{ name }}</p>
  `
})
export class AppComponent {
  name = 'John';

  changeName() {
    setTimeout(() => {
      this.name = 'Jane';
    }, 1000);
  }
}
```

當我們點擊按鈕時，會呼叫 **changeName** 方法，並呼叫 **setTimeout** 函式。**setTimeout** 函式將等待 1 秒，然後呼叫回呼函式。回呼函式會將名稱變更為 **Jane**。

現在，讓我們新增與之前相同的想像中的 Angular 底層程式碼。

> ⚠️ 想像中的 Angular 底層程式碼：

```typescript
component.changeName(); // 在內部使用 setTimeout
// 此程式碼會在呼叫 changeName 方法後立即執行
angular.runChangeDetection();
```

由於**呼叫堆疊**，**setTimeout** 回呼將在 **angular.runChangeDetection** 方法之後被呼叫。因此，Angular 執行了變更偵測，但名稱尚未變更。這就是視圖不會更新的原因。這是一個損壞的應用程式 ⚠️。(實際上並非如此，因為我們有 👇)

**Zone.js 來救援**
-------------------------

Zone.js 自 Angular 2.0 的早期就已存在。它是一個 monkey patch 瀏覽器 API 的程式庫，使我們能夠掛接到瀏覽器事件的生命週期。這是什麼意思？這表示我們可以在瀏覽器事件之前和之後執行程式碼。

```typescript
setTimeout(() => {
  console.log('Hello world');
}, 1000);
```

上面的程式碼將在 1 秒後列印 **Hello world**。但是，如果我們想在 **setTimeout** 回呼之前或之後執行一些程式碼呢？_你知道，出於業務原因_ 😄。一個名為 **Angular** 的框架可能想在 **setTimeout** 回呼之前和之後執行一些程式碼。

**Zone.js** 使我們能夠做到這一點。我們可以建立一個區域（Angular 也會建立一個區域）並掛接到 **setTimeout** 回呼。

```typescript
const zone = Zone.current.fork({
  onInvokeTask: (delegate, current, target, task, applyThis, applyArgs) => {
    console.log('Before setTimeout');
    delegate.invokeTask(target, task, applyThis, applyArgs);
    console.log('After setTimeout');
  }
});
```

要在區域內執行我們的 **setTimeout**，我們需要使用 **zone.run()** 方法。

```typescript
zone.run(() => {
  setTimeout(() => {
    console.log('Hello world');
  }, 1000);
});
```

現在，當我們執行上面的程式碼時，我們將看到以下輸出。

```
Before setTimeout
Hello world
After setTimeout
```

這就是 zone.js 的運作方式。它會 monkey patch 瀏覽器 API，使我們能夠掛接到瀏覽器事件的生命週期。

**Zone.js + Angular**
---------------------

Angular 在每個應用程式中預設載入 **zone.js** 並建立一個名為 **NgZone** 的區域。**NgZone** 包含一個名為 **onMicrotaskEmpty** 的 **Observable**。當佇列中不再有微任務時，此 observable 會發出值。這就是 Angular 用來知道何時所有非同步程式碼都已完成執行，並且可以安全地執行變更偵測的方式。

![Image 74](https://miro.medium.com/v2/resize:fit:700/1*3Oe8-a_qFIOnAGEZsfM3vg.png)

NgZone 今天包裝了每個 angular 應用程式

讓我們看看底層的[程式碼](https://github.com/angular/angular/blob/c4de4e1f894001d8f80b70297c5e576f2d11ec6f/packages/core/src/change_detection/scheduling/ng_zone_scheduling.ts#L31)：

```typescript
// ng_zone_scheduling.ts NgZoneChangeDetectionScheduler
this._onMicrotaskEmptySubscription = this.zone.onMicrotaskEmpty.subscribe({
    next: () => this.zone.run(() => this.applicationRef.tick())
});
```

我們在上面的程式碼中看到的是，當 **onMicrotaskEmpty** observable 發出值時，Angular 將呼叫 [**applicationRef.tick()**](https://github.com/angular/angular/blob/c4de4e1f894001d8f80b70297c5e576f2d11ec6f/packages/core/src/application/application_ref.ts#L537) 方法。這個 **tick** 方法是什麼 🤔？你還記得想像中的 Angular 底層程式碼中的 **runChangeDetection** 方法嗎？嗯，**tick** 方法就是 **runChangeDetection** 方法。它會 **_同步_** 執行整個元件樹的變更偵測。

但是現在，Angular 知道所有非同步程式碼都已完成執行，並且可以 **_安全地_** 執行變更偵測。

```typescript
tick(): void {
    // 為了簡潔起見，已移除程式碼
    for (let view of this._views) {
        // 為單一元件執行變更偵測
        view.detectChanges();
    }
}
```

**tick** 方法將迭代所有根視圖（大多數情況下，我們只有一個根視圖/元件，即 **AppComponent**）並同步執行 **detectChanges**。

**元件髒標記**
---------------------------

Angular 會做的另一件事是，當它知道元件內的某些內容發生變更時，它會將元件標記為髒。

以下是將元件標記為髒的事件：

*   事件（click、mouseover 等）

每次我們在範本中點擊帶有監聽器的按鈕時，Angular 都會使用一個名為 [**wrapListenerIn_markDirtyAndPreventDefault**](https://github.com/angular/angular/blob/c4de4e1f894001d8f80b70297c5e576f2d11ec6f/packages/core/src/render3/instructions/listener.ts#L260) 的函式包裝回呼函式。正如我們從函式的名稱中看到的那樣 😅，它會將元件標記為髒。

```typescript
function wrapListener(): EventListener {
  return function wrapListenerIn_markDirtyAndPreventDefault(e: any) {
    // ...為了簡潔起見，已移除程式碼
    markViewDirty(startView); // 將元件標記為髒
  };
}
```

*   變更的輸入

此外，在執行變更偵測時，Angular 會檢查元件的輸入值是否已變更（**===** 檢查）。如果已變更，它會將元件標記為髒。[原始碼在此](https://github.com/angular/angular/blob/c4de4e1f894001d8f80b70297c5e576f2d11ec6f/packages/core/src/render3/component_ref.ts#L348)。

```typescript
setInput(name: string, value: unknown): void {
    // 如果與最後一個值相同，則不要設定輸入
    if (Object.is(this.previousInputValues.get(name), value)) {
        return;
    }// 為了簡潔起見，已移除程式碼
    setInputsForProperty(lView[TVIEW], lView, dataValue, name, value);
    markViewDirty(childComponentLView); // 將元件標記為髒
}
```

*   輸出發射

為了在 Angular 中監聽輸出發射，我們在範本中註冊一個事件。正如我們之前看到的，回呼函式將被包裝，並且當發射事件時，元件將被標記為髒。

讓我們看看這個 [**markViewDirty**](https://github.com/angular/angular/blob/c4de4e1f894001d8f80b70297c5e576f2d11ec6f/packages/core/src/render3/instructions/mark_view_dirty.ts#L24) 函式的作用。

```typescript
/**
 * 將目前視圖和所有祖先標記為髒。
 */
export function markViewDirty(lView: LView): LView|null {
  while (lView) {
    lView[FLAGS] |= LViewFlags.Dirty;
    const parent = getLViewParent(lView);
    // 一旦找到未附加到任何容器的根視圖，就停止向上遍歷
    if (isRootView(lView) && !parent) {
      return lView;
    }
    // 否則繼續
    lView = parent!;
  }
  return null;
}
```

正如我們可以從註解中讀到的，**markViewDirty** 函式會將目前的視圖和所有祖先標記為髒。讓我們看看下面的圖片，以便更好地了解這意味著什麼。

![Image 75](https://miro.medium.com/v2/resize:fit:700/1*vWR8BluUbvyEdbezRbhYLw.png)

將元件及其祖先標記為髒，直到根元件

因此，當我們點擊按鈕時，Angular 會呼叫我們的回呼函式（_changeName_），並且因為它使用 **wrapListenerIn_markDirtyAndPreventDefault** 函式包裝，它會將元件標記為髒。

正如我們之前所說，Angular 使用 zone.js 並使用它包裝我們的應用程式。

![Image 76](https://miro.medium.com/v2/resize:fit:700/1*qlIO2Wh0s9dSGhchJ9rh7g.png)

NgZone 包裝 Angular 應用程式

在將髒標記傳遞到頂部後，**wrapListenerIn_markDirtyAndPreventDefault** 會觸發並觸發 zone.js

![Image 77](https://miro.medium.com/v2/resize:fit:700/1*DmRTN6SwNtGGqZA_E16Y_g.png)

事件監聽器通知 zone.js

因為 Angular 正在監聽 **onMicrotaskEmpty** observable，並且因為 **(click)** 會註冊一個事件監聽器，而該監聽器已由 zone 包裝，所以 zone 會知道事件監聽器已完成執行，並且可以向 **onMicrotaskEmpty** observable 發出一個值。

![Image 78](https://miro.medium.com/v2/resize:fit:700/1*I6ceCK5GilF6J65lvyqoAw.png)

當不再有任何微任務執行時，**onMicrotaskEmpty** 會觸發

**onMicrotaskEmpty** 告訴 Angular 是時候執行變更偵測了。

**元件綁定重新整理**
-----------------------------

當 Angular 執行變更偵測時，它會從上到下檢查每個元件。它會遍歷所有元件（**髒**和**非髒**）並檢查它們的綁定。如果綁定已變更，它會更新視圖。

![Image 79](https://miro.medium.com/v2/resize:fit:700/1*x08OitRwQ2qT7gIs3DF9bA.png)

但是，為什麼 Angular 要檢查所有元件 🤔？為什麼它不只檢查髒元件 🤔？

嗯，這是因為變更偵測策略。

**OnPush 變更偵測**
---------------------------

Angular 有一個名為 **OnPush** 的變更偵測策略。當我們使用此策略時，Angular 只會為標記為髒的元件執行變更偵測。

首先，讓我們將變更偵測策略變更為 **OnPush**。

```typescript
@Component({
  // ...
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserCard {}
```

讓我們看看下面的圖形，以便更好地了解 **OnPush** 策略的變更偵測是如何運作的。

![Image 80](https://miro.medium.com/v2/resize:fit:700/1*zELCaBs7IW92PvceLa_IOA.png)

現在，某些元件標記為 **OnPush（且其子元件隱式為 OnPush 元件）**

讓我們執行與之前相同的操作。點擊元件中的按鈕並變更名稱。

首先，我們會有**髒標記階段**。

![Image 81](https://miro.medium.com/v2/resize:fit:700/1*AeeeFm-_fHGjbFAACMr1hA.png)

然後，事件監聽器會通知 zone.js。

![Image 82](https://miro.medium.com/v2/resize:fit:700/1*iP28hqjA-YdazbKm-zg_tA.png)

事件通知 zone.js

當所有非同步執行完成後，**onMicrotaskEmpty** 會觸發。

![Image 83](https://miro.medium.com/v2/resize:fit:700/1*frosM42uC9BbaC8P3fIS5g.png)

現在，Angular 將執行 **tick** 方法，它將執行的是從上到下遍歷所有元件並檢查每個元件。

如果元件是：

*   **OnPush** + **非髒** -> **跳過**
*   **OnPush** + **髒** -> **檢查綁定** -> **重新整理綁定** -> **檢查子元件**

![Image 84](https://miro.medium.com/v2/resize:fit:700/1*Clr0BonprSL5sPAE-5JRTw.png)

正如我們所見，透過使用 **OnPush**，我們可以跳過我們知道沒有任何變更的樹狀結構部分。

**OnPush + Observables + async pipe**
-------------------------------------

當我們使用 Angular 時，observables 一直是我們管理資料和狀態變更的首選工具。為了支援 observables，Angular 提供了 **async** pipe。**async** pipe 訂閱一個 observable 並傳回最新值。為了讓 Angular 知道該值已變更，它會呼叫來自 **ChangeDetectorRef** 類別（元件的 **ChangeDetectorRef**）的 **markForCheck** 方法。

```typescript
@Pipe()
export class AsyncPipe implements OnDestroy, PipeTransform {
  constructor(ref: ChangeDetectorRef) {}
  transform<T>(obj: Observable<T>): T|null {
    // 為了簡潔起見，已移除程式碼
  }

private _updateLatestValue(async: any, value: Object): void {
    // 為了簡潔起見，已移除程式碼
    this._ref!.markForCheck(); // <\-- 將元件標記為檢查
  }
}
```

我在此處撰寫了更多關於它的內容（從頭開始建立 async pipe 並了解其運作方式）：

而 **markForCheck** 方法的作用是，它只會呼叫我們之前看到的 **markViewDirty** 函式。

```typescript
// view_ref.ts
markForCheck(): void {
  markViewDirty(this._cdRefInjectingView || this._lView);
}
```

因此，與之前相同，如果我們在範本中使用帶有 **async** pipe 的 observables，它的作用將與我們使用 **(click)** 事件時相同。它會將元件標記為髒，而 Angular 會執行變更偵測。

![Image 85](https://miro.medium.com/v2/resize:fit:700/1*LRpedou2RqKPanLTPRUbTQ.png)

data$ | async pipe 將元件標記為髒

OnPush + Observables + 誰在觸發 zone.js？
-------------------------------------------------

如果我們的資料在沒有我們互動（click、mouseover 等）的情況下發生變更，則很可能在底層的某處執行了 **setTimeout** 或 **setInterval** 或 HTTP 呼叫，從而觸發了 zone.js。

以下是如何輕鬆地破壞它的方法 🧨

```typescript
@Component({
  selector: 'todos',
  standalone: true,
  imports: [AsyncPipe, JsonPipe],
  template: `{{ todos$ | async | json }}`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class TodosComponent {
  private http = inject(HttpClient);
  private ngZone = inject(NgZone);

  todos$ = of([] as any[]);

  ngOnInit() {
    this.ngZone.runOutsideAngular(() => {
      setTimeout(() => {
        // 這將會更新，但沒有任何內容觸發 zonejs
        this.todos$ = this.getTodos();
      });
    });
  }

  getTodos() {
    return this.http
      .get<any>('https://jsonplaceholder.typicode.com/todos/1')
      .pipe(shareReplay(1));
  }
}
```

我們在這裡所做的是：

*   在 **ngOnInit 中，**我們使用了 **ngZone.runOutsideAngular()**，這是一個允許我們在 Angular 區域外執行操作的 API。
*   我們使用 **setTimeout**（跳過要執行的第一個任務，也因為 Angular 預設會至少執行一次變更偵測），並且在 **setTimeout** 內部，我們為 observable 指派一個值（耶，我們有變更）。
*   由於 **setTimeout** 不會在區域內執行，因此 API 呼叫也會在區域外進行，因為程式碼在 **runOutsideAngular** 內執行，因此沒有任何內容通知 zonejs 發生了變更。
*   在您的應用程式中執行此程式碼，並查看瀏覽器中只會顯示 **「[]」**。
*   **損壞的狀態 🧨！**

不太好 😄！但是，我們開始質疑的另一件事是：

**為什麼我們需要將所有祖先都標記為髒？**
---------------------------------------------------

原因很簡單，如果我們不將所有祖先都標記為髒，我們甚至可以更快地獲得損壞的狀態。如何？

讓我們再次看看上面的範例，但是現在，只將元件及其子元件標記為髒。

![Image 86](https://miro.medium.com/v2/resize:fit:700/1*WR3R89nCi2ej-J0JDBUKjg.png)

因此，我們只將點擊的元件及其子元件標記為檢查。當 **tick** 發生時，它會到達父元件，即 **OnPush**，檢查它是否未髒，並跳過它。

![Image 87](https://miro.medium.com/v2/resize:fit:700/1*eTO0ASGOnxsQm9sW-i6EVQ.png)

如果我們在使用 markForCheck 時不將祖先標記為髒，則會造成損壞的狀態

這就是我們再次進入損壞狀態的原因 🧨！

**為什麼我們不能只為標記為髒的元件執行變更偵測？**
-----------------------------------------------------------------------------------------

我們可以使用 **ChangeDetectorRef** 類別中的 **detectChanges** 方法來執行此操作。但它有其缺點。由於該方法會同步執行變更偵測，因此可能會導致效能問題。由於所有操作都將在同一個瀏覽器任務中完成，因此可能會封鎖主執行緒並導致卡頓。想像一下每隔 1 或 2 秒偵測 100 個項目的清單的變更。這對瀏覽器來說是一項繁重的工作。

**markForCheck vs detectChanges（合併執行 vs 同步執行）**
------------------------------------------------------------------------

當我們使用 markForCheck 時，我們只是告訴 Angular 一個元件是髒的，並且沒有其他事情發生，因此即使我們呼叫 markForCheck 1000 次，也不會成為問題。但是，當我們使用 **detectChanges** 時，Angular 會執行實際的工作，例如檢查綁定並在需要時更新視圖。這就是為什麼我們應該使用 **markForCheck** 而不是 **detectChanges** 的原因。

**我們是否可以安排** detectChanges **在下一個瀏覽器任務中執行？**
--------------------------------------------------------------------------

我們可以，這就是來自 [**rx-angular**](https://www.rx-angular.io/) 的 [**push pipe**](https://www.rx-angular.io/docs/template/api/push-pipe) 或 [**rxLet**](https://www.rx-angular.io/docs/template/api/rx-let-directive) 指令的作用。它會在下一個瀏覽器任務中安排變更偵測。但是，對每個元件執行此操作並不是一個好主意。因為，如果我們有 100 個項目的清單，並且我們為每個項目安排變更偵測，我們將會有 100 個瀏覽器任務。這對效能也沒有好處。

Signals 🚦
----------

前端世界正朝著 signals 發展。Solid.js、Svelte、Vue 和 Angular 正在建立其 signals 實作。這是因為 signals 是**管理狀態**和**狀態變更**的更好方法。

Angular 中的 Signals 帶來了很多 **DX** 好處。我們可以輕鬆建立和衍生狀態，並且還可以在狀態變更時使用 effects 執行副作用。我們不必訂閱它們，我們不必取消訂閱它們，我們也不必擔心記憶體洩漏 🧯。

我們只需呼叫它們，它們就會傳回目前的值。

```typescript
const name = signal('John'); // 建立具有初始值的 signal
const upperCaseName = computed(() => name().toUpperCase()); // 建立一個計算的 signal

effect(() => {
  console.log(name() + ' ' + upperCaseName()); // 當 name 或 upperCaseName 變更時執行副作用
});

name.set('Jane'); // 變更名稱

// 輸出：
// John JOHN
// Jane JANE
```

我們也可以在範本中使用 signals，就像一般的函式呼叫一樣。

```typescript
@Component({
  template: `
    <button (click)="name.set('Jane')">Change name</button>
    <p>{{ name() }}</p>
  `
})
export class AppComponent {
  name = signal('John');
}
```

如果你問在範本中呼叫函式是否是一個好主意，我會說如果函式呼叫很便宜，那麼這是一個好主意，而呼叫 signal 很便宜。它只是一個傳回值的函式呼叫（而不計算任何內容）。

我也在這裡寫了關於它的內容：

Signals 和變更偵測
----------------------------

在 v17 中，Angular 變更偵測得到了升級 🚀！

Angular 範本現在將 signals 理解為比函式呼叫更多的內容。以下是使其成為現實的 PR 之一。

在之前，我們使用 **async** pipe，因此它會呼叫 **markForCheck** 方法，而使用 signals，我們只需正常呼叫它們即可。Angular 現在會註冊一個 **effect**（_消費者_），它將監聽此 signal，並在每次 signal 變更時將範本標記為檢查。

![Image 88](https://miro.medium.com/v2/resize:fit:700/1*JvVZ73v8E5zXRzyCOnt3Vg.png)

第一個好處是我們不再需要 async pipe 了 🎉。

第二個改進變更偵測的 PR 是這個：

它解決了一個與 signals 無關但與變更偵測本身相關的問題（我不會詳細解釋）。

透過使用它引入的機制，我們得到了第 3 個 PR，它新增了 **Glo-cal** 變更偵測（全域 + 局部變更偵測）（由我的朋友 @[Matthieu Riegler](https://twitter.com/Jean__Meche) 創造的術語）

因此，讓我們更好地了解 glo-cal（局部）變更偵測 👇

**局部變更偵測（目標模式）**
------------------------------------------

我上面連結的其中一個 PR 在 Angular 中引入了兩個新旗標。

![Image 89](https://miro.medium.com/v2/resize:fit:522/1*gvQx3FEpysqIW8RpfX91Pw.png)

兩個新旗標（RefreshView 和 HAS_CHILD_VIEWS_TO_REFRESH）

它們如何運作？

當範本 effect 執行時，Angular 會執行一個名為 **markViewForRefresh** 的函式，該函式會將目前的元件旗標設定為 **RefreshView**，然後呼叫 **markAncestorsForTraversal**，它會將所有祖先標記為 **HAS_CHILD_VIEWS_TO_REFRESH**。

```typescript
/**
 * 從 lView 中新增 `RefreshView` 旗標，並更新父系的 HAS_CHILD_VIEWS_TO_REFRESH 旗標。
 */
export function markViewForRefresh(lView: LView) {
  if (lView[FLAGS] & LViewFlags.RefreshView) {
    return;
  }
  lView[FLAGS] |= LViewFlags.RefreshView;
  if (viewAttachedToChangeDetector(lView)) {
    markAncestorsForTraversal(lView);
  }
}
```

這就是它在圖形中的樣子（更新的樹狀結構以展示更多邊緣案例）👇

![Image 90](https://miro.medium.com/v2/resize:fit:700/1*DRDqyuI6pLlELqlA48m5vw.png)

因此，具有 signal 變更的元件會以橘色邊框標記，而父系現在具有 ⏬ 圖示，以告知它們具有要重新整理的子視圖。

> 注意：我們仍然依賴 zone.js 來觸發變更偵測。

當 zone.js 觸發時（與之前相同的原因），它會呼叫 **appRef.tick()**，然後我們將進行自上而下的變更偵測，其中會有一些差異和新規則！

目標模式規則
------------------

NgZone 以 **全域模式**觸發變更偵測（它會自上而下檢查和重新整理所有元件）

在 **全域模式**中，我們會檢查 **CheckAlways** _（未設定任何變更偵測策略的常規元件）_ 和 **髒的 OnPush** 元件

**什麼會觸發目標模式？**

*   當在 **全域模式**中遇到 **非髒的 OnPush** 元件時，我們會切換到 **目標模式！**

在 **目標模式**中：

*   僅當視圖設定了 **RefreshView** 旗標時才重新整理視圖
*   **請勿**重新整理 **CheckAlways** 或常規的 **髒** 旗標視圖
*   如果我們到達具有 **RefreshView** 旗標的視圖，則以 **全域模式**遍歷子元件

讓我們逐一了解元件。

1.  根元件是一個常規元件（**CheckAlways**），因此我們只需在需要時檢查和重新整理綁定，然後繼續檢查其子元件。

![Image 91](https://miro.medium.com/v2/resize:fit:700/1*MwEruHykzCIvzX901X4WOw.png)

2.  所有 **CheckAlways** 元件將繼續像以前一樣運作。

![Image 92](https://miro.medium.com/v2/resize:fit:700/1*JbNnofQxP7dnAqRpjmro_Q.png)

所有 CheckAlways 元件都會重新整理

3.  OnPush 將繼續像以前一樣運作，因此如果未將其標記為髒，則不會檢查它。

4.  如果我們檢查另一個 **OnPush** + **HAS_CHILD_VIEWS_TO_REFRESH** 但未髒的元件，我們會得到 **目標模式** 的觸發器（請檢查上述規則）

![Image 93](https://miro.medium.com/v2/resize:fit:700/1*XmdIz4ykbWYzPGWeAYfMqg.png)

6.  元件本身不會重新整理，讓我們前往子元件

![Image 94](https://miro.medium.com/v2/resize:fit:700/1*OltY0HzHDhj1MDLYEMabuw.png)

**CheckAlways** 元件上的 **目標模式** -> 跳過

7.  然後我們到達一個 **RefreshView** 元件，並且我們處於 **目標模式**，這表示我們會重新整理綁定。我們也會轉換為 **全域模式**，以確保 **CheckAlways** 子元件也能正確重新整理。

![Image 95](https://miro.medium.com/v2/resize:fit:700/1*67Qjlpzdqai93xdGgH8Img.png)

8.  現在我們處於 **全域模式**，並且我們有一個 **CheckAlways** 元件，因此我們只需正常重新整理）

![Image 96](https://miro.medium.com/v2/resize:fit:700/1*uvzUYErJ3jjenEzC9h2itA.png)

這就是所有關於新的目標變更偵測的內容。

如果我們查看最終樹狀結構，我們可以發現，當我們到達一個未髒的 OnPush 元件時，我們跳過了比以前更多的元件。

![Image 97](https://miro.medium.com/v2/resize:fit:700/1*IAhmVb58R8oaE-IrGid43A.png)

> 目標變更偵測 = 沒有「地雷」的 OnPush 🔫

你可以使用 [Mathieu Riegler](https://github.com/JeanMeche) 🔨 在這個應用程式中玩轉所有這些變更偵測規則

![Image 98](https://miro.medium.com/v2/resize:fit:700/1*eqLX13lrKYs_4tKk9uk-hQ@2x.png)

了解 Angular 變更偵測

**無區域的 Angular —** 讓我們從 Angular 中移除 zone.js
--------------------------------------------------------

當我們從 Angular 中移除 zone.js 時，我們剩下的程式碼會執行，但不會更新視圖中的任何內容（也會移除 zone.js 的引導時間及其在瀏覽器中施加的所有壓力！我們也從套件大小中移除了 `15kb` 😎）。因為沒有任何東西在觸發 **appRef.tick()**。

但是，Angular 有一些 API 會告訴它某些內容發生了變更。是哪些？

*   markForCheck（由 async pipe 使用）
*   signal 變更
*   將視圖標記為髒的事件處理常式
*   在使用 **setInput()** 動態建立的元件上設定輸入

此外，OnPush 元件已經使用它需要告訴 Angular 某些內容發生變更的想法來運作