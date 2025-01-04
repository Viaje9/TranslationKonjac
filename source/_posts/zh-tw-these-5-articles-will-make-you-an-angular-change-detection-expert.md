---
title: These 5 articles will make you an Angular Change Detection expert
date: 2025-01-04 16:50:34
tags:
---

原文：[https://angular.love/these-5-articles-will-make-you-an-angular-change-detection-expert](https://angular.love/these-5-articles-will-make-you-an-angular-change-detection-expert)

# 5 篇文章讓你成為 Angular Change Detection 專家

在過去的 8 個月裡，我將大部分空閒時間花在逆向工程 Angular 上。其中最讓我著迷的主題是 change detection。我認為這是框架中最重要的一部分，因為它負責「可見」的工作，像是 DOM 更新、輸入綁定和查詢列表更新。我的探索產生了一系列深入的文章，這些文章突顯了 change detection 機制的主要概念，並深入探討了實作細節。在這篇文章中，我將它們全部整理在一起，並簡要說明你可以在每篇文章中找到什麼。一旦你完成閱讀它們，你將獲得 change detection 的啟發。

<!-- more -->

理解 change detection
------------------------------

這裡列出了 5 篇深入的文章（以及一篇新的），它們將顯著擴展你對 Angular 中 change detection 過程的認知邊界。每篇文章都建立在前一篇解釋的資訊之上，所以我建議你按照這裡列出的順序閱讀它們。

[**Angular 的 $digest**](https://angular.love/angulars-digest-is-reborn-in-the-newer-version-of-angular/) **[在新版 Angular 中重生](https://angular.love/en/angulars-digest-is-reborn-in-the-newer-version-of-angular/)**

這篇文章比較了 AngularJS 中的 `$digest` 過程和 Angular 中的 change detection 過程。它解釋了兩者存在的必要性，並展示了它們如何使用相同的 [dirty checking](https://teropa.info/blog/2015/03/02/change-and-its-detection-in-javascript-frameworks.html#angularjs-dirty-checking) 概念來建構。然後，它提供了一些範例，示範如何在 Angular 中使用生命週期鉤子作為 AngularJS 中 `$watch` 的等效機制。它還展示了 Angular 與 AngularJS 的不同之處，因為它現在強制執行所謂的從上到下的單向資料流。這篇文章解釋了強制執行背後的理由、其好處以及它對架構的限制。這篇文章對於希望遷移到 Angular 的經驗豐富的 AngularJS 開發人員尤其有用。

**[Angular 變更檢測的入門介紹](/2025/01/04/zh-tw-a-gentle-introduction-into-change-detection-in-angular/)**

這篇文章提供了對 change detection 機制的「淺顯」解釋。它將讓你對其組成部分和機制有一個高層次的概述：用於表示元件的內部資料結構、綁定的作用以及作為過程一部分執行的操作。我也會提到 zones，並確切地告訴你這個功能如何在 Angular 中啟用自動 change detection。

**[Angular 中的變更偵測是否仍然需要 NgZone (zone.js)？](/2025/01/03/zh-tw_do-you-still-think-that-ngzone-zone-js-is-required-for-change-detection-in-angular/)**

這篇文章描述了 [NgZone](https://angular.io/api/core/NgZone) 如何在 [zone.js](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/) 函式庫之上實作，並解釋了 NgZone 在框架中扮演的角色。與普遍的看法相反，它不是 change detection 過程的一部分，而是用於觸發它。文章首先示範了 Angular 如何在沒有 NgZone 和 zone.js 的情況下檢測變更並執行渲染。然後，它繼續展示 NgZone 帶來的價值以及它是如何實作的。文章的很大一部分致力於解釋常用的公共 API，如 `isStable`、`onUnstable` 和 `onMicrotaskEmpty`。文章最後解釋了使用 GoogleAPI 等第三方函式庫時常見的變更無法檢測到的問題。

**[關於 Angular 中 change detection 你需要知道的一切](/2025/01/04/zh-tw_everything-you-need-to-know-about-change-detection-in-angular/)**

如果你想紮實掌握 change detection 機制，這篇文章是必讀的。它提供了引擎如何實作的高層次概述，並提供相關連結以供進一步探索。文章首先介紹了一個稱為 View 的內部元件表示，並解釋了 change detection 過程在 View 上執行。然後，它列出了在 change detection 期間按執行順序執行的所有操作。這些操作包括更新 View 狀態、渲染、處理輸入綁定和呼叫生命週期鉤子。最後，它解釋了 [ChangeDetectorRef](https://angular.io/api/core/ChangeDetectorRef) 公共 API，如 `detach`、`detectChanges` 和 `markForCheck`，並提供了這些方法用法的簡短範例。

**[Angular 中 DOM 更新的機制](/2025/01/04/zh-tw-the-mechanics-of-dom-updates-in-angular/)**

這篇文章深入探討了將應用程式模型與 DOM 同步的過程的實作細節，也就是單向資料綁定或 DOM 渲染。此操作在 change detection 過程中佔據中心位置，因為它正是使元件變更在 DOM 中渲染的原因。文章首先揭露了關於 View 概念的更多細節，特別是 View Factory 和幾種基本的 View 節點類型。然後，它展示了 change detection 機制如何針對這些節點執行插值或輸入綁定設置的 DOM 更新。

**[Angular 中屬性綁定更新的機制](https://angular.love/en/the-mechanics-of-property-bindings-update-in-angular/)**

與前一篇關於 DOM 更新的文章類似，這篇文章探討了負責更新子元件和指令的輸入綁定的過程的實作細節。它介紹了 Binding Definition 的概念及其在 change detection 過程中的作用。然後，它繼續示範當編譯器處理[屬性綁定的範本語法](https://angular.io/guide/template-syntax#property-binding--property-)時，這些綁定定義是如何產生的。最後，它概述了在 View 節點上執行 change detection 並更新子元件和指令的輸入屬性的逐步過程。

避免常見的混淆
--------------------------

這裡列出了一些額外的有價值的文章，它們澄清了我經常在 stackoverflow 上看到的與 change detection 相關的一些混淆。

**[認為 change detection 是深度優先的人和認為它是廣度優先的人通常都是對的](https://angular.love/en/he-who-thinks-change-detection-is-depth-first-and-he-who-thinks-its-breadth-first-are-both-usually-right/)**

這篇文章回答了一個有趣的問題，即 Angular 是先檢查當前元件的兄弟元件（廣度優先順序），還是先檢查其子元件（深度優先順序）。它展示了 Angular 如何在實際開始檢查之前觸發兄弟元件上的生命週期鉤子，並解釋了這種行為如何導致你得出錯誤的答案。

**[你真的知道 Angular 中的單向資料流是什麼意思嗎](https://angular.love/en/do-you-really-know-what-unidirectional-data-flow-means-in-angular/)**

這篇文章解釋了雙向資料綁定和單向資料流之間的差異。它示範了 Angular 中更新輸入綁定的過程與 AngularJS 的不同之處，以及這種差異的重要性。

**[關於 \`ExpressionChangedAfterItHasBeenCheckedError\` 錯誤你需要知道的一切](https://angular.love/en/everything-you-need-to-know-about-the-expressionchangedafterithasbeencheckederror-error/)**

這篇文章解釋了 Angular 社群中常見且經常被誤解的錯誤背後的理由和機制。雖然有些開發人員將其視為錯誤，但實際上這是一個設計決策，旨在透過將 change detection 運行限制為單次運行而不是 AngularJS 中的多次運行（`$digest` 運行）來提高效能。這篇文章解釋了拋出錯誤如何幫助防止模型資料和 UI 之間的不一致，以便不會在頁面上向使用者顯示錯誤或過時的資料。這篇文章主要包含兩個部分，第一部分探討錯誤的原因，第二部分建議可能的修復方法。它還解釋了為什麼在生產模式下不會拋出錯誤。

**[如果你認為 \`ngDoCheck\` 表示你的元件正在被檢查](https://angular.love/en/if-you-think-ngdocheck-means-your-component-is-being-checked-read-this-article/)**

這篇文章詳細解答了為什麼即使這些元件上的輸入綁定沒有更改，也會為使用 `OnPush` 策略的元件觸發 `ngDoCheck` 生命週期鉤子的問題。它解釋了通常令人意想不到的事實，即當檢查父元件時，會為子元件觸發鉤子，並展示了該機制如何負責觸發 `ngDoCheck`，即使表面上沒有理由這樣做。文章的第二部分透過示範一些用例來回答為什麼我們需要 `ngDoCheck` 的問題。

**[Angular 中 Constructor 和 ngOnInit 的本質區別](https://angular.love/en/the-essential-difference-between-constructor-and-ngoninit-in-angular/)**

這篇文章詳細解答了 stackoverflow 上最受歡迎的 Angular 問題之一，瀏覽量超過 10 萬次，即[Constructor 和 ngOnInit 之間的區別](https://stackoverflow.com/questions/35763730/difference-between-constructor-and-ngoninit)。這篇文章給出了全面的比較，突顯了使用上的差異，也深入探討了元件的初始化過程。
