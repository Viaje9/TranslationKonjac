---
title: Signals in Angular：deep dive for busy developers
date: 2024-12-24 18:40:14
tags:
---

原文：<https://angular.love/signals-in-angular-deep-dive-for-busy-developers>

# Angular 中的 Signals：忙碌開發者的深入探討

建構複雜的使用者介面是一項艱鉅的任務。在現代網路應用程式中，UI 狀態很少由簡單的獨立值組成。它更像是依賴於複雜層級的其他值或計算狀態的複雜計算狀態。管理這種狀態需要大量的工作：開發者必須儲存、計算、使失效和同步這些值。

多年來，網路開發中引入了各種框架和基本概念，以簡化這項任務。其中大多數的核心主題是響應式程式設計，它提供了管理應用程式狀態的基礎架構，讓開發者可以專注於業務邏輯，而不是重複的狀態管理任務。

最近加入的是 **signals**，這是一種「響應式」基本概念，表示動態變化的值，並且可以在值改變時通知感興趣的消費者。這些消費者反過來可以執行重新計算或各種副作用，例如建立/銷毀元件、執行網路請求、更新 DOM 等。

我們可以在不同的框架中找到 signals 的不同實現。現在甚至有努力[將 signals 標準化](https://github.com/tc39/proposal-signals)：

> _… 這項工作重點是讓 JavaScript 生態系統保持一致。一些框架作者正在這裡合作建立一個通用模型，這個模型可以支援他們的響應式核心。目前的草案是基於來自_ [_Angular_](https://angular.io/)_,_ [_Bubble_](https://bubble.io/)_,_ [_Ember_](https://emberjs.com/)_,_ [_FAST_](https://www.fast.design/)_,_ [_MobX_](https://mobx.js.org/)_,_ [_Preact_](https://preactjs.com/)_,_ [_Qwik_](https://qwik.dev/)_,_ [_RxJS_](https://rxjs.dev/)_,_ [_Solid_](https://www.solidjs.com/)_,_ [_Starbeam_](https://www.starbeamjs.com/)_,_ [_Svelte_](https://svelte.dev/)_,_ [_Vue_](https://vuejs.org/)_,_ [_Wiz_](https://blog.angular.io/angular-and-wiz-are-better-together-91e633d8cd5a)_, 以及更多框架的作者/維護者的設計輸入。_

Angular 中 signals 的實現非常類似於提案中提供的實現，因此我可能會在本文中對兩者進行交叉引用。

<!-- more -->

Signals 作為基本概念
---------------------

一個 signal 代表一個可能隨時間改變的資料單元。Signals 可以是「狀態」（只是手動設定的值）或「計算」（可以想像成基於其他 signals 的公式）。

計算的 signals 的運作方式是自動追蹤它們在評估期間讀取了哪些其他 signals。當讀取一個計算的 signal 時，它會檢查其先前記錄的任何依賴項是否已變更，並在變更時重新評估自身。

例如，這裡我們有一個狀態 signal `counter` 和一個計算的 signal `isEven`。我們將 `counter` 的初始值設定為 `0`，稍後將其變更為 `1`。你可以看到計算的 signal `isEven` 通過在 `counter` signal 更新之前和之後產生兩個不同的值來對變更做出反應：

```typescript
import { computed, signal } from '@angular/core';

// 狀態/可寫入的 signal
const counter = signal(0);

// 計算的 signal
const isEven = computed(() => (counter() & 1) == 0);

counter() // 0
isEven() // true

counter.set(1)

counter() // 1
isEven() // false
```

另請注意，在上面的範例中，`isEven` signal 並未明確訂閱來源 `counter` signal。相反，它只是在其計算函數中使用 `counter()` 來呼叫來源 signal。這足以連結這兩個 signals。因此，每當來源 `counter` signal 更新為新值時，派生的 signal 也會自動更新。

狀態和計算的 signals 都被視為值的生產者。生產者表示產生值並可以傳遞變更通知的 signals。

當通過 API 呼叫更新值時，狀態 signal 會變更（產生）其值，而當回呼中使用的依賴項變更時，計算的 signal 會自動產生新值。

計算的 signals 也可以是消費者，因為它們可能依賴於一些生產者（消費）。在其他響應式實現中，例如 Rx，消費者也被稱為接收器。

當生產者 signal 的值變更時，相依的消費者（例如，計算的 signals）的值不會立即更新。當讀取計算的 signal 時，它會檢查其先前記錄的任何依賴項是否已變更，並在必要時重新評估自身。

這使得計算的 signals 變成惰性或基於提取的，這表示它們只有在存取時才被評估，即使基礎狀態較早發生變更也是如此。在上面的範例中，計算的 signal 值只有在我們呼叫 `isEven()` 時才被評估，儘管對基礎依賴項 `counter` 的更新較早發生，當我們執行 `counter.set()` 時。

除了常規的可寫入和計算的 signals 之外，還有監看者（effects）的概念。與計算的 signals 的基於提取的評估相反，變更生產者 signal 會立即通知監看者，**同步**呼叫監看者的通知回呼，有效地「推送」通知。框架將監看者包裝成暴露給使用者的 effects。Effects 通過排程延遲使用者程式碼的通知。

與 Promises 不同，signals 中的所有內容都是同步執行的：

*   將 signal 設定為新值是同步的，並且在之後讀取任何依賴於它的計算的 signal 時，會立即反映出來。沒有內建的批次處理這種變更。
*   讀取計算的 signals 是同步的 — 它們的值始終可用。
*   監看者是同步通知的，但是包裝那些監看者的 effects 可以選擇通過排程進行批次處理和延遲通知。

實現細節
----------------------

在內部，signals 的實現定義了我在本文中要解釋的一些概念：響應式環境、依賴關係圖和 effects（監看者）。讓我們先從**響應式環境**開始。

要討論響應式環境，可以想像成[堆疊框架](https://stackoverflow.com/questions/79923/what-and-where-are-the-stack-and-heap/80113#80113)（執行框架），它定義了在其中評估和執行 JavaScript 程式碼的環境。特別是，它定義了函數可以使用的物件（變數）。你可以說，這些物件的可用性定義了一個環境。例如，在 Web Worker 環境中執行的函數無權存取 `document` 全域物件。

**響應式環境定義了一個活動的消費者物件**，它依賴於生產者，並且在讀取其值時，可以供其存取器函數使用。例如，這裡我們有一個消費者 `isEvent`，它依賴於 `counter` 生產者（消費其值）。此依賴項是通過在 `computed` 回呼內存取 `counter` 的值來定義的：

```typescript
isEvent = computed(() => (counter() & 1) === 0)
```

當 `computed` 回呼執行時，它會自動執行 `counter` signal 的存取器函數以取得其值。**我們可以說，在這種情況下，`counter` signal 是在 `isEvent` 消費者的響應式環境中執行的。**因此，如果存在一個活動的消費者依賴於此生產者的值，則生產者將在響應式環境中執行。

為了實現響應式環境的這種機制，每次存取消費者的值時，但在重新計算它之前（在執行 `computed` 回呼之前），我們可以將此消費者設定為活動的消費者。這可以通過簡單地將該消費者物件分配給全域變數，並在執行回呼時將其保留在那裡來完成。此全域變數將可供在執行 `computed` 回呼期間查詢的所有生產者使用，並且它將定義此消費者所依賴的所有生產者的響應式環境。

這正是 Angular 正在做的事情。當執行計算的回呼時，它將首先將目前節點設定為 [`producerRecomputeValue`](https://github.com/angular/angular/blob/a5b5b7d5ef84b9852d2115dd7a764f4ab3299379/packages/core/primitives/signals/src/computed.ts#L111) 中的活動消費者：

```typescript
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

Angular 從 [`createComputed`](https://github.com/angular/angular/blob/a5b5b7d5ef84b9852d2115dd7a764f4ab3299379/packages/core/primitives/signals/src/computed.ts#L53) 工廠函數中的 `producerUpdateValueVersion` 中到達這裡：

```typescript
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

此呼叫堆疊也清楚地展示了此實現：

![圖片 22](https://wp.angular.love/wp-content/uploads/2024/08/1_o_jhWzaf9toobEpVXB37CQ-300x141.webp)

因此，在執行計算的回呼時，在該消費者處於活動狀態期間查詢的每個生產者，都將知道它們是在響應式環境中執行的。**在特定消費者的響應式環境中執行的所有生產者，都會被新增為該消費者的依賴項。**這構成了響應式圖形。

Angular 中大多數現有的功能都是在非響應式環境中執行的。你可以通過簡單地搜尋使用 `setActiveConsumer` 與 `null` 值的情況來觀察到這一點：

![圖片 23](https://wp.angular.love/wp-content/uploads/2024/08/1_AWAf6lmiOOr8VCNtGI3jYg-300x176.webp)

例如，在執行生命週期掛鉤之前，Angular 會清除響應式環境：

```typescript
/**
 * 執行單個生命週期掛鉤，確保：
 * - 它是在非響應式環境中呼叫的；
 * - 註冊設定檔資料。
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

Angular 樣板函數（元件視圖）和 effects 在響應式環境中執行。

響應式圖形
--------------

響應式圖形是通過消費者和生產者之間的依賴關係來建立的。通過值存取器的響應式環境實現，使得 signal 依賴項可以被自動且隱式地追蹤。使用者不需要宣告依賴項陣列，而且特定環境的依賴項集合也不需要在執行之間保持靜態。

當執行生產者時，它會將自身新增到目前活動消費者的依賴項中（定義目前響應式環境的消費者）。這會在 [`producerAccessed`](https://github.com/angular/angular/blob/1081c8d6233ba1ff09187b95a09b0644e130cdf8/packages/core/primitives/signals/src/graph.ts#L238) 函數內發生：

```typescript
export function producerAccessed(node: ReactiveNode): void {
  ...
  // 此生產者是 `activeConsumer` 的第 `idx` 個依賴項。
    const idx = activeConsumer.nextProducerIndex++;
    if (activeConsumer.producerNode[idx] !== node) {
      // 我們是消費者的一個新依賴項（在 `idx` 處）。
      activeConsumer.producerNode[idx] = node;
      // 如果活動的消費者是即時的，則將其新增為即時消費者。如果不是，則使用 0 作為佔位符值。
      activeConsumer.producerIndexOfThis[idx] = consumerIsLive(activeConsumer)
        ? producerAddLiveConsumer(node, activeConsumer, idx)
        : 0;
    }
```

生產者和消費者都參與響應式圖形。此依賴關係圖是雙向的，但是在每個方向上追蹤的依賴項方面存在差異。

生產者是通過 `producerNode` 屬性追蹤為消費者的依賴項，從**消費者建立到生產者的**邊：

```typescript
interface ConsumerNode extends ReactiveNode {
  producerNode: NonNullable<ReactiveNode['producerNode']>;
  producerIndexOfThis: NonNullable<ReactiveNode['producerIndexOfThis']>;
  producerLastReadVersion: NonNullable<ReactiveNode['producerLastReadVersion']>;
```

某些消費者也被追蹤為「即時」消費者，並且在另一個方向上建立邊，**從生產者建立到消費者**。這些邊用於在更新生產者的值時傳播變更通知：

```typescript
interface ProducerNode extends ReactiveNode {
  liveConsumerNode: NonNullable<ReactiveNode['liveConsumerNode']>;
  liveConsumerIndexOfThis: NonNullable<ReactiveNode['liveConsumerIndexOfThis']>;
}
```

消費者始終追蹤它們所依賴的生產者。生產者僅追蹤來自被視為「即時」的消費者的依賴項。當消費者將 `consumerIsAlwaysLive` 屬性設定為 `true` 時，或是一個被即時消費者依賴的生產者時，該消費者被視為「即時」。

在 Angular 中，兩種節點類型被定義為即時消費者：

*   [watch](https://github.com/angular/angular/blob/a5b5b7d5ef84b9852d2115dd7a764f4ab3299379/packages/core/primitives/signals/src/watch.ts#L137) 節點（用於 effects 中）
*   響應式 [LView](https://github.com/angular/angular/blob/4c7d5d8acd8a714fe89366f76dc69f91356f0a06/packages/core/src/render3/reactive_lview_consumer.ts#L51) 節點（用於變更偵測中）

這是它們的定義：

```typescript
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

在某些情況下，`computed` signals 可能會變成「即時」消費者，例如，當在 `effect` 回呼中使用時。

以下程式碼設定：

```typescript
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

    // 需要等待變更偵測來通知 effect
    setTimeout(() => {
      // effect 依賴於 B
      const depEToB = e.watcher[SIGNAL].producerNode[0] === nodes[B];

      // 即時消費者連結從生產者 A 到 B，
      // 以及從 B 到 E，因為 E (effect) 是即時消費者
      const depLiveAToB = nodes[A].liveConsumerNode[0] === nodes[B];
      const depLiveBToE = nodes[B].liveConsumerNode[0] === e.watcher[SIGNAL];

      console.log(depLiveAToB, depLiveBToE, depEToB);
    });
  }
}
```

將產生以下圖形：

![圖片 24](https://wp.angular.love/wp-content/uploads/2024/08/1_vDJV8hF-twTuf1GckjPnZQ-300x123.webp)

通過活動消費者實現的響應式環境能夠實現**動態依賴項追蹤**。當將某個消費者設定為活動狀態時，被評估的生產者會通過這些生產者呼叫的順序來動態定義。每次在消費者的響應式環境中存取生產者時，都可以為 `ActiveConsumer` 重新排列依賴項清單。

為了實現這一點，消費者的依賴項會追蹤在 `producerNode` 陣列中：

```typescript
interface ConsumerNode extends ReactiveNode {
  producerNode: NonNullable<ReactiveNode['producerNode']>;
  producerIndexOfThis: NonNullable<ReactiveNode['producerIndexOfThis']>;
  producerLastReadVersion: NonNullable<ReactiveNode['producerLastReadVersion']>;
```

當重新執行特定消費者的計算時，該陣列中的指標（索引）`producerIndexOfThis` 會初始化為索引 `0`，並且每個讀取的依賴項都會與先前執行中指標目前位置的依賴項進行比較。如果存在不匹配，則表示自上次執行以來，依賴項已變更，並且可以刪除舊的依賴項並替換為新的依賴項。在執行結束時，可以刪除任何剩餘的不匹配依賴項。

這表示，如果你只有一個分支需要依賴項，並且先前的計算採用了另一個分支，那麼即使在提取時，對該臨時未使用的值進行變更也不會導致重新計算計算的 signal。這導致了在一次執行到下一次執行時存取的不同 signals 集合的可能性。

例如，此計算的 signal `dynamic` 會根據 `useA` signal 的值讀取 `dataA` 或 `dataB`：

```typescript
const dynamic = computed(() => useA() ? dataA() : dataB());
```

在任何給定的時間點，它將具有 `[useA, dataA]` 或 `[useA, dataB]` 的依賴項集合，並且它永遠不能同時依賴於 `dataA` 和 `dataB`。

此程式碼類似於 Angular 中的[此測試案例](https://github.com/proposal-signals/signal-polyfill/blob/4cf87cef28aa89e938f079e4d82e9bf10f6d0a4c/tests/behaviors/dynamic-dependencies.test.ts#L4)，清楚地展示了這一點：

```typescript
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

如你所見，圖形沒有單個起始頂點。由於每個消費者都保留一個依賴生產者的清單，而這些生產者又可能具有依賴項，例如計算的 signal，因此你可以說，每個消費者在被存取時都是圖形的根頂點。

兩階段更新
-----------------

早期基於推送的響應式模型面臨著冗餘計算的問題：如果對狀態 signal 的更新導致計算的 signal 急切地執行，則最終可能會將更新推送至 UI。但是，如果在下一個框架之前，源狀態 signal 將再次發生變更，則此寫入 UI 的操作可能為時過早。

例如，對於這樣的圖形，此問題涉及無意中評估 `A -> B -> D` 和 `C`，然後由於 `C` 已變更而重新評估 `D`。重新評估 `D` 兩次是低效的，並且可能導致使用者明顯地看到故障。

![圖片 25](https://wp.angular.love/wp-content/uploads/2024/08/0_IgUGB8HjKNyQRqhV.webp)

這被稱為菱形問題。

有時，由於這種[故障](https://en.wikipedia.org/wiki/Reactive_programming#Glitches)，甚至會向最終使用者顯示不準確的中間值。Signals 通過基於提取（惰性）而不是基於推送的方式來避免這種動態行為：當框架排程 UI 呈現時，它將提取適當的更新，從而避免在計算以及寫入 DOM 中浪費工作。

考慮以下範例：

```typescript
const a = signal(0);

const b = computed(() => a() + 'b');
const c = computed(() => a() + 'c');
const d = computed(() => b() + c() + 'd');

// 執行計算的回呼以設定依賴項
d();

// 更新圖形頂部的 signal
setTimeout(() => a.set(1), 2000);
```

一旦 `a` 更新，就不會發生傳播。僅更新節點的值和版本：

```typescript
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

當我們稍後存取 `d()` 的值時，signals 實現會向上輪詢 `d` 的依賴項，通過 `consumerPollProducersForChange` 來確定是否需要重新計算。

為了有效地處理，所有響應式節點都會記錄依賴項節點的版本。要確定變更，只需簡單地比較生產者節點的已儲存版本與節點上的實際版本：

```typescript
interface ConsumerNode extends ReactiveNode {
...
producerLastReadVersion: NonNullable<ReactiveNode['producerLastReadVersion']>;
}

function consumerPollProducersForChange(node) {
...
// 輪詢生產者以了解變更。
for (let i = 0; i < node.producerNode.length; i++) {
const producer = node.producerNode[i];
const seenVersion = node.producerLastReadVersion[i];
// 首先檢查版本。不匹配表示自上次讀取以來，生產者的值已知已變更。
if (seenVersion !== producer.version) {
return true;
}
```

如果這些版本不同，則表示生產者已發生變更，並且實現將通過 `producerRecomputeValue` 執行計算回呼的重新計算：

```typescript
export function producerUpdateValueVersion(node: ReactiveNode): void {
  ...

  if (!node.producerMustRecompute(node) && !consumerPollProducersForChange(node)) {
    // 自上次讀取以來，我們沒有任何生產者報告變更，因此無需重新計算我們的值，並且我們可以認為自己是乾淨的。
    node.dirty = false;
    node.lastCleanEpoch = epoch;
    return;
  }

  node.producerRecomputeValue(node);

  // 重新計算值後，我們不再是髒的。
  node.dirty = false;
  node.lastCleanEpoch = epoch;
}
```

這將對 `C` 的依賴項重複該過程。通過這種方式，它將到達節點 `A`，此時將導致評估分支 `D->C->A`。但是由於 `D` 也依賴於 `B` 生產者，因此它將在計算 `D` 之前重新評估 `B`。通過這種方式，不會出現 `D` 的雙重計算問題。

但是，有時你可能需要急切地通知某些消費者。正如你可能已經猜到的，這些被稱為「即時」消費者。在這種情況下，只要更新生產者的值，變更通知就會通過圖形傳播，通知依賴於生產者的即時消費者。

其中一些消費者可能是衍生值，因此也是生產者，這會使其快取值失效，然後繼續將變更通知傳播到自己的即時消費者，依此類推。最終，此通知會到達 effects，這些 effects 會排程自身以重新執行。

**至關重要的是，在此階段，不會執行任何副作用，也不會執行任何中間或衍生值的重新計算，僅執行快取值的失效。這允許變更通知到達圖形中的所有受影響節點，而不會出現觀察到中間或錯誤狀態的可能性。**

如果需要，一旦此變更傳播完成（同步），此階段可以接著我們上面看到的惰性評估。

要查看此通知階段的實際運作情況，讓我們在我們的設定中新增一個即時消費者，例如監看者。當更新 `a` 時，更新會傳播到相依的即時消費者：

```typescript
import { computed, signal } from '@angular/core';
import { createWatch } from '@angular/core/primitives/signals';

const a = signal(0);
const b = computed(() => a() + 'b');
const c = computed(() => a() + 'c');
const d = computed(() => b() + c() + 'd');

setTimeout(() => a.set(1), 3000);

// 監看者將設定對 `d` 的依賴項
const watcher = createWatch(
  () => console.log(d()),
  () => setTimeout(watcher.run, 1000),
  false
);

watcher.notify();
```

一旦我們更新 `a.set(1)` 的值，我們就可以看到即時消費者正在運作中的通知：

![圖片 26](https://wp.angular.love/wp-content/uploads/2024/08/1_39Jgsu7ROYU5MEoWGnMF8w-300x181.webp)

節點 `b` 和 `c` 是節點 `a` 的即時消費者，因此當執行 `a` 的更新時，Angular 將遍歷 `node.liveConsumerNode` 並通知這些節點有關變更。

但是，正如前面提到的，這裡沒有真正發生任何事情。該節點只是被標記為髒的，並通過 `producerNotifyConsumers` 將通知傳播到其即時消費者：

```typescript
function consumerMarkDirty(node) {
  node.dirty = true;
  producerNotifyConsumers(node);
  node.consumerMarkedDirty?.(node);
}
```

所有這些都會一路傳遞到依賴於 `d` 的監看者 (effect)。與常規響應式節點相反，監看節點在其 `consumerMarkedDirty` 方法中實現排程：

```typescript
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

在這裡，通知階段和圖形遍歷會停止。

這個兩階段的過程有時被稱為「推送/提取」演算法：「骯髒」會在變更來源 signal 時通過圖形急切推送，但是重新計算是惰性執行的，僅當通過讀取它們的 signals 來提取值時才執行。

變更偵測
----------------

為了將基於 signals 的通知整合到變更偵測過程中，Angular 依賴於即時消費者的機制。元件樣板會編譯為樣板表達式（JS 程式碼），並在該元件視圖的響應式環境中執行。在這種情況下，執行 signal 將傳回該值，但也將 signal 註冊為該元件視圖的依賴項。

**由於樣板表達式是即時消費者，因此 Angular 將建立從生產者到樣板表達式節點的連結。只要生產者的值更新，該生產者就會立即同步通知樣板節點。收到通知後，Angular 會標記元件及其所有祖先以進行檢查。**

你可能已經[從我的其他文章](https://angular.love/change-detection-and-component-trees-in-angular-applications)中了解到，每個元件的樣板在內部都表示為 `LView` 物件。以下是元件的外觀：

```typescript
@Component({...})
export class AppComponent {
  value = signal(0);
}
```

編譯時，它看起來像常規的 JS 函數 `AppComponent_Template`，在該元件的變更偵測期間執行：

```typescript
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

當 Angular 將 signals 新增到其變更偵測實現時，它將所有元件視圖（樣板函數）包裝在 `ReactiveLViewConsumer` 節點中：

```typescript
export interface ReactiveLViewConsumer extends ReactiveNode {
  lView: LView | null;
}
```

此介面由 [`REACTIVE_LVIEW_CONSUMER_NODE`](https://github.com/angular/angular/blob/4c7d5d8acd8a714fe89366f76dc69f91356f0a06/packages/core/src/render3/reactive_lview_consumer.ts#L51) 節點實現：

```typescript
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

你可以將此過程想像成每個視圖都獲得自己的 `ReactiveLViewConsumer` **消費者**節點，該節點定義了在樣板函數內存取的所有 signals 的響應式環境。

在我們的範例中，每當樣板函數作為變更偵測的一部分執行時，它都會在樣板函數節點的環境中執行 `ctx.value()` 生產者，而該節點是一個 `ActiveConsumer`：

![圖片 27](https://wp.angular.love/wp-content/uploads/2024/08/1_qeeISKeap4-Oo6WlikyPZQ-300x204.webp)

這將導致將樣板表達式節點（消費者）作為**即時**依賴項新增至生產者 `value()`：

![圖片 28](https://wp.angular.love/wp-content/uploads/2024/08/1_O9Cqy1y_Q8rnwxpu84BieA-300x173.webp)

此依賴項確保一旦生產者 `counter` 的值變更，它將立即通知消費者節點（樣板表達式）。

當生產者的值變更時，即時消費者會同步呼叫 `consumerMarkDirty` 方法：

```typescript
/**
 * 將骯髒的通知傳播到此生產者的即時消費者。
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

function consumerMarkDirty(node: ReactiveNode): void {
  node.dirty = true;
  producerNotifyConsumers(node);
  node.consumerMarkedDirty?.(node);
}
```

在 `consumerMarkedDirty` 內部，樣板表達式節點將使用 `markAncestorsForTraversal` 來標記要刷新的祖先，其方式與之前 `markForCheck()` 的方式類似：

```typescript
const REACTIVE_LVIEW_CONSUMER_NODE: Omit<ReactiveLViewConsumer, 'lView'> = {
  ...
  consumerMarkedDirty: (node: ReactiveLViewConsumer) => {
    markAncestorsForTraversal(node.lView!);
  },
};

function markAncestorsForTraversal(lView: LView) {
  let parent = getLViewParent(lView);
  while (parent !== null) {
    ...
    parent[FLAGS] |= LViewFlags.HasChildViewsToRefresh;
    parent = getLViewParent(parent);
  }
}
```

最後一個問題是 Angular 何時將目前的 `LView` 消費者節點設定為 `ActiveConsumer`？這一切都發生在你可能已經從我之前的文章中了解到的 [`refreshView`](https://github.com/angular/angular/blob/4c7d5d8acd8a714fe89366f76dc69f91356f0a06/packages/core/src/render3/instructions/change_detection.ts#L192) 函數中。

此函數對每個 `LView` 執行變更偵測，並執行常見的變更偵測操作：執行樣板函數、執行掛鉤、刷新查詢和設定主機綁定。基本上，在 Angular 執行所有這些操作之前，已經新增了整段程式碼來處理響應性。

以下是它的外觀：

```typescript
function refreshView<T>(tView, lView, templateFn, context) {
  ...

  // 啟動元件響應式環境
  enterView(lView);
  let returnConsumerToPool = true;
  let prevConsumer: ReactiveNode | null = null;
  let currentConsumer: ReactiveLViewConsumer | null = null;
  if (!isInCheckNoChangesPass) {
    if (viewShouldHaveReactiveConsumer(tView)) {
      currentConsumer = getOrBorrowReactiveLViewConsumer(lView);
      prevConsumer = consumerBeforeComputation(currentConsumer);
    } else {... }

    ...

    try {
      ...
      if (templateFn !== null) {
        executeTemplate(tView, lView, templateFn, RenderFlags.Update, context);
      }
  }
```

由於此程式碼是在 Angular 在 `executeTemplate` 程式碼中執行元件的樣板函數之前執行的，因此，當執行元件樣板中使用的 signals 的存取器函數時，已經設定了響應式環境。

Effects 和監看者
--------------------

Effect 是一種專門的工具，旨在根據應用程式的狀態執行具有副作用的操作。Effects 是使用在響應式環境中執行的回呼定義的即時消費者。會擷取此函數的 signal 依賴項，並且每當其任何依賴項產生新值時，都會通知 effect。

在大多數應用程式程式碼中很少需要 effects，但在特定情況下可能會很有用。以下是 Angular 文件中建議的一些使用範例：

*   記錄資料或使其與 `window.localStorage` 同步
*   新增無法使用樣板語法表達的自訂 DOM 行為，例如對 `<canvas>` 元素執行自訂呈現

**Angular 不會在變更偵測機制中使用 effects 來觸發元件的 UI 更新。如變更偵測部分所述，對於此功能，它依賴於即時消費者的機制。**

雖然 signal 演算法是標準化的，但尚未定義 effects 應如何運作的詳細資訊，並且在不同的框架之間會有所不同。這是因為 effect 排程的細微性質，它通常會與框架呈現週期和其他高階、框架特定的狀態或 JavaScript 無法存取的策略整合。

但是，signal 提案定義了一組基本概念，即 [watch](https://github.com/angular/angular/blob/main/packages/core/primitives/signals/README.md#side-effects-createwatch) API，框架作者可以使用這些基本概念來建立自己的 effects。`Watcher` 介面用於監看響應式函數，並在該函數的依賴項變更時接收通知。

在 Angular 中，`effect` 是 `watcher` 的包裝器。首先，讓我們探索監看者如何運作，我們將看到它們如何用於建構 `effect` 基本概念。

首先，我們將從 Angular 基本概念中匯入 `watcher`，並使用它來實現通知機制：

```typescript
import { createWatch } from '@angular/core/primitives/signals';

const counter = signal(0);

const watcher = createWatch(
  // 執行使用者提供的回呼並設定追蹤
  // 這將執行 2 次
  // 第一次在 `watcher.notify()` 之後，第二次在 `this.counter.set(1)` 之後
  () => counter(),
  // 這由 `notify` 方法呼叫
  // 或由消費者本身通過 consumerMarkDirty 方法呼叫，
  // 會排程使用者提供的回呼在 1000 毫秒後執行
  () => setTimeout(watcher.run, 1000),
  false
);

// 將監看者標記為骯髒的 (過時的)，以強制使用者提供的回呼
// 執行並設定 `counter` signal 的追蹤
// `notify` 方法會在底層呼叫 `consumerMarkDirty`
watcher.notify();

// 當值變更時，會執行 consumerMarkDirty
// 它會排程使用者提供的回呼執行
setTimeout(() => this.counter.set(1), 3000);
```

當我們執行 `watcher.notify()` 時，Angular 會在監看者節點上同步呼叫 `consumerMarkDirty` 方法。但是，使用者定義的通知回呼不會在收到通知後立即執行。相反，它被排程在未來某個時間通過 `watcher.run` 執行。當 `watch` 接收到「markDirty」通知時，它將僅呼叫此排程操作。

你可以在實際運作中看到：

![圖片 29](https://wp.angular.love/wp-content/uploads/2024/08/1_FwpksG4QvcTYICQ6mHRChg-300x96.webp)

當我們執行 `this.counter.set(1)` 時，相同的呼叫鏈會導致排程使用者提供的回呼。

為了建構 `effect()` 函數，Angular 將監看者包裝在 `EffectHandle` 類別中：

```typescript
export function effect(effectFn,options): EffectRef {
  const handle = new EffectHandle();
  ...
  return handle;
}

class EffectHandle implements EffectRef, SchedulableEffect {
  unregisterOnDestroy: (() => void) | undefined;
  readonly watcher: Watch;

  constructor(...) {
    this.watcher = createWatch(
      (onCleanup) => this.runEffect(onCleanup),
      () => this.schedule(),
      allowSignalWrites,
    );
    this.unregisterOnDestroy = destroyRef?.onDestroy(() => this.destroy());
  }
```

你可以看到 `EffectHandle` 類別是設定監看者的地方。對於我們之前使用監看者的上述範例，使用 `effect` 函數將大大簡化設定：

```typescript
import { Component, effect, signal } from '@angular/core';

@Component({...})
export class AppComponent {
  counter = null;

  constructor() {
    this.counter = signal(0);

    // 這將執行 2 次
    effect(() => this.counter());

    setTimeout(() => this.counter.set(1), 3000);
  }
}
```

當我們直接使用 `effect` 函數時，我們只傳遞一個回呼。這是使用者定義的回呼，它設定依賴項，並且在更新依賴項時由 Angular 排程執行。

Angular effects 中使用的目前排程器是 `ZoneAwareEffectScheduler`，它在變更偵測週期之後作為微任務佇列的一部分執行更新：

```typescript
export class ZoneAwareEffectScheduler implements EffectScheduler {
  private queuedEffectCount = 0;
  private queues = new Map<Zone | null, Set<SchedulableEffect>>();
  private readonly pendingTasks = inject(PendingTasks);
  private taskId: number | null = null;

  scheduleEffect(handle: SchedulableEffect): void {
      this.enqueue(handle);
      if (this.taskId === null) {
        const taskId = (this.taskId = this.pendingTasks.add());
        queueMicrotask(() => {
          this.flush();
          this.pendingTasks.remove(taskId);
          this.taskId = null;
        });
      }
    }
```

Angular 必須實現一個有趣的特性來「初始化」effect。正如我們在監看者的實現中所看到的，我們需要通過手動呼叫 `watcher.notify()` 一次來啟動追蹤。Angular 也需要這樣做，並且在第一次執行變更偵測時這樣做。

以下是它的實現方式。

當你在元件的注入環境中執行 `effect` 函數時，Angular 會將通知回呼新增至元件的視圖物件 `LView[EFFECTS_TO_SCHEDULE]`：

```typescript
export function effect(
  effectFn: (onCleanup: EffectCleanupRegisterFn) => void,
  options?: CreateEffectOptions,
): EffectRef {
  ...
  const handle = new EffectHandle();

  // 需要手動將 effects 標記為骯髒的，以觸發它們的初始執行。此標記的時機很重要，因為 effects 可能會讀取追蹤元件輸入的 signals，這些 signals 僅在這些元件進行首次更新傳遞後才可用。
  // ...
  const cdr = injector.get(ChangeDetectorRef, null, {optional: true}) as ViewRef<unknown> | null;
  if (!cdr || !(cdr._lView[FLAGS] & LViewFlags.FirstLViewPass)) {
    // 此 effect 要么未在視圖注入器中執行，要么視圖已經進行了第一次變更偵測傳遞，這對於設定任何所需的輸入是必要的。
    handle.watcher.notify();
  } else {
    // 將 effect 的初始化延遲到視圖完全初始化之後。
    (cdr._lView[EFFECTS_TO_SCHEDULE] ??= []).push(handle.watcher.notify);
  }

  return handle;
}
```

以這種方式新增的通知函數將在此元件視圖在 `refreshView` 函數中首次執行變更偵測時執行一次：

```typescript
export function refreshView<T>(tView,lView,templateFn,context) {
   ...
  
   // 排程任何正在等待此視圖更新傳遞的 effects。
    if (lView[EFFECTS_TO_SCHEDULE]) {
      for (const notifyEffect of lView[EFFECTS_TO_SCHEDULE]) {
        notifyEffect();
      }

      // 一旦它們被執行，我們可以刪除陣列。
      lView[EFFECTS_TO_SCHEDULE] = null;
    }
}
```

呼叫 `notifyEffect` 將觸發底層監看者的 `consumerMarkDirty` 通知回呼，而該監看者又將使用現有的排程器（在變更偵測之後）來排程 effect（使用者提供的回呼）執行：

![圖片 30](https://wp.angular.love/wp-content/uploads/2024/08/1_BwgXXo4piUtwmCpAB9dWXw-300x87.webp)

這就是整個故事 🙂
