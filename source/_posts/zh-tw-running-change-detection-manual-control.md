---
title: Running change detection – Manual control
date: 2025-01-04 20:57:03
tags:
---

原文：[https://angular.love/running-change-detection-manual-control](https://angular.love/running-change-detection-manual-control)

# 在 Angular 中執行變更偵測 - 應用程式中手動控制偵測

手動控制
--------------

即使 Angular 會自動執行變更偵測，有時你可能需要手動執行變更偵測。
如果元件的變更偵測器與元件視圖分離，或更新發生在 Angular zone 之外，就會發生這種情況。

Angular 有兩種手動觸發變更偵測的方式：

*   透過 [ApplicationRef](https://github.com/angular/angular/blob/99723fc64ed6ded3635ae5e19648550dc42f1bc1/packages/core/src/application_ref.ts) 呼叫 [tick](https://github.com/angular/angular/blob/ff84c7360334792a9a422bd0fa0353e0cf670525/packages/core/src/application_ref.ts#L1001)
*   透過 [ChangeDetectorRef](https://github.com/angular/angular/blob/c14c701775c900ce9ac8781c08fc76da067910c5/packages/core/src/change_detection/change_detector_ref.ts#L63) 呼叫 [detectChanges](https://github.com/angular/angular/blob/a98fa489fb498273bdfde2fa4598893d1d53582e/packages/core/src/render3/view_ref.ts#L273)

<!-- more -->

`tick` 方法會從根元件開始為整個應用程式執行變更偵測，
而 `detectChanges` 會執行局部變更偵測，從對應於 `ChangeDetectorRef` 實例的元件開始，並向下進行。

讓我們更詳細地探討這兩種方法。

tick
----

當 Angular 從 `NgZone` 收到關於沒有未完成的微任務的通知時，
Angular [使用此方法](https://github.com/angular/angular/blob/ff84c7360334792a9a422bd0fa0353e0cf670525/packages/core/src/application_ref.ts#L769)
來執行應用程式範圍的變更偵測：

```typescript
export class ApplicationRef {
  constructor() {
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

但是 `tick` 方法本身與 zone 或 `NgZone` 沒有任何關係。
它只是為整個應用程式調用變更偵測。

如果我們[深入研究](https://github.com/angular/angular/blob/ff84c7360334792a9a422bd0fa0353e0cf670525/packages/core/src/application_ref.ts#L1001)，
可以看到它只是簡單地迭代所有頂層（根）視圖，並在每個視圖上觸發 `detectChanges`：

```typescript
export class ApplicationRef {
	tick(): void {
	  try {
	    this._runningTick = true;
	    for (let view of this._views) {
	      view.detectChanges();
	    }
	    if (typeof ngDevMode === 'undefined' || ngDevMode) {
	      for (let view of this._views) {
	        view.checkNoChanges();
	      }
	    }
	  } catch (e) { ... } finally { ... }
	}
}
```

我們還可以發現，在開發模式下，`tick` 還會執行 `checkNoChanges`，
這會執行第二次變更偵測週期，以確保沒有偵測到進一步的變更。
如果在第二次週期中檢測到其他變更，
則表示應用程式中的綁定具有無法在單一變更偵測過程中解決的副作用。
在這種情況下，Angular 會拋出錯誤 ExpressionChanged 錯誤，
因為 Angular 會強制執行[單向數據流](https://angular.love/change-detection-big-picture-unidirectional-data-flow/)。

為了查看 `tick` 方法的實際應用，讓我們使用以下範例：

```typescript
@Component({
  selector: 'i-cmp',
  template: `
    {{ title }}
    <button (click)="changeName()">Change name</button>
  `,
})
export class I {
  title = 'Original';

  changeName() {
    this.title = 'Updated';
  }
}
```

這裡我們有一個點擊處理程序，可以更新 `title` 屬性。沒什麼特別的。
現在讓我們像這樣在 `main.ts` 中禁用 `NgZone`：

```typescript
platformBrowserDynamic().bootstrapModule(AppModule, { ngZone: 'noop' });
```

當我們運行程式碼時，我們可以發現當我們點擊按鈕時，螢幕不會更新：

![Image 25: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/1-94.gif)

現在讓我們注入 `ApplicationRef` 並在處理程序內部觸發 `tick` 方法：

```typescript
@Component({
  selector: 'i-cmp',
  template: `
    {{ title }}
    <button (click)="changeName()">Change name</button>
  `,
})
export class I {
  constructor(private appRef: ApplicationRef) {}

  title = 'Original';

  changeName() {
    this.title = 'Updated';
    this.appRef.tick();
  }
}
```

讓我們測試一下：

![Image 26: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/2.gif)

透過此變更，一切都如預期般運作。

有時你可能會看到建議使用 `NgZone.run` 作為全局執行變更偵測的方法。
但是，正如 [使用 zones 自動執行](https://angular.love/en/running-change-detection-autorun-with-zones/)
章節中所解釋的，`run` 方法只是在 Angular zone 內部評估回呼函式。內部沒有顯式調用 `ApplicationRef.tick()`。
這表示如果當回呼函式執行完成時沒有來自 Angular zone 的事件通知，
變更偵測將不會自動發生。

detectChanges
-------------

此方法在 Angular 為每個元件建立的變更偵測器服務上可用。
它用於顯式處理變更偵測及其對元件樹的副作用，
從你在其上觸發 `detectChanges()` 的元件開始。

這種所謂的**局部變更偵測週期**在許多情況下都很有用，除了在防止自動檢查時手動觸發變更偵測之外。
例如，如果你在具有比後代更多的祖先的元件中變更狀態，你可能會
通過使用 `detectChanges()` 獲得效能提升，因為你不會在元件的祖先上不必要地運行變更偵測。
另一個案例是分離的變更偵測器，我們將在
[分離視圖](https://angular.love/en/running-change-detection-detached-views/)
章節中詳細探討。

在底層，`detectChanges` 只是調用
[refreshView](https://github.com/angular/angular/blob/5f9c7ceb907be47dff3e203dd837fd6ee9133fcb/packages/core/src/render3/instructions/shared.ts#L354)
函式，我們在
[操作](https://angular.love/en/change-detection-big-picture-operations/)
章節中簡要探討過：

```typescript
export class ViewRef implements ChangeDetectorRef_interface {
  constructor(public _lView: LView, ...) {}

  detectChanges(): void {
    detectChangesInternal(this._lView[TVIEW], this._lView, this.context);
  }
}

export function detectChangesInternal(tView, lView, context, ...) {
  try {
    refreshView(tView, lView, tView.template, context);
  } catch (error) {.... } finally {... }
}
```

正如你從上面的程式碼片段中看到的，
變更偵測器服務基本上是透過
[LView](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L79)
實現的元件容器的淺層包裝。
當為元件建立 `ViewRef` 時，與元件關聯的 `LView` 會被注入到建構子中。
當為嵌入式視圖建立 `ViewRef` 時，它接收的 `LView` 也描述嵌入式視圖，而不是元件。

以下是如何想像兩個 `A` 元件實例的層次結構：

![Image 27: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/3.png)

對於每個[視圖類型](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L492)，Angular 實現了 `ViewRef` 的不同子類型：
在下面的螢幕截圖中，我們可以看到 `ViewRef` 用於元件視圖，
`EmbeddedViewRef` 用於嵌入式視圖，而 `InternalViewRef` 用於根/主機視圖：

![Image 28: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/4.png)

使用變更偵測器服務
-----------------------------

為了查看 `detectChanges` 方法的實際應用，讓我們使用之前帶有按鈕的範例。
這次，我們將注入 `ChangeDetectorRef` 而不是 `ApplicationRef`：

```typescript
export class I {
  constructor(private cdRef: ChangeDetectorRef) {}

  title = 'Original';

  changeName() {
    this.title = 'Updated';
    this.cdRef.detectChanges();
  }
}
```

讓我們測試一下：

![Image 29: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/5.gif)

如你所見，一切都如預期般運作。

這次與使用 `ApplicationRef.tick()` 的唯一區別在於變更偵測不會為祖先元件運行，
特別是對於根 `AppComponent`：

```typescript
@Component({
  selector: 'app-root',
  template: `<i-cmp></i-cmp>`,
})
export class AppComponent {}
```

如果我們只是將日誌記錄行為新增至 `refreshView` 函式，這很容易檢查：

![Image 30: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/6.png)

我在這裡使用條件斷點而不是日誌點，因為
我不希望在視圖類型為根時記錄輸出。我們只對元件視圖感興趣。

這是使用 `detectChanges` 時的日誌記錄輸出：

![Image 31: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/8.gif)

僅檢查 `I` 元件。

當我們使用 `ApplicationRef` 時：

![Image 32: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/7.gif)

你可以在這裡看到變更偵測被觸發兩次，
一次用於常規變更偵測週期，另一次用於 `checkNoChanges` 流程。

### 令人驚訝的 ngDoCheck 行為

有一個與 `detectChanges` 相關的意外行為。
`ngDoCheck` hook 不會為你觸發 `detectChanges` 的元件觸發。
發生這種情況的原因是生命週期 hook 在檢查其父元件時在子元件上執行，
而不是在發出呼叫的當前元件上執行。
請參閱關於[操作](https://angular.love/en/change-detection-big-picture-operations/)的章節。

這樣設計的原因之一是允許從 `ngDoCheck` hook 手動控制 `OnPush` 邏輯。
如果子元件被定義為 `onPush`，並且沒有輸入綁定發生變化，你仍然可以從此子元件的
[`ngDoCheck`](https://angular.love/if-you-think-ngdocheck-means-your-component-is-being-checked-read-this-article/)
調用 `markForCheck()`，將該元件標記為髒。

markForCheck
------------

變更偵測器服務實現了 `markForCheck` 方法，該方法非常有用但又有些令人困惑。
與 `detectChanges()` 相反，它不執行變更偵測，
而是顯式地將當前元件（視圖）及其祖先標記為已變更（髒）。
下次任何祖先執行變更偵測週期時，保證會檢查此標記的元件視圖。
這表示如果需要在某些其他操作之前同步更新視圖，
你需要使用 `detectChanges()`，因為呼叫 `markForCheck()` 可能不會及時更新。

`markForCheck` 方法主要用於使用 `OnPush` [變更偵測策略](https://angular.love/deep-dive-into-the-onpush-change-detection-strategy-in-angular/)的元件。
當輸入綁定的值發生變化或在元件的模板中觸發了與 UI 相關的事件時，元件通常會被標記為髒（需要重新渲染）。
如果沒有發生這兩個觸發器，你需要顯式呼叫此方法以確保下次發生變更偵測時將元件包含在檢查中。

由於 Angular 僅檢查物件引用，如果引用保持不變，它可能不會偵測到物件屬性輸入變更。
一種解決方案是使用不可變物件。
另一種解決方案是在 `ngDoCheck` 方法內執行所需的檢查，並顯式地將視圖標記為髒。

讓我們看看這個範例。我們有一個父元件 `O`，它將一個項目新增到 `items` 陣列，
但對陣列的引用保持不變。
定義為 `OnPush` 的子元件 `O1` 採用此項目陣列作為輸入並呈現它。
預設情況下，當 `O` 新增項目時，Angular 不會偵測到變更，也不會更新視圖。
這就是為什麼我們要手動檢查陣列的長度，並在 `O1` 中使用 `markForCheck` 來顯式地將元件標記為髒的原因：

```typescript
@Component({
  selector: 'o-cmp',
  template: '<o1-cmp [items]="items"></o1-cmp>',
})
export class O {
  items = [1, 2, 3];

  constructor() {
    setTimeout(() => {
      this.items.push(4);
    }, 2000);
  }
}

@Component({
  selector: 'o1-cmp',
  template: '<div *ngFor="let item of items">{{item}}</div>',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class O1 {
  @Input() items = [];
  prevLength = 0;

  constructor(private cdRef: ChangeDetectorRef) {}

  ngDoCheck() {
    if (this.items.length !== this.prevLength) {
      this.prevLength = this.items.length;
      this.cdRef.markForCheck();
    }
  }
}
```

當呼叫 `markForCheck` 方法時，
[在底層](https://github.com/angular/angular/blob/5f9c7ceb907be47dff3e203dd837fd6ee9133fcb/packages/core/src/render3/instructions/shared.ts#L1745)
Angular 只是從當前元件視圖開始向上迭代，
並為每個父元件（直到根元件）啟用檢查：

```typescript
export function markViewDirty(lView: LView): LView | null {
  while (lView) {
    lView[FLAGS] |= LViewFlags.Dirty;
    const parent = getLViewParent(lView);
    // Stop traversing up as soon as you find a root view
    // that wasn't attached to any container
    if (isRootView(lView) && !parent) {
      return lView;
    }
    // continue otherwise
    lView = parent!;
  }
  return null;
}
```

最重要的部分是這個賦值：

```typescript
lView[FLAGS] |= LViewFlags.Dirty;
```

程式碼在這裡使用布林值 `OR` 來設定 `LViewFlags.Dirty`
[標誌](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L333)：

```typescript
export const enum LViewFlags {
  /** Whether this view has default change detection strategy (checks always) or onPush */
  CheckAlways = 0b00000010000,

  /** Whether or not this view is currently dirty (needing check) */
  Dirty = 0b00000100000,
}
```

使用布林值 `OR` 的想法是，如果任何輸入為 `1`，則結果位元遮罩值將為 `1`：

```typescript
1 | 1 = 1
0 | 1 = 1
```

Angular 在 `refreshComponent` 函式內部檢查此標誌，以確定元件是否需要檢查。
該函式本身從 `refreshView` 執行，而 `refreshView` 是變更偵測的核心：

```typescript
function refreshComponent(hostLView, componentHostIdx): void {
  ...
  const tView = componentView[TVIEW];
  if (componentView[FLAGS] & (LViewFlags.CheckAlways | LViewFlags.Dirty)) {
    refreshView(tView, componentView, tView.template, componentView[CONTEXT]);
  }
}
```

當變更影響多個元件，並且你確定變更偵測運行會接著進行時，
透過呼叫 `markForCheck` 而不是 `detectChanges`，你實際上是
透過將未來的運行合併到一個週期來減少變更偵測的呼叫次數。

`markForCheck` 方法也可能適用於合併和避免在當前檢查週期中間觸發新的變更偵測週期時拋出的錯誤。
如果你不確定是否會發生這種情況，請使用 `cd.markForCheck()`。
