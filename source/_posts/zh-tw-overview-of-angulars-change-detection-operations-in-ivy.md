---
title: Overview of Angular’s Change Detection operations in Ivy
date: 2025-01-04 16:07:40
tags:
---

原文：<https://angular.love/overview-of-angulars-change-detection-operations-in-ivy>

# Ivy 中 Angular 的變更偵測操作概述

不久前，我[寫了一篇文章](https://angular.love/en/everything-you-need-to-know-about-change-detection-in-angular/)，詳細探討了 Angular 的變更偵測在變更偵測期間執行的所有操作。隨著 Angular 在 v12 中切換到名為 Ivy 的新渲染引擎，文章中呈現的資訊在某種程度上已經過時。

在本文中，我想概述 Angular 在新的 Ivy 引擎中變更偵測期間執行的所有操作。

<!-- more -->

> ****本文節錄自我的 [Angular 深度探索課程系列](https://angular.love/en/change-detection-big-picture-operations/)****

當 Angular 為特定元件（視圖）執行變更偵測時，它會執行許多操作。這些操作有時被稱為副作用，就像計算邏輯的副作用一樣：

> 在電腦科學中，如果操作、函式或表達式修改了其本地環境之外的某些狀態變數值，則稱其具有副作用，也就是說，如果它具有除了將值回傳給操作的呼叫者之外的任何可觀察到的效果。

****在 Angular 中，變更偵測的主要副作用是將應用程式狀態呈現到目標平台。大多數時候，目標平台是瀏覽器，應用程式狀態具有元件屬性的形式，而呈現涉及更新 DOM。****

當檢查一個元件時，Angular 會執行一些其他操作。我們可以透過探索 [refreshView](https://github.com/angular/angular/blob/02f3d12a0dc2c1b6f5ae06fff019058036fa5edc/packages/core/src/render3/instructions/shared.ts#L357%3E) 函式來識別它們。一個經過簡化的函式主體，附帶我的解釋性註解如下所示：

```
function refreshView(tView, lView, templateFn, context) {
  enterView(lView);

  try {
    if (templateFn !== null) {
      // 更新子元件的 input bindings
      // 執行 ngOnInit、ngOnChanges 和 ngDoCheck hooks
      // 更新目前元件上的 DOM
      executeTemplate(tView, lView, templateFn, RenderFlags.Update, context);
    }

    // 執行 ngOnInit、ngOnChanges 和 ngDoCheck hooks
    // 如果它們尚未從 template 函式執行
    const preOrderCheckHooks = tView.preOrderCheckHooks;
    if (preOrderCheckHooks !== null) {
      executeCheckHooks(lView, preOrderCheckHooks, null);
    }

    // 首先將在此 lView 中宣告的 transplant 視圖標記為需要在其插入點進行刷新。
    // 這是為了避免 template 在此 `LView` 中定義，但其宣告出現在插入元件之後的情況。
    markTransplantedViewsForRefresh(lView);

    // 刷新透過 ViewContainerRef.createEmbeddedView() 添加的視圖
    refreshEmbeddedViews(lView);

    // 內容查詢結果必須在內容 hooks 呼叫之前刷新。
    if (tView.contentQueries !== null) {
      refreshContentQueries(tView, lView);
    }

    // 執行內容 hooks (AfterContentInit, AfterContentChecked)
    const contentCheckHooks = tView.contentCheckHooks;
    if (contentCheckHooks !== null) {
      executeCheckHooks(lView, contentCheckHooks);
    }

    // 執行透過 @HostBinding() 添加的邏輯
    processHostBindingOpCodes(tView, lView);

    // 刷新子元件視圖。
    const components = tView.components;
    if (components !== null) {
      refreshChildComponents(lView, components);
    }

    // 視圖查詢必須在刷新子元件後執行，因為此視圖中的 template 可能會插入到子元件中。
    // 如果視圖查詢在子元件刷新之前執行，則 template 可能尚未插入。
    const viewQuery = tView.viewQuery;
    if (viewQuery !== null) {
      executeViewQueryFn<T>(RenderFlags.Update, viewQuery, context);
    }

    // 執行視圖 hooks (AfterViewInit, AfterViewChecked)
    const viewCheckHooks = tView.viewCheckHooks;
    if (viewCheckHooks !== null) {
      executeCheckHooks(lView, viewCheckHooks);
    }

    // 在檢查元件後重置 dirty 狀態
    if (!isInCheckNoChangesPass) {
      lView[FLAGS] &= ~(LViewFlags.Dirty | LViewFlags.FirstLViewPass);
    }

    // 這一個很棘手 :) 需要它自己的章節，我們稍後會探討它
    if (lView[FLAGS] & LViewFlags.RefreshTransplantedView) {
      lView[FLAGS] &= ~LViewFlags.RefreshTransplantedView;
      updateTransplantedViewCount(lView[PARENT] as LContainer, -1);
    }

  } finally {
    leaveView();
  }
}
```

我們將在「內部渲染引擎」部分詳細了解所有這些操作。

現在，讓我們瀏覽一下從我上面顯示的函式推斷出的 ****在變更偵測期間執行的核心操作****。以下是這些操作的清單，****按照指定的順序排列：****

1.  以更新模式為目前視圖執行 template 函式
    – 檢查並[更新](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L154)子元件/指令實例上的 input 屬性
    – 如果綁定已變更，則在子元件上執行 hooks `ngOnInit`、[`ngDoCheck`](https://angular.love/if-you-think-ngdocheck-means-your-component-is-being-checked-read-this-article/) 和 `ngOnChanges`
    – 如果目前視圖元件實例上的屬性已變更，則[更新](https://hackernoon.com/the-mechanics-of-dom-updates-in-angular-3b2970d5c03d)目前視圖的 DOM 插值
2.  如果尚未在上一步中執行，則執行 executeCheckHooks
    – 如果綁定已變更，則在子元件上[呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L202) `OnChanges` [生命週期 hook](https://angular.love/complete-guide-angular-lifecycle-hooks/)
    – 在子元件上[呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L202) `ngDoCheck`（`OnInit` 僅在第一次檢查期間呼叫）
3.  markTransplantedViewsForRefresh
    – 尋找需要沿 `Lview` 鏈刷新的 transplant 視圖
4.  refreshEmbeddedViews
    – 為透過 ViewContainerRef API 建立的視圖[執行變更偵測](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/view.ts#L327)（大多重複此清單中的步驟）
5.  refreshContentQueries
    – [更新](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/query.ts#L91)子視圖元件實例上的 `ContentChildren` 查詢清單
6.  執行 Content CheckHooks
    – 在子元件實例上[呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L503) `AfterContentChecked` 生命週期 hooks（`AfterContentInit` 僅在第一次檢查期間呼叫）
7.  processHostBindingOpCodes
    – 檢查並更新透過元件類別內部的 `@HostBinding()` 語法添加到主機 DOM 元素上的 DOM 屬性
8.  refreshChildComponents
    – 為目前元件 template 中引用的子元件[執行變更偵測](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/view.ts#L327)。如果 `OnPush` 元件不髒，則跳過它們
9.  executeViewQueryFn
    – [更新](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/query.ts#L91)目前視圖元件實例上的 `ViewChildren` 查詢清單
10. 執行 View CheckHooks (AfterViewInit, AfterViewChecked)
    – 在子元件實例上[呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L503) `AfterViewChecked` 生命週期 hooks（`AfterViewInit` 僅在第一次檢查期間呼叫）

觀察
------------

根據上面列出的操作，有幾件事需要強調。

****目前視圖的 Change detection 負責啟動子視圖的 Change detection 。**** 這遵循 `refreshChildComponents` 操作（上面清單中的第 8 項）。對於每個子元件，Angular 執行 [refreshComponent](https://github.com/angular/angular/blob/02f3d12a0dc2c1b6f5ae06fff019058036fa5edc/packages/core/src/render3/instructions/shared.ts#L1687) 函式：

```
function refreshComponent(hostLView, componentHostIdx) {
  const componentView = getComponentLViewByIndex(componentHostIdx, hostLView);
  // 僅應刷新附加的 CheckAlways 元件
  // 或 OnPush 和 dirty 的元件
  if (viewAttachedToChangeDetector(componentView)) {
    const tView = componentView[TVIEW];
    if (componentView[FLAGS] & (LViewFlags.CheckAlways | LViewFlags.Dirty)) {
      refreshView(tView, componentView, tView.template, componentView[CONTEXT]);
    } else if (componentView[TRANSPLANTED_VIEWS_TO_REFRESH] > 0) {
      // 僅應刷新附加的 CheckAlways 元件
      // 或 OnPush 和 dirty 的元件
      refreshContainsDirtyView(componentView);
    }
  }
}
```

有一個條件定義了是否會檢查元件：

```
 if (viewAttachedToChangeDetector(componentView)) { ... }
 if (componentView[FLAGS] & (LViewFlags.CheckAlways | LViewFlags.Dirty)) {...}
```

主要條件是元件的 [changeDetectorRef](https://github.com/angular/angular/blob/c14c701775c900ce9ac8781c08fc76da067910c5/packages/core/src/change_detection/change_detector_ref.ts#L63) 必須附加到元件樹。如果未附加，則不會檢查元件本身及其子元件或包含的 transplant 視圖。

如果主要條件成立，則如果元件不是 `OnPush`，或者如果它是 `OnPush` 且是 dirty，則會檢查該元件。在 `refreshView` 函式的末尾有一個邏輯，用於重置 [`OnPush` 元件](https://angular.love/deep-dive-into-the-onpush-change-detection-strategy-in-angular/)上的 dirty 標誌：

```
// 在檢查元件後重置 dirty 狀態
if (!isInCheckNoChangesPass) {
  lView[FLAGS] &= ~(LViewFlags.Dirty | LViewFlags.FirstLViewPass);
}
```

最後，如果元件包含 transplant 視圖，也會檢查它們：

```
if (componentView[TRANSPLANTED_VIEWS_TO_REFRESH] > 0) {
  // 僅應刷新附加的 CheckAlways 元件或 OnPush 和 dirty 的元件
  refreshContainsDirtyView(componentView);
}
```

Template 函式

Angular 在變更偵測期間首先執行的 `executeTemplate` 函式的工作是執行元件定義中的 template 函式。此 template 函式是由編譯器為每個元件產生的。對於 `A` 元件：

```
@Component({
  selector: 'a-cmp',
  template: `<b-cmp [b]="1"></b-cmp> {{updateTemplate()}}`,
})
export class A {
  ngDoCheck() {
    console.log('A: ngDoCheck');
  }
  ngAfterContentChecked() {
    console.log('A: ngAfterContentChecked');
  }
  ngAfterViewChecked() {
    console.log('A: ngAfterViewChecked');
  }
  updateTemplate() {
    console.log('A: updateTemplate');
  }
}
```

其定義如下所示：

```
import {
  ɵɵdefineComponent as defineComponent,
  ɵɵelement as element,
  ɵɵtext as text,
  ɵɵproperty as property,
  ɵɵadvance as advance,
  ɵɵtextInterpolate1 as textInterpolate1
} from '@angular/core';

export class A {}
export class B {}

A.ɵfac = function A_Factory(t) { return new (t || A)(); };
A.ɵcmp = defineComponent({
    type: A,
    selectors: [["a-cmp"]],
    decls: 2,
    vars: 2,
    consts: [[3, "b"]],
    template: function A_Template(rf, ctx) {
      if (rf & 1) {
        element(0, "b-cmp", 0);
        text(1);
      }
      if (rf & 2) {
        property("b", 1);
        advance(1);
        textInterpolate1(" ", ctx.updateTemplate(), "");
      }
    },
    dependencies: function() { return [B]; },
    encapsulation: 2
  }
);
```

> _所有從 import 匯入的函式都使用前綴_ `__ɵɵ__` _匯出，以將它們識別為私有的。_

此 template 可以包含[各種指令](https://github.com/angular/angular/blob/40c138c13d17b638908999faafc9eb4cca0202fb/packages/core/src/render3/instructions/all.ts)。在我們的例子中，它包含在初始化階段執行的創建指令 `element` 和 `text`，以及在變更偵測階段執行的 `property`、`advance` 和 `textInterpolate1`：

```
template: function A_Template(rf, ctx) {
  if (rf & 1) {
    element(0, "b-cmp", 0);
    text(1);
  }
  if (rf & 2) {
    property("b", 1);
    advance(1);
    textInterpolate1(" ", ctx.updateTemplate(), "");
  }
}
```

### 生命週期 hooks

重要的是要了解，****大多數生命週期 hooks 在 Angular 為目前元件執行變更偵測時在子元件上呼叫。**** 只有 `ngAfterViewChecked` hook 的行為略有不同。

如果您有以下元件階層：`A -> B -> C`，以下是 hooks 呼叫和綁定更新的順序：

```
進入視圖：A
    B: updateBinding
    B: ngOnChanges
    B: ngDoCheck
        A: updateTemplate
    B: ngAfterContentChecked
    進入視圖：B
        С: updateBinding
        C: ngOnChanges
        С: ngDoCheck
            B: updateTemplate
        С: ngAfterContentChecked
        進入視圖：C
            С: updateTemplate
            С: ngAfterViewChecked
    B: ngAfterViewChecked
A: ngAfterViewChecked
```

目前就這樣。****我會不斷地向課程添加內容****，包括像我上面寫的免費材料。按一下[這裡](https://angular.love/en/change-detection-big-picture-overview)以探索課程或閱讀文章「[深入 Angular 課程的早鳥優惠](https://medium.com/angular-in-depth/early-bird-option-for-the-most-in-depth-angular-course-29a88a2534cd)」，我在其中更詳細地討論了課程內容和目標受眾。
