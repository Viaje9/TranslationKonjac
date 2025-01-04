---
title: Signals in Angular：deep dive for busy developers
date: 2024-12-24 18:40:14
tags:
---

原文：<https://angular.love/signals-in-angular-deep-dive-for-busy-developers>

# Angular 中的 Signals：忙碌開發人員的深入探討

多年來，Web 開發中引入了各種框架和基本元素，以簡化這項任務。其中大多數的核心主題是反應式程式設計，它為管理應用程式狀態提供了基礎設施，使開發人員可以專注於業務邏輯，而不是重複的狀態管理任務。

最近新增的是 **signals**，這是一個「反應式」基本元素，它表示一個動態變化的值，並且可以在值更改時通知感興趣的消費者。這些消費者反過來可以執行重新計算或各種副作用，例如建立/銷毀元件、執行網路請求、更新 DOM 等。

我們可以在不同的框架中找到不同的 signals 實現。現在甚至有人努力 [將 signals 標準化](https://github.com/tc39/proposal-signals)：

> _... 這項努力的重點是使 JavaScript 生態系統保持一致。一些框架作者正在這裡合作開發一個通用模型，該模型可以支援他們反應性的核心。目前的草案基於來自 _[Angular](https://angular.io/)_、_[Bubble](https://bubble.io/)_、_[Ember](https://emberjs.com/)_、_[FAST](https://www.fast.design/)_、_[MobX](https://mobx.js.org/)_、_[Preact](https://preactjs.com/)_、_[Qwik](https://qwik.dev/)_、_[RxJS](https://rxjs.dev/)_、_[Solid](https://www.solidjs.com/)_、_[Starbeam](https://www.starbeamjs.com/)_、_[Svelte](https://svelte.dev/)_、_[Vue](https://vuejs.org/)_、_[Wiz](https://blog.angular.io/angular-and-wiz-are-better-together-91e633d8cd5a)_ 等作者/維護者的設計輸入。_

<!-- more -->

Angular 中的 signals 實作非常類似於提案中提供的實作，因此我可能會在本文中在這兩者之間進行交叉參考。

Signals 作為基本元素
---------------------

signal 代表一個數據單元，該數據單元可能會隨著時間的推移而變化。Signals 可以是「狀態」(僅僅是手動設定的值) 或「計算」(認為是基於其他 signals 的公式)。

計算的 signals 通過自動追蹤在其評估期間讀取了哪些其他 signals 來發揮作用。當讀取計算的 signal 時，它會檢查其先前記錄的任何依賴項是否已更改，如果已更改，則重新評估自身。

例如，這裡我們有一個狀態 signal `counter` 和一個計算的 signal `isEven`。我們將 `counter` 的初始值設定為 `0`，然後將其更改為 `1`。你可以看到，計算的 signal `isEven` 通過在更新 `counter` signal 之前和之後產生兩個不同的值來對更改做出反應：

```
import { computed, signal } from '@angular/core';

// state/writable signal
const counter = signal(0);

// computed signal
const isEven = computed(() => (counter() & 1) == 0);

counter() // 0
isEven() // true

counter.set(1)

counter() // 1
isEven() // false
```

另請注意，在上面的範例中，`isEven` signal 沒有明確訂閱來源 `counter` signal。相反，它只是在其計算函式中使用 `counter()` 呼叫來源 signal。這足以連結這兩個 signals。因此，每當 `counter` 來源 signal 使用新值更新時，衍生的 signal 也會自動更新。

狀態和計算的 signals 都被視為值的生產者。生產者代表產生值並可以傳遞變更通知的 signals。

當通過 API 呼叫更新值時，狀態 signal 會變更(產生)其值，而當回呼中使用的依賴項變更時，計算的 signal 會自動產生新值。

計算的 signals 也可以是消費者，因為它們可能依賴於一些生產者(消費)。在其他反應式實作中，例如 Rx，消費者也稱為接收器。

當生產者 signal 的值變更時，依賴消費者的值(例如計算的 signals)不會立即更新。當讀取計算的 signal 時，它會檢查其先前記錄的任何依賴項是否已更改，並在必要時重新評估自身。

這使得計算的 signals 變得延遲或基於拉取，這意味著即使底層狀態先前已變更，它們也僅在存取時才被評估。在我們上面的範例中，計算的 signal 值僅在我們呼叫 `isEven()` 時才被評估，儘管對底層依賴項 `counter` 的更新較早發生，當時我們執行了 `counter.set()`。

除了常規的可寫和計算的 signals 外，還有監看器(effects)的概念。與基於拉取的計算的 signals 評估相反，變更生產者 signal 將立即通知監看器，**同步**呼叫監看器的通知回呼，有效地「推送」通知。框架將監看器包裝到公開給使用者的 effects 中。Effects 通過排程延遲使用者程式碼的通知。

與 Promises 不同，signals 中的所有內容都是同步執行的：

*   將 signal 設定為新值是同步的，並且在之後讀取任何依賴於它的計算的 signal 時會立即反映這一點。沒有內建的批次處理這種變異。
*   讀取計算的 signals 是同步的 - 它們的值始終可用。
*   監看器是同步通知的，但包裝這些監看器的 effects 可能會選擇通過排程來批次處理和延遲通知。

實作詳細資訊
--------------

在內部，signals 的實作定義了許多我想在本文中解釋的概念：反應式上下文、依賴圖和 effects(監看器)。讓我們從**反應式上下文**開始。

要討論反應式上下文，請考慮[堆疊框架](https://stackoverflow.com/questions/79923/what-and-where-are-the-stack-and-heap/80113#80113)(執行框架)，它定義了在其中評估和執行 JavaScript 程式碼的環境。特別是，它定義了哪些物件(變數)可用於該函式。你可以說這些物件的可用性定義了一個上下文。例如，在 Web Worker 上下文中執行的函式無法存取 `document` 全域物件。

**反應式上下文定義了一個作用中的消費者物件**，該物件依賴於生產者，並且每當讀取其值時都可用於其存取函式。例如，我們在這裡有一個消費者 `isEvent`，它依賴於 `counter` 生產者(消耗其值)。這種依賴關係是通過存取 `computed` 回呼中的 `counter` 值來定義的：

```
isEvent = computed(() => (counter() & 1) === 0)
```

當 computed 回呼執行時，它將自動執行 `counter` signal 的存取函式以取得其值。**我們可以說在這種情況下，`counter` signal 是在 `isEvent` 消費者的反應式上下文中執行的。**因此，如果有一個活動的消費者依賴於此生產者的值，則在反應式上下文中執行生產者。

為了實現這種反應式上下文機制，每次存取消費者的值時，但在其重新計算 _之前_(在執行 `computed` 回呼之前)，我們可以將此消費者設定為活動的消費者。這可以通過簡單地將該消費者物件分配給全域變數並在執行回呼時將其保留在那裡來完成。這個全域變數將可供在執行 `computed` 回呼期間查詢的所有生產者使用，並且它將定義此消費者所依賴的所有生產者的反應式上下文。

這正是 Angular 正在做的事情。當執行 computed 回呼時，它將首先在 [`producerRecomputeValue`](https://github.com/angular/angular/blob/a5b5b7d5ef84b9852d2115dd7a764f4ab3299379/packages/core/primitives/signals/src/computed.ts#L111) 中將目前節點設定為活動消費者：

```
function producerRecomputeValue(node: ComputedNode<unknown>): void {
  ...
  const prevConsumer = consumerBeforeComputation(node);
  let newValue: unknown;
  try {
    newValue = node.computation();
  } catch (err) {...} finally {...}

function consumerBeforeComputation(node: ReactiveNode | null) {
  node && (node.nextProducerIndex = 0);
  return setActiveConsumer(node);
}
```

Angular 從 [`createComputed`](https://github.com/angular/angular/blob/a5b5b7d5ef84b9852d2115dd7a764f4ab3299379/packages/core/primitives/signals/src/computed.ts#L53) 工廠函式中的 `producerUpdateValueVersion` 取得此處：

```
function createComputed<T>(computation: () => T): ComputedGetter<T> {
  ...
  const computed = () => {
    producerUpdateValueVersion(node);
    ...
  };
}

function producerUpdateValueVersion(node: ReactiveNode): void {
  ...
  node.producerRecomputeValue(node);
  ...
}
```

此呼叫堆疊也清楚地示範了此實作：

![Image 27](https://wp.angular.love/wp-content/uploads/2024/08/1_o_jhWzaf9toobEpVXB37CQ-300x141.webp)

因此，當執行計算的回呼時，在該消費者活動期間查詢的每個生產者都會知道它們是在反應式上下文中執行的。**在特定消費者的反應式上下文中執行的所有生產者都會作為消費者的依賴項新增。**這就構成了反應式圖。

Angular 中大多數預先存在的功能都在非反應式上下文中執行。你可以通過簡單地搜尋使用帶有 `null` 值的 `setActiveConsumer` 來觀察到這一點：

![Image 28](https://wp.angular.love/wp-content/uploads/2024/08/1_AWAf6lmiOOr8VCNtGI3jYg-300x176.webp)

例如，在執行生命週期掛鉤之前，Angular 會清除反應式上下文：

```
/**
 * Executes a single lifecycle hook, making sure that:
 * - it is called in the non-reactive context;
 * - profiling data are registered.
 */
function callHookInternal(directive: any, hook: () => void) {
  profiler(ProfilerEvent.LifecycleHookStart, directive, hook);
  const prevConsumer = setActiveConsumer(null);
  try {
    hook.call(directive);
  } finally {
    setActiveConsumer(prevConsumer);
    profiler(ProfilerEvent.LifecycleHookEnd, directive, hook);
  }
}
```

Angular 範本函式(元件檢視)和 effects 在反應式上下文中執行。

反應式圖
--------------

反應式圖是通過消費者和生產者之間的依賴關係建立的。通過值存取器的反應式上下文實作使 signal 依賴關係可以自動且隱含地追蹤。使用者無需宣告依賴項陣列，特定上下文的依賴項集也無需在執行之間保持靜態。

當執行生產者時，它會將自身新增到目前活動消費者(定義目前反應式上下文的消費者)的依賴項中。這發生在 [`producerAccessed`](https://github.com/angular/angular/blob/1081c8d6233ba1ff09187b95a09b0644e130cdf8/packages/core/primitives/signals/src/graph.ts#L238) 函式內部：

```
export function producerAccessed(node: ReactiveNode): void {
  ...
  // This producer is the `idx`th dependency of `activeConsumer`.
    const idx = activeConsumer.nextProducerIndex++;
    if (activeConsumer.producerNode[idx] !== node) {
      // We're a new dependency of the consumer (at `idx`).
      activeConsumer.producerNode[idx] = node;
      // If the active consumer is live, then add it as a live consumer. If not, then use 0 as a
      // placeholder value.
      activeConsumer.producerIndexOfThis[idx] = consumerIsLive(activeConsumer)
        ? producerAddLiveConsumer(node, activeConsumer, idx)
        : 0;
    }
```

生產者和消費者都參與反應式圖。此依賴關係圖是雙向的，但每個方向追蹤的依賴關係有所不同。

生產者通過 `producerNode` 屬性被追蹤為消費者的依賴項，從**消費者到生產者**建立邊：

```
interface ConsumerNode extends ReactiveNode {
  producerNode: NonNullable<ReactiveNode['producerNode']>;
  producerIndexOfThis: NonNullable<ReactiveNode['producerIndexOfThis']>;
  producerLastReadVersion: NonNullable<ReactiveNode['producerLastReadVersion']>;
```

某些消費者也被追蹤為「活動」消費者，並在另一個方向上建立邊，**從生產者到消費者**。這些邊用於在更新生產者值時傳播變更通知：

```
interface ProducerNode extends ReactiveNode {
  liveConsumerNode: NonNullable<ReactiveNode['liveConsumerNode']>;
  liveConsumerIndexOfThis: NonNullable<ReactiveNode['liveConsumerIndexOfThis']>;
}
```

消費者始終追蹤它們所依賴的生產者。生產者僅追蹤來自被視為「活動」消費者的消費者的依賴關係。當消費者將 `consumerIsAlwaysLive` 屬性設定為 `true` 時，或者當消費者依賴於活動消費者時，該消費者是「活動的」。

在 Angular 中，兩種節點類型被定義為活動消費者：

*   [watch](https://github.com/angular/angular/blob/a5b5b7d5ef84b9852d2115dd7a764f4ab3299379/packages/core/primitives/signals/src/watch.ts#L137) 節點(在 effects 中使用)
*   反應式 [LView](https://github.com/angular/angular/blob/4c7d5d8acd8a714fe89366f76dc69f91356f0a06/packages/core/src/render3/reactive_lview_consumer.ts#L51) 節點(在變更偵測中使用)

以下是它們的定義：

```
const WATCH_NODE: Partial<WatchNode> = /* @__PURE__ */ (() => {
  return {
    ...REACTIVE_NODE,
    consumerIsAlwaysLive: true,
    consumerAllowSignalWrites: false,
    consumerMarkedDirty: (node: WatchNode) => {
      if (node.schedule !== null) {
        node.schedule(node.ref);
      }
    },
    hasRun: false,
    cleanupFn: NOOP_CLEANUP_FN,
  };
})();

const REACTIVE_LVIEW_CONSUMER_NODE: Omit<ReactiveLViewConsumer, 'lView'> = {
  ...REACTIVE_NODE,
  consumerIsAlwaysLive: true,
  consumerMarkedDirty: (node: ReactiveLViewConsumer) => {
    markAncestorsForTraversal(node.lView!);
  },
  consumerOnSignalRead(this: ReactiveLViewConsumer): void {
    this.lView![REACTIVE_TEMPLATE_CONSUMER] = this;
  },
};
```

在某些上下文中，`computed` signals 可能會變成「活動」消費者，例如，當在 `effect` 回呼中使用時。

以下程式碼設定：

```
import { ChangeDetectorRef, Component, computed, effect, signal } from '@angular/core';
import { SIGNAL } from '@angular/core/primitives/signals';

@Component({
  standalone: true,
  selector: 'app-root',
  template: 'Angular Love',
  styles: []
})
export class AppComponent {
  constructor(private cdRef: ChangeDetectorRef) {
    const a = signal(0);

    const b = computed(() => a() + 'b');
    const c = computed(() => a() + 'c');
    const d = computed(() => b() + c() + 'd');

    const nodes = [a[SIGNAL], b[SIGNAL], c[SIGNAL], d[SIGNAL]] as any[];

    d();

    const A = 0, B = 1, C = 2, D = 3;

    const depBToA = nodes[B].producerNode[0] === nodes[A];
    const depCToA = nodes[C].producerNode[0] === nodes[A];
    const depDToB = nodes[D].producerNode[0] === nodes[B];
    const depDToC = nodes[D].producerNode[1] === nodes[C];

    console.log(depBToA, depCToA, depDToB, depDToC);

    const e = effect(() => b()) as any;

    // need to wait for change detection to notify the effect
    setTimeout(() => {
      // effect depends on B
      const depEToB = e.watcher[SIGNAL].producerNode[0] === nodes[B];

      // live consumers link from producer A to B,
      // and from B to E, because E (effect) is a live consumer
      const depLiveAToB = nodes[A].liveConsumerNode[0] === nodes[B];
      const depLiveBToE = nodes[B].liveConsumerNode[0] === e.watcher[SIGNAL];

      console.log(depLiveAToB, depLiveBToE, depEToB);
    });
  }
}
```

將產生以下圖表：

![Image 29](https://wp.angular.love/wp-content/uploads/2024/08/1_vDJV8hF-twTuf1GckjPnZQ-300x123.webp)

通過活動消費者的反應式上下文實作可實現**動態依賴關係追蹤**。當將特定消費者設定為活動狀態時，將通過這些生產者呼叫的順序動態定義正在評估的生產者。對於 `ActiveConsumer`，每次在此消費者的反應式上下文中存取生產者時，都可能會重新排列依賴項清單。

為了實現這一點，消費者的依賴關係在 `producerNode` 陣列中追蹤：

```
interface ConsumerNode extends ReactiveNode {
  producerNode: NonNullable<ReactiveNode['producerNode']>;
  producerIndexOfThis: NonNullable<ReactiveNode['producerIndexOfThis']>;
  producerLastReadVersion: NonNullable<ReactiveNode['producerLastReadVersion']>;
```

當重新執行特定消費者的計算時，該陣列中的指標(索引) `producerIndexOfThis` 會初始化為索引 `0`，並且讀取的每個依賴項都會根據指標目前位置的上次執行中的依賴項進行比較。如果存在不匹配，則表示自上次執行以來依賴項已變更，並且可以捨棄舊的依賴項並替換為新的依賴項。在執行結束時，可以捨棄任何剩餘的不匹配的依賴項。

這意味著，如果只有一個分支需要依賴項，並且先前的計算採用了另一個分支，那麼對該暫時未使用的值的變更將不會導致重新計算計算的 signal，即使在拉取時也是如此。這會導致從一個執行到下一個執行可能存取不同的 signal 集。

例如，這個計算的 signal `dynamic` 會根據 `useA` signal 的值讀取 `dataA` 或 `dataB`：

```
const dynamic = computed(() => useA() ? dataA() : dataB());
```

在任何給定的時間點，它將具有 `[useA, dataA]` 或 `[useA, dataB]` 的依賴項集，並且永遠不能同時依賴 `dataA` 和 `dataB`。

這個程式碼類似於 Angular 中的[這個測試案例](https://github.com/proposal-signals/signal-polyfill/blob/4cf87cef28aa89e938f079e4d82e9bf10f6d0a4c/tests/behaviors/dynamic-dependencies.test.ts#L4)，清楚地示範了這一點：

```
import { computed, signal } from '@angular/core';
import { SIGNAL} from '@angular/core/primitives/signals';

const states = Array.from('abcdefgh').map((s) => signal(s));
const sources = signal(states);

const vComputed = computed(() => {
  let str = '';
  for (const state of sources()) str += state();
  return str;
});

const n = vComputed[SIGNAL] as any;
expectEqual(vComputed(), 'abcdefgh');
expectEqualArrayElements(n.producerNode.slice(1), states.map(s => s[SIGNAL]));

sources.set(states.slice(0, 5));
expectEqual(vComputed(), 'abcde');
expectEqualArrayElements(n.producerNode.slice(1), states.slice(0, 5).map(s => s[SIGNAL]));

sources.set(states.slice(3));
expectEqual(vComputed(), 'defgh');
expectEqualArrayElements(n.producerNode.slice(1), states.slice(3).map(s => s[SIGNAL]));

function expectEqual(v1, v2): any {
  if (v1 !== v2) throw new Error(`Expected ${v1} to equal ${v2}`);
}
function expectEqualArrayElements(v1, v2): any {
  if (v1.length !== v2.length) throw new Error(`Expected ${v1} to equal ${v2}`);
  for (let i = 0; i < v1.length; i++) {
    if (v1[i] !== v2[i]) throw new Error(`Expected ${v1} to equal ${v2}`);
  }
}
```

如你所見，該圖沒有一個單一的起點頂點。由於每個消費者都保留一個依賴生產者的清單，而生產者反過來又可能具有依賴關係，例如計算的 signal，因此你可以說，在存取時，每個消費者都是圖表的根頂點。

雙階段更新
-----------------

先前反應性的基於推送的模型面臨冗餘計算的問題：如果對狀態 signal 的更新導致計算的 signal 急切執行，最終這可能會將更新推送至 UI。但是，如果狀態 signal 在下一幀之前要進行另一個變更，則對 UI 的寫入可能是不成熟的。

例如，對於像這樣的圖表，此問題涉及無意中評估 `A -> B -> D` 和 `C`，然後因為 `C` 已變更而重新評估 `D`。兩次重新評估 `D` 是低效的，並且可能導致使用者明顯的故障。

![Image 30](https://wp.angular.love/wp-content/uploads/2024/08/0_IgUGB8HjKNyQRqhV.webp)

這被稱為菱形問題。

有時，由於此類[故障](https://en.wikipedia.org/wiki/Reactive_programming#Glitches)，不準確的中間值甚至會顯示給終端使用者。Signals 通過基於拉取(延遲)而不是基於推送來避免這種動態：當框架排程 UI 的呈現時，它將拉取適當的更新，避免在計算以及寫入 DOM 中浪費工作。

請考慮以下範例：

```
const a = signal(0);

const b = computed(() => a() + 'b');
const c = computed(() => a() + 'c');
const d = computed(() => b() + c() + 'd');

// run the computed callback to set up dependencies
d();

// update the signal at the top of the graph
setTimeout(() => a.set(1), 2000);
```

一旦更新 `a`，就不會發生傳播。僅更新節點的值和版本：

```
function signalSetFn(node, newValue) {
  ...
  if (!node.equal(node.value, newValue)) {
    node.value = newValue;
    signalValueChanged(node);
  }
}

function signalValueChanged(node) {
  node.version++;
  ...
}
```

當我們稍後存取 `d()` 的值時，signals 實作會通過 `consumerPollProducersForChange` 向上輪詢 `d` 的依賴項，以判斷是否需要重新計算。

為了實現高效處理，所有反應式節點都會記錄依賴節點的版本。為了判斷變更，只需比較生產者節點的儲存版本和節點上的實際版本就足夠了：

```
interface ConsumerNode extends ReactiveNode {
...
producerLastReadVersion: NonNullable<ReactiveNode['producerLastReadVersion']>;
}

function consumerPollProducersForChange(node) {
...
// Poll producers for change.
for (let i = 0; i < node.producerNode.length; i++) {
const producer = node.producerNode[i];
const seenVersion = node.producerLastReadVersion[i];
// First check the versions. A mismatch means that the producer's value is known to have
// changed since the last time we read it.
if (seenVersion !== producer.version) {
return true;
}
```

如果這些版本不同，則表示生產者發生了變更，並且實作將通過 `producerRecomputeValue` 執行計算的回呼的重新計算：

```
export function producerUpdateValueVersion(node: ReactiveNode): void {
  ...

  if (!node.producerMustRecompute(node) && !consumerPollProducersForChange(node)) {
    // None of our producers report a change since the last time they were read, so no
    // recomputation of our value is necessary, and we can consider ourselves clean.
    node.dirty = false;
    node.lastCleanEpoch = epoch;
    return;
  }

  node.producerRecomputeValue(node);

  // After recomputing the value, we're no longer dirty.
  node.dirty = false;
  node.lastCleanEpoch = epoch;
}
```

這將對 `C` 的依賴項重複此過程。這樣，它將到達節點 `A`，此時將導致評估分支 `D->C->A`。但是，由於 `D` 也依賴於 `B` 生產者，它會在計算 `D` 之前重新評估該生產者。這樣，`D` 就沒有雙重計算的問題。

不過，有時你可能需要急切通知某些消費者。正如你可能猜到的，這些消費者被稱為「活動」消費者。在這種情況下，變更通知會立即通過圖表傳播，因為生產者值已更新，從而通知依賴於該生產者的活動消費者。

其中一些消費者可能是派生的值，因此也是生產者，它們會使快取的值失效，然後繼續將變更通知傳播到它們自己的活動消費者，依此類推。最終，此通知會到達 effects，這些 effects 會排程自己重新執行。

**至關重要的是，在此階段，不會執行任何副作用，並且不會執行中間值或衍生值的重新計算，僅會使快取的值失效。這使得變更通知可以到達圖表中所有受影響的節點，而不會觀察到中間狀態或故障狀態的可能性。**

如果需要，一旦此變更傳播完成(同步)，此階段可以接著我們上面看到的延遲評估。

為了查看此通知階段的運作情況，讓我們在我們的設定中新增一個活動消費者，例如監看器。當更新 `a` 時，更新會傳播到依賴的活動消費者：

```
import { computed, signal } from '@angular/core';
import { createWatch } from '@angular/core/primitives/signals';

const a = signal(0);
const b = computed(() => a() + 'b');
const c = computed(() => a() + 'c');
const d = computed(() => b() + c() + 'd');

setTimeout(() => a.set(1), 3000);

// watcher will setup a dependency on `d`
const watcher = createWatch(
  () => console.log(d()),
  () => setTimeout(watcher.run, 1000),
  false
);

watcher.notify();
```

一旦我們更新 `a.set(1)` 的值，我們就可以看到活動消費者在運作中的通知：

![Image 31](https://wp.angular.love/wp-content/uploads/2024/08/1_39Jgsu7ROYU5MEoWGnMF8w-300x181.webp)

節點 `b` 和 `c` 是節點 `a` 的活動消費者，因此當運行 `a` 的更新時，Angular 將遍歷 `node.liveConsumerNode` 並通知這些節點有關變更。

但是，正如前面提到的，這裡沒有真正發生任何事情。該節點只是被標記為已變更，並通過 `producerNotifyConsumers` 將通知傳播給其活動消費者：

```
function consumerMarkDirty(node) {
  node.dirty = true;
  producerNotifyConsumers(node);
  node.consumerMarkedDirty?.(node);
}
```

所有這些都會一直下降到依賴於 `d` 的監看器(effect)。與常規反應式節點相反，監看節點在其 `consumerMarkedDirty` 方法中實作排程：

```
const WATCH_NODE: Partial<WatchNode> = (() => {
  return {
    ...REACTIVE_NODE,
    consumerIsAlwaysLive: true,
    consumerAllowSignalWrites: false,
    consumerMarkedDirty: (node: WatchNode) => {
      if (node.schedule !== null) {
        node.schedule(node.ref);
      }
    },
    hasRun: false,
    cleanupFn: NOOP_CLEANUP_FN,
  };
})();
```

在此，通知階段和圖遍歷停止。

這種分兩個階段的過程有時被稱為「推送/拉取」演算法：當變更來源 signal 時，會急切地通過圖表推送「髒狀態」，但只有當通過讀取其 signals 來拉取值時，才會延遲執行重新計算。

變更偵測
----------------

為了將基於 signals 的通知整合到變更偵測過程中，Angular 依賴於活動消費者的機制。元件範本會編譯為範本表達式(JS 程式碼)，並在該元件檢視的反應式上下文中執行。在此類上下文中，執行 signal 將傳回值，但也將 signal 註冊為元件檢視的依賴項。

**由於範本表達式是活動消費者，Angular 將建立從生產者到範本表達式節點的連結。一旦更新生產者的值，該生產者將立即同步通知範本節點。收到通知後，Angular 將標記該元件及其所有祖先進行檢查。**

你可能已經從[我的其他文章](https://angular.love/change-detection-and-component-trees-in-angular-applications)中了解，每個元件的範本在內部都表示為 `LView` 物件。以下是元件的外觀：

```
@Component({...})
export class AppComponent {
  value = signal(0);
}
```

編譯後，它看起來像常規的 JS 函式 `AppComponent_Template`，該函式在此元件的變更偵測期間執行：

```
this.ɵcmp = defineComponent({
  type: AppComponent,
  ...
  template: function AppComponent_Template(rf, ctx) {
    if (rf & 1) {
      ɵɵtext(0);
    }
    if (rf & 2) {
      ɵɵtextInterpolate1("", ctx.value(), "\n");
    }
  },
});
```

當 Angular 將 signals 新增到其變更偵測實作時，它將所有元件檢視(範本函式)包裝在 `ReactiveLViewConsumer` 節點中：

```
export interface ReactiveLViewConsumer extends ReactiveNode {
  lView: LView | null;
}
```

該介面由 [`REACTIVE_LVIEW_CONSUMER_NODE`](https://github.com/angular/angular/blob/4c7d5d8acd8a714fe89366f76dc69f91356f0a06/packages/core/src/render3/reactive_lview_consumer.ts#L51) 節點實作：

```
const REACTIVE_LVIEW_CONSUMER_NODE: Omit<ReactiveLViewConsumer, 'lView'> = {
  ...REACTIVE_NODE,
  consumerIsAlwaysLive: true,
  consumerMarkedDirty: (node: ReactiveLViewConsumer) => {
    markAncestorsForTraversal(node.lView!);
  },
  consumerOnSignalRead(this: ReactiveLViewConsumer): void {
    this.lView![REACTIVE_TEMPLATE_CONSUMER] = this;
  },
};
```

你可以將此過程視為每個檢視取得自己的 `ReactiveLViewConsumer` **消費者**節點，該節點定義了在範本函式內存取的所有 signals 的反應式上下文。

在我們的案例中，每當範本函式作為變更偵測的一部分執行時，它將在範本函式節點(作為 `ActiveConsumer` )的上下文中執行 `ctx.value()` 生產者：

![Image 32](https://wp.angular.love/wp-content/uploads/2024/08/1_qeeISKeap4-Oo6WlikyPZQ-300x204.webp)

這將導致範本表達式節點(消費者)作為 **活動** 依賴項新增到生產者 `value()`：

![Image 33](https://wp.angular.love/wp-content/uploads/2024/08/1_O9Cqy1y_Q8rnwxpu84BieA-300x173.webp)

此依賴關係確保一旦生產者 `counter` 的值變更，它將立即通知消費者節點(範本表達式)。

活動消費者實作 `consumerMarkDirty` 方法，該方法在生產者值變更時由生產者同步呼叫：

```
/**
 * Propagate a dirty notification to live consumers of this producer.
 */
function producerNotifyConsumers(node: ReactiveNode): void {
  ...
  try {
    for (const consumer of node.liveConsumerNode) {
      if (!consumer.dirty) {
        consumerMarkDirty(consumer);
      }
    }
  } finally {
    inNotificationPhase = prev;
  }
}

function consumer