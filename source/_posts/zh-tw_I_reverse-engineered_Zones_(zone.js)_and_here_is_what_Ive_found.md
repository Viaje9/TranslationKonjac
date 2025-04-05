---
title: 我對 Zones (zone.js) 進行了反向工程，這是我的發現
date: 2024-12-31 17:01:14
tags:
---

原文：<https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found>

# zone.js 反向工程 - zone.js 中有什麼有趣的地方？

Zones 是一種新的機制，可以幫助開發人員處理多個邏輯上相關的非同步操作。Zones 的工作方式是將每個非同步操作與一個 zone 關聯起來。開發人員可以利用這種綁定來：

*   將一些資料與 zone 關聯，類似於其他語言中的執行緒本機儲存，該資料可供 zone 內部的任何非同步操作存取。
*   自動追蹤給定 zone 內的未完成非同步操作，以執行清理或渲染或測試斷言步驟
*   測量在 zone 中花費的總時間，用於分析或現場分析
*   處理 zone 內的所有未捕獲的異常或未處理的 promise 拒絕，而不是讓它們傳播到頂層

網路上大多數文章要么描述過時的 API，要么使用過於簡化的類比來解釋 Zones。在本文中，我將使用最新的 API，並盡可能接近實作來詳細探索重要的 API。我將先描述 API，然後展示非同步任務關聯機制，並繼續介紹開發人員可以使用的攔截鉤子，以執行上面列出的任務。在文章的最後，我簡要說明了 Zones 在底層的工作原理。

<!-- more -->

Zones 目前是 EcmaScript 標準的第 0 階段[提案](https://github.com/domenic/zones)，目前正被 [Node 阻止](https://github.com/nodejs/TSC/issues/340)。Zones 通常被稱為 `Zone.js`，這是 [github 儲存庫](https://github.com/angular/zone.js) 和 [npm 套件](https://www.npmjs.com/package/zone.js) 的名稱。但是，在本文中，我將使用規格中指定的名稱 `Zone`。****請注意，本文不是關於 NgZone，而是關於 NgZone 所建立的機制 — Zones (zone.js)。**** 透過了解我在本文中介紹的內容，你將能夠建立自己的 NgZone 或了解現有的 NgZone 如何運作。若要深入了解 NgZone，請閱讀[你是否仍然認為在 Angular 中進行變更偵測需要 NgZone (zone.js)？](https://angular.love/en/do-you-still-think-that-ngzone-zone-js-is-required-for-change-detection-in-angular/)

相關的 Zone API
---------------

讓我們首先看看使用 Zones 時最相關的方法。該類別具有以下介面：

```typescript
class Zone {
  constructor(parent: Zone, zoneSpec: ZoneSpec);
  static get current();
  get name();
  get parent();

  fork(zoneSpec: ZoneSpec);
  run(callback, applyThis, applyArgs, source);
  runGuarded(callback, applyThis, applyArgs, source);
  wrap(callback, source);

}
```

Zones 有一個 **current zone** 的概念，這至關重要。current zone 是非同步內容，會與所有非同步操作傳播。它表示與目前正在執行的堆疊框架/非同步任務關聯的 zone。可以使用靜態 getter `Zone.current` 存取此 current zone。

每個 zone 都有一個 `name`，主要用於工具和除錯目的。它還定義了旨在操作 zone 的方法：

*   `z.run(callback, ...)` 在給定的 zone 中同步調用函式。它會在執行 `callback` 時將 current zone 設定為 `z`，並在回呼執行完成後將其重設為先前的值。在 zone 中執行回呼通常稱為「進入」zone。
*   `z.runGuarded(callback, ...)` 與 `run` 相同，但會捕獲運行時錯誤並提供一種攔截它們的機制。如果任何父 Zone 都未處理錯誤，則會重新拋出錯誤。
*   `z.wrap(callback)` 會產生一個新函式，該函式會在閉包中捕獲 `z`，並在本質上執行 `z.runGuarded(callback)`（當執行時）。如果回呼稍後傳遞給 `other.run(callback)`，它仍然會在 `z` zone 中執行，而不是 `other`。該機制在概念上類似於 JavaScript 中 `Function.prototype.bind` 的工作方式。

在下一節中，我們將詳細討論 `fork` 方法。Zone 還有許多方法可以執行、排程和取消任務：

```typescript
class Zone {
  runTask(...);
  scheduleTask(...);
  scheduleMicroTask(...);
  scheduleMacroTask(...);
  scheduleEventTask(...);
  cancelTask(...);
}
```

這些是開發人員很少使用的低階方法，因此我不會在本文中詳細討論它們。排程任務是 Zone 的內部操作，對於開發人員而言，它通常意味著簡單地調用某些非同步操作，例如 `setTimeout`。

跨呼叫堆疊持續保留 zone
------------------------

JavaScript VM 會在它[自己的堆疊框架](https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf#6e11)中執行每個函式。因此，如果你有如下程式碼：

```typescript
function c() {
    // 捕獲堆疊追蹤
    try {
        new Function('throw new Error()')();
    } catch (e) {
        console.log(e.stack);
    }
}

function b() { c() }
function a() { b() }

a();
```

在 `c` 函式內部，它具有以下呼叫堆疊：

```
at c (index.js:3)
at b (index.js:10)
at a (index.js:14)
at index.js:17
```

我在 `c` 函式中用於捕獲堆疊追蹤的方法在 [MDN 網站](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/stack) 上有說明。

呼叫堆疊可以像這樣繪製：

![圖片 23](https://wp.angular.love/wp-content/uploads/2024/07/content-image1-585.png)

因此，我們的函式呼叫有 3 個堆疊框架，全域內容有一個堆疊。

在常規的 JavaScript 環境中，函式 `c` 的堆疊框架與函式 `a` 的堆疊框架沒有任何關聯。Zone 允許我們做的是將每個堆疊框架與特定的 zone 關聯起來。例如，我們可以將堆疊框架 `a` 和 `c` 與同一個 zone 關聯，有效地將它們連結在一起。因此，我們最終會得到以下結果：

![圖片 24](https://wp.angular.love/wp-content/uploads/2024/07/content-image2-497.png)

我們稍後會看到如何做到這一點。

### 使用 zone.fork 建立子 zone

Zones 最常用的功能之一是使用 `fork` 方法建立新的 zone。Forking zone 會建立新的子 zone，並將其 `parent` 設定為用於 forking 的 zone：

```typescript
const c = z.fork({name: 'c'});
console.log(c.parent === z); // true
```

`fork` 方法在底層只是使用類別建立一個新的 zone：

```typescript
new Zone(targetZone, zoneSpec);
```

因此，為了完成將 `a` 和 `c` 函式與同一個 zone 關聯的任務，我們首先需要建立該 zone。為此，我們將使用我上面展示的 `fork` 方法：

```typescript
const zoneAC = Zone.current.fork({name: 'AC'});
```

我們傳遞給 `fork` 方法的物件稱為 zone 規格 (ZoneSpec)，它具有以下屬性：

```typescript
interface ZoneSpec {
    name: string;
    properties?: { [key: string]: any };
    onFork?: ( ... );
    onIntercept?: ( ... );
    onInvoke?: ( ... );
    onHandleError?: ( ... );
    onScheduleTask?: ( ... );
    onInvokeTask?: ( ... );
    onCancelTask?: ( ... );
    onHasTask?: ( ... );
}
```

`name` 定義 zone 的名稱，`properties` 用於將資料與 zone 關聯。所有其他屬性都是攔截鉤子，允許父 zone 攔截子 zone 的某些操作。重要的是要了解 forking 會建立 zone 階層，並且可以使用鉤子讓父 zone 攔截 Zone 類別中所有操作 zone 的方法。在本文的後面，我們將看到如何使用 `properties` 在非同步操作之間共用資料，以及使用鉤子來實作任務追蹤。

讓我們建立一個子 zone：

```typescript
const zoneB = Zone.current.fork({name: 'B'});
```

現在我們有了兩個 zone，我們可以使用它們在特定的 zone 內執行函式。為此，我們可以使用 `zone.run()` 方法。

### 使用 zone.run 切換 zones

若要讓特定的堆疊框架與 zone 關聯，我們需要使用 `run` 方法在該 zone 中執行函式。如你所知，它會在指定的 zone 中同步執行回呼，並在其完成後還原 zone。

因此，讓我們運用該知識並稍微修改我們的範例：

```typescript
function c() {
    console.log(Zone.current.name);  // AC
}
function b() {
    console.log(Zone.current.name);  // B
    zoneAC.run(c);
}
function a() {
    console.log(Zone.current.name);  // AC
    zoneB.run(b);
}
zoneAC.run(a);
```

現在每個呼叫堆疊都與 zone 關聯：

![圖片 25](https://wp.angular.love/wp-content/uploads/2024/07/content-image3-456.png)

如你從上面的程式碼中看到的，我們使用 `run` 方法執行每個函式，該方法直接指定要使用的 zone。你現在可能想知道，如果我們不使用 `run` 方法而只是在 zone 內執行函式會發生什麼事？

****重要的是要了解，在函式內部排程的所有函式呼叫和非同步任務都將在與該函式相同的 zone 中執行。****

我們知道 zones 環境始終具有根 zone。因此，如果我們不使用 `zone.run` 切換 zones，我們預期所有函式都會在 `root` zone 中執行。讓我們看看是否是這樣：

```typescript
function c() {
    console.log(Zone.current.name);  // <root>
}

function b() {
    console.log(Zone.current.name);  // <root>
    c();
}

function a() {
    console.log(Zone.current.name);  // <root>
    b();
}

a();
```

是的，就是這樣。這是圖表：

![圖片 26](https://wp.angular.love/wp-content/uploads/2024/07/content-image4-375.png)

如果我們只在 `a` 函式中使用一次 `zoneAB.run`，則 `b` 和 `c` 將在 `AB` zone 中執行：

```typescript
const zoneAB = Zone.current.fork({name: 'AB'});

function c() {
    console.log(Zone.current.name);  // AB
}

function b() {
    console.log(Zone.current.name);  // AB
    c();
}

function a() {
    console.log(Zone.current.name);  // <root>
    zoneAB.run(b);
}

a();
```

![圖片 27](https://wp.angular.love/wp-content/uploads/2024/07/content-image5-325.png)

你可以看到我們在 `AB` zone 中明確呼叫 `b` 函式。但是，`c` 函式也會在此 zone 中執行。

跨非同步任務持續保留 zone
--------------------------

JavaScript 開發的獨特特徵之一是非同步程式設計。可能大多數新的 JS 開發人員都會使用 `setTimeout` 方法來熟悉這種範例，該方法允許延後執行函式。Zone 將 `setTimeout` 非同步操作稱為任務。具體而言，是巨集任務。另一種類型的任務是微任務，例如 `promise.then`。瀏覽器在內部使用此術語，[Jake Archibald](https://twitter.com/jaffathecake) 在 [任務、微任務、佇列和排程](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)中深入解釋了此術語。

因此，現在讓我們看看 Zone 如何處理非同步任務，例如 `setTimeout`。為此，我們將只使用我們上面使用的程式碼，但我們將回呼傳遞給 `setTimeout` 函式，而不是立即呼叫函式 `c`。因此，此函式將在未來（大約 2 秒後）****在單獨的呼叫堆疊中****執行：

```typescript
const zoneBC = Zone.current.fork({name: 'BC'});

function c() {
    console.log(Zone.current.name);  // BC
}

function b() {
    console.log(Zone.current.name);  // BC
    setTimeout(c, 2000);
}

function a() {
    console.log(Zone.current.name);  // <root>
    zoneBC.run(b);
}

a();
```

我們在上面了解到，如果我們在 zone 內部呼叫一個函式，該函式將在同一個 zone 中執行。此行為也適用於非同步操作。如果我們排程一個非同步任務並指定回呼函式，則此函式將在排程該任務的同一個 zone 中執行。

因此，如果我們繪製呼叫歷史，我們將有以下結果：

![圖片 28](https://wp.angular.love/wp-content/uploads/2024/07/content-image6-250.png)

這非常好，但是，此圖表隱藏了重要的實作詳細資訊。在底層，Zone 必須為即將執行的每個任務****還原****正確的 zone。為此，它必須記住應該執行此任務的 zone，並且透過在任務上保留對相關 zone 的引用來做到這一點。然後，此 zone 用於從根 zone 中的處理常式呼叫任務。

這表示每個非同步任務的呼叫堆疊始終以根 zone 開始，該根 zone 使用與任務相關的資訊來還原正確的 zone，然後呼叫任務。因此，以下是更準確的表示：

![圖片 29](https://wp.angular.love/wp-content/uploads/2024/07/content-image7-227.png)

跨非同步任務傳播內容
----------------------

Zone 具有開發人員可以利用的多種有趣的功能。其中一項功能是內容傳播。它僅表示我們可以將資料附加到 zone，並在該 zone 內部執行的任何任務中存取此資料。

讓我們使用我們的最後一個範例，並示範如何跨 `setTimeout` 非同步任務持續保留資料。如你先前了解到的，當 forking 新的 zone 時，我們傳遞 zone 規格物件。此物件可以具有可選的屬性 `properties`。我們可以使用此屬性將資料與 zone 關聯，如下所示：

```typescript
const zoneBC = Zone.current.fork({
    name: 'BC',
    properties: {
        data: 'initial'
    }
});
```

然後可以使用 `zone.get` 方法存取：

```typescript
function a() {
    console.log(Zone.current.get('data')); // 'initial'
}

function b() {
    console.log(Zone.current.get('data')); // 'initial'
    setTimeout(a, 2000);
}

zoneBC.run(b);
```

`properties` 屬性指向的物件是淺層不可變的，這表示你無法新增/移除此物件屬性。這主要是因為 Zone 沒有提供任何方法來執行此操作。因此，在上面的範例中，我們無法為 `properties.data` 設定不同的值。

但是，我們可以將物件傳遞給 `properties.data` 而不是基本類型，然後我們將能夠修改資料：

```typescript
const zoneBC = Zone.current.fork({
    name: 'BC',
    properties: {
        data: {
            value: 'initial'
        }
    }
});

function a() {
    console.log(Zone.current.get('data').value); // 'updated'
}

function b() {
    console.log(Zone.current.get('data').value); // 'initial'
    Zone.current.get('data').value = 'updated';
    setTimeout(a, 2000);
}

zoneBC.run(b);
```

有趣的是，使用 `fork` 方法建立的子 zone 會繼承父 zone 的屬性：

```typescript
const parent = Zone.current.fork({
    name: 'parent',
    properties: { data: 'data from parent' }
});

const child = parent.fork({name: 'child'});

child.run(() => {
    console.log(Zone.current.name); // 'child'
    console.log(Zone.current.get('data')); // 'data from parent'
});
```

追蹤未完成的任務
----------------

另一項可能更有趣且有用的功能是追蹤未完成的非同步巨集和微型任務的能力。Zone 會將所有未完成的任務保留在佇列中。若要取得此佇列狀態何時變更的通知，我們可以使用 zone 規格的 `onHasTask` 鉤子。這是它的簽名：

```typescript
onHasTask(delegate, currentZone, targetZone, hasTaskState);
```

由於父 zone 可以攔截子 zone 事件，Zone 會提供 `currentZone` 和 `targetZone` 參數，以區分任務佇列中發生變更的 zone 和攔截事件的 zone。因此，例如，如果你需要確保你正在攔截目前 zone 的事件，只需比較 zone：

```typescript
// 我們只對來自我們 zone 的事件感興趣
if (currentZone === targetZone) { ... }
```

傳遞給鉤子的最後一個參數是 `hasTaskState`，它描述了任務佇列的狀態。這是它的簽名：

```typescript
type HasTaskState = {
    microTask: boolean; 
    macroTask: boolean; 
    eventTask: boolean; 
    change: 'microTask'|'macroTask'|'eventTask';
};
```

因此，如果你在 zone 內部呼叫 `setTimeout`，你將會取得具有下列值的 `hasTaskState` 物件：

```typescript
{
    microTask: false; 
    macroTask: true; 
    eventTask: false; 
    change: 'macroTask';
}
```

這表示佇列中有一個待處理的巨集任務，並且佇列中的變更來自 `macrotask`。

因此，讓我們在實作中看到此內容：

```typescript
const z = Zone.current.fork({
    name: 'z',
    onHasTask(delegate, current, target, hasTaskState) {
        console.log(hasTaskState.change);          // "macroTask"
        console.log(hasTaskState.macroTask);       // true
        console.log(JSON.stringify(hasTaskState));
    }
});

function a() {}

function b() {
    // 使用
    // change === "macroTask" 同步觸發 `onHasTask` 事件，因為 `setTimeout` 是一個巨集任務
    setTimeout(a, 2000);
}

z.run(b);
```

我們會得到以下輸出：

```
macroTask
true
{
    "microTask": false,
    "macroTask": true,
    "eventTask": false,
    "change": "macroTask"
}
```

每當兩秒鐘後 timeout 完成執行時，會再次觸發 `onHasTask`：

```
macroTask
false
{
    "microTask": false,
    "macroTask": false,
    "eventTask": false,
    "change": "macroTask"
}
```

但是，有一個注意事項。你只能使用 `onHasTask` 鉤子來追蹤****整個任務佇列****的 `empty/non-empty` 狀態。你 \*\*\*\*無法使用它來追蹤個別的任務。\*\*\*\* 如果你執行以下程式碼：

```typescript
let timer;

const z = Zone.current.fork({
    name: 'z',
    onHasTask(delegate, current, target, hasTaskState) {
        console.log(Date.now() - timer);
        console.log(hasTaskState.change);
        console.log(hasTaskState.macroTask);
    }
});

function a1() {}
function a2() {}

function b() {
    timer = Date.now();
    setTimeout(a1, 2000);
    setTimeout(a2, 4000);
}

z.run(b);
```

你將會取得以下輸出：

```
1
macroTask
true

4006
macroTask
false
```

你可以看到，對於在 `2` 秒內完成的 `setTimeout` 任務沒有事件。當排程第一個 `setTimeout` 且任務佇列狀態從 `non-empty` 變更為 `empty` 時，`onHasTask` 鉤子會觸發一次，並且當最後一個 `setTimeout` 回呼完成時，它會在 `4` 秒內第二次觸發。

如果想要追蹤個別任務，你需要使用 `onSheduleTask` 和 `onInvoke` 鉤子。

### onSheduleTask 和 onInvokeTask

Zone 規格定義了兩個可用於追蹤個別任務的鉤子：

*   onScheduleTask  
    每當偵測到非同步操作（如 `setTimeout`）時執行
*   onInvokeTask  
    當執行傳遞給非同步操作（如 `setTimeout(callback)`）的回呼時執行

以下是如何使用這些鉤子來追蹤個別任務：

```typescript
let timer;

const z = Zone.current.fork({
    name: 'z',
    onScheduleTask(delegate, currentZone, targetZone, task) {
      const result = delegate.scheduleTask(targetZone, task);
      const name = task.callback.name;
      console.log(
          Date.now() - timer, 
         `task with callback '${name}' is added to the task queue`
      );
      return result;
    },
    onInvokeTask(delegate, currentZone, targetZone, task, ...args) {
      const result = delegate.invokeTask(targetZone, task, ...args);
      const name = task.callback.name;
      console.log(
        Date.now() - timer, 
       `task with callback '${name}' is removed from the task queue`
     );
     return result;
    }
});

function a1() {}
function a2() {}

function b() {
    timer = Date.now();
    setTimeout(a1, 2000);
    setTimeout(a2, 4000);
}

z.run(b);
```

以下是預期的輸出：

```
1 "task with callback ‘a1’ is added to the task queue"
2 "task with callback ‘a2’ is added to the task queue"
2001 "task with callback ‘a1’ is removed from the task queue"
4003 "task with callback ‘a2’ is removed from the task queue"
```

### 使用 onInvoke 攔截 zone「進入」

可以透過呼叫 `z.run()` 明確地進入 (切換) zone，也可以透過呼叫任務隱含地進入。在上一節中，我解釋了 `onInvokeTask` 鉤子，當 Zone 在內部執行與非同步任務關聯的回呼時，可以使用該鉤子來攔截 zone 進入。還有另一個鉤子 `onInvoke`，你可以使用它來取得何時透過執行 `z.run()` 進入 zone 的通知。

以下是如何使用它的範例：

```typescript
const z = Zone.current.fork({
    name: 'z',
    onInvoke(delegate, current, target, callback, ...args) {
        console.log(`entering zone '${target.name}'`);
        return delegate.invoke(target, callback, ...args);
    }
});

function b() {}

z.run(b);
```

輸出為：

```
entering zone ‘z’
```

`Zone.current` 在底層如何工作
------------------------------------

目前的 zone 是使用 [\_currentZoneFrame](https://github.com/angular/zone.js/blob/b9c0d9c09776bc253f3156ea511c9a1a5c952c31/dist/zone.js#L627) 變數追蹤的，該變數會在 [此處](https://github.com/angular/zone.js/blob/b9c0d9c09776bc253f3156ea511c9a1a5c952c31/dist/zone.js#L64) 捕獲到閉包中，並由 `Zone.current` getter 回傳。因此，為了切換 zone，只需要更新 `__currentZoneFrame` 變數。你現在知道，可以透過執行 `z.run()` 或呼叫任務來切換 zone。

因此，[這裡](https://github.com/angular/zone.js/blob/b9c0d9c09776bc253f3156ea511c9a1a5c952c31/dist/zone.js#L139) 是 `run` 方法更新變數的位置：

```typescript
class Zone {
   ...
   run(callback, applyThis, applyArgs,source) {
      ...
      _currentZoneFrame = {parent: _currentZoneFrame, zone: this};
```

而 `runTask` 會在 [此處](https://github.com/angular/zone.js/blob/b9c0d9c09776bc253f3156ea511c9a1a5c952c31/dist/zone.js#L185) 更新變數：

```typescript
class Zone {
   ...
   runTask(task, applyThis, applyArgs) {
      ...
      _currentZoneFrame = { parent: _currentZoneFrame, zone: this };
```

`runTask` 方法是由每個任務都有的 `invokeTask` 方法呼叫的：

```typescript
class ZoneTask {
    invokeTask() {
         _numberOfNestedTaskFrames++;
      try {
          self.runCount++;
          return self.zone.runTask(self, this, arguments);
```

每個任務在建立時都會將其 zone 保存在 `zone` 屬性中。這正是用於在 `invokeTask` 中 `runTask` 的 zone（`self` 在這裡指的是任務實例）：

```typescript
self.zone.runTask(self, this, arguments);
```
