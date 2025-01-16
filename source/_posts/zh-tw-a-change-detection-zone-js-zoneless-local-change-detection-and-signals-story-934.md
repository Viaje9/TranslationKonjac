---
title: A change detection, zone.js, zoneless, local change detection, and signals story 📚
tags:
---

原文：[https://itnext.io/a-change-detection-zone-js-zoneless-local-change-detection-and-signals-story-9344079c3b9d](https://itnext.io/a-change-detection-zone-js-zoneless-local-change-detection-and-signals-story-9344079c3b9d)

# 一個關於變更偵測、zone.js、無 Zone、局部變更偵測和 Signals 的故事 📚


![Image 71](https://miro.medium.com/v2/resize:fit:700/1*N0FoEeuERFDvvkuOzhgOJA.png)


Photo by [Mark Boss](https://unsplash.com/@vork?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/photos/traffic-light-gHkKgHX0fbE?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)

<!-- more -->

Angular 是一個元件驅動的框架。就像其他所有框架一樣，它應該向使用者顯示資料，並在資料變更時刷新視圖。

![Image 72](https://miro.medium.com/v2/resize:fit:667/1*eVpyKXodMU9thMSDn1XY4Q.png)

一個在範本中顯示使用者資料的 UserCard 元件

隨著時間的推移，我們建立越來越多的元件並將它們組合在一起，最終可能會得到如下的元件樹。

![Image 73](https://miro.medium.com/v2/resize:fit:700/1*QbEmmWPDvOTKw60oW-RKmw.png)

元件結構

但是，Angular 如何知道何時刷新視圖？它如何知道資料何時變更？它如何知道何時執行變更偵測？

**使用同步程式碼進行變更偵測**
------------------------------------------

讓我們先從一個簡單的範例開始。我們有一個具有 **name** 屬性和 **changeName** 方法的元件。當我們點擊按鈕時，我們呼叫 **changeName** 方法並變更 **name** 屬性。

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

當我們點擊按鈕時，會呼叫 **changeName** 方法，而且因為所有內容都由 Angular 包裝，我們可以安全地假設，在名稱變更後，Angular 可以執行一些程式碼來更新視圖（並且所有內容都將同步）。

> ⚠️ 想像中的 Angular 底層程式碼：

```typescript
component.changeName();
// 執行此程式碼 Angular 將為整個元件樹執行變更偵測，因為我們可能已在其他元件使用的服務中更新了一些資料
angular.runChangeDetection();
```

這樣可以正常運作！但是，大多數時候，當我們變更資料時，我們不會同步執行。我們通常會發出 HTTP 請求，或使用一些計時器，或等待其他事件發生後再更新資料。這就是問題開始的地方。

**使用非同步程式碼進行變更偵測**
-------------------------------------------

現在，假設我們想要在 1 秒後變更名稱。我們可以使用 **setTimeout** 函式來完成。

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

現在，讓我們加入與之前相同的想像中 Angular 底層程式碼。

> ⚠️ 想像中的 Angular 底層程式碼：

```typescript
component.changeName(); // 在內部使用 setTimeout
// 此程式碼在呼叫 changeName 方法後立即執行
angular.runChangeDetection();
```

由於 **call stack**，**setTimeout** 回呼將在 **angular.runChangeDetection** 方法之後呼叫。因此，Angular 執行了變更偵測，但名稱尚未變更。這就是為什麼視圖不會更新的原因。這是一個損壞的應用程式 ⚠️。（實際上並非如此，因為我們有 👇）

**Zone.js 來救援**
-------------------------

Zone.js 自 Angular 2.0 的早期就已存在。它是一個猴子補丁瀏覽器 API 的程式庫，使我們能夠加入瀏覽器事件的生命週期。這是什麼意思？這表示我們可以在瀏覽器事件之前和之後執行我們的程式碼。

```typescript
setTimeout(() => {
  console.log('Hello world');
}, 1000);
```

上面的程式碼會在 1 秒後印出 **Hello world**。但是，如果我們想在 **setTimeout** 回呼之前或之後執行一些程式碼呢？_你知道，出於商業原因_ 😄。一個稱為 **Angular** 的框架可能想在 **setTimeout** 回呼之前和之後執行一些程式碼。

**Zone.js** 使我們能夠做到這一點。我們可以建立一個 zone（Angular 也會建立一個 zone），並加入 **setTimeout** 回呼。

```typescript
const zone = Zone.current.fork({
  onInvokeTask: (delegate, current, target, task, applyThis, applyArgs) => {
    console.log('Before setTimeout');
    delegate.invokeTask(target, task, applyThis, applyArgs);
    console.log('After setTimeout');
  }
});
```

若要在 zone 內執行我們的 **setTimeout**，我們需要使用 **zone.run()** 方法。

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

這就是 zone.js 的運作方式。它會猴子補丁瀏覽器 API，並使我們能夠加入瀏覽器事件的生命週期。

**Zone.js + Angular**
---------------------

Angular 預設會在每個應用程式中載入 **zone.js**，並建立一個名為 **NgZone** 的 zone。**NgZone** 包含一個名為 **onMicrotaskEmpty** 的 **Observable**。當佇列中沒有其他微任務時，此 observable 會發出值。Angular 就是利用這一點來知道何時所有非同步程式碼都已執行完成，並且可以安全地執行變更偵測。

![Image 74](https://miro.medium.com/v2/resize:fit:700/1*3Oe8-a_qFIOnAGEZsfM3vg.png)

NgZone 今天包裝著每個 Angular 應用程式

讓我們看看底層的 [程式碼](https://github.com/angular/angular/blob/c4de4e1f894001d8f80b70297c5e576f2d11ec6f/packages/core/src/change_detection/scheduling/ng_zone_scheduling.ts#L31)：

```typescript
// ng_zone_scheduling.ts NgZoneChangeDetectionScheduler
this._onMicrotaskEmptySubscription = this.zone.onMicrotaskEmpty.subscribe({
    next: () => this.zone.run(() => this.applicationRef.tick())
});
```

我們在上面的程式碼中看到的是，當 **onMicrotaskEmpty** observable 發出值時，Angular 將呼叫 [**applicationRef.tick()**](https://github.com/angular/angular/blob/c4de4e1f894001d8f80b70297c5e576f2d11ec6f/packages/core/src/application/application_ref.ts#L537) 方法。這個 **tick** 方法是什麼 🤔？你還記得想像中的 Angular 底層程式碼中的 **runChangeDetection** 方法嗎？嗯，**tick** 方法就是 **runChangeDetection** 方法。它會為整個元件樹 **_同步_** 執行變更偵測。

但現在，Angular 知道所有非同步程式碼都已執行完成，並且可以 **_安全地_** 執行變更偵測。

```typescript
tick(): void {
    // 為了簡潔起見，已移除程式碼
    for (let view of this._views) {
        // 為單一元件執行變更偵測
        view.detectChanges();
    }
}
```

**tick** 方法會反覆運算所有根視圖（大多數時候我們只有一個根視圖/元件，即 **AppComponent**），並同步執行 **detectChanges**。

**元件的髒標記**
---------------------------

Angular 所做的另一件事是，當它知道元件內部的某些內容已變更時，會將元件標記為髒。

以下是將元件標記為髒的內容：

*   事件（點擊、滑鼠移過等等）

每次我們點擊範本中帶有監聽器的按鈕時，Angular 都會使用一個名為 [**wrapListenerIn\_markDirtyAndPreventDefault**](https://github.com/angular/angular/blob/c4de4e1f894001d8f80b70297c5e576f2d11ec6f/packages/core/src/render3/instructions/listener.ts#L260) 的函式來包裝回呼函式。正如我們從函式名稱 😅 可以看到的那樣，它會將元件標記為髒。

```typescript
function wrapListener(): EventListener {
  return function wrapListenerIn_markDirtyAndPreventDefault(e: any) {
    // ... 為了簡潔起見，已移除程式碼
    markViewDirty(startView); // 將元件標記為髒
  };
}
```

*   已變更的輸入

此外，在執行變更偵測時，Angular 將檢查元件的輸入值是否已變更（**\===** 檢查）。如果已變更，它會將元件標記為髒。[此處提供原始碼](https://github.com/angular/angular/blob/c4de4e1f894001d8f80b70297c5e576f2d11ec6f/packages/core/src/render3/component_ref.ts#L348)。

```typescript
setInput(name: string, value: unknown): void {
    // 如果輸入與最後的值相同，則不要設定輸入
    if (Object.is(this.previousInputValues.get(name), value)) {
        return;
    }
    // 為了簡潔起見，已移除程式碼
    setInputsForProperty(lView[TVIEW], lView, dataValue, name, value);
    markViewDirty(childComponentLView); // 將元件標記為髒
}
```

*   輸出發射

若要在 Angular 中監聽輸出發射，我們會在範本中註冊事件。正如我們之前所看到的，回呼函式將被包裝，並且當事件發射時，元件將被標記為髒。

讓我們看看這個 [**markViewDirty**](https://github.com/angular/angular/blob/c4de4e1f894001d8f80b70297c5e576f2d11ec6f/packages/core/src/render3/instructions/mark_view_dirty.ts#L24) 函式的作用。

```typescript
/**
 * 將目前的視圖和所有祖先標記為髒。
 */
export function markViewDirty(lView: LView): LView|null {
  while (lView) {
    lView[FLAGS] |= LViewFlags.Dirty;
    const parent = getLViewParent(lView);
    // 當您找到未附加到任何容器的根視圖時，停止向上遍歷
    if (isRootView(lView) && !parent) {
      return lView;
    }
    // 否則繼續
    lView = parent!;
  }
  return null;
}
```

正如我們從註解中讀到的，**markViewDirty** 函式會將目前的視圖和所有祖先標記為髒。讓我們看看下面的影像，以便更好地了解這表示什麼。

![Image 75](https://miro.medium.com/v2/resize:fit:700/1*vWR8BluUbvyEdbezRbhYLw.png)

將元件及其祖先標記為髒到根

因此，當我們點擊按鈕時，Angular 會呼叫我們的回呼函式 ( _changeName_ )，並且因為它使用 **wrapListenerIn\_markDirtyAndPreventDefault** 函式包裝，它會將元件標記為髒。

正如我們之前所說，Angular 使用 zone.js 並使用它包裝我們的應用程式。

![Image 76](https://miro.medium.com/v2/resize:fit:700/1*qlIO2Wh0s9dSGhchJ9rh7g.png)

NgZone 包裝著 Angular 應用程式

在將髒標記到最上層之後，**wrapListenerIn\_markDirtyAndPreventDefault** 會觸發並觸發 zone.js

![Image 77](https://miro.medium.com/v2/resize:fit:700/1*DmRTN6SwNtGGqZA_E16Y_g.png)

事件監聽器通知 zone.js

因為 Angular 正在監聽 **onMicrotaskEmpty** observable，並且因為 **(click)** 註冊了一個事件監聽器，而 zone 已包裝了此事件監聽器，所以 zone 會知道事件監聽器已完成執行，並且可以向 **onMicrotaskEmpty** observable 發出一個值。

![Image 78](https://miro.medium.com/v2/resize:fit:700/1*I6ceCK5GilF6J65lvyqoAw.png)

當沒有微任務在執行時，**onMicrotaskEmpty** 會觸發

**onMicrotaskEmpty** 告訴 Angular 是時候執行變更偵測了。

**元件繫結刷新**
---------------------------

當 Angular 執行變更偵測時，它會從上到下檢查每個元件。它會檢查所有元件（**髒**和**非髒**），並檢查它們的繫結。如果繫結已變更，它會更新視圖。

![Image 79](https://miro.medium.com/v2/resize:fit:700/1*x08OitRwQ2qT7gIs3DF9bA.png)

但是，為什麼 Angular 要檢查所有元件 🤔？為什麼不只檢查髒的元件 🤔？

嗯，這是因為變更偵測策略。

**OnPush 變更偵測**
---------------------------

Angular 有一個名為 **OnPush** 的變更偵測策略。當我們使用此策略時，Angular 將僅針對標記為髒的元件執行變更偵測。

首先，讓我們將變更偵測策略變更為 **OnPush**。

```typescript
@Component({
  // ...
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserCard {}
```

讓我們看看下面的圖形，以便更好地了解 **OnPush** 策略的變更偵測運作方式。

![Image 80](https://miro.medium.com/v2/resize:fit:700/1*zELCaBs7IW92PvceLa_IOA.png)

現在有些元件標記為 **OnPush（及其子元件隱含地為 OnPush 元件）**

讓我們做與之前相同的事情。點擊元件中的按鈕並變更名稱。

首先，我們將有**髒標記階段**。

![Image 81](https://miro.medium.com/v2/resize:fit:700/1*AeeeFm-_fHGjbFAACMr1hA.png)

然後，事件監聽器將通知 zone.js。

![Image 82](https://miro.medium.com/v2/resize:fit:700/1*iP28hqjA-YdazbKm-zg_tA.png)

事件通知 zone.js

當所有非同步內容都執行完成後，**onMicrotaskEmpty** 將會觸發。

![Image 83](https://miro.medium.com/v2/resize:fit:700/1*frosM42uC9BbaC8P3fIS5g.png)

現在，Angular 將執行 **tick** 方法，它將從上到下遍歷所有元件並檢查每個元件。

如果元件是：

*   **OnPush** + **非髒** -> **略過**
*   **OnPush** + **髒** -> **檢查繫結** -> **刷新繫結** -> **檢查子元件**

![Image 84](https://miro.medium.com/v2/resize:fit:700/1*Clr0BonprSL5sPAE-5JRTw.png)

正如我們所看到的，藉由使用 **OnPush**，我們可以略過我們知道沒有任何變更的樹狀結構部分。

**OnPush + Observables + async pipe**
-------------------------------------

當我們使用 Angular 時，observables 一直是我們管理資料和狀態變更的首選工具。為了支援 observables，Angular 提供了 **async** pipe。**async** pipe 會訂閱 observable 並傳回最新的值。為了讓 Angular 知道值已變更，它會呼叫來自 **ChangeDetectorRef** 類別（元件的 **ChangeDetectorRef**）的 **markForCheck** 方法。

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

我在此處撰寫了更多相關資訊（從頭開始建立 async pipe 並了解它的運作方式）：

**markForCheck** 方法的作用是，它只會呼叫我們之前看到的 **markViewDirty** 函式。

```typescript
// view_ref.ts
markForCheck(): void {
  markViewDirty(this._cdRefInjectingView || this._lView);
}
```

因此，與之前相同，如果我們在範本中使用帶有 **async** pipe 的 observables，它的作用方式會與我們使用 **(click)** 事件時相同。它會將元件標記為髒，而 Angular 會執行變更偵測。

![Image 85](https://miro.medium.com/v2/resize:fit:700/1*LRpedou2RqKPanLTPRUbTQ.png)

data$ | async pipe 將元件標記為髒

OnPush + Observables + 誰在觸發 zone.js？
-------------------------------------------------

如果我們的資料在沒有我們互動的情況下變更（點擊、滑鼠移過等等），則可能在底層的某處有 **setTimeout** 或 **setInterval** 或正在發出的 HTTP 呼叫觸發了 zone.js。

以下是我們可以輕鬆破壞它的方式 🧨

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

我們在這裡所做的事情是：

*   在 **ngOnInit** 中，我們使用了 **ngZone.runOutsideAngular()**，這是一個允許我們在 Angular zone 之外執行內容的 API。
*   我們使用 **setTimeout**（略過第一個要執行的任務，而且因為 Angular 預設至少會執行一次變更偵測），並且在 **setTimeout** 內部，我們將值指派給 observable（耶，我們有變更）。
*   因為 **setTimeout** 不會在 zone 內執行，所以 API 呼叫也會在 zone 之外執行，因為程式碼在 **runOutsideAngular** 內部執行，所以沒有任何內容通知 zonejs 某些內容已變更。
*   在您的應用程式中執行此程式碼，並查看瀏覽器中只會顯示 "**\[\]**"。
*   **損壞的狀態 🧨！**

不太好 😄！但是，我們開始質疑的另一件事是：

**為什麼我們需要將所有祖先標記為髒？**
---------------------------------------------------

這樣做的原因很簡單，如果我們不將所有祖先標記為髒，我們甚至可以更快地進入損壞的狀態。如何？

讓我們再次查看上面的範例，但現在，只將元件及其子元件標記為髒。

![Image 86](https://miro.medium.com/v2/resize:fit:700/1*WR3R89nCi2ej-J0JDBUKjg.png)

因此，我們只將發生點擊的元件及其子元件標記為檢查。**tick** 發生時，它將到達身為 **OnPush** 的父元件，檢查它是否未變更，然後略過它。

![Image 87](https://miro.medium.com/v2/resize:fit:700/1*eTO0ASGOnxsQm9sW-i6EVQ.png)

如果我們在使用 markForCheck 時不將祖先標記為髒，則會損壞狀態

這就是我們如何再次進入損壞狀態 🧨！

**為什麼我們不能只針對標記為髒的元件執行變更偵測？**
-----------------------------------------------------------------------------------------

我們可以使用 **ChangeDetectorRef** 類別中的 **detectChanges** 方法來做到這一點。但是它有其缺點。因為該方法會同步執行變更偵測，所以可能會導致效能問題。因為所有內容都將在相同的瀏覽器任務中完成，所以可能會封鎖主執行緒並導致卡頓。想像一下，每 1 或 2 秒偵測 100 個項目的清單的變更。這對瀏覽器來說是很多工作。

**markForCheck 與 detectChanges (合併執行與同步執行)**
--------------------------------------------------------------------------

當我們使用 markForCheck 時，我們只會告訴 Angular 有一個元件是髒的，不會發生其他任何事情，因此即使我們呼叫 markForCheck 1000 次，也不會有問題。但是，當我們使用 **detectChanges** 時，Angular 會執行實際工作，例如檢查繫結，並在需要時更新視圖。這就是為什麼我們應該使用 **markForCheck** 而不是 **detectChanges** 的原因。

**我們是否無法在下一個瀏覽器任務中排定** detectChanges **的行程？**
--------------------------------------------------------------------------------------

我們可以，這就是 [**push pipe**](https://www.rx-angular.io/docs/template/api/push-pipe) 或 [**rxLet**](https://www.rx-angular.io/docs/template/api/rx-let-directive) 指示詞（來自 [**rx-angular**](https://www.rx-angular.io/)）的作用。它會在下一個瀏覽器任務中排定變更偵測的行程。但是，針對每個元件執行此操作並不是一個好主意。因為如果我們有 100 個項目的清單，並且我們為每個項目排定變更偵測的行程，我們將有 100 個瀏覽器任務。這對效能也不好。

Signals 🚦
----------

前端世界正朝向 signals 發展。Solid.js、Svelte、Vue 和 Angular 正在建立其 signal 實作。這是因為 signals 是**管理狀態**和**狀態變更**的更好方式。

Angular 中的 Signals 帶來了許多 **DX** 優勢。我們可以輕鬆建立和衍生狀態，並在狀態使用 effects 變更時執行副作用。我們不必訂閱它們，不必取消訂閱它們，也不必擔心記憶體流失 🧯。

我們只需要呼叫它們，它們就會傳回目前的值。

```typescript
const name = signal('John'); // 使用初始值建立 signal
const upperCaseName = computed(() => name().toUpperCase()); // 建立計算的 signal
effect(() => {
  console.log(name() + ' ' + upperCaseName()); // 當名稱或 upperCaseName 變更時，執行副作用
});

name.set('Jane'); // 變更名稱

// 輸出：
// John JOHN
// Jane JANE
```

我們也可以在範本中使用 signals，就像正常的函式呼叫一樣。

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

如果您問說，在範本中呼叫函式是否是個好主意，我會說，如果函式呼叫很便宜，那就是個好主意，而呼叫 signal 很便宜。它只是一個傳回值的函式呼叫（沒有計算任何內容）。

我在此處也撰寫了相關資訊：

Signals 和變更偵測
----------------------------

在 v17 中，Angular 的變更偵測得到了升級 🚀！

Angular 範本現在將 signals 理解為不只是函式呼叫。以下是使此成為現實的其中一個 PR。

在我們使用 **async** pipe 之前，它會呼叫 **markForCheck** 方法，而使用 signals，我們只需要正常呼叫它們即可。Angular 現在將註冊一個 **effect** ( _consumer_ )，它將監聽此 signal，並在每次 signal 變更時將範本標記為檢查。

![Image 88](https://miro.medium.com/v2/resize:fit:700/1*JvVZ73v8E5zXRzyCOnt3Vg.png)

第一個好處是我們不再需要 async pipe 🎉。

改進變更偵測的第二個 PR 是這個：

它解決了一個與 signals 無關但與變更偵測本身相關的問題（我將不會詳細說明）。

藉由使用它引入的機制，我們得到了第三個 PR，它加入了 **Glo-cal** 變更偵測（全域 + 局部變更偵測）（我的朋友 @[Matthieu Riegler](https://twitter.com/Jean__Meche) 創造的詞語）

因此，讓我們更好地了解 glo-cal (局部) 變更偵測 👇

**局部變更偵測（目標模式）**
------------------------------------------

我上面連結的其中一個 PR 在 Angular 中引入了兩個新的旗標。

![Image 89](https://miro.medium.com/v2/resize:fit:522/1*gvQx3FEpysqIW8RpfX91Pw.png)

兩個新的旗標 (RefreshView 和 HAS\_CHILD\_VIEWS\_TO\_REFRESH)

它們如何運作？

當範本 effect 執行時，Angular 會執行一個名為 **markViewForRefresh** 的函式，該函式會將目前元件的旗標設定為 **RefreshView**，然後呼叫 **markAncestorsForTraversal**，它會將所有祖先標記為 **HAS\_CHILD\_VIEWS\_TO\_REFRESH**。

```typescript
/**
 * 從 lView 新增 `RefreshView` 旗標，並更新父系的 HAS_CHILD_VIEWS_TO_REFRESH 旗標。
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

以下是它在圖形中的外觀（已更新樹狀結構以展示更多邊緣案例）👇

![Image 90](https://miro.medium.com/v2/resize:fit:700/1*DRDqyuI6pLlELqlA48m5vw.png)

因此，具有 signal 變更的元件會以橘色邊框標記，而父元件現在具有 ⏬ 圖示，表示它們有子視圖需要刷新。

> 注意：我們仍然依賴 zone.js 來觸發變更偵測。

當 zone.js 啟動時（與之前的原因相同），它將呼叫 **appRef.tick()**，然後我們將會有自上而下的變更偵測，其中包含一些差異和新規則！

目標模式規則
-----------------

NgZone 在 **GlobalMode** 中觸發變更偵測（它將自上而下檢查並刷新所有元件）

在 **GlobalMode** 中，我們會檢查 **CheckAlways** _(沒有設定任何變更偵測策略的正常元件)_ 和 **Dirty OnPush** 元件

**什麼觸發了 TargetedMode？**

*   當在 **GlobalMode** 中，我們遇到 **Non-Dirty OnPush** 元件時，我們會切換到 **TargetedMode！**

在 **TargetedMode 中：**

*   僅當視圖設定了 **RefreshView** 旗標時才刷新視圖
*   **請勿**刷新 **CheckAlways** 或一般的 **Dirty** 旗標視圖
*   如果我們到達具有 **RefreshView** 旗標的視圖，則以 **GlobalMode** 遍歷子元件

讓我們逐一查看元件。

1.  根元件是一個正常元件 (**CheckAlways**)，因此我們只在需要時檢查並刷新繫結，然後繼續到其子元件。

![Image 91](https://miro.medium.com/v2/resize:fit:700/1*MwEruHykzCIvzX901X4WOw.png)

2\. 所有 **CheckAlways** 元件將繼續以與之前相同的方式運作。

![Image 92](https://miro.medium.com/v2/resize:fit:700/1*JbNnofQxP7dnAqRpjmro_Q.png)

所有 CheckAlways 元件都會刷新

3\. OnPush 將繼續以相同的方式運作，因此如果未標記為髒，則不會檢查它。

4\. 如果我們檢查另一個是 **OnPush** + **HAS\_CHILD\_VIEWS\_TO\_REFRESH** 但不是髒的元件，我們會觸發 **TargetedMode**（檢查上面的規則）

![Image 93](https://miro.medium.com/v2/resize:fit:700/1*XmdIz4ykbWYzPGWeAYfMqg.png)

6\. 元件本身不會刷新，讓我們前往子元件

![Image 94](https://miro.medium.com/v2/resize:fit:700/1*OltY0HzHDhj1MDLYEMabuw.png)

**TargetedMode** 在 **CheckAlways** 元件上 -> 略過

7\. 然後我們到達一個 **RefreshView** 元件，我們處於 **TargetedMode**，這表示我們刷新繫結。我們也會轉換為 **GlobalMode**，以確保 **CheckAlways** 子元件也正確刷新。

![Image 95](https://miro.medium.com/v2/resize:fit:700/1*67Qjlpzdqai93xdGgH8Img.png)

8\. 現在我們處於 **GlobalMode**，我們有一個 **CheckAlways** 元件，因此我們只需正常刷新即可）

![Image 96](https://miro.medium.com/v2/resize:fit:700/1*uvzUYErJ3jjenEzC9h2itA.png)

這就是關於新的目標變更偵測的全部內容。

如果我們查看最終樹狀結構，我們可以看到，當我們到達一個不是髒的 OnPush 元件時，我們略過的元件比以前更多。

![Image 97](https://miro.medium.com/v2/resize:fit:700/1*IAhmVb58R8oaE-IrGid43A.png)

> 目標變更偵測 = 沒有陷阱的 OnPush 🔫

您可以在此應用程式中體驗所有這些變更偵測規則，此應用程式由 [Mathieu Riegler](https://github.com/JeanMeche) 🔨 提供

![Image 98](https://miro.medium.com/v2/resize:fit:700/1*eqLX13lrKYs_4tKk9uk-hQ@2x.png)

了解 Angular 變更偵測

**無 Zone Angular —** 讓我們從 Angular 中移除 zone.js
--------------------------------------------------------

當我們從 Angular 中移除 zone.js 時，我們會留下可以執行的程式碼，但不會更新視圖中的任何內容（zone.js 的引導時間及其在瀏覽器中施加的所有壓力也會被移除！我們還從套件大小中移除了 `15kb` 😎）。因為沒有任何內容觸發 **appRef.tick()**。

但是，Angular 有一些 API 可以告知它某些內容已變更。哪些？

*   markForCheck（由 async pipe 使用）
*   signal 變更
*   將視圖標記為髒的事件處理常式
*   在使用 **setInput()

設定以動態方式建立的元件的輸入

此外，OnPush 元件已經使用以下概念運作：它需要告知 Angular 某些內容已變更。

因此，我們可以讓 Angular 在知道某些內容已變更時排定 **tick()** 的行程，而不是讓 zone.js 排定 **tick()** 的行程。

在此 PR（**實驗性**）中，我們可以看見 **markViewDirty** 現在將通知 **changeDetectionScheduler** 某些內容已變更。

```typescript
export function markViewDirty(lView: LView): LView|null {
  lView[ENVIRONMENT].changeDetectionScheduler?.notify();
  // ... 為了簡潔起見，已移除程式碼
}
```

排程器應該排定 tick() 的行程，正如我們可以在此 zoneless 排程器的其他 **實驗性** 實作中看到的那樣。

```typescript
@Injectable({providedIn: 'root'})
class ChangeDetectionSchedulerImpl implements ChangeDetectionScheduler {
  private appRef = inject(ApplicationRef);
  private taskService = inject(PendingTasks);
  private pendingRenderTaskId: number|null = null;

  notify(): void {
    if (this.pendingRenderTaskId !== null) return;

    this.pendingRenderTaskId = this.taskService.add();
    setTimeout(() => {
      try {
        if (!this.appRef.destroyed) {
          this.appRef.tick();
        }
      } finally {
        const taskId = this.pendingRenderTaskId!;
        this.pendingRenderTaskId = null;
        this.taskService.remove(taskId);
      }
    });
  }
}
```

這是實驗性的，但我們可以看到，基本上，它會合併所有 notify() 呼叫，並僅執行一次（在此程式碼中，每個巨任務（setTimeout）僅執行一次，但也許我們每個微任務（Promise.resolve()）只執行一次）

**我們應該從中了解什麼？**

目前使用 OnPush 變更偵測策略的應用程式在無 Zone Angular 世界中將正常運作。

**無 Zone Angular !== Glo-cal (局部) 變更偵測**
---------------------------------------------------------

無 Zone Angular 與局部變更偵測不同。無 Zone Angular 只是從 Angular 中移除 **zone.js**，並使用 Angular 已擁有的 API 來排定 tick() 的行程。

**真正的局部變更偵測**是一個新功能，它將允許我們僅針對目前使用 OnPush 變更偵測策略的元件子樹（而不是整個元件樹）執行變更偵測。

**Signals 變更偵測（無 OnPush、無 Zone.js、僅限 Signals）**
------------------------------------------------------------------

Signals 變更偵測將帶來的一件事是原生單向資料流（雙向資料繫結，無任何問題）。

觀看此影片 **Rethinking Reactivity w/ Alex Rickabaugh | Angular Nation**

雖然使用 OnPush 和無 Zone 的 glo-cal 變更偵測很棒，但僅使用 signal 元件，我們可能會做得更好。

如果我們不必使用 **OnPush** 呢？或是使用 **HAS\_CHILD\_VIEWS\_TO\_REFRESH** 標記父項，並針對整個元件樹執行變更偵測呢？如果我們只能針對已變更的元件內部的視圖執行變更偵測呢？

在 RFC 中閱讀更多資訊：

**2024 年新年快樂！🎉 Angular 的 Signals 年 🚦！**
------------------------------------------------------------

今年，Angular 為我們帶來了許多新功能。而且我確信明年會更好。

請務必查看

*   [2024 年的 Angular 路線圖](https://angular.dev/roadmap) 🎉
*   [2023 年的 Angular 聖誕節日曆](https://angularchristmascalendar.com/)

應有的功勞：
-----------------------

*   [Alex Rickabaugh](https://twitter.com/synalx) 回答了我所有關於 Angular 的問題
*   [Andrew Scott](https://twitter.com/AScottAngular) 至今為止他建立的每個 PR
*   [Chau Tran](https://twitter.com/Nartc1410) 和 [Matthieu Riegler](https://twitter.com/Jean__Meche) 關於 Angular 內部的討論
*   [Michael Hladky](https://twitter.com/Michael_Hladky) [Julian Jandl](https://twitter.com/hoebbelsB) [Kirill Karnaukhov](https://twitter.com/kh_kirill) [Edouard Bozon](https://twitter.com/edbzn) [Lars Gyrup Brink Nielsen](https://twitter.com/LayZeeDK) (RxAngular 團隊)
*   整個 Angular 團隊所做的精彩工作！

更擅長現代 Angular ⚡️
-----------------------------

掌握最新的功能，以建構現代應用程式。了解如何使用獨立元件、函式守衛和攔截器、signals、新的 inject 方法等等。

🔗 [來自 Push-Based.io 的現代 Angular 工作坊](https://push-based.io/workshop/modern-angular)

![Image 99](https://miro.medium.com/v2/resize:fit:700/1*DOmhfx3sEzAXtWrLaibVlg@2x.png)

現代 Angular 工作坊議程

感謝您的閱讀！
-------------------

我經常在推特和部落格上發布關於 **Angular** 的文章（最新消息、signals、影片、podcast、更新、RFC、pull request 等等）。💎

如果這篇文章對您而言有趣且有用，並且您想了解更多關於 Angular 的資訊，請追蹤我的 [@Enea\_Jahollari](https://twitter.com/Enea_Jahollari) 或 [Medium](https://eneajahollari.medium.com/)。📖

如果您能[買一杯咖啡支持我](https://ko-fi.com/eneajahollari) ☕️，我會很感激。先感謝您 🙌
