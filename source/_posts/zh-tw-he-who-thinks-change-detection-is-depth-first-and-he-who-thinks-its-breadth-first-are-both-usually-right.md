---
title: He who thinks change detection is depth-first and he who thinks it’s breadth-first are both usually right
date: 2025-01-13 16:40:56
tags:
---


原文：<https://angular.love/he-who-thinks-change-detection-is-depth-first-and-he-who-thinks-its-breadth-first-are-both-usually-right>

# 認為變更檢測是深度優先的人，與認為變更檢測是廣度優先的人，通常都是對的

我曾經被問到 [Angular 中的變更偵測](https://angular.love/these-5-articles-will-make-you-an-angular-change-detection-expert/)是深度優先還是廣度優先。這基本上意味著 Angular 是先檢查當前元件的兄弟元件（廣度優先），還是先檢查其子元件（深度優先）。我之前沒有考慮過這個問題，所以我只是憑直覺和對內部機制的了解來回答。我宣稱它是深度優先的。後來，為了驗證我的斷言，我建立了一個元件樹，並在 `ngDoCheck` 鉤子中加入了一些日誌記錄邏輯：

<!-- more -->

```
@Component({
    selector: 'r-comp',
    template: `{{addRender()}}`
})
export class RComponent {
    ngDoCheck() {
        // 依序保存所有呼叫並記錄到控制台
        calls.ngDoCheck.push('R');
    }
```

令我驚訝的是，結果顯示某些兄弟元件先被檢查，如下圖所示：

![Image 16](https://wp.angular.love/wp-content/uploads/2024/07/content-image1-79.gif)

所以你在這裡看到 Angular 先檢查 `K`，然後是 `V`、`L`，然後是 `C`，依此類推。那麼我錯了嗎？它真的是一個廣度優先的演算法嗎？嗯，不完全是。首先要注意的是，上面的表示不是一個正確的廣度優先演算法。該演算法的傳統實作檢查****同一層級上的所有兄弟元件****，而在上圖中，你可以看到該演算法確實檢查了 `L` 和 `C` 兄弟元件，但它並沒有檢查 `X` 和 `F`，而是向下檢查 `J` 和 `O`。此外，廣度優先圖遍歷演算法的實作是明確定義的，但我無法在原始碼中找到它。因此，我決定進行另一個實驗，並在變更偵測評估表達式時呼叫的自訂函數中加入日誌記錄邏輯：

```
@Component({
    selector: 'r-comp',
    template: `{{addRender()}}`
})
export class RComponent {
    addRender() {
        calls.render.push('R');
    }
}
```

這次我得到了不同的結果：

![Image 17](https://wp.angular.love/wp-content/uploads/2024/07/content-image2-71.gif)

這是一個正確的深度優先圖遍歷演算法。那麼這裡發生了什麼事？其實很簡單，讓我來解釋一下。

變更偵測操作
-----------

為了了解行為上的差異，我們需要看看變更偵測機制在檢查元件時執行的操作。如果你讀過 [我其他關於變更偵測的文章](https://angular.love/these-5-articles-will-make-you-an-angular-change-detection-expert/)，你可能知道變更偵測執行的主要操作如下：

*   更新子元件的屬性
*   在****子元件上****呼叫 `NgOnChanges` 和 [`NgDoCheck`](https://angular.love/if-you-think-ngdocheck-means-your-component-is-being-checked-read-this-article/) 生命周期鉤子
*   在****當前元件上****更新 DOM
*   ****對子元件執行變更偵測****

我上面重點標示了一個有趣的細節 — 當 Angular 檢查當前元件時，它會呼叫****子元件上****的生命週期鉤子，但會為****當前元件****渲染 DOM。這是一個非常重要的區別。這正是使得如果我們將日誌記錄放入 `NgDoCheck` 鉤子中，會讓人覺得該演算法以廣度優先方式運作的原因。當 Angular 檢查一個當前元件時，它會呼叫其所有作為兄弟元件的子元件的生命週期鉤子。假設 Angular 現在檢查 `K` 元件，並在 `L` 和 `C` 上呼叫 `NgDoCheck` 生命周期鉤子。因此，我們得到以下結果：

![Image 18](https://wp.angular.love/wp-content/uploads/2024/07/content-image3-48.gif)

看起來像是廣度優先演算法。然而，請記住，Angular 仍在檢查 `K` 元件的過程中。因此，在完成 `K` 元件的所有操作後，它不會像廣度優先實作那樣繼續檢查兄弟元件 `V`。相反，它會繼續檢查 `K` 的子元件 `L`。這是變更偵測演算法的深度優先實作。而且，我們現在知道它會在 `J` 和 `O` 元件上呼叫 `ngDoCheck`，這正是發生的情況：

![Image 19](https://wp.angular.love/wp-content/uploads/2024/07/content-image4-53.gif)

所以，最後，我的直覺並沒有讓我失望。變更偵測機制在內部是以深度優先方式實作的，但會先在兄弟元件上呼叫 `ngDoCheck` 生命周期鉤子。順帶一提，我已經在 [如果你認為 \`ngDoCheck\` 表示你的元件正在被檢查 — 請閱讀這篇文章](https://angular.love/if-you-think-ngdocheck-means-your-component-is-being-checked-read-this-article/) 中深入描述了這個邏輯。

### Stackblitz 演示

[這裡](https://stackblitz.com/edit/depth-or-breadth-first) 你可以看到在不同位置記錄日誌的演示。
