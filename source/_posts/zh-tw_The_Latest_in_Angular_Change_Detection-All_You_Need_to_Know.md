---
title: Angular 最新的變更偵測 – 所有你需要知道的
date: 2024-12-24 18:40:14
tags:
---

原文：<https://angular.love/the-latest-in-angular-change-detection-zoneless-signals>

# Angular 最新的 Change Detection—所有你需要知道的



 Change Detection 一直是一個熱門話題，這並不奇怪  它是任何框架的核心概念之一。不管他如何設計或實現， Change Detection 始終是框架的基本任務之一，同時也是開發者選擇這些框架的主要原因之一。

在 Angular 中，這個話題特別具有爭議性，主要是因為由 **Zone.js** 支援的 Change Detection 機制具有某種「魔法般」的性質。最近這部分有重大更新，特別是新增了 **signals**（信號），並提供了 **不使用 Zone.js** 的選項。我們一起來探索這些變化是如何展開的。


<!-- more -->

## （不）快速的回顧

在深入介紹新變化之前，我們需要對已存在多年的 Change Detection 機制有基本的了解。這個基礎將幫助我們更好地掌握新概念，因為它們是在現有機制之上構建的，而不是引入完全獨立或全新的東西（是一種演進，而非革命）。如果你對引入 **signals** 和 **Zoneless**（無 Zone.js）的 Change Detection 已經非常熟悉，可以考慮跳過本章。

當談到 Change Detection 時，我認為將其執行劃分為「**When**」與「**How**」兩個方面是非常有幫助的。理解這兩個部分對於掌握 Angular 的整體 Change Detection 流程至關重要。


### When？

「**When**」的部分與 Change Detection 的排程有關。它涉及 Change Detection 何時發生以及導致其執行的因素。

除了手動觸發 Change Detection 的選項外，主要由 **scheduler**（調度器）負責為我們處理這個問題。自 Angular 問世以來， Change Detection 的調度器一直基於 ***Zone.js***。該庫通過[補丁瀏覽器 API](https://github.com/angular/zone.js/blob/master/STANDARD-APIS.md)並攔截任務執行來跟踪各種操作。

簡單來說，可以說 Angular 的 zone (***NgZone***) 在每次攔截的操作完成後，檢查 microtask queue 是否為空。如果是，則會觸發一個特殊事件，這個事件被 scheduler 使用，最終觸發 Change Detection 的運行。我們不會深入探討更多細節，但如果你對此機制不熟悉，強烈建議閱讀[這篇文章](https://angular.love/running-change-detection-autorun-with-zones)（或者如果你真的感興趣，可以閱讀[所有相關文章](https://angular.love/search?q=zone.js)）。

關鍵點在於理解 ***Zone.js*** 向 Angular 提供關於完成操作的通知，促使框架通過運行 Change Detection 做出響應。需要注意的是，這種配置並不知道 Template 綁定的資料實際上是否有改變，因此刷新可能必要也可能不必要。


### How？

「**How**」部分聚焦於 Change Detection 執行的機制，包括組件樹的遍歷以及檢查變更的過程。一旦排定了 Change Detection 執行，就必須了解這對我們的應用程式意味著什麼、執行方式爲何，以及可能的結果。

以下是 Change Detection 基本知識的簡要回顧，這些知識將幫助我們進一步討論。你也可以考慮閱讀[這篇概述](https://angular.love/change-detection-big-picture-overview)，它簡要解釋了該主題。

Angular 的組件結構形成一棵樹。在默認配置下，以下規則以高層次的方式描述了該流程的執行方式：

- 該過程從根組件開始。

- 整個組件樹都會檢查是否有變更，意味著每個節點都會被訪問。

- 遍歷方向為自上而下。

- 節點的訪問順序遵循深度優先搜索（DFS）算法。

以下動畫說明了這個過程：

![](https://wp.angular.love/wp-content/uploads/2024/10/Change-Detection.gif)

當**訪問**或**偵測到變更**單個節點時，會發生什麼？這涉及多個操作，包括執行 lifecycle hooks 、更新綁定以及必要時刷新畫面。我們不會深入這些細節，因為它們屬於更廣泛的主題。對於本文而言，只需記住：如果發生數據變更，component view 會在 Change Detection 的過程中更新即可。


###  Change Detection 策略 – OnPush

在 Angular 中，有兩種 Change Detection 策略：***Default*** 和 ***OnPush*****。「How」章節中描述的規則實際上就是 *Default*** 策略的行為規範。那麼，*OnPush* 有何不同？

基本的樹遍歷規則保持不變，但 ***OnPush*** 策略允許我們在 Change Detection 期間「剪除」或跳過一些樹的分支。這減少了執行的操作數量，從而提高了性能。

當一個組件使用 ***OnPush*** 策略時，它（以及其子組件）不會總是被檢測。它們僅在被標記為「dirty」時才會被檢測。如果組件沒有被標記為 dirty，則整個分支都會被跳過。

可以在應用中混合使用 ***Default*** 和 ***OnPush*** 組件。例如，如果父組件是 ***OnPush***，而子組件是 ***Default***，只要父組件被標記為 dirty 並未被剪除，子組件仍然會被檢測。

此時，我們可以重新審視「整個組件樹都會被檢查，意味著每個節點都會被訪問」的規則。如果我們使用 ***OnPush*** 組件，這一點將不再成立，這構成了一個例外。

以下是該過程的動畫展示：

![](https://wp.angular.love/wp-content/uploads/2024/10/onPush.gif)


### Dirty 標記

在上一節中，我們提到 ***OnPush*** 策略依賴於組件的「dirty」狀態來進行 Change Detection 。一個常見的問題是：什麼會使組件變得「dirty」？這裡有一組封閉的操作集可以將組件設為這種狀態：

- 輸入值改變（immutable）。

- 發生模板綁定事件（包括 output emit 和 host listener）。

- 一個值被異步管道（async pipe）使用。

- 直接呼叫 `ChangeDetectorRef.markForCheck()`。

- `@defer` 區塊的狀態變化。

另一個重要的點是，這些操作會「向上冒泡」穿過組件層級結構。這由 `markViewDirty` 函數負責。這意味著不僅變更來源的組件會被標記為 dirty（例如上圖中的 F），其所有的父組件（如 E 和 A）也會被標記為 dirty。這確保了嵌套的組件（例如 F）在 Change Detection 中總會被訪問，不會被任何 ***OnPush*** 且非 dirty 的父組件（如 E）剪除。

唯一的例外是當組件由於來自父組件綁定的輸入值變化而被標記為 dirty。在這種情況下，父組件已經處於其自身的 Change Detection 檢查過程中，因此無需再次將其標記為 dirty。此時，僅接收輸入的組件會被標記，允許 Change Detection 在父組件完成後處理它。


## 改善「如何」 – 信號（Signals）

現在我們已經回顧了 Angular 長期以來的 Change Detection 規則，可以探索該過程如何得到改進。

最近，信號（signals）成為框架許多令人興奮改進背後的推動力， Change Detection 也不例外。在改善「如何」這方面，信號比標準流程有了顯著提升。讓我們更詳細地了解它的工作原理，以及開發者需要進行哪些調整。

本文假設你已對信號、信號圖（signal graphs）及相關概念有基本了解。如果沒有，強烈建議先閱讀 Max Koretskyi 的深入文章：[Signals in Angular – Deep Dive for Busy Developers](https://angular.love/signals-in-angular-deep-dive-for-busy-developers)。

第一個關鍵概念是，從信號圖的角度來看，組件視圖是作為一個反應式消費者（reactive consumer）運行的，這意味著它會響應模板中信號的讀取。我們可以將其可視化如下：

![](https://wp.angular.love/wp-content/uploads/2024/10/reactive-consumer.png)

當模板中讀取的信號接收到新值時，它會將反應式消費者標記為 dirty。需要強調的是，這裡標記為 dirty 的是附加到組件視圖的反應式消費者，而不是組件視圖本身（即組件樹中的節點）。否則，它將與普通的 ***OnPush*** 策略類似。由於這一區別，我們可以得到不同的結果，稍後你會看到。

但將反應式消費者標記為 dirty 對於我們的應用意味著什麼？這取決於你所使用的 Angular 版本。

在 Angular 17 之前，將反應式消費者標記為 dirty 也會將組件視圖標記為 dirty，其行為與使用 **AsyncPipe** 類似。

在這種情況下，我們能實現的最佳效果是為所有組件使用 ***OnPush*** 策略，從而將 Change Detection 限制到單一路徑：

![](https://wp.angular.love/wp-content/uploads/2024/10/one-path.gif)

然而，如我們剛剛所發現的，這與 **AsyncPipe** 的工作方式完全相同，因此看起來並沒有什麼特別的好處。

情況在 Angular 17 或更高版本中發生了重大改變。在這些版本中，將反應式消費者標記為 dirty 不再將整個組件標記為 dirty。相反，會觸發一個新函數 `markAncestorsForTraversal`。這個函數從組件開始，向上遍歷其祖先直到根組件（與 `markViewDirty` 類似），但不同之處在於，當前組件保持不變（因為反應式消費者已標記為 dirty），而其祖先則被標記為新標誌 **HasChildViewsToRefresh**。這樣的操作如下圖所示：

![](https://wp.angular.love/wp-content/uploads/2024/10/mark-traversal.gif)

 Change Detection 的遍歷機制也被更新。現在，當 Change Detection 從這樣的樹結構開始時，它會通過 A 和 E 組件，而不對它們進行 Change Detection 。原因是它們是 ***OnPush*** 且不是 dirty 的。得益於新標誌 **HasChildViewsToRefresh**，Angular 繼續訪問標有此標誌的節點，搜尋需要 Change Detection 的組件（在此例中為標記有 dirty 的反應式消費者的組件 F）。當到達 F 組件時，發現其反應式消費者為 dirty，該組件因此進行 Change Detection ——且僅此一個！

![](https://wp.angular.love/wp-content/uploads/2024/10/one-node.gif)

很酷，對吧？我們已經從檢測整個組件路徑變更到只檢測單個組件。雖然這是簡化的組件樹例子，但在實際應用中，性能提升更加顯著。

這種基於信號的全新 Change Detection 方法——僅檢測一個組件的方式，通常被稱為「**semi-local**」（半本地化）或「**global-local/glocal**」（全局-本地化） Change Detection 。


### 注意事項

然而，有一些注意事項需要考慮。我們來看一個例子，其中用戶點擊觸發了信號值的更改：

![](https://wp.angular.love/wp-content/uploads/2024/10/caveat1.gif)

點擊按鈕後，我們注意到與之前描述的場景不同，我們的組件及其所有祖先都被標記為 dirty，導致它們全都被 Change Detection 。為什麼會這樣？原因在於舊的 Change Detection 規則仍在生效。你還記得哪些行為會將組件標記為 dirty 嗎？其中之一是模板綁定事件，而這正是此處發生的事情。如果你調試這樣的例子，會發現儘管 `markAncestorsForTraversal` 被調用，但 `markViewDirty` 也被調用。

這導致我們得出結論：信號值更改的來源很重要。如果它來自於某些會將組件標記為 dirty 的操作，那麼與標準的 ***OnPush*** 策略相比並沒有優勢。為了受益於半本地化 Change Detection ，觸發必須來自不會將組件標記為 dirty 但仍能為我們排定 Change Detection 的操作。例如：

- `setInterval`、`setTimeout`

- `Observable.subscribe`、`toSignal(Observable)`，例如 `HttpClient` 調用

- RxJS 的 `fromEvent` 或 `Renderer2.listen`

另一個更明顯但仍可能阻礙我們努力的注意事項是應用中某些組件未使用 ***OnPush*** 策略。假設信號值變更的觸發源不會將組件標記為 dirty，下面這棵樹中 ABCD 子樹仍會進行 Change Detection ，因為這些組件未使用 ***OnPush***，且每次運行中都會檢查它們的變更（默認策略仍適用）。

![](https://wp.angular.love/wp-content/uploads/2024/10/caveat2a.gif)


## 改善「何時」—Zoneless（無 Zone.js）

我們剛剛探討了信號在改善 Angular  Change Detection 流程「**如何**」方面的重大作用。現在，我們來看看「**何時**」部分可以如何改進。Angular 團隊的答案是轉向「無 Zone.js」（***Zoneless***）。


### 為什麼移除 Zone.js？

首先，讓我們考慮一下為什麼要完全移除 ***Zone.js***。這裡有幾個原因：

請不要誤解我的意思——我認為 ***Zone.js*** 是一個令人著迷的工具（至少在它推出的時候是這樣的），它自始至終對 Angular 的成功起到了重要作用。它讓開發者更輕鬆地使用框架，即便是新手開發者也能簡單地專注於他們的任務，剩下的「魔法」由框架處理。

然而，正如我們所見，這種「魔法」是有代價的。雖然許多應用可能在使用 ***Zone.js*** 時運行得非常好，但當面對更複雜的解決方案，並需要提升性能時，尋找替代方案是一個合理的選擇。


### Zoneless 調度器

在 Angular 17.1 中推出的新 ***Zoneless*** 調度器不再依賴於 ***Zone.js*** 事件，而是等待框架其他部分的顯式通知來處理變更。這一轉變非常重要：框架不再基於「當某些操作剛剛發生，並且可能發生了某些改變」來觸發 Change Detection ，而是基於「當收到數據已改變的通知時」來進行觸發。

為實現這一點，新調度器提供了一個特殊的 `notify` 方法，在以下場景中會被調用：

- 當模板中的信號讀取到新值（具體來說，當 `markAncestorsForTraversal` 被調用時）。

- 當通過 `markViewDirty` 函數將組件標記為 dirty 時。這可能發生在組件通過 **AsyncPipe** 接收到新值、模板綁定事件、調用 `ComponentRef.setInput` 或顯式調用 `ChangeDetectorRef.markForCheck` 等情況下。

- 當一個 `afterRender` 鉤子被註冊、視圖重新附加到 Change Detection 樹或從 DOM 中移除時。在這些情況下，`notify` 方法會被調用，但它僅執行鉤子，並不刷新視圖。

你可能會擔心 Change Detection 是否會過於頻繁地運行，比如當多個信號值快速變更或事件頻繁觸發時。幸運的是，調度器的實現設計得非常高效。它會在短時間內收集通知，並排程單次 Change Detection 運行，而不是多次觸發。這種行為基於 `setTimeout` 和 `requestAnimationFrame` 的競態條件，但我們不會深入這些細節。關鍵在於這些 `notify` 調用會被合併，確保最佳性能。

要啟用新調度器，你需要按如下方式更新你的應用（關鍵新增項是 `provideExperimentalZonelessChangeDetection`，其餘部分取決於你的現有設置）：

```typescript
import { ApplicationConfig, provideExperimentalZonelessChangeDetection } from '@angular/core';
import { bootstrapApplication } from "@angular/platform-browser";
import { AppComponent } from "./app.component";

export const appConfig: ApplicationConfig = {
  providers: [
    provideExperimentalZonelessChangeDetection(),
  ]
};

bootstrapApplication(AppComponent, appConfig)
  .catch((err) => console.error(err));

// 從 angular.json 中移除以下內容：
//
// "polyfills": [
//   "zone.js"
// ],
```

以下動畫展示了在使用 ***OnPush*** 策略和信號的應用中，新 ***Zoneless*** 調度器的運作方式：

![](https://wp.angular.love/wp-content/uploads/2024/10/zoneless.gif)

這當然是最佳情況：信號值變更只標記了一個組件進行 Change Detection 並觸發檢測運行。這意味著我們僅在必要的地方進行 Change Detection ，且僅在需要的時候執行。


### 區域化/混合調度器

在上一章中，我們提到 ***Zoneless*** 調度器在 Angular 17.1 中首次引入。Angular 18 不僅顯著改進了這一實現，還為現有的基於 ***Zone.js*** 的調度器帶來了一項令人興奮的增強功能，即加入了我們在 ***Zoneless*** 調度器中看到的顯式通知機制。這意味著，調度器現在不僅監聽 ***Zone.js*** 事件，還會因 `notify` 方法的調用而觸發。

在 Angular 18 之前，如果信號值在 Zone.js 外發生變更， Change Detection 不會被排程——只有在 Zone.js 內執行的操作才能保證排程。例如，以下代碼不會導致視圖刷新：

```typescript
zone.runOutsideAngular(() => {
  httpClient.get<Post>('https://jsonplaceholder.typicode.com/posts/1')
    .pipe(
      delay(3000),
      map((res) => res.title)
    ).subscribe((title) => {
      this.title.set(title);
    });
});
```

使用增強的混合調度器，在相同的場景下，我們仍然可以實現視圖刷新。這是因為信號值的更改會調用 `notify` 方法，無論我們處於 Zone.js 內部還是外部。

這為尚未準備完全轉向 ***Zoneless*** 的應用程序提供了許多新可能性，同時也能受益於漸進式的過渡。


## 總結

希望本文能幫助你理解 Angular  Change Detection 機制的最新進展，以及這些概念如何協同工作。我也希望你能從中得出結論：不必一次性全面採用這些新功能。如果這成為你使用最新功能的阻礙，請記住，你可以穩步地逐步適配信號功能，使更多組件兼容 ***OnPush*** 策略，最終轉向 ***Zoneless*** 方法。在此期間，你仍然可以收穫其中的好處。

