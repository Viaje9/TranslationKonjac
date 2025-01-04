---
title: Do you still think that NgZone (zone.js) is required for change detection in Angular?
date: 2025-01-03 10:57:14
tags:
---

原文：<https://angular.love/do-you-still-think-that-ngzone-zone-js-is-required-for-change-detection-in-angular>

# Angular 中的變更偵測是否仍然需要 NgZone (zone.js)？

我看到的大部分文章都強烈地將 `Zone` (`zone.js`) 和 `NgZone` 與 Angular 中的變更偵測聯繫起來。雖然它們確實相關，但嚴格來說它們並不是一個整體的一部分。是的，`Zone` 和 `NgZone` 用於自動觸發非同步操作所導致的變更偵測。但是，由於變更偵測是一個獨立的機制，它可以成功地在沒有 `Zone` 和 `NgZone` 的情況下工作。在第一章中，我將展示如何在沒有 `zone.js` 的情況下使用 Angular。本文的第二部分解釋了 Angular 和 zone.js 如何透過 `NgZone` 相互作用。最後，我還會說明為什麼自動變更偵測有時在使用 [Google API Client Library](https://developers.google.com/api-client-library/) (gapi) 等第三方函式庫時不起作用。

我寫過許多關於 Angular 中變更偵測的深入文章，而這篇文章完善了這個概念。如果你正在尋找關於變更偵測如何運作的全面概述，我建議你閱讀所有文章，從這篇開始 [這 5 篇文章將讓你成為 Angular 變更偵測專家](https://angular.love/these-5-articles-will-make-you-an-angular-change-detection-expert/)。****請注意，本文不是關於 Zones (zone.js)，而是關於 Angular 如何在 NgZone 的實作中使用 Zones 以及它與變更偵測機制的關係。****要了解更多關於 Zones 的資訊，請閱讀 [我對 Zones (zone.js) 進行了逆向工程，這是我發現的](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/)。

<!-- more -->

在沒有 Zone (zone.js) 的情況下使用 Angular
------------------------------------

為了證明 Angular 可以在沒有 Zone 的情況下成功運作，我最初計劃提供一個什麼都不做的模擬 Zone 物件。但即將推出的 Angular 版本 5 為我簡化了操作。它現在 [提供了一種方法](https://github.com/angular/angular/commit/344a5ca) 來使用一個 [noop Zone](https://github.com/angular/angular/blob/30d5a2ca83c9cf44f602462597a58547b05b75dd/packages/core/src/zone/ng_zone.ts#L318) ，該 Zone 不會透過設定執行任何操作。

為此，我們先移除對 `zone.js` 的依賴。我將使用 stackblitz 來展示應用程式，由於它使用 Angular-CLI，我將從 `polyfils.ts` 檔案中移除以下導入：

``` typescript
/* Zone JS 是 Angular 本身所需的。 */
import 'zone.js/dist/zone';  // 包含在 Angular CLI 中。
```

之後，我將設定 Angular 以使用 noop Zone 實作，如下所示：

``` typescript
platformBrowserDynamic()
    .bootstrapModule(AppModule, {
        ngZone: 'noop'
    });
```

如果你現在 [執行應用程式](https://stackblitz.com/edit/angular-jmlwb7)，你會看到變更偵測已完全運作，並在 DOM 中呈現 `name` 元件屬性。

現在，如果我們使用 `setTimeout` 更新此屬性：

``` typescript
export class AppComponent  {
    name = 'Angular 4';

    constructor() {
        setTimeout(() => {
            this.name = 'updated';
        }, 1000);
    }
```

你會看到變更沒有更新。這是預期的，因為沒有使用 `NgZone`，因此不會自動觸發變更偵測。但是，如果我們手動觸發它，它仍然可以正常工作。這可以透過注入 [ApplicationRef](https://angular.io/api/core/ApplicationRef) 並觸發 `tick` 方法來啟動變更偵測來完成：

``` typescript
export class AppComponent  {
    name = 'Angular 4';

    constructor(app: ApplicationRef) {
        setTimeout(()=>{
            this.name = 'updated';
            app.tick();
        }, 1000);
    }
```

現在你可以看到 [更新已成功呈現](https://stackblitz.com/edit/angular-lr1rss)。

總之，上述示範的重點是向你展示 `zone.js`，尤其是 `NgZone`，並不是變更偵測實作的一部分。它是一種**非常方便**的機制，可以透過呼叫 `app.tick()` 來自動觸發變更偵測，而不是在某些點手動執行。我們將在一分鐘內看到這些點是什麼。

NgZone 如何使用 Zones
---------------------

在我的 [先前關於 Zone (zone.js) 的文章](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/) 中，我深入描述了 Zone 提供的內部運作和 API。我在那裡解釋了 [forking a zone](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/) 和 [在特定 zone 中執行任務](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/) 的核心概念。我將在此處引用這些概念。

我也展示了 Zone 提供的兩種能力 — [context propagation](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/) 和出色的 [非同步任務追蹤](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/)。Angular 實作了 [NgZone](https://github.com/angular/angular/blob/30d5a2ca83c9cf44f602462597a58547b05b75dd/packages/core/src/zone/ng_zone.ts#L86) 類別，該類別嚴重依賴任務追蹤機制。

NgZone 只是 [forked 子 zone](https://github.com/angular/angular/blob/30d5a2ca83c9cf44f602462597a58547b05b75dd/packages/core/src/zone/ng_zone.ts#L252) 的包裝：

``` typescript
function forkInnerZoneWithAngularBehavior(zone: NgZonePrivate) {
    zone._inner = zone._inner.fork({
        name: 'angular',
        ...
```

forked zone 保存在 `_inner` 屬性中，通常稱為 Angular zone。當你執行 `NgZone.run()` 時，這個 zone 用於執行回呼：

``` typescript
run(fn, applyThis, applyArgs) {
    return this._inner.run(fn, applyThis, applyArgs);
}
```

在 forking Angular zone 時的目前 zone 保存在 `_outer` 屬性中，當你執行 `NgZone.runOutsideAngular()` 時，它用於執行回呼：

``` typescript
runOutsideAngular(fn) {
    return this._outer.run(fn);
}
```

這個方法通常用於在 Angular zone 之外執行效能密集型操作，以避免不斷地 [觸發變更偵測](https://angular.love/these-5-articles-will-make-you-an-angular-change-detection-expert/)。

NgZone 具有 `isStable` 屬性，表示沒有未完成的微任務或巨任務。它還定義了四個事件：

```
+------------------+-------------------------------------------------------+
|      事件         |                     描述                               
+------------------+-------------------------------------------------------+
| onUnstable       | 當程式碼進入 Angular Zone 時發出通知。                     
|                  | 這會在 VM Turn 上首先觸發。                               
|                  |                                                        
| onMicrotaskEmpty | 當目前 VM Turn 中沒有更多微任務排隊時發出通知。               
|                  | 這是 Angular 進行變更偵測的提示，可能會使更多微任務排隊。      
|                  | 因此，這個事件可能會在每個 VM Turn 中觸發多次。              
|                  |                                                        
| onStable         | 當最後一次 `onMicrotaskEmpty` 執行且沒有更多微任務時發出通知， 
|                  | 這表示我們即將放棄 VM Turn。                              
|                  | 這個事件只會被呼叫一次。                                   
|                  |                                                        
| onError          | 發出已傳遞錯誤的通知。                 ㅤ                  
+------------------+-------------------------------------------------------+
```

Angular 在 [ApplicationRef](https://github.com/angular/angular/blob/30d5a2ca83c9cf44f602462597a58547b05b75dd/packages/core/src/application_ref.ts#L364) 內部使用 `onMicrotaskEmpty` 事件來自動觸發變更偵測：

``` typescript
this._zone.onMicrotaskEmpty.subscribe(
    {next: () => { this._zone.run(() => { this.tick(); }); }});
```

如果你還記得上一節的內容，`tick()` 方法用於執行應用程式的變更偵測。

NgZone 如何實作 onMicrotaskEmpty 事件
--------------------------------------------

現在讓我們看看 `NgZone` 如何實作 `onMicrotaskEmpty` 事件。該事件從 [checkStable](https://github.com/angular/angular/blob/30d5a2ca83c9cf44f602462597a58547b05b75dd/packages/core/src/zone/ng_zone.ts#L233) 函數發出：

``` typescript
function checkStable(zone: NgZonePrivate) {
  if (zone._nesting == 0 && !zone.hasPendingMicrotasks && !zone.isStable) {
    try {
      zone._nesting++;
      zone.onMicrotaskEmpty.emit(null); <-------------------
```

這個函數會定期從三個 Zone hooks 觸發：

*   [onHasTask](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/)
*   [onInvokeTask](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/)
*   [onInvoke](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/)

正如關於 Zones 的文章中所解釋的，當觸發最後兩個 hook 時，微任務佇列中可能會發生變更，因此 Angular 每次觸發 hook 時都必須執行 `stable` 檢查。 `onHasTask` hook 也用於執行檢查，因為它可以追蹤整個佇列變更。

常見的陷阱
---------------

在 stackoveflow 上，與變更偵測相關的最常見問題之一是為什麼有時在使用第三方函式庫時，元件中的變更不會套用。這裡有一個 [此類問題的範例](https://stackoverflow.com/a/46286400/2545680)，其中涉及 [Google API Client Library](https://developers.google.com/api-client-library/) (gapi)。解決此類問題的常見方法是簡單地在 Angular zone 內執行回呼，如下所示：

``` javascript
gapi.load('auth2', () => {
    zone.run(() => {
        ...
```

然而，一個有趣的問題是，為什麼 Zone **不**註冊請求，導致在其中一個 hook 中**沒有**通知？而且沒有通知，`NgZone` 不會自動觸發變更偵測。

為了瞭解這一點，我深入研究了 `gapi` 的 **縮小**原始碼，發現它使用 [JSONP](http://schock.net/articles/2013/02/05/how-jsonp-really-works-examples/) 發出網路請求。這種方法不使用常見的 AJAX API，例如 [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) 或 [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)，這些 API 由 Zones 修補和追蹤。相反，它會建立一個帶有源 URL 的 `script` 標籤，並定義一個全域回呼，以便在從伺服器取得請求的帶有資料的腳本時觸發。Zones 無法修補或偵測到這種情況，因此框架仍然無法知道使用此技術執行的請求。

以下是來自 [gapi 縮小版本](https://apis.google.com/js/api.js) 的相關程式碼片段，以供好奇的人參考：

``` javascript
Ja = function(a) {
    var b = L.createElement(Z);
    b.setAttribute(“src”, a);
    a = Ia();
    null !== a && b.setAttribute(“nonce”, a);
    b.async = “true”;
    (a = L.getElementsByTagName(Z)[0]) ? 
        a.parentNode.insertBefore(b, a) : 
        (L.head || L.body || L.documentElement).appendChild(b)
}
```

`Z` 變數等於 `"script"`，而參數 `a` 保存請求 URL：

``` javascript
https://apis.google.com/_.../cb=gapi.loaded_0
```

URL 的最後一段是 `gapi.loaded_0` 全域回呼：

``` javascript
typeof gapi.loaded_0 
“function”
```

**就這樣。感謝閱讀！**
