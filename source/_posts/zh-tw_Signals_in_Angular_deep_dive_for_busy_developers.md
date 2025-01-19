---
title: Signals in Angular：deep dive for busy developers
date: 2024-12-24 18:40:14
tags:
---

原文：<https://angular.love/signals-in-angular-deep-dive-for-busy-developers>

# Angular 中的 Signals：忙碌開發者的深度解析

建構複雜的使用者介面是一項艱鉅的任務。在現代的 web 應用程式中，UI 的 state 很少由簡單的獨立值組成。它更像是一個複雜的 computed state，取決於其他值或 computed state 的複雜層次結構。管理這個 state 涉及大量的工作：開發人員必須儲存、計算、使無效和同步這些值。

多年來，web 開發中引入了各種框架和基本要素來簡化這項任務。它們大多數的一個核心主題是 reactive programming，它提供了管理應用程式 state 的基礎架構，讓開發人員可以專注於商業邏輯，而不是重複的 state 管理任務。

最新加入的是 signals，一種 "reactive" 的基本要素，它代表一個動態變化值，並可以在值發生變化時通知感興趣的 consumers。這些 consumers 反過來可以運行重新計算或各種 side-effect，例如建立/銷毀 components、運行網路請求、更新 DOM 等。

我們可以在不同的框架中找到 signals 的不同實現。現在甚至有標準化 signals 的努力：

... 這項努力的重點是使 JavaScript 生態系統保持一致。一些框架作者正在這裡合作開發一個通用模型，該模型可以支持他們的核心 reactivity。目前的草案基於來自 Angular、Bubble、Ember、FAST、MobX、Preact、Qwik、RxJS、Solid、Starbeam、Svelte、Vue、Wiz 等作者/維護者的設計輸入 ...

Angular 中 signals 的實現與提案中提供的實現非常相似，因此我可能會在本文中在這兩者之間進行交叉引用。

<!-- more -->

**Signals 作為基本要素**

一個 signal 代表一個資料單元，它可能會隨著時間而改變。Signals 可以是 "state" (只是一個手動設定的值) 或 "computed" (可以想像成基於其他 signals 的公式)。

Computed signals 的運作方式是自動追蹤它們在評估期間讀取了哪些其他 signals。當讀取 computed signal 時，它會檢查其先前記錄的任何 dependencies 是否已變更，如果已變更，則會重新評估自身。

例如，這裡我們有一個 state signal `counter` 和一個 computed signal `isEven`。我們將 `counter` 的初始值設定為 0，然後將其更改為 1。你可以看到 computed signal `isEven` 通過在 `counter` signal 更新之前和之後產生兩個不同的值來對變更做出反應：

```typescript
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

另請注意，在上面的範例中，`isEven` signal 沒有明確地訂閱來源 `counter` signal。相反，它只是在其 computed function 中使用 `counter()` 呼叫來源 signal。這足以連結兩個 signals。因此，每當 `counter` 來源 signal 更新為新值時，派生的 signal 也會自動更新。

State 和 computed signals 都被視為值的 producers。Producers 代表產生值並可以傳遞變更通知的 signals。

當通過 API 呼叫更新值時，state signal 會變更 (產生) 其值，而當 callback 中使用的 dependencies 變更時，computed signal 會自動產生新值。

Computed signals 也可以是 consumers，因為它們可能取決於一些 producers (consume)。在其他 reactive 的實現中，例如 Rx，consumers 也被稱為 sinks。

當 producer signal 的值變更時，dependent 的 consumers (例如 computed signals) 的值不會立即更新。當讀取 computed signal 時，它會檢查其先前記錄的任何 dependencies 是否已變更，如有必要，則會重新評估自身。

這使得 computed signals 具有 lazy 或 pull-based 的特性，意味著它們僅在被存取時才會被評估，即使底層的 state 較早發生變更。在我們上面的範例中，computed signal 的值僅在我們呼叫 `isEven()` 時才會被評估，儘管對底層 dependency `counter` 的更新較早發生，即當我們執行 `counter.set()` 時。

除了 regular writable 和 computed signals 之外，還有一個 watchers (effects) 的概念。與 computed signals 的 pull based 評估相反，變更 producer signal 將會立即通知 watcher，同步呼叫 watcher 的通知 callback，有效地 "push" 通知。框架將 watchers 包裝到向使用者公開的 effects 中。Effects 通過排程來延遲使用者程式碼的通知。

與 Promises 不同，signals 中的所有內容都是同步運行的：

*   將 signal 設定為新值是同步的，並且在之後讀取任何依賴於它的 computed signal 時會立即反映出來。沒有內建的批次處理此 mutation。
*   讀取 computed signals 是同步的 - 它們的值始終可用。
*   Watchers 會同步收到通知，但包裝這些 watchers 的 effects 可以選擇通過排程來批次處理和延遲通知。

**實現細節**

在內部，signals 的實作定義了一些概念，我想在本文中解釋這些概念：reactive context、dependency graph 和 effects (watchers)。讓我們從 reactive context 開始。

要討論 reactive context，可以想像一個堆疊框架 (execution frame)，它定義了 JavaScript 程式碼被評估和執行的環境。具體來說，它定義了哪些 objects (variables) 可供 function 使用。你可以說，這些 objects 的可用性定義了一個 context。例如，在 web worker context 中運行的 function 無法存取 `document` 全域 object。

Reactive context 定義了一個 active consumer object，它依賴於 producers，並且在它們的值被讀取時可供其存取 function 使用。例如，這裡我們有一個 consumer `isEvent`，它依賴於 `counter` producer (consume 其值)。該 dependency 是通過存取 computed callback 內部的 `counter` 值來定義的：

```typescript
isEvent = computed(() => (counter() & 1) === 0)
```

當 computed callback 運行時，它將自動執行 `counter` signal 的 accessor function 來取得其值。我們可以說，在這種情況下，`counter` signal 是在 `isEvent` consumer 的 reactive context 中執行的。因此，如果有一個 active consumer 依賴於此 producer 的值，則 producer 會在 reactive context 中執行。

為了實現 reactive context 的這種機制，每次當存取 consumer 的值時，但在重新計算之前 (在 computed callback 運行之前)，我們可以將此 consumer 設定為 active consumer。這可以通過簡單地將該 consumer object 指派給全域變數並在 callback 執行時將其保留在那裡來完成。這個全域變數將可供在 computed callback 執行期間查詢的所有 producers 使用，並且它將為此 consumer 依賴的所有 producers 定義 reactive context。

這正是 Angular 正在做的事情。當 computed callback 執行時，它會先將目前的 node 設定為 `producerRecomputeValue` 中的 active consumer：

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

Angular 從 `createComputed` factory function 內部的 `producerUpdateValueVersion` 取得：

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

這個呼叫堆疊也清楚地展示了這種實現：

![alt text](https://wp.angular.love/wp-content/uploads/2024/08/1_o_jhWzaf9toobEpVXB37CQ-300x141.webp)

因此，當 computed 的 callback 正在執行時，在該 consumer 處於 active 狀態期間查詢的每個 producer 都會知道它們是在 reactive context 中執行的。在特定 consumer 的 reactive context 中執行的所有 producers 都會作為 consumer 的 dependencies 添加。這構成了一個 reactive graph。

Angular 中的大多數預先存在的功能都是在非 reactive context 中執行的。你可以通過簡單地搜尋 `setActiveConsumer` 的 null 值用法來觀察到這一點：

![alt text](https://wp.angular.love/wp-content/uploads/2024/08/1_AWAf6lmiOOr8VCNtGI3jYg-300x176.webp)

例如，在運行 lifecycle hooks 之前，Angular 會清除 reactive context：

```typescript
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

Angular template functions (component views) 和 effects 在 reactive contexts 中運行。

**Reactive graph**

Reactive graph 是通過 consumers 和 producers 之間的 dependencies 來建構的。通過值 accessors 實現的 Reactive context 使 signal dependencies 可以自動且隱含地被追蹤。使用者不需要宣告 dependencies 的陣列，特定 context 的 dependencies 集合也不需要在 execution 之間保持靜態。

當執行 producer 時，它會將自身添加到目前 active consumer 的 dependencies 中 (定義目前 reactive context 的 consumer)。這發生在 `producerAccessed` function 內部：

```typescript
export function producerAccessed(node: ReactiveNode): void {
  ...
  // This producer is the idxth dependency of activeConsumer.
  const idx = activeConsumer.nextProducerIndex++;
  if (activeConsumer.producerNode[idx] !== node) {
    // We're a new dependency of the consumer (at idx).
    activeConsumer.producerNode[idx] = node;
    // If the active consumer is live, then add it as a live consumer. If not, then use 0 as a
    // placeholder value.
    activeConsumer.producerIndexOfThis[idx] = consumerIsLive(activeConsumer)
      ? producerAddLiveConsumer(node, activeConsumer, idx)
      : 0;
  }
```

Producers 和 consumers 都參與 reactive graph。這個 dependency graph 是雙向的，但每個方向追蹤的 dependencies 有所不同。

Producers 作為 consumer 的 dependencies 通過 `producerNode` property 追蹤，從 consumers 建立到 producers 的邊：

```typescript
interface ConsumerNode extends ReactiveNode {
  producerNode: NonNullable<ReactiveNode['producerNode']>;
  producerIndexOfThis: NonNullable<ReactiveNode['producerIndexOfThis']>;
  producerLastReadVersion: NonNullable<ReactiveNode['producerLastReadVersion']>;
}
```

某些 consumers 也被追蹤為 "live" consumers，並在另一個方向上建立邊，從 producer 到 consumer。這些邊用於在更新 producer 的值時傳播變更通知：

```typescript
interface ProducerNode extends ReactiveNode {
  liveConsumerNode: NonNullable<ReactiveNode['liveConsumerNode']>;
  liveConsumerIndexOfThis: NonNullable<ReactiveNode['liveConsumerIndexOfThis']>;
}
```

Consumers 始終追蹤它們依賴的 producers。Producers 只追蹤來自被視為 "live" 的 consumers 的 dependencies。當 consumer 的 `consumerIsAlwaysLive` property 設定為 true，或者是一個依賴於 live consumer 的 producer 時，consumer 就是 "live" 的。

在 Angular 中，有兩種 node 類型被定義為 live consumers：

*   watch nodes (用於 effects)
*   reactive LView nodes (用於 change detection)

這是它們的定義：

```typescript
const WATCH_NODE: Partial<WatchNode> = /* @PURE */ (() => {
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

在某些情況下，computed signals 可能會成為 "live" consumers，例如，當在 effect callback 中使用時。

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

將產生以下 graph：

![alt text](https://wp.angular.love/wp-content/uploads/2024/08/1_vDJV8hF-twTuf1GckjPnZQ-300x123.webp)

通過 active consumer 實現的 reactive context 可以實現動態 dependency 追蹤。當將特定 consumer 設定為 active 時，被評估的 producers 是通過這些 producers 呼叫的順序動態定義的。每次在該 consumer 的 reactive context 中存取 producer 時，都可以為 ActiveConsumer 重新排列 dependency 列表。

為了實現這一點，consumer 的 dependencies 會在 `producerNode` 陣列中追蹤：

```typescript
interface ConsumerNode extends ReactiveNode {
  producerNode: NonNullable<ReactiveNode['producerNode']>;
  producerIndexOfThis: NonNullable<ReactiveNode['producerIndexOfThis']>;
  producerLastReadVersion: NonNullable<ReactiveNode['producerLastReadVersion']>;
}
```

當特定 consumer 的計算重新運行時，一個指標 (index) `producerIndexOfThis` 將初始化為 index 0，並且每個讀取的 dependency 都會與上一次運行時指標目前位置的 dependency 進行比較。如果存在不匹配，則表示 dependencies 自上次運行以來已變更，並且可以刪除舊的 dependency 並替換為新的 dependency。在運行結束時，可以刪除任何剩餘的不匹配 dependencies。

這表示，如果你只有一個分支需要 dependency，並且先前的計算採用了另一個分支，那麼即使在 pull 時，對該暫時未使用的值進行變更也不會導致重新計算 computed signal。這導致可能從一個 execution 到下一個 execution 存取不同的 signal 集合。

例如，此 computed signal 會根據 `useA` signal 的值動態讀取 `dataA` 或 `dataB`：

```typescript
const dynamic = computed(() => useA() ? dataA() : dataB());
```

在任何給定的時間點，它都將具有 `[useA, dataA]` 或 `[useA, dataB]` 的 dependency 集合，並且永遠不能同時依賴於 `dataA` 和 `dataB`。

此程式碼類似於 Angular 中的此測試案例，清楚地表明了這一點：

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

如你所見，graph 沒有一個單一的起始頂點。由於每個 consumer 都保留了 dependency producers 的列表，而這些 producers 又可能具有 dependencies，例如 computed signal，因此你可以說，每個 consumer 在被存取時都是 graph 的根頂點。

**兩階段更新**

早期的 push-based reactivity 模型面臨著冗餘計算的問題：如果對 state signal 的更新導致 computed signal 急切運行，最終可能會將更新 push 到 UI。但是，如果下一個 frame 之前還有另一個對原始 state signal 的變更，則對 UI 的寫入可能會過早。

例如，對於像這樣的 graph，這個問題涉及無意中評估 `A -> B -> D` 和 `C`，然後由於 `C` 已變更而重新評估 `D`。重新評估 `D` 兩次是低效的，並且可能會導致使用者明顯的 glitches。

![alt text](https://wp.angular.love/wp-content/uploads/2024/08/0_IgUGB8HjKNyQRqhV.webp)

這被稱為鑽石問題。

有時，由於這種 glitches，甚至會向終端使用者顯示不準確的中間值。Signals 通過 pull-based (lazy) 而不是 push-based 來避免這種動態：在框架排程 UI 渲染時，它會 pull 適當的更新，避免在計算和寫入 DOM 中浪費工作。

考慮這個範例：

```typescript
const a = signal(0);

const b = computed(() => a() + 'b');
const c = computed(() => a() + 'c');
const d = computed(() => b() + c() + 'd');

// run the computed callback to set up dependencies
d();

// update the signal at the top of the graph
setTimeout(() => a.set(1), 2000);
```

一旦更新 `a`，就不會發生傳播。僅更新 node 的值和版本：

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

當我們稍後存取 `d()` 的值時，signals 的實作會向上 poll `d` 的 dependencies，通過 `consumerPollProducersForChange` 來確定是否需要重新計算。

為了實現有效率的處理，所有 reactive nodes 都會記錄 dependency node 的版本。為了確定變更，只需將 producer node 的已儲存版本與 node 上的實際版本進行比較就足夠了：

```typescript
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
}
```

如果這些不同，則表示 producer 已變更，並且實作將通過 `producerRecomputeValue` 運行 computed callback 的重新計算：

```typescript
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

這將對 `C` 的 dependencies 重複此過程。這樣，它將到達 node `A`，此時將導致評估分支 `D->C->A`。但是由於 `D` 也依賴於 `B` producer，因此它會在計算 `D` 之前重新評估該 producer。這樣，`D` 就沒有重複計算的問題。

但是，有時你可能需要急切地通知某些 consumers。你可能已經猜到了，這些被稱為 "live" consumers。在這種情況下，一旦更新 producer value，變更通知就會通過 graph 傳播，通知依賴於 producer 的 live consumers。

其中一些 consumers 可能是派生值，因此也是 producers，它們會使快取的值無效，然後繼續將變更通知傳播到它們自己的 live consumers，依此類推。最終，此通知到達 effects，這些 effects 會排程自身以重新執行。

至關重要的是，在此階段，不會運行任何 side effects，並且不會執行中間值或派生值的重新計算，只會使快取值無效。這允許變更通知到達 graph 中的所有受影響 nodes，而不會出現觀察到中間或 glitches 的 states 的可能性。

如果需要，一旦此變更傳播完成 (同步)，此階段之後可以是我們上面看到的 lazy 評估。

要查看此通知階段的運行情況，讓我們在我們的設定中新增一個 live consumer，例如一個 watcher。當 `a` 更新時，更新將會傳播到 dependent 的 live consumers：

```typescript
import { computed, signal } from '@angular/core';
import { createWatch } from '@angular/core/primitives/signals';

const a = signal(0);
const b = computed(() => a() + 'b');
const c = computed(() => a() + 'c');
const d = computed(() => b() + c() + 'd');

setTimeout(() => a.set(1), 3000);

// watcher will setup a dependency on d
const watcher = createWatch(
  () => console.log(d()),
  () => setTimeout(watcher.run, 1000),
  false
);

watcher.notify();
```

一旦我們更新 `a.set(1)` 的值，我們就可以看到 live consumers 的通知正在執行：

![alt text](https://wp.angular.love/wp-content/uploads/2024/08/1_39Jgsu7ROYU5MEoWGnMF8w-300x181.webp)

Nodes `b` 和 `c` 是 node `a` 的 live consumers，因此當為 `a` 運行更新時，Angular 將會遍歷 `node.liveConsumerNode` 並通知這些 nodes 變更。

但是，如前所述，這裡沒有真正發生任何事情。該 node 只是被標記為 dirty，並通過 `producerNotifyConsumers` 將通知傳播到其 live consumers：

```typescript
function consumerMarkDirty(node) {
  node.dirty = true;
  producerNotifyConsumers(node);
  node.consumerMarkedDirty?.(node);
}
```

所有這些都會一直追溯到依賴於 `d` 的 watcher (effect)。與 regular reactive nodes 相反，watch node 在其 `consumerMarkedDirty` method 中實現了排程：

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

在這裡，通知階段和 graph 遍歷停止。

這個兩階段過程有時被稱為 "push/pull" 演算法：當變更來源 signal 時，"dirtiness" 會被急切地 push 到 graph 中，但重新計算是 lazy 地執行的，僅當通過讀取其 signals 來 pull 值時才會執行。

**Change detection**

為了將基於 signals 的通知整合到 change detection 過程中，Angular 依賴於 live consumers 的機制。Component templates 被編譯為 template expressions (JS 程式碼)，並在該 component 視圖的 reactive context 中執行。在這種 contexts 中，執行 signal 將會返回 value，但也會將 signal 註冊為 component 視圖的 dependency。

由於 template expressions 是 live consumers，因此 Angular 會建立從 producer 到 template expression node 的連結。一旦更新 producer 的值，該 producer 將會立即且同步地通知 template node。收到通知後，Angular 將會標記 component 及其所有 ancestors 以進行檢查。

正如你可能從我的其他文章中了解到的那樣，每個 component 的 template 在內部都表示為 `LView` object。以下是 component 的外觀：

```typescript
@Component({...})
export class AppComponent {
  value = signal(0);
}
```

編譯後，它看起來像常規的 JS function `AppComponent_Template`，該 function 在此 component 的 change detection 期間執行：

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

當 Angular 將 signals 新增到其 change detection 實作中時，它將所有 component views (template function) 包裝在 `ReactiveLViewConsumer` node 中：

```typescript
export interface ReactiveLViewConsumer extends ReactiveNode {
  lView: LView | null;
}
```

該介面由 `REACTIVE_LVIEW_CONSUMER_NODE` node 實現：

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

你可以將此過程視為每個視圖取得自己的 `ReactiveLViewConsumer` consumer node，該 node 定義在 template function 內部存取的所有 signals 的 reactive context。

在我們的範例中，每當 template function 作為 change detection 的一部分運行時，它會在 template function node 的 context 中執行 `ctx.value()` producer，該 node 作為 `ActiveConsumer`：

![alt text](https://wp.angular.love/wp-content/uploads/2024/08/1_qeeISKeap4-Oo6WlikyPZQ-300x204.webp)

這將導致 template expression node (consumer) 作為 live dependency 添加到 producer `value()`：

![alt text](https://wp.angular.love/wp-content/uploads/2024/08/1_O9Cqy1y_Q8rnwxpu84BieA-300x173.webp)

此 dependency 可確保一旦 producer `counter` 的值發生變更，它將立即通知 consumer node (template expression)。

Live consumers 實現了 `consumerMarkDirty` method，該 method 在 producer 的值變更時由 producer 同步呼叫：

```typescript
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

function consumerMarkDirty(node: ReactiveNode): void {
  node.dirty = true;
  producerNotifyConsumers(node);
  node.consumerMarkedDirty?.(node);
}
```

在 `consumerMarkedDirty` 內部，template expression node 將使用 `markAncestorsForTraversal` 以類似於之前 `markForCheck()` 的方式標記 ancestors 以進行重新整理：

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

最後一個問題是 Angular 何時將目前的 `LView` consumer node 設定為 `ActiveConsumer`？所有這些都發生在 `refreshView` function 內部，你可能已經從我之前的文章中了解了這個 function。

此 function 對每個 `LView` 運行 change detection，並運行常見的 change detection 操作：執行 template function、執行 hooks、重新整理 queries 和設定 host bindings。基本上，在 Angular 運行所有這些操作之前，已新增了一整段處理 reactivity 的程式碼。

以下是它的外觀：

```typescript
function refreshView<T>(tView, lView, templateFn, context) {
  ...

  // Start component reactive context
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
```

由於此程式碼是在 Angular 在 `executeTemplate` 程式碼中運行 component 的 template function 之前執行的，因此當 component template 中使用的 signals 的 accessor functions 執行時，已經設定了 reactive context。

**Effects 和 watchers**

Effect 是一種專門的工具，旨在基於應用程式的 state 執行具有 side-effect 的操作。Effects 是使用在 reactive context 中執行的 callback 定義的 live consumers。捕獲此 function 的 signal dependencies，並且每當其任何 dependencies 產生新值時都會通知 effect。

Effects 在大多數應用程式程式碼中很少需要，但在特定情況下可能有用。以下是 Angular 文件中建議的一些使用範例：

*   記錄資料或使其與 window.localStorage 保持同步
*   新增無法使用 template 語法表達的自訂 DOM 行為，例如執行對 `<canvas>` 元素的自訂渲染

Angular 不使用 effects 在 change detection 機制中觸發 component 的 UI 更新。如 change detection 章節中所述，此功能依賴於 live consumers 的機制。

雖然 signal 演算法是標準化的，但 effects 應如何運作的細節尚未定義，並且在不同的框架中會有所不同。這是由於 effect 排程的微妙性質，它通常與框架渲染週期和其他高層次的、框架特定的 states 或 JavaScript 無法存取的策略整合在一起。

但是，signals 提案定義了一組基本要素，即 watch API，框架作者可以使用它們來建立自己的 effects。Watcher 介面用於觀察 reactive function，並在該 function 的 dependencies 變更時接收通知。

在 Angular 中，effect 是 watcher 的包裝器。首先讓我們探索 watchers 的運作方式，我們將了解它們如何用於建立 effect 基本要素。

首先，我們將從 Angular 基本要素匯入 watcher，並使用它來實現通知機制：

```typescript
import { createWatch } from '@angular/core/primitives/signals';

const counter = signal(0);

const watcher = createWatch(
  // run the user provided callback and set up tracking
  // this will be executed 2 times
  // 1st after watcher.notify() and 2nd time after this.counter.set(1)
  () => counter(),
  // this is called by the notify method
  // or by the consumer itself through through consumerMarkDirty method,
  // schedules the user provided callback to run in 1000ms
  () => setTimeout(watcher.run, 1000),
  false
);

// mark the watcher as dirty (stale) to force the user provided callback
// to run and set up tracking for the counter signal
// notify method will call consumerMarkDirty under the hood
watcher.notify();

// when the value changes, consumerMarkDirty is executed
// which schedules the user provided callback to run
setTimeout(() => this.counter.set(1), 3000);
```

當我們運行 `watcher.notify()` 時，Angular 會同步呼叫 watcher node 上的 `consumerMarkDirty` method。但是，使用者定義的通知 callback 不會在通知後立即執行。相反，它被排程為在未來通過 `watcher.run` 運行。當 watch 收到 "markDirty" 通知時，它將簡單地呼叫此排程操作。

您可以在這裡看到實際操作：

![alt text](https://wp.angular.love/wp-content/uploads/2024/08/1_FwpksG4QvcTYICQ6mHRChg-300x96.webp)

當我們運行 `this.counter.set(1)` 時，相同的呼叫鏈會導致排程使用者提供的 callback 運行。

為了建立 `effect()` function，Angular 將 watcher 包裝在 `EffectHandle` class 中：

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

你可以看到，`EffectHandle` class 是設定 watcher 的地方。對於我們之前使用 watchers 的範例，使用 `effect` function 將會大大簡化設定：

```typescript
import { Component, effect, signal } from '@angular/core';

@Component({...})
export class AppComponent {
  counter = null;

  constructor() {
    this.counter = signal(0);

    // this will be executed 2 times
    effect(() => this.counter());

    setTimeout(() => this.counter.set(1), 3000);
  }
}
```

當我們直接使用 `effect` function 時，我們只會傳遞一個 callback。這是使用者定義的 callback，它會設定 dependencies，並在 dependencies 更新時由 Angular 排程運行。

Angular effects 中使用的目前 scheduler 是 `ZoneAwareEffectScheduler`，它會在 change detection 週期後作為 microtask 佇列的一部分運行更新：

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

Angular 必須實現一個有趣的怪癖才能 "初始化" effect。正如我們在 watcher 的實作中看到的那樣，我們需要手動呼叫 `watcher.notify()` 一次來啟動追蹤。Angular 也需要這樣做，並且會在第一次運行 change detection 時這樣做。

以下是它的實現方式。

當你在 component 的 injection context 內部執行 `effect` function 時，Angular 會將通知 callback 新增到 component 的 view object `LView[EFFECTS_TO_SCHEDULE]`：

```typescript
export function effect(
  effectFn: (onCleanup: EffectCleanupRegisterFn) => void,
  options?: CreateEffectOptions,
): EffectRef {
  ...
  const handle = new EffectHandle();

  // Effects need to be marked dirty manually to trigger their initial run. The timing of this
  // marking matters, because the effects may read signals that track component inputs, which are
  // only available after those components have had their first update pass.
  // ...
  const cdr = injector.get(ChangeDetectorRef, null, {optional: true}) as ViewRef<unknown> | null;
  if (!cdr || !(cdr._lView[FLAGS] & LViewFlags.FirstLViewPass)) {
    // This effect is either not running in a view injector, or the view has already
    // undergone its first change detection pass, which is necessary for any required inputs to be
    // set.
    handle.watcher.notify();
  } else {
    // Delay the initialization of the effect until the view is fully initialized.
    (cdr._lView[EFFECTS_TO_SCHEDULE] ??= []).push(handle.watcher.notify);
  }

  return handle;
}
```

以這種方式新增的通知 functions 將在此 component 的視圖在 `refreshView` function 內部第一次運行 change detection 時執行一次：

```typescript
export function refreshView<T>(tView,lView,templateFn,context) {
  ...

  // Schedule any effects that are waiting on the update pass of this view.
  if (lView[EFFECTS_TO_SCHEDULE]) {
    for (const notifyEffect of lView[EFFECTS_TO_SCHEDULE]) {
      notifyEffect();
    }

    // Once they've been run, we can drop the array.
    lView[EFFECTS_TO_SCHEDULE] = null;
  }
```

呼叫 `notifyEffect` 將觸發底層 watcher 的 `consumerMarkDirty` 通知 callback，而它反過來將使用現有的 scheduler (在 change detection 之後) 排程 effect (使用者提供的 callback) 運行：

![alt text](https://wp.angular.love/wp-content/uploads/2024/08/1_BwgXXo4piUtwmCpAB9dWXw-300x87.webp)

這就是全部的故事了 🙂
