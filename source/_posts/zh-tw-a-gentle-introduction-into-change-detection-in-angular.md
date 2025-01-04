---
title: A gentle introduction into change detection in Angular
date: 2025-01-04 16:33:14
tags:
---

原文：<https://angular.love/a-gentle-introduction-into-change-detection-in-angular>

# Angular 變更檢測的入門介紹

現代的網路應用程式是互動式的。應用程式的狀態可能隨時因按鈕點擊或伺服器返回的請求而改變。當狀態改變時，程式碼需要偵測到該變更並在使用者介面中反映出來。[這是變更偵測機制的主要工作](https://angular.love/en/what-every-front-end-developer-should-know-about-change-detection-in-angular-and-react/)。

> 如果你偏好觀看而不是閱讀，[可以看看我在 AngularConnect 上發表的演講](https://www.youtube.com/watch?v=DsBy9O0c6eo)。

過去一年，我寫了[許多關於 Angular 中 **change detection** 機制的深入文章](https://angular.love/en/these-5-articles-will-make-you-an-angular-change-detection-expert/)。它們提供了詳細的解釋並涵蓋了許多內部細節。但是，它們也需要花費大量時間才能仔細閱讀。對於那些沒有時間但仍然好奇的你來說：這篇文章提供了對變更偵測機制的「較輕鬆」的解釋。它將讓你對其組成部分和機制有一個高階的概述：用於表示元件的內部資料結構、綁定的作用以及作為該過程一部分執行的操作。我也會提到 zones，並確切地展示此功能如何在 Angular 中啟用自動變更偵測。

當事情出錯時，了解變更偵測的內部運作原理將有助於你更有效率地除錯錯誤，例如 [`ExpressionChangedAfterItHasBeenCheckedError`](https://angular.love/everything-you-need-to-know-about-the-expressionchangedafterithasbeencheckederror-error/)，並避免[一些常見的混淆](https://angular.love/if-you-think-ngdocheck-means-your-component-is-being-checked-read-this-article/)。在這篇文章中，我將演示幾個導致錯誤的設定，並使用它們來解釋變更偵測的內部運作原理。

<!-- more -->

初次相遇
---------------

讓我們從這個簡單的 Angular 元件開始。它會在應用程式中發生變更偵測時，渲染當時的時間。時間戳記具有毫秒精度。點擊按鈕會觸發變更偵測：

![Image 25](https://wp.angular.love/wp-content/uploads/2024/07/content-image1-75.gif)

這是實作方式：

```typescript
@Component({
    selector: 'my-app',
    template: `
        <h3>
            Change detection is triggered at:
            <span [textContent]="time | date:'hh:mm:ss:SSS'"></span>
        </h3>
        <button (click)="0">Trigger Change Detection</button>
    `
})
export class AppComponent {
    get time() {
        return Date.now();
    }
}
```

如你所見，它相當基本。有一個名為 `time` 的 getter 會回傳目前的時間戳記。而且，我將它綁定到 HTML 中的 `span` 元素。

> Angular 不允許空表達式，所以我將 `0` 作為點擊回呼。

你可以在[這裡](https://stackblitz.com/edit/angular-hqbenm?file=src/app/app.component.ts)使用它。當 Angular 執行變更偵測時，它會取得 `time` 屬性的值，透過 `date` pipe 傳遞它，並使用結果來更新 DOM。一切都如預期般運作。但是，當我檢查主控台時，我看到 `ExpressionChangedAfterItHasBeenCheckedError` 錯誤：

![Image 26](https://wp.angular.love/wp-content/uploads/2024/07/content-image2-362.png)

這其實相當令人驚訝。通常，這個錯誤會出現在更複雜的實作中。那麼，我們如何能以如此簡單的功能得到它呢？別擔心，我們現在要來調查它。

讓我們從錯誤訊息開始：

> Expression has changed after it was checked. Previous value: “textContent: 1542375826274”. Current value: “textContent: 1542375826275”.

它告訴我們，`textContent` 綁定的表達式產生的值是不同的。是的，毫秒確實不同。因此，Angular 評估了表達式 `time | date:’hh:mm:ss:SSS` 兩次，並比較了結果。它偵測到差異，這就是導致錯誤的原因。

**但是，為什麼 Angular 要執行該比較？**  
**或者，它究竟何時執行？**

這些問題激發了我的好奇心，並最終引導我深入研究變更偵測的內部運作原理。因為，為了找到這些問題的答案，我必須開始除錯。而我一直在除錯，除錯，嗯，我想它持續了大概...幾個月？讓我們從第二個問題開始，也就是錯誤何時被拋出。但首先，我需要與你分享我的一些發現，這將有助於我們理解我們剛才觀察到的行為。

元件視圖和綁定
----------------------------

Angular 中的變更偵測有兩個主要組成部分：

*   元件視圖
*   相關聯的綁定

Angular 中的每個元件都有一個包含 HTML 元素的模板。當 Angular 建立 DOM 節點以在螢幕上渲染模板的內容時，它需要一個地方來儲存這些 DOM 節點的參考。為此，內部有一個稱為 View 的資料結構。它也用於儲存元件實例的參考和綁定表達式的先前值。元件和視圖之間存在一對一的關係。這是展示視圖的圖表：

![Image 27](https://wp.angular.love/wp-content/uploads/2024/07/content-image3-333.png)

當編譯器分析模板時，它會識別出在變更偵測期間可能需要更新的 DOM 元素的屬性。對於每個這樣的屬性，編譯器都會建立一個 **binding**。綁定定義要更新的屬性名稱，以及 Angular 用於取得新值的表達式。

在我們的案例中，屬性 `time` 用於屬性 `textContent` 的表達式中。因此，Angular 會建立一個綁定並將其與 `span` 元素關聯：

![Image 28](https://wp.angular.love/wp-content/uploads/2024/07/content-image4-274.png)

> 在實際的實作中，綁定並不是一個包含所有必要資訊的單一物件。`viewDefinition` 定義了範本元素的實際綁定和要更新的屬性。用於綁定的表達式放置在 `updateRenderer` 函式中。

檢查元件視圖
-------------------------

如你所知，在 Angular 中，會針對每個元件執行變更偵測。現在我們知道元件在內部被表示為視圖，我們可以說變更偵測是針對每個視圖執行的。

當 Angular 檢查一個視圖時，它會簡單地執行編譯器為視圖產生的所有綁定。它會評估表達式並將其結果與儲存在視圖的 `oldValues` 陣列中的值進行比較。這就是髒檢查名稱的由來。如果它偵測到差異，它會更新與綁定相關的 DOM 屬性。它還需要將新值放入視圖的 `oldValues` 陣列中。就是這樣。你現在有了一個更新的使用者介面。一旦 Angular 完成檢查目前的元件，它會對子元件重複完全相同的步驟。

在我們的應用程式中，在 `App` 元件中，只有一個對 `span` 元素的 `textContent` 屬性的綁定。因此，在變更偵測期間，Angular 會讀取元件的屬性 `time` 的值，套用 `date` pipe，並將其與儲存在視圖中的先前值進行比較。如果它偵測到差異，Angular 會更新 span 的 `textContent` 屬性和 `oldValues` 陣列。

**但是，錯誤從何而來？**

在每個變更偵測週期之後，在開發模式下，Angular 會 **同步地** 執行另一次檢查，以確保表達式產生與前一個變更偵測執行期間相同的值。此檢查不是原始變更偵測週期的一部分。它會在整個元件樹的檢查完成 **之後** 執行，並執行完全相同的步驟。但是，這次，當 Angular 偵測到差異時，它不會更新 DOM。相反，它會拋出 `ExpressionChangedAfterItHasBeenCheckedError`。

![Image 29](https://wp.angular.love/wp-content/uploads/2024/07/content-image5-238.png)

原因
-------

所以現在我們知道錯誤何時被拋出。**但是，為什麼 Angular 需要此檢查？** 好吧，想像一下，在變更偵測執行期間，某些元件的屬性已被更新。結果，表達式會產生與使用者介面中呈現的不一致的新值。那麼，Angular 會做什麼？它當然可以執行另一個變更偵測週期，以將應用程式狀態與使用者介面同步。但是，如果在該過程中再次更新某些屬性會怎樣？看到模式了嗎？Angular 實際上可能會陷入無限的變更偵測執行迴圈。實際上，[這在 AngularJS 中經常發生](https://docs.angularjs.org/error/$rootScope/infdig)。

為了避免這種情況，Angular 強加了所謂的[單向資料流](https://angular.love/en/do-you-really-know-what-unidirectional-data-flow-means-in-angular/)。而這個在變更偵測後執行的檢查以及導致的 `ExpressionChangedAfterItHasBeenCheckedError` 錯誤是強制機制。一旦 Angular 處理完目前元件的綁定，你就不再可以更新用於綁定表達式的元件屬性。

修正錯誤
-----------------

為了防止錯誤，我們需要確保在變更偵測執行期間和後續檢查中，表達式回傳的值相同。在我們的案例中，我們可以像這樣將評估部分移出 `time` getter：

```typescript
export class AppComponent {
    _time;
    get time() {  return this._time; }

    constructor() {
        this._time = Date.now();
    }
}
```

但是，使用此實作，getter `time` 的值將始終相同。我們仍然需要更新該值。我們稍早了解到，產生錯誤的檢查會在變更偵測週期後 **同步** 執行。因此，如果我們 **非同步地** 更新它，我們將避免錯誤。因此，為了每毫秒更新一次值，我們可以使用 `setInterval` 函式，並將延遲設定為 `1` 毫秒，如下所示：

```typescript
export class AppComponent {
    _time;
    get time() {  return this._time; }

    constructor() {
        this._time = Date.now();
        
        setInterval(() => {
            this._time = Date.now();
        }, 1);
    }
}
```

此實作解決了我們原先的問題。但是，不幸的是，它引入了一個新的問題。所有計時事件，例如 `setInterval`，都會在 Angular 中觸發變更偵測。這表示使用此實作，我們最終會陷入無限的變更偵測週期迴圈中。**為了避免這種情況，我們需要一種方法來執行** `setInterval` \*\*\*\*而不觸發變更偵測。\*\*\*\*幸運的是，我們有辦法做到這一點。為了了解如何做到這一點，我們首先需要了解為什麼 `setInterval` 會在 Angular 中觸發變更偵測。

使用 zones 進行自動變更偵測
------------------------------------------

與 React 相反，Angular 中的變更偵測可以完全自動觸發，這是由於瀏覽器中的任何非同步事件所致。這是透過使用名為 `[**zone.js**](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b)` 的函式庫來實現的，該函式庫引入了 zones 的概念。與流行的觀點相反，zones 並不是 Angular 中變更偵測機制的一部分。事實上，[Angular 可以不用它們也能運作](https://angular.love/en/do-you-still-think-that-ngzone-zone-js-is-required-for-change-detection-in-angular/)。該函式庫只是提供了一種攔截非同步事件（例如 `setInterval`）並通知 Angular 的方法。根據該通知，Angular 會執行變更偵測。

有趣的是，你可以在網頁上擁有許多不同的 zones。其中一個將是 `NgZone`。它是在 Angular 引導時建立的。這是 Angular 應用程式在其內部執行的 zone。而 Angular **只會** 收到在此 zone 內發生的事件的通知。

![Image 30](https://wp.angular.love/wp-content/uploads/2024/07/content-image6-181.png)

但是，`zone.js` 也提供了一個 API，可以在 Angular zone 以外的 zone 中執行一些程式碼。Angular 不會收到其他 zones 中發生的非同步事件的通知。沒有通知表示沒有變更偵測。執行此操作的方法名稱稱為 `runOutsideAngular`，它由 `NgZone` 服務實作。

這是實作方法，它會注入 `NgZone` 並在 Angular zone 外部執行 `setInterval`：

```typescript
export class AppComponent {
    _time;
    get time() {
        return this._time;
    }

    constructor(zone: NgZone) {
        this._time = Date.now();

        zone.runOutsideAngular(() => {
            setInterval(() => {
                this._time = Date.now()
            }, 1);
        });
    }
}
```

現在我們不斷地更新時間，_但我們是以非同步方式並在 Angular zone 外部執行此操作。這保證了在變更偵測和後續檢查期間，getter `time` 會回傳相同的值。當 Angular 在下一個變更偵測週期中讀取 `time` 值時，該值將會被更新，並且變更將會反映在螢幕上。_

> 使用 NgZone 在 Angular 外部執行一些程式碼以避免觸發變更偵測是一種常見的優化技術。

除錯
---------

你可能想知道是否有任何方法可以在 Angular 內部查看此視圖和綁定。事實上，有。在 `[**@angular/core**](https://github.com/maximusk/angular/blob/24cf8b326918e0694dfcdcf48635b5325060827a/packages/core/src/view/view.ts#L350)` [模組](https://github.com/maximusk/angular/blob/24cf8b326918e0694dfcdcf48635b5325060827a/packages/core/src/view/view.ts#L350)中，有一個名為 `checkAndUpdateView` 的函式[。](https://github.com/maximusk/angular/blob/24cf8b326918e0694dfcdcf48635b5325060827a/packages/core/src/view/view.ts#L350) 它會執行元件樹中的每個視圖（元件），並對每個視圖執行檢查。當我遇到變更偵測問題時，我總是會開始除錯此函式。

試著自己除錯它。前往[這個 stackblitz 示範應用程式](https://angular-eobrrh.stackblitz.io/)，然後開啟主控台。找到該函式並在那裡設定中斷點。點擊按鈕以觸發變更偵測。檢查 `view` 變數。這是我執行此操作的錄製：

![Image 31](https://wp.angular.love/wp-content/uploads/2024/07/content-image7-33.gif)

第一個 `view` 將是主機視圖。它是 Angular 建立的用於託管我們的應用程式元件的某種根元件。我們需要繼續執行以到達它的子視圖，這將是為我們的 `AppComponent` 建立的視圖。**探索它**。`component` 屬性會保留對 `App` 元件實例的參考。`nodes` 屬性會保留對為 `App` 元件範本中的元素建立的 DOM 節點的參考。`oldValues` 陣列會儲存綁定表達式的結果。

運算順序
-----------------

我們剛剛了解到，由於單向資料流限制，你不能在檢查元件後，在變更偵測期間變更元件的某些屬性。最常見的情況是，當 Angular 對子元件執行變更偵測時，此更新是透過共用服務或同步事件廣播來進行的。但是，也可以將父元件直接注入到子元件中，並在生命週期鉤子中更新父元件狀態。以下是一些演示此操作的程式碼：

```typescript
@Component({
    selector: 'my-app',
    template: `
        <div [textContent]="text"></div>
        <child-comp></child-comp>
    `
})
export class AppComponent {
    text = 'Original text in parent component';
}

@Component({
    selector: 'child-comp',
    template: `<span>I am child component</span>`
})
export class ChildComponent {
    constructor(private parent: AppComponent) {}

    ngAfterViewChecked() {
        this.parent.text = 'Updated text in parent component';
    }
}
```

你可以在[這裡](https://stackblitz.com/edit/angular-zntusy)使用它。基本上，我們定義了兩個元件的簡單階層。父元件宣告了用於綁定的 `text` 屬性。子元件將父元件注入建構函式中，並在其 `ngAfterViewChecked` 生命周期鉤子中更新其屬性。你能猜到我們會在主控台中看到什麼嗎？?

沒錯，是熟悉的 `ExpressionChangedAfterItWasChecked` 錯誤。這是因為當 Angular 為子元件呼叫 `ngAfterViewChecked` 生命周期鉤子時，它已經檢查過父 `App` 元件的綁定。但是在檢查之後，我們會更新綁定中使用的父元件的屬性 `text`。

但是，有趣的部分來了。如果我現在變更鉤子會怎樣？比如說，改成 `ngOnInit`。你認為我們還會看到錯誤嗎？

```typescript
export class ChildComponent {
    constructor(private parent: AppComponent) {}

    ngOnInit() {
        this.parent.text = 'Updated text in parent component';
    }
}
```

**嗯，這次它不在那裡。** [檢查演示](https://stackblitz.com/edit/angular-ti5abu)。事實上，我們可以將程式碼放入任何其他鉤子中（除了 `AfterViewInit` 和 `AfterViewChecked`），而且我們不會在主控台中看到錯誤。那麼這裡發生了什麼事？為什麼 `ngAfterViewChecked` 鉤子很特別？

為了理解此行為，我們需要知道 Angular 在變更偵測期間執行的操作及其順序。而且，我們已經知道可以在哪裡找到它們：我先前向你展示的 `checkAndUpdateView` 函式。以下是你在函式主體中可以找到的部分程式碼：

```typescript
function checkAndUpdateView(view, ...) {
    ...       
    // update input bindings on child views (components) & directives,
    // call NgOnInit, NgDoCheck and ngOnChanges hooks if needed
    Services.updateDirectives(view, CheckType.CheckAndUpdate);
    
    // DOM updates, perform rendering for the current view (component)
    Services.updateRenderer(view, CheckType.CheckAndUpdate);
    
    // run change detection on child views (components)
    execComponentViewsAction(view, ViewAction.CheckAndUpdate);
    
    // call AfterViewChecked and AfterViewInit hooks
    callLifecycleHooksChildrenFirst(…, NodeFlags.AfterViewChecked…);
    ...
}
```

如你所見，Angular 還會在變更偵測過程中觸發生命週期鉤子。\*\*\*\*有趣的是，某些鉤子是在 Angular 處理綁定時的渲染部分之前呼叫，而某些鉤子是在渲染部分之後呼叫。\*\*\*\*這是展示當 Angular 為父元件執行變更偵測時會發生什麼的圖表：

![](https://wp.angular.love/wp-content/uploads/2024/07/content-image8-141.png)

讓我們逐步了解它。首先，它會更新 **子** 元件的輸入綁定。然後，它會在 **子** 元件上呼叫 `OnInit`、`DoCheck` 和 `OnChanges` 鉤子。這是有道理的，因為它剛剛更新了輸入綁定，並且 Angular 需要通知子元件輸入綁定已初始化。**然後，Angular 會對目前元件執行渲染。** 之後，它會對子元件執行變更偵測。這表示它基本上會對子視圖重複這些操作。最後，它會在 **子** 元件上呼叫 `AfterViewChecked` 和 `AfterViewInit` 鉤子，以告知它已被檢查。

我們在這裡可以注意到的是，Angular 會在處理父元件的綁定 **之後**，為子元件 **呼叫** `AfterViewChecked` 生命週期鉤子。另一方面，`OnInit` **鉤子是在** 處理綁定 **之前** 呼叫。因此，即使 `OnInit` 中的 `text` 值發生變更，在後續檢查期間它仍然會相同。這就解釋了在 `ngOnInit` 鉤子中沒有錯誤的看似怪異的行為。謎團解開了 ?。

總結
-------

好的，現在讓我們總結一下我們剛剛學到的內容。Angular 中的所有元件在內部都以稱為視圖的資料結構表示。Angular 的編譯器會解析範本並建立綁定。每個綁定都會定義要更新的 DOM 元素的屬性，以及用於取得值的表達式。在變更偵測期間用於比較的先前值會儲存在視圖的 `oldValues` 屬性中。在變更偵測期間，Angular 會執行綁定、評估表達式、將它們與先前的值進行比較，並在必要時更新 DOM。在每個變更偵測週期之後，Angular 會執行檢查，以確保元件狀態與使用者介面同步。此檢查會同步執行，並且可能會拋出 `ExpressionChangedAfterItWasChecked` 錯誤。

### 我們學到了很多！我希望這些知識能喚醒你的好奇心。你接下來該做什麼？

**[這 5 篇文章會讓你成為 Angular 變更偵測專家](https://angular.love/en/these-5-articles-will-make-you-an-angular-change-detection-expert/)**

如果你正在尋找關於 Angular 中變更偵測的更深入解釋，本文是一個很好的起點。它是一系列深入探討變更偵測主題的文章，例如 zones、DOM 更新機制、單向資料流和 `ExpressionChangedAfterItWasChecked` 錯誤。

**[提升你的逆向工程技能](https://angular.love/en/level-up-your-reverse-engineering-skills/)**

我今天與你分享的大部分內容都是透過逆向工程發現的。對我來說，這是學習最有價值的方式，但它可能是最具挑戰性的。本文總結了我的逆向工程經驗，並概述了一些有助於你開始自行探索來源的指南和原則。
