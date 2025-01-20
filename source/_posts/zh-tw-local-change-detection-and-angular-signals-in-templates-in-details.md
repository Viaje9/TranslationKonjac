---
title: Local change detection and Angular Signals in templates in details
date: 2025-01-20 17:11:56
tags:
---



# 標題：深入探討 Local Change Detection 和 Angular Signals 在模板中的應用

原文：<https://medium.com/angularwave/local-change-detection-and-angular-signals-in-templates-in-details-948283adc36d>

大家好 👋！讓我們繼續探討 Angular Signals 系統及其內部運作原理 🔍。以下是文章目錄，方便您快速導航：

*   [Introduction to Angular Signals](https://medium.com/angularwave/introduction-to-angular-signals-e20dba5737db)
*   [Deep dive into the Angular Signals: Part 1](https://medium.com/angularwave/deep-dive-into-the-angular-signals-part-1-c6f9c62aea0e)
*   [Deep dive into the Angular Signals: Part 2](https://medium.com/angularwave/deep-dive-into-the-angular-signals-part-2-274c08c42ca0)
*   [Effect: what is it and how does it work?](https://medium.com/p/9778788a40bc)

今天，讓我們來談談一個因 Angular Signals 而實現的驚人特性 — local change detection。我非常期待與您分享這一切。這是一個引人入勝的主題，我對此感到非常興奮。幾天前，[我在推特上發布了相關內容](https://x.com/sharikov_vlad/status/1731023880641491303?s=20)。這些推文簡要地描述了 Angular Signals 的運作方式。我迫不及待地想分享更多細節！

<!-- more -->

![Image 29](https://miro.medium.com/v2/resize:fit:700/0*aRPk8IXYXy9ayPkQ)

目前已有一種關於 change detection 的工作方式，它的速度已經相當快了。而有了 signals，Angular 可以只檢查必要的組件，從而可以用更少的工作量來達成相同的結果。

> Angular change detection 已經很快了。有了 Angular Signals，它的速度將如同閃電般迅速 🚀🚀🚀。因為現在它可以在執行 change detection 時更具針對性。

如前所述，Angular Signals 與其他響應式系統有許多共通之處。這些工具中使用的通用方法名稱是 fine-grained reactivity。使用這樣的工具可以讓 Angular 只執行必要的操作。讓我們詳細了解一下這在 Angular 中是如何實現的。我將按照以下方式組織這篇文章：

*   解釋用於討論此主題的範例
*   簡述：目前的 change detection 如何運作？
*   當您在組件的模板中使用 signal 時會發生什麼，以及 Angular 如何追蹤它？
*   Angular 如何精確地實現 local change detection？

重點總結
-------

*   ✅ 使用 Signals，Angular 可以檢查更少的 view 並更快地執行 change detection。
*   ✅ 現在，每個 view（大致上每個組件）都有一個對應的 signal。
*   ✅ 這個 signal 就像一個 computed signal 或 effect，它可以追蹤組件中使用的 signal，並在其發生變化時調用某些邏輯。
*   ✅ 當任何 signal 發生變化時，view 的祖先會被標記一個特殊的標誌，這有助於 Angular 進行 change detection 過程。
*   ✅ 當任何 signal 發生變化時，view 對應的 signal 會被標記，這為 Angular 提供了必須檢查其變化的資訊。
*   ✅ 底層有一個響應式圖。它允許 signals 之間相互通訊。該機制使 Angular 能夠實現 Local change deteciton。

範例
---

我將在以下範例中討論這個主題。您可以在 [Stackblitz 上查看](https://stackblitz.com/edit/stackblitz-starters-hjwued?devToolsHeight=33&file=src%2Fmain.ts)。這是一個具有根組件的應用程式，它渲染一個中間層組件。中間層組件會多次渲染自身，最後渲染一個目標組件。目標組件有一個將要被改變的狀態。此外，為了視覺化 change detection 過程，我添加了 `visualizeCd` getter 並將其放在模板中。它輸出一個空字符串，但它也使用 `console.log` 來顯示模板已被訪問。中間層組件輸出訪問的事實並用它們的 `depth` 標記它。目標組件也執行相同的操作並對其進行標記。需要這些 getter 來了解哪些確切的組件被訪問。

在這個範例中，我試圖更接近一個具有許多組件的「真實世界」應用程式。在真實世界的應用程式中，您通常擁有大量組件。它們就像一棵有很多樹枝和樹葉的樹。Angular 在底層也有這樣一棵樹，並且它會使用它。

[**Matthieu Riegler**](https://twitter.com/Jean__Meche) 開發了[一個很棒的工具](https://twitter.com/Jean__Meche/status/1728599408734851245)，可以幫助您了解 change detection 在不同模式下的運作方式。它會顯示哪些組件被檢查。使用右側分支並在該組件中執行標記檢查和/或 signal 更改以查看差異：

![Image 30](https://miro.medium.com/v2/resize:fit:448/1*JDe_vNa5ZYRlgGzjDjl1WA.png)

您可以在 view 上看到諸如 HasChildViewsToRefresh 或 Dirty 或 `Consumer dirty` 之類的標誌。要在組件上執行操作後模擬 change detection，請執行 `AppRef.tick()`。您將看到 Angular 如何檢查 view。這些標誌是什麼意思，Angular 如何知道要檢查什麼？讓我們來詳細說明。我們先來看看它現在是如何運作的。

目前的 Change Detection 如何運作？
-----------------------------------

在這裡，我將簡要概述它的運作原理。這是因為您可以撰寫不止一篇關於 change detection 機制本身的文章。已經有幾篇關於它的文章，因此如果您有興趣，可以查看它們（[範例](https://medium.com/angularwave/onpush-your-new-default-ba3fd5bc9f6e)）。在 Angular 內部，有不同的結構來維護框架中發生的所有事情。有一組 view。如果您簡化（我在這個範例中就是這樣做的），您可以說應用程式中的每個組件都有一個 view。所有這些 view 形成一個樹狀結構。Angular 使用該結構來執行 change detection。view 包含有關組件的大量技術資訊，並在執行不同操作時使用該資訊。您可以將 view 視為包含大量資訊的容器，例如輸入的當前狀態、模板、指令、當前綁定狀態等。Angular 操作這些 view，而不是組件。

change detection 有兩個部分。第一部分是 change detection 應該以某種方式啟動。這就是 `zone.js` 發揮作用的時候。它追蹤所有這些奇特的非同步瀏覽器事件（例如 XHR、DOM 事件、`setInterval`、`setTimeout` 等），並允許您獲得有關某些非同步事件發生的通知。Angular 有一個 `NgZone` 模組，它使用 `zone.js` 來追蹤此類非同步互動。當發生非同步事件時，change detection 會被觸發，Angular 會遍歷 view 的樹。您可以在 [Angular 的核心](https://github.com/angular/angular/blob/21741384f46012f7e8cdbab03281b664b9b954cb/packages/core/src/application_ref.ts#L1237-L1243)中找到該程式碼。它開始樹的遍歷。第二部分是在 change detection 期間應該檢查的內容。

Angular 執行 change detection 有兩種策略。它可以是 `OnPush` 或 `Default` 策略。在預設情況下，它會遍歷樹並訪問每個 view。

使用 `OnPush` 優化，它的工作方式更精細，但仍然很廣泛。在實際應用程式中，您不會像我在相對簡單的範例中那樣擁有一棵只有一個分支的 view 樹。您有很多分支。`OnPush` 策略允許框架切斷整個 view 分支而不檢查它們。這使得應用程式的效能更高。

> 有兩種 ChangeDetection 策略。在預設情況下，每個 view 都會在 change detection 期間被檢查。在 `OnPush` 情況下，只有被標記的 view 才會在 change detection 期間被檢查。

在 OnPush 策略中，只有「標記」的 view 會被檢查。有幾種方法可以標記組件以進行檢查：輸入綁定發生變化、事件偵聽器觸發或調用了 `ChangeDetectoRef` 中的 `markForCheck`。

> 要在 OnPush 中標記要更新的組件，您需要更改輸入、觸發事件（或從輸出中發出）或手動調用 markForCheck（AsyncPipe 執行相同的操作）

在我的範例中，我使用的是 `OnPush` 策略，因此實際上，應用程式已經進行了相當的優化。我想將這種方法與使用 signals 的新方法進行比較。出於示範目的，我在範例中使用了 `markForCheck` 方法。您可以在 `TargetComponent` 的 `ngOnInit` 中看到它：

```typescript
@Component({
  selector: 'app-target',
  standalone: true,
  template: `state: {{ state }} {{ visualizeCd }}`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class TargetComponent {
  state = 0;
  private cdr = inject(ChangeDetectorRef);

  ngOnInit() {
    setTimeout(() => {
      console.log('>> change state');
      this.state += 1;
      // 👇
      this.cdr.markForCheck();
    }, 3000);
  }

  get visualizeCd() {
    console.log('>> cd happened in target');
    return '';
  }
}
```

`markForCheck` 調用會標記當前 view 及其所有父 view，直到根 view 以進行檢查。讓我們將其視覺化。當前的根組件和中間層組件的模板是：

```html
<!-- app-root -->
<app-mid />

<!-- app-mid -->
{{ visualizeCd }}
@if (depth < 5) {
  <app-mid [depth]="depth + 1" />
} @else {
  <app-target />
}
```

這會產生如下所示的樹：

![Image 31](https://miro.medium.com/v2/resize:fit:700/1*SwnoXgTmJZWkUPno58eTfg.png)

如果您將根組件更改為以下內容，會發生什麼？

```html
<!-- root -->
<app-mid />
<app-mid [logicIsDisabled]="true" />
```

讓我們假設 `app-mid` 組件有這樣的輸入並將其向下傳遞給子組件，直到目標組件。如果該輸入為 `true`，則目標組件中的狀態根本不會更改，並且 view 將不會被標記為需要檢查。view 的樹將如下所示：

![Image 32](https://miro.medium.com/v2/resize:fit:700/1*NMIBCzH7_e35eSwutcYGmg.png)

現在，第二個分支已禁用更改。讓我們檢查一下在狀態更改後調用 `markForCheck` 時會發生什麼。`app-target` view 被標記為需要檢查。此外，它還會標記所有父 view 以進行檢查，直到根 view。以下是發生的情況：

![Image 33](https://miro.medium.com/v2/resize:fit:700/1*mb4MZvz9uhy56m_u5VnSaw.png)

現在，由於 `setTimeout` 是一個非同步操作，並且 `zone.js` 會追蹤它。Angular 觸發 change detection 並且僅檢查綠色的 view。

在 change detection 期間究竟發生了什麼？每個 view 都包含對組件模板函數的引用。每個組件都有一個模板。在構建期間，這些組件會被編譯成 JavaScript 函數。以下是[該函數的範例（來自 Angular 程式碼中的某些 README）](https://github.com/angular/angular/blob/21741384f4/packages/core/src/render3/VIEW_DATA.md?plain=1#L90-L109)：

```typescript
function(rf: RenderFlags, ctx: MyApp) {
  if (rf & RenderFlags.Create) {
    ɵɵelementStart(0, 'div');
    ɵɵtext(1);
    ɵɵelementEnd();
  }
  if (rf & RenderFlags.Update) {
    ɵɵproperty('title', ctx.name);
    ɵɵadvance(1);
    ɵɵtextInterpolate1('Hello ', ctx.name, '!');
  }
  ...
}
```

現在，為了執行 change detection，Angular 會訪問每個 view，如果它被標記為需要檢查，它會執行該組件的模板函數。現在，它將新值與先前計算的值進行比較，如果存在差異，Angular 會渲染更改。

> 為了執行 change detection，Angular 會遍歷 view 的樹並在**每個**組件上執行模板函數。

讓我們檢查一下我在之前提供的範例中發生了什麼，您就會明白為什麼需要 `visualizeCd` 輔助函數：

![Image 34](https://miro.medium.com/v2/resize:fit:700/1*BMblzGAZucqCzkFvfsxs5w.png)

您可以看到 `cd happened` 日誌在 `change state` 日誌之前出現了很多次。它會針對 `MidLayerComponent` 的每個實例出現一次，針對 `TargetComponent` 出現一次。更改狀態後，您可以看到日誌出現的次數相同。這表示在每個組件上都調用了 `visualizeCd`。現在有意義了嗎？這與上述內容相關。整個 view 分支都被標記為需要檢查。Angular 通過為每個 view 執行模板函數來檢查每個組件。

> 在 OnPush 情況下，模板會在每個被標記的 view 上執行。

這就是當前的 change detection 世界。別忘了我在上面提供的用於檢查 change detection 如何運作的示範。您可以檢查 OnPush 發生了什麼並將其與 Angular Signals 進行比較。現在，讓我們回顧一下 Angular Signals 發生了什麼以及為什麼。

Angular Signals 如何改變了這一點？
----------------------------------------

讓我們繼續看[第二個範例](https://stackblitz.com/edit/stackblitz-starters-rwh9sp?devToolsHeight=33&file=src%2Fmain.ts)，我在其中使用 signals 而不是簡單的變數。目標組件的程式碼略有更改：

```typescript
@Component({
  selector: 'app-target',
  standalone: true,
  template: `state: {{ state() }} {{ visualizeCd }}`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class TargetComponent {
  state = signal(0);

  ngOnInit() {
    setTimeout(() => {
      console.log('>> change state');
      // 👇 no markForCheck
      this.state.update((v) => v + 1);
    }, 3000);
  }

  get visualizeCd() {
    console.log('>> cd happened in target');
    return '';
  }
}
```

這是一個非常簡單的更改，但它帶來了很多好處 🤩！您可以看到現在 `state` 實例成員已更改為 signal。它以前是一個簡單的變數，現在變成了一個 signal。此外，在 `ngOnInit` 函數中，我使用 `this.state.update` 而不是簡單的 `this.state += 1`。請注意，現在沒有 `markForCheck` 調用。此外，在模板中，我使用了 `state()`。如果您想獲取 signal 值，您需要調用一個函數。該程式碼可以正常運作。

在考慮將範例遷移到 signals 時，讓我們考慮以下問題：

*   ❓ 當您將 signal 放入 Angular 組件的模板中時會發生什麼？
*   ❓當模板中使用的 signal 發生更改時會發生什麼？
*   ❓全域 change detection 傳播機制是如何更改的？

讓我們分解一下在模板中使用該 signal 時在技術上發生了什麼。這個 signal 是組件模板的一部分。從上一節中，您知道在底層，Angular 操作 view 和模板函數。最後，`state: {{ state() }} {{ visualizeCd }}` 模板將變成一個模板函數，如下所示：

```typescript
function TargetComponent_Template(rf, ctx) {
  if (rf & 1) {
    ɵɵtext(0);
  }
  if (rf & 2) {
                                           👇
    ɵɵtextInterpolate2("state: ", ctx.state(), " ", ctx.visualizeCd, "");
  }
}
```

`rf & 1` 代表創建，`rf & 2` 部分代表更新。這表示它在任何更新期間都會執行。所以，這是一個基本的 JavaScript 函數。必須從某個地方調用它。這發生在 [executeTemplate](https://github.com/angular/angular/blob/21741384f4/packages/core/src/render3/instructions/shared.ts#L261) [函數](https://github.com/angular/angular/blob/21741384f4/packages/core/src/render3/instructions/shared.ts#L261)中，該函數是從 [refreshView](https://github.com/angular/angular/blob/21741384f4/packages/core/src/render3/instructions/change_detection.ts#L158) [函數](https://github.com/angular/angular/blob/21741384f4/packages/core/src/render3/instructions/change_detection.ts#L158)中調用的。讓我們仔細看看：

```typescript
// render3/instructions/change_detection.ts
export function refreshView<T>(tView: TView, lView: LView, templateFn: ComponentTemplate<{}>|null, context: T) {
  // ...
  let prevConsumer: ReactiveNode|null = null;
  let currentConsumer: ReactiveLViewConsumer|null = null;
  if (!isInCheckNoChangesPass && viewShouldHaveReactiveConsumer(tView)) {
    // 👇👇👇
    currentConsumer = getOrBorrowReactiveLViewConsumer(lView);
    prevConsumer = consumerBeforeComputation(currentConsumer);
  }
  try {
    // ...
    if (templateFn !== null) {
      // 👇👇👇
      executeTemplate(tView, lView, templateFn, RenderFlags.Update, context);
    }
    // ....
  } finally {
    if (currentConsumer !== null) {
      // 👇👇👇
      consumerAfterComputation(currentConsumer, prevConsumer);
      maybeReturnReactiveLViewConsumer(currentConsumer);
    }
    leaveView();
  }
}

// render3/reactive_lview_customer.ts
export function getOrBorrowReactiveLViewConsumer(lView: LView): ReactiveLViewConsumer {
  return lView[REACTIVE_TEMPLATE_CONSUMER] ?? borrowReactiveLViewConsumer(lView);
}

function borrowReactiveLViewConsumer(lView: LView): ReactiveLViewConsumer {
  const consumer: ReactiveLViewConsumer =
      freeConsumers.pop() ?? Object.create(REACTIVE_LVIEW_CONSUMER_NODE);
  // 👇
  consumer.lView = lView;
  return consumer;
}

const REACTIVE_LVIEW_CONSUMER_NODE: Omit<ReactiveLViewConsumer, 'lView'|'slot'> = {
  ...REACTIVE_NODE,
  // 👇
  consumerIsAlwaysLive: true,
  consumerMarkDirty: (node: ReactiveLViewConsumer) => {
    // 👇
    markAncestorsForTraversal(node.lView!);
  },
  consumerOnSignalRead(this: ReactiveLViewConsumer): void {
    this.lView![REACTIVE_TEMPLATE_CONSUMER] = this;
  },
};
```

有很多程式碼，但請注意上面的幾行。`currentConsumer = getOrBorrowReactiveLViewConsumer(lView)` 獲取現有的 `lView[REACTIVE_TEMPLATE_CONSUMER]` 或基於 `REACTIVE_LVIEW_CONSUMER_NODE` 結構創建一個新的響應式節點。另請注意，`lView` 已添加到創建的 consumer 中。這是一件重要的事情，因為它將在以後使用。

> 每個 view 都有一個與之關聯的響應式節點

如果您閱讀了之前的文章，那麼 `prevConsumer = consumerBeforeComputation(currentConsumer);` 對您來說應該很熟悉。這將使用 `currentConsumer` 設置當前的 `activeConsumer` 狀態。之後，將調用 `executeTemplate`，並執行其中的模板函數。在模板函數中，有一個 `ctx.state()`，表示訪問 signal。現在，將調用 [producerAccessed](https://github.com/angular/angular/blob/21741384f4/packages/core/primitives/signals/src/graph.ts#L199-L246) [函數](https://github.com/angular/angular/blob/21741384f4/packages/core/primitives/signals/src/graph.ts#L199-L246)，在該函數中將創建兩個響應式節點之間的連結。請查看之前的文章，我在其中詳細解釋了這一點。它位於 [Deep dive into the Angular Signals: Part 1](https://medium.com/angularwave/deep-dive-into-the-angular-signals-part-1-c6f9c62aea0e) 中的**延遲評估和自動依賴項圖**部分。讓我們簡要地檢查一下，因為您現在可能沒有閱讀之前的文章，如果是的話，重複學習也是好的：

```typescript
export function producerAccessed(node: ReactiveNode): void {
  // ...
  if (activeConsumer.producerNode[idx] !== node) {
    // We're a new dependency of the consumer (at `idx`).
    activeConsumer.producerNode[idx] = node;
    // If the active consumer is live, then add it as a live consumer. If not, then use 0 as a
    // placeholder value.
    // 👇
    activeConsumer.producerIndexOfThis[idx] =
        consumerIsLive(activeConsumer) ? producerAddLiveConsumer(node, activeConsumer, idx) : 0;
  }
  activeConsumer.producerLastReadVersion[idx] = node.version;
}

function consumerIsLive(node: ReactiveNode): boolean {
  return node.consumerIsAlwaysLive || (node?.liveConsumerNode?.length ?? 0) > 0;
}
```

簡而言之，該方法會更改 `activeConsumer`（組件 view 中的 consumer）節點的狀態，並將當前訪問的節點（`TargetComponent` 中的 `state` signal）註冊到其中。此外，如果 `activeConsumer` 為 live，則將調用 `producerAddLiveConsumer`，這會將 `activeConsumer` 添加到 `state` signal 的 `liveConsumersNode` 陣列中。由於執行組件模板之前創建的響應式節點的配置，`activeConsumer` 為 live。請查看前面程式碼區塊中的 `REACTIVE_LVIEW_CONSUMER_NODE` 定義。

> view 的響應式節點成為組件模板中訪問的每個 signal 的 live consumer。這與 effect 或 computed signal 非常相似

所以，再次瀏覽所有這些內容。在組件模板中使用 signal 會發生什麼：

1. 在執行模板之前，Angular 會創建或獲取現有的 view 的響應式 consumer。
2. 如果模板中將有 signals，它會將其設置為 `activeConsumer`。
3. Angular 執行模板並訪問 signal。
4. 在內部，會在 view 的 consumer 和 `state` signal 之間建立依賴項。
5. signal 的值會返回到模板函數，以便它可以在瀏覽器中輸出它。

> 當執行模板時，會在模板中訪問的所有 signals 和 view 的響應式節點之間建立依賴項。

Angular 如何知道 signals 中的更改？
-----------------------------------------------

這是因為在第一次執行模板函數後，在該 signal 和 view 的 consumer 之間建立了依賴項。它允許響應式系統節點進行通訊。當您使用 `set` 或 `update` 方法更新 signal 時，會在內部調用 `signalSetFn` 函數。這會導致 [signalValueChanged](https://github.com/angular/angular/blob/21741384f4/packages/core/primitives/signals/src/signal.ts#L109-L115) 調用，進而導致 [producerNotifiesConsumers](https://github.com/angular/angular/blob/21741384f4/packages/core/primitives/signals/src/graph.ts#L292-L309) 調用。讓我們仔細看看：

```typescript
export function producerNotifyConsumers(node: ReactiveNode): void {
  if (node.liveConsumerNode === undefined) {
    return;
  }
  // Prevent signal reads when we're updating the graph
  const prev = inNotificationPhase;
  inNotificationPhase = true;
  try {
    for (const consumer of node.liveConsumerNode) {
      if (!consumer.dirty) {
        // 👇
        consumerMarkDirty(consumer);
      }
    }
  } finally {
    inNotificationPhase = prev;
  }
}

export function consumerMarkDirty(node: ReactiveNode): void {
  // 👇
  node.dirty = true;
  producerNotifyConsumers(node);
  // 👇
  node.consumerMarkedDirty?.(node);
}
```

該函數會遍歷 `node` 的 `liveConsumerNode` 陣列。如果它為 dirty，則會調用 `consumerMarkDirty` 函數。它將節點設置為 dirty，遞迴調用 `producerNotifyConsumers` 並調用 `node` 的 `consumerMarkedDirty` 方法。將節點設置為 dirty 和 `consumerMarkedDirty` 方法是目前最重要的。該方法在 `REACTIVE_LVIEW_CONSUMER_NODE` 定義中定義。它使用 `node.lView` 參數調用 [markAncestorsForTraversal](https://github.com/angular/angular/blob/21741384f4/packages/core/src/render3/util/view_utils.ts#L223-L243) 函數。這就是為什麼在響應式節點中需要 `lView` 的原因。它是在上面提到的 `borrowReactiveLViewConsumer` 中分配的。讓我們檢查一下標記函數：

```typescript
export function markAncestorsForTraversal(lView: LView) {
  let parent = lView[PARENT];
  while (parent !== null) {
    // We stop adding markers to the ancestors once we reach one that already has the marker. This
    // is to avoid needlessly traversing all the way to the root when the marker already exists.
    if ((isLContainer(parent) && (parent[FLAGS] & LContainerFlags.HasChildViewsToRefresh) ||
         (isLView(parent) && parent[FLAGS] & LViewFlags.HasChildViewsToRefresh))) {
      break;
    }
    if (isLContainer(parent)) {
      parent[FLAGS] |= LContainerFlags.HasChildViewsToRefresh;
    } else {
      // 👇
      parent[FLAGS] |= LViewFlags.HasChildViewsToRefresh;
      if (!viewAttachedToChangeDetector(parent)) {
        break;
      }
    }
    parent = parent[PARENT];
  }
}
```

該函數會修改從當前 view 到根 view 的每個 view 的 FLAGS。它與 `markForCheck` 調用發生的情況幾乎相同，但標誌不同。讓我們比較一下。[ChangeDetectorRef](https://github.com/angular/angular/blob/21741384f4/packages/core/src/render3/view_ref.ts#L143-L145) 的 [markForCheck](https://github.com/angular/angular/blob/21741384f4/packages/core/src/render3/view_ref.ts#L143-L145) [方法](https://github.com/angular/angular/blob/21741384f4/packages/core/src/render3/view_ref.ts#L143-L145)會導致 [markViewDirty](https://github.com/angular/angular/blob/21741384f4/packages/core/src/render3/instructions/mark_view_dirty.ts#L24C1-L36C2) [工具](https://github.com/angular/angular/blob/21741384f4/packages/core/src/render3/instructions/mark_view_dirty.ts#L24C1-L36C2)：

```typescript
export function markViewDirty(lView: LView): LView|null {
  while (lView) {
    // 👇   
    lView[FLAGS] |= LViewFlags.Dirty;
    const parent = getLViewParent(lView);
    // Stop traversing up as soon as you find a root view that wasn't attached to any container
    if (isRootView(lView) && !parent) {
      return lView;
    }
    // continue otherwise
    lView = parent!;
  }
  return null;
}
```

它與 `markAncestorsForTraversal` 執行幾乎相同的操作，但標誌不同。

所以，讓我們確認一下當 signal 更改時會發生什麼：

> 當組件模板中使用的 signal 更改時，它會通知 view 的響應式 live consumer 更改。這會導致將該 consumer 標記為 dirty。此外，它會將其所有祖先標記為具有 `HasChildViewsToRefresh` 標誌，直到根 view。

現在，正如您已經知道的，change detection 機制有兩個部分。第一部分需要確定應該檢查哪些 view。第二部分是遍歷 view 的樹。讓我們檢查第二部分。第二部分由 `zone.js` 觸發。

> 全域自上而下的 change detection 傳播仍然由 `zone.js` 觸發

樹遍歷機制如何更改以與響應式系統互動？

響應式系統如何實現 Local Change Detection？
------------------------------------------------------------

讓我們檢查一下基於 signals 的第二個範例記錄了什麼：

![Image 35](https://miro.medium.com/v2/resize:fit:700/1*BbOFlDXj0nPHKRWCqk-boQ.png)

您可以看到更改後只有一個日誌。看起來只有 TargetComponent 的一個模板函數被執行。這正是發生的事情。這可能是因為 Angular 最近對使用 signals 的組件進行了更改。在上一節中，我將所有內容分解到您可以清楚地看到當 signal 更改時，其 view 和所有父 view 直到根 view 都會被標記一個特殊的標誌。現在，該標誌將用於檢測是否應該檢查組件，或者換句話說，是否應該執行 view 的模板函數。讓我們詳細看看它是如何運作的。

signal 的狀態在傳遞給 `setTimeout` 調用的函數中更改。這是一個非同步操作，這表示 `zone.js` 知道該操作。Angular 的核心有一個機制可以追蹤此類非同步操作，並在這些操作之後觸發全域 change detection 機制。這也發生在我們的範例中。順便說一下，無論您是否使用 signals，`zone.js` 都會觸發全域 change detection。這可能會在將來使用 `signals: true` 組件時發生變化，但它們還沒有上線。所以，Angular 開始自上而下地遍歷 view 的樹。現在，請記住 `markAncestorsForTraversal`。它設置了 `HasChildViewsToRefresh` 標誌並將目標組件 view 標記為 dirty。您可以簡單地嘗試在程式碼中找到此標誌並查看它在哪裡使用的一種方法。另一種方法是您可以將 `debugger` 語句放入 `TargetComponent` 中的 `visualizeCd` getter 中。讓我們檢查調用堆疊：

![Image 36](https://miro.medium.com/v2/resize:fit:700/1*ngYhSl4HcbZWGm0qHEAmYw.png)

您可以看到熟悉的 `refreshView` 和 `executeTemplate` 函數。您還可以看見 `TargetComponent_Template` 函數，它是我們目標組件的模板函數。此外，還有許多類似 `detectChanges*` 的函數，它們是 change detection 機制的一部分。神奇的事情一定發生在其中一個函數中。您可以觀察那裡發生了什麼，您可以打開範例並將 `debugger` 語句放入 `visualizeCd` getter 中。讓我們嘗試在程式碼中搜尋該標誌。順便說一下，我對所有範例都使用 `21741384f4` 提交雜湊。程式碼會更改，因此為了保持一致性並使所有連結保持有效，我將顯示來自一個特定提交的所有內容。我使用 `.HasChildViewsToRefresh` 查詢進行搜尋，我能夠找到兩個檔案：`view_utils.ts` 和 `instructions/change_detection.ts`。第一個是工具函數，第二個是關於 change detection 過程的。我們絕對想檢查第二個。此外，第一個已經很熟悉了，因為 `markAncestorsForTraversal` 工具函數來自那裡。`HasChildViewsToRefresh` 在兩個函數中：`detectChangesInView` 和 `detectChangesInEmbeddedViews`。第二個函數是從第一個函數中調用的。讓我們檢查一下它們：

```typescript
function detectChangesInView(lView: LView, mode: ChangeDetectionMode) {
  const isInCheckNoChangesPass = ngDevMode && isInCheckNoChangesMode();
  const tView = lView[TVIEW];
  const flags = lView[FLAGS];
  const consumer = lView[REACTIVE_TEMPLATE_CONSUMER];
  let shouldRefreshView: boolean =
      !!(mode === ChangeDetectionMode.Global && flags & LViewFlags.CheckAlways);
  shouldRefreshView ||= !!(
      flags & LViewFlags.Dirty && mode === ChangeDetectionMode.Global && !isInCheckNoChangesPass);
  shouldRefreshView ||= !!(flags & LViewFlags.RefreshView);
  // Refresh views when they have a dirty reactive consumer, regardless of mode.
  // 👇
  shouldRefreshView ||= !!(consumer?.dirty && consumerPollProducersForChange(consumer));if (consumer) {
    consumer.dirty = false;
  }
  lView[FLAGS] &= ~(LViewFlags.HasChildViewsToRefresh | LViewFlags.RefreshView);
  if (shouldRefreshView) {
    // 👇
    refreshView(tView, lView, tView.template, lView[CONTEXT]);
  } else if (flags & LViewFlags.HasChildViewsToRefresh) {
    // 👇
    detectChangesInEmbeddedViews(lView, ChangeDetectionMode.Targeted);
    const components = tView.components;
    if (components !== null) {
      // 👇
      detectChangesInChildComponents(lView, components, ChangeDetectionMode.Targeted);
    }
  }
}

function detectChangesInEmbeddedViews(lView: LView, mode: ChangeDetectionMode) {
  for (let lContainer = getFirstLContainer(lView); lContainer !== null;
       lContainer = getNextLContainer(lContainer)) {
    lContainer[FLAGS] &= ~LContainerFlags.HasChildViewsToRefresh;
    for (let i = CONTAINER_HEADER_OFFSET; i < lContainer.length; i++) {
      const embeddedLView = lContainer[i];
      detectChangesInViewIfAttached(embeddedLView, mode);
    }
  }
}
```

實際上，這些函數解釋了一切。這些函數在應用程式中的每個 view 上遞迴調用。當 Angular 執行 change detection 過程時，它會遍歷每個 view 並調用該程式碼。從前面的章節中，您知道在 signal 更改的情況下，view 會被標記 `HasChildViewsToRefresh` 標誌。此外，每個 view 都有一個可以為 dirty 的響應式 consumer。那裡發生了什麼？讓我們來詳細說明。

我正在嘗試（我真的在努力，各位）簡化事情。坦率地說，Angular 有兩種類型的 view，並且它同時使用它們：`tView` 和 `lView`。為了簡化，讓我們現在專注於已知的標誌，我省略了 `detectChangesInView` 函數的某些部分。因為這篇文章不是關於 change detection 的。

首先，該函數檢測是否應該刷新 view。它檢查 view 和模式的不同標誌。讓我們檢查一下這些行：

```typescript
// ...
shouldRefreshView ||= !!(
                       👇
      flags & LViewFlags.Dirty && mode === ChangeDetectionMode.Global && !isInCheckNoChangesPass);

shouldRefreshView ||=
                👇
    !!(consumer?.dirty && consumerPollProducersForChange(consumer));

// ...
```

第一個表達式檢查 view 是否具有 `LViewFlags.Dirty` 標誌和 `mode`。我將在這篇文章中簡化 `mode` 部分。在 `OnPush` + `markForCheck` 模式的情況下，您通常會使用 `ChangeDetectionMode.Global`。在使用 signals 模式和精細更新的情況下，您將使用 `ChangeDetectionMode.Targeted`。`flags & LViewFlags.Dirty` 是一個按位運算，該機制在 Angular 中用於執行極快的檢查。第一個表達式是關於 `Dirty` 標誌的，我之前提到它在 `ChangeDetectorRef` 的 `markForCheck` 方法中使用。所以，如果它被標記為需要檢查，它將為 true，但現在情況並非如此。

> 如果 view 被標記了 RefreshView 標誌，Angular 會執行模板函數。這是 `markViewDirty` 調用（`ChangeDetectorRef` 的 `markForCheck`）的情況。

第二個表達式檢查 view 的響應式 consumer 是否為 dirty。在我的第二個使用 signals 的範例中，view 的響應式節點被標記為 dirty。它發生在 `state` signal 更改時，view 的響應式節點被通知並且它變為 dirty。因此，當 Angular 遍歷到該 view 時，此檢查將在該 view 上返回 true。

> 如果 view 的響應式節點被標記為 dirty，Angular 會執行模板函數。當模板中使用的任何 signals 由於響應式依賴項圖而更新時，就會發生這種情況。

例如，在 computed signal 的情況下，需要 `consumerPollProducersForChange(consumer)` 部分。從深入探討的第一部分中，您可能還記得（如果您還沒有閱讀，請查看它；它很酷，您會讓我高興 :)) 可能存在複雜的情況。讓我們檢查一下那部分：

```typescript
lView[FLAGS] &= ~(LViewFlags.HasChildViewsToRefresh | LViewFlags.RefreshView);
```

這也是一個按位運算符。它從 view 中刪除了 `HasChildViewsToRefresh` 和 `RefreshView` 標誌。之後，您可以看到一個 `if {} else {}` 條件。讓我們檢查一下它：

```typescript
if (shouldRefreshView) {
    refreshView(tView, lView, tView.template, lView[CONTEXT]);
  } else if (flags & LViewFlags.HasChildViewsToRefresh) {
    detectChangesInEmbeddedViews(lView, ChangeDetectionMode.Targeted);
    const components = tView.components;
    if (components !== null) {
      detectChangesInChildComponents(lView, components, ChangeDetectionMode.Targeted);
    }
  }
```

在 `shouldRefreshView` 為 `true` 的情況下，它會轉到 `refreshView` 方法並執行模板。在我們的範例中，當目標組件中的 `state` signal 的狀態更新時，就會執行此操作。

否則，它會檢查 view 是否具有 `HasChildViewsToRefresh` 標誌。此標誌在 `markAncestorsForTraversal` 函數中設置。如果是，它會繼續檢查子項而不執行模板函數。`else if` 分支中的兩個函數都將再次導致 `detectChangesInView` 函數。它以遞迴方式工作，並將在當前 view 的每個子項上調用，依此類推。

讓我們總結一下。當在組件中更新 signal 時，這是如何工作的：

1. 在組件中更新 signal 作為非同步操作（如 `setTimeout`）的一部分。
2. Angular 自上而下觸發全域 change detection。
3. 作為其中的一部分，在應用程式中 view 樹的每個 view 上調用 `detectChangesInView`。
4. 它檢查 view 的 consumer 是否為 dirty 或其依賴項是否已更改，如果為 true，則執行模板。
5. 如果不是，它會檢查 view 上是否有 `HasChildViewsToRefresh` 標誌，這表示 Angular 必須繼續檢查子項。在這種情況下，它不會執行 view 的模板。它會跳過它。
6. 如果由於 signal 更改而標記了 view，Angular 不會在每個 view 上執行模板函數。
7. 此外，請注意，在 `Dirty` 標誌的情況下，此函數不會轉到子 view。它只是停止執行 `detectChangesInView` 並退出。這就是在 `markForCheck` 方法的情況下如何切斷 view 分支。

該演算法在樹中的每個 view 上執行。在以下情況下，當在

重點說明：

> 在使用 signals 的情況下，Change detection 的工作方式更優化，因為 Angular 不會在標記的 view 集合中的每個 view 上執行模板函數。它以精細的方式執行此操作。它僅針對其響應式節點為 dirty 的 view 執行模板函數。

一個溫和但重要的說明
-----------------------------

當組件使用 `OnPush` 策略時，有幾種方法可以標記 view 以進行檢查。它們在應用程式中經常使用，因此組件可以相互通訊。此外，像 `(click)` 這樣的 DOM 事件也會出現在那裡。[有一段程式碼](https://github.com/angular/angular/blob/21741384f46012f7e8cdbab03281b664b9b954cb/packages/core/src/render3/instructions/listener.ts#L260)在輸出的情況下將 view 及其父項標記為 dirty。另一方面，還有許多東西，如 WebSockets、Server-Side Events、通過服務進行通訊和計時器。因此，在編寫應用程式時，請考慮此細節。讓我們看看 Angular 團隊在未來版本中向我們提出了什麼建議。根據 RFC，看起來未來將會有更多有趣的工具。

所以，我們今天討論了什麼：

*   ✅ Angular 如何在 Angular Signals 之上構建 Local Change Detection 機制？
*   ✅ 它如何將 Angular Signals 集成到現有的 change detection 機制中？
*   ✅ 當在組件的模板中使用 signal 時會發生什麼？

總之，這就是我想說的關於使用 Angular Signals 實現的 Local change detection 機制的所有內容。我認為這是 Angular 演進中的重要一步，分解這部分內容真是太棒了 🚀。感謝來自 AngularWave 的 [Roman Sedov](https://twitter.com/marsibarsi) 和 [Alex Inkin](https://twitter.com/Waterplea) 的審閱。這有很大幫助！請繼續關注[我的 Twitter](https://twitter.com/sharikov_vlad)，下次見！
