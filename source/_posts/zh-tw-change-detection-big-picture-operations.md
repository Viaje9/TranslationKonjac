---
title: Change Detection Big Picture – Operations
date: 2025-01-04 21:20:43
tags:
---

原文：[https://angular.love/change-detection-big-picture-operations](https://angular.love/change-detection-big-picture-operations)

# Change Detection 大觀 - 運算

運算
----------

當 Angular 為特定的 component（視圖）執行 change detection 時，它會執行一些運算。
這些運算有時被稱為副作用，如同計算邏輯的副作用：

> 在電腦科學中，如果一個運算、函式或表達式修改了其局部環境之外的某些狀態變數值，則稱其具有副作用，也就是說，如果它除了將值返回給運算的調用者之外，還有任何可觀察到的影響。

在 Angular 中，change detection 的主要副作用是將應用程式狀態渲染到目標平台上。
大多數情況下，目標平台是瀏覽器，應用程式狀態的形式是 component 屬性，而渲染涉及更新 DOM。

在檢查 component 時，Angular 會執行其他一些運算。
我們可以透過探索 [`refreshView`](https://github.com/angular/angular/blob/02f3d12a0dc2c1b6f5ae06fff019058036fa5edc/packages/core/src/render3/instructions/shared.ts#L357) 函式來識別它們。
經過簡化的函式主體，以及我的解釋性註解如下：

<!-- more -->

```typescript
function refreshView(tView, lView, templateFn, context) {
  enterView(lView);

  try {
    if (templateFn !== null) {
      // 更新子 component 的 input bindings
      // 執行 ngOnInit、ngOnChanges 和 ngDoCheck hooks
      // 更新當前 component 的 DOM
      executeTemplate(tView, lView, templateFn, RenderFlags.Update, context);
    }

    // 執行 ngOnInit、ngOnChanges 和 ngDoCheck hooks
    // 如果它們沒有從 template 函式中執行
    const preOrderCheckHooks = tView.preOrderCheckHooks;
    if (preOrderCheckHooks !== null) {
      executeCheckHooks(lView, preOrderCheckHooks, null);
    }

    // 首先，將在此 lView 中宣告的 transplanted views 標記為在其插入點需要刷新。
    // 這是為了避免在 `LView` 中定義 template，但其宣告出現在插入 component 之後的情況。
    markTransplantedViewsForRefresh(lView);

    // 刷新透過 ViewContainerRef.createEmbeddedView() 加入的視圖
    refreshEmbeddedViews(lView);

    // Content query 結果必須在呼叫 content hooks 之前刷新。
    if (tView.contentQueries !== null) {
      refreshContentQueries(tView, lView);
    }

    // 執行 content hooks (AfterContentInit, AfterContentChecked)
    const contentCheckHooks = tView.contentCheckHooks;
    if (contentCheckHooks !== null) {
      executeCheckHooks(lView, contentCheckHooks);
    }

    // 執行透過 @HostBinding() 加入的邏輯
    processHostBindingOpCodes(tView, lView);

    // 刷新子 component 視圖。
    const components = tView.components;
    if (components !== null) {
      refreshChildComponents(lView, components);
    }

    // View query 必須在刷新子 component 之後執行，因為此視圖中的 template 可以插入到子 component 中。
    // 如果 view query 在子 component 刷新之前執行，則 template 可能尚未插入。
    const viewQuery = tView.viewQuery;
    if (viewQuery !== null) {
      executeViewQueryFn<T>(RenderFlags.Update, viewQuery, context);
    }

    // 執行 view hooks (AfterViewInit, AfterViewChecked)
    const viewCheckHooks = tView.viewCheckHooks;
    if (viewCheckHooks !== null) {
      executeCheckHooks(lView, viewCheckHooks);
    }

    // 在檢查 component 後重置 dirty 狀態
    if (!isInCheckNoChangesPass) {
      lView[FLAGS] &= ~(LViewFlags.Dirty | LViewFlags.FirstLViewPass);
    }

    // 這一個很棘手 :) 需要自己的章節，我們稍後會探討它
    if (lView[FLAGS] & LViewFlags.RefreshTransplantedView) {
      lView[FLAGS] &= ~LViewFlags.RefreshTransplantedView;
      updateTransplantedViewCount(lView[PARENT] as LContainer, -1);
    }

  } finally {
    leaveView();
  }
}
```

我們將在「渲染引擎內部」章節中詳細介紹所有這些運算。

現在，讓我們回顧一下從我上面展示的函式中推斷出的**change detection 期間執行的核心運算**。
以下是按指定順序排列的此類運算列表：

1.  在當前視圖的更新模式中執行 template 函式
    1.  檢查並[更新](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L154) 子 component/directive 實例上的 input 屬性
    2.  如果 bindings 發生變化，則在子 [component](https://angular.love/component-initialization-without-ngoninit-with-async-pipes-for-observables-and-ngonchanges) 上執行 `ngOnInit`、`ngDoCheck` 和 `ngOnChanges` hooks
    3.  如果目前視圖 component 實例上的屬性發生變化，則[更新](https://hackernoon.com/the-mechanics-of-dom-updates-in-angular-3b2970d5c03d)目前視圖的 DOM 插值
2.  如果沒有在上一步中運行，則執行 executeCheckHooks
    1.  如果 bindings 發生變化，則在子 component 上 [呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L202) `OnChanges` 生命週期 hook
    2.  在子 component 上 [呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L202) [`ngDoCheck`](https://angular.love/if-you-think-ngdocheck-means-your-component-is-being-checked-read-this-article/) （OnInit 僅在第一次檢查期間呼叫）
3.  markTransplantedViewsForRefresh
    1.  尋找需要向下刷新 LView 鏈的 transplanted views，並將其標記為 dirty
4.  refreshEmbeddedViews
    1.  為透過 `ViewContainerRef` APIs 建立的視圖[運行 change detection](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/view.ts#L327) （主要重複此列表中的步驟）
5.  refreshContentQueries
    1.  [更新](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/query.ts#L91) 子視圖 component 實例上的 `ContentChildren` query 列表
6.  執行 Content CheckHooks (`AfterContentInit`、`AfterContentChecked`)
    1.  [呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L503) 子 component 實例上的 `AfterContentChecked` 生命周期 hooks（`AfterContentInit` 僅在第一次檢查期間呼叫）
7.  processHostBindingOpCodes
    1.  檢查並更新透過 component 類別內部的 `@HostBinding()` 語法新增的 host DOM 元素上的 DOM 屬性
8.  refreshChildComponents
    1.  為當前 component 的 template 中引用的子 component [運行 change detection](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/view.ts#L327)。如果 `OnPush` component 不 dirty，則會跳過它們
9.  executeViewQueryFn
    1.  [更新](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/query.ts#L91) 當前視圖 component 實例上的 `ViewChildren` query 列表
10. 執行 View CheckHooks (`AfterViewInit`、`AfterViewChecked`)
    1.  [呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L503) 子 component 實例上的 `AfterViewChecked` 生命周期 hooks（`AfterViewInit` 僅在第一次檢查期間呼叫）

這是說明上述運算的圖表：

![Image 13: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/2-6.png)

觀察
------------

根據上面列出的運算，有幾件事需要強調。第一個也是最有趣的是，**當前視圖的 change detection 負責啟動子視圖的 change detection。**

**[檢查什麼是 ngTemplateOutlet](https://angular.love/ngtemplateoutlet-the-secret-to-customisation/)！**

這遵循 `refreshChildComponents` 運算（上面列表中的 #8）。
對於每個子 component，Angular 執行
[`refreshComponent`](https://github.com/angular/angular/blob/02f3d12a0dc2c1b6f5ae06fff019058036fa5edc/packages/core/src/render3/instructions/shared.ts#L1687) 函式：

```typescript
function refreshComponent(hostLView, componentHostIdx) {
  const componentView = getComponentLViewByIndex(componentHostIdx, hostLView);
  // 只有已附加且為 CheckAlways 或 OnPush 且為 dirty 的 component 才應刷新
  if (viewAttachedToChangeDetector(componentView)) {
    const tView = componentView[TVIEW];
    if (componentView[FLAGS] & (LViewFlags.CheckAlways | LViewFlags.Dirty)) {
      refreshView(tView, componentView, tView.template, componentView[CONTEXT]);
    } else if (componentView[TRANSPLANTED_VIEWS_TO_REFRESH] > 0) {
      // 只有已附加且為 CheckAlways 或 OnPush 且為 dirty 的 component 才應刷新
      refreshContainsDirtyView(componentView);
    }
  }
}
```

有一個條件定義是否會檢查 component：

```typescript
  if (viewAttachedToChangeDetector(componentView)) { ... }
  if (componentView[FLAGS] & (LViewFlags.CheckAlways | LViewFlags.Dirty)) { ... }
```

主要條件是 component 的 `changeDetectorRef` 必須附加到 component 樹。
如果它未附加，則既不會檢查 component 本身，也不會檢查其子項或包含的 transplanted views。

如果主要條件成立，則如果 component 不是 `OnPush`，或者如果它是 `OnPush` 且為 dirty，則將檢查該 component。
在 `refreshView` 函式的末尾有一個邏輯可以重置 <a href="https://wp.angular.love/deep-dive-into-the-onpush-change-detection-strategy-in-angular/">OnPush</a> component 上的 dirty 標誌：

```typescript
// 在檢查 component 後重置 dirty 狀態
if (!isInCheckNoChangesPass) {
  lView[FLAGS] &= ~(LViewFlags.Dirty | LViewFlags.FirstLViewPass);
}
```

最後，如果 component 包含 transplanted views，它們也會被檢查：

```typescript
if (componentView[TRANSPLANTED_VIEWS_TO_REFRESH] > 0) {
  // 只有已附加且為 CheckAlways 或 OnPush 且為 dirty 的 component 才應刷新
  refreshContainsDirtyView(componentView);
}
```

### Template 函式

Angular 在 change detection 期間首先運行的 `executeTemplate` 函式的工作是執行來自 component 定義的 template 函式。
這個 template 函式是由編譯器為每個 component 生成的。
對於 `A` component：

```typescript
@Component({
  selector: 'a-cmp',
  template: `<b-cmp [b]="1"></b-cmp> {{ updateTemplate() }}`,
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

定義如下：

```typescript
import {
  ɵɵdefineComponent as defineComponent,
  ɵɵelement as element,
  ɵɵtext as text,
  ɵɵproperty as property,
  ɵɵadvance as advance,
  ɵɵtextInterpolate as textInterpolate
} from '@angular/core';

export class A {}

A.ɵfac = function A_Factory(t) { return new (t || A)(); };
A.ɵcmp = defineComponent({
    type: A,
    selectors: [["a-cmp"]],
    template: function A_Template(rf, ctx) {
      if (rf & 1) {
        element(0, "b-cmp", 0);
        text(1);
      }
      if (rf & 2) {
        property("b", 1);
        advance(1);
        textInterpolate(" ", ctx.updateTemplate(), "");
      }
    },
    ...
  }
);
```

所有導入的函式都以 `ɵɵ` 前綴匯出，將它們標識為私有。

此 template 可以包含各種指令。在我們的例子中，它包含在初始化階段執行的創建指令 `element` 和 `text`，以及在 change detection 階段執行的 `property`、`advance` 和 `textInterpolate`：

```typescript
function A_Template(rf, ctx) {
  if (rf & 1) {
    element(0, 'b-cmp', 0);
    text(1);
  }
  if (rf & 2) {
    property('b', 1);
    advance(1);
    textInterpolate(' ', ctx.updateTemplate(), '');
  }
}
```

這些是在 change detection 期間按順序執行的指令。

### 生命周期 hook

Angular component 可以使用生命週期 hook 方法來利用 component 或 directive 生命週期中的關鍵事件。
當 Angular 創建 component 類別並渲染 component 視圖及其子視圖時，生命週期就會開始。
當 Angular 檢查資料綁定的屬性何時發生變化時，生命週期會繼續進行 change detection，並根據需要更新視圖和 component 實例。
當 Angular 銷毀 component 實例並從 DOM 中移除其渲染的 template 時，生命週期會結束。

**重要的是要理解，生命週期 hook 方法是在 change detection 期間執行的。**

以下是可用的生命週期方法的完整列表：

*   onChanges
*   onInit
*   doCheck
*   afterContentInit
*   afterContentChecked
*   afterViewInit
*   afterViewChecked
*   ngOnDestroy

其中 `onInit`、`afterContentInit` 和 `afterViewInit` 僅在初始 change detection 運行（第一次運行）期間執行。
`ngOnDestroy` hook 僅在銷毀 component 之前執行一次。
其餘 4 個方法在每個 change detection 週期執行：

*   onChanges
*   doCheck
*   afterContentChecked
*   afterViewChecked

您也可以將 component 的 `constructor` 視為一種生命週期事件，它在創建 component 實例時執行。
然而，從 component 初始化階段的角度來看，建構子和生命週期方法之間存在巨大差異。

Angular 啟動過程由兩個主要階段組成：

*   建構 component 樹
*   運行 change detection

並且在 Angular 建構 component 樹時會呼叫 component 的建構子。包括 `ngOnInit` 在內的所有生命週期 hooks 都會作為後續 change detection 階段的一部分被呼叫。

同樣重要的是要理解，**當 Angular 為當前 component 運行 change detection 時，大多數生命週期 hooks 會在子 component 上呼叫。**
只有 `ngAfterViewChecked` hook 的行為略有不同。

我們可以將此 component 的結構用於定義一個由 3 個 component 組成的層次結構 `[A -> B -> C]`，以記錄方法呼叫的順序：

```typescript
@Component({
  selector: 'a-cmp',
  template: `<b-cmp [b]="1"></b-cmp> {{ updateTemplate() }}`,
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

@Component({
  selector: 'b-cmp',
  template: `<c-cmp [b]="1"></c-cmp> {{updateTemplate()}}`,
})
export class B {}

@Component({
  selector: 'c-cmp',
  template: `{{ updateTemplate() }}`,
})
export class C {}
```

這是 hooks 呼叫和綁定更新的順序：

```
Entering view: A
    B: updateBinding
    B: ngOnChanges
    B: ngDoCheck
        A: updateTemplate
    B: ngAfterContentChecked
    Entering view: B
        С: updateBinding
        C: ngOnChanges
        С: ngDoCheck
            B: updateTemplate
        С: ngAfterContentChecked
        Entering view: C
            С: updateTemplate
            С: ngAfterViewChecked
    B: ngAfterViewChecked
A: ngAfterViewChecked
```

在此處[檢查](https://stackblitz.com/edit/angular-ivy-jkvjgz?file=src%2Fapp%2Fcomponents.ts,src%2Fapp%2Fapp.module.ts,src%2Fapp%2Fapp.component.ts)。

如您所見，當 Angular 正在為 `A` component 運行檢查時，生命週期方法 `ngOnChanges` 和 `ngDoCheck` 在 `B` component 上執行。這有點出乎意料，但完全合乎邏輯。當 Angular 為 `A` component 運行 change detection 時，它會在 template 內部運行指令，這會更新子 component `B` 上的 bindings。因此，一旦更新了 B component 上的屬性，透過在 component 上運行 `ngOnChanges` 來通知 `B` component 關於這一點是有意義的。

### View 和 Content Queries

View 和 Content query 是我們在運行時取得 component template 中元素的方法。
通常，評估 query 的結果可在 `ngAfterViewChecked` 或 `ngAfterContentChecked` 生命周期方法中取得。
如果我們查看運算順序，我們可以看出原因：

```typescript
// Content query 結果必須在呼叫 content hooks 之前刷新。
if (tView.contentQueries !== null) {
  refreshContentQueries(tView, lView);
}

// 執行 content hooks (AfterContentInit, AfterContentChecked)
const contentCheckHooks = tView.contentCheckHooks;
if (contentCheckHooks !== null) {
  executeCheckHooks(lView, contentCheckHooks);
}

...

// View query 結果必須在呼叫 content hooks 之前刷新。
const viewQuery = tView.viewQuery;
if (viewQuery !== null) {
  executeViewQueryFn<T>(RenderFlags.Update, viewQuery, context);
}

// 執行 view hooks (AfterViewInit, AfterViewChecked)
const viewCheckHooks = tView.viewCheckHooks;
if (viewCheckHooks !== null) {
  executeCheckHooks(lView, viewCheckHooks);
}
```

這些是指上面列表中 Content Queries 的運算 `#5` 和 `#6`，以及 View Queries 的運算 `#9` 和 `#10`。

Angular 透過執行透過 component 定義定義的函式來更新 query 結果。
對於如下的 component 定義

```typescript
@Component({
  selector: 'c-cmp',
  template: ``,
})
export class C {
  @ViewChild('ctrl') viewChild: any;
  @ContentChild('ctrl') contentChild: any;

  title = 'c-comp is here';
}
```

更新 query 的函式如下：

```typescript
C.ɵcmp = defineComponent({
  type: C,
  selectors: [["c-cmp"]],
  contentQueries: function C_ContentQueries(rf, ctx, dirIndex) {
    if (rf & 1) {
      contentQuery(dirIndex, _c0, 5);
    }
    if (rf & 2) {
      let _t;
      queryRefresh(_t = ["loadQuery"]()) && (ctx.contentChild = _t.first);
    }
  },
  viewQuery: function C_Query(rf, ctx) {
    if (rf & 1) {
      viewQuery(_c0, 5);
    }
    if (rf & 2) {
      let _t;
      queryRefresh(_t = ["loadQuery"]()) && (ctx.viewChild = _t.first);
    }
  },
  template: function C_Template(rf, ctx) {},
  ...
});
```

### Embedded Views

Angular 提供了一種機制，透過使用視圖容器在 component 的 template 中實現動態行為。
您可以使用 component template 內的 `ng-container` 標籤建立視圖容器，並使用 `@ViewChild` query 存取它。
這些容器提供了一種實例化 template 程式碼的方法，允許它被重複使用並根據需要輕鬆修改。

視圖容器提供 API 來建立、操作和移除動態視圖。
我將它們稱為動態視圖，而不是框架為 template 中找到的靜態 component 建立的靜態視圖。
Angular 不使用視圖容器來處理靜態視圖，而是在特定於子 component 的節點內保留對子視圖的引用。
我們將在「渲染引擎內部」章節中更多地討論視圖類型的差異。

嵌入式視圖是透過使用 `viewContainerRef.createEmbeddedView()` 方法實例化 `TemplateRef` 從 template 建立的。
視圖容器也可以包含透過使用 `createComponent()` 方法實例化 component 建立的 host 視圖。
視圖容器實例可以包含其他視圖容器，從而建立視圖層次結構。
所有結構指令（如 `ngIf` 和 `ngFor`）都使用視圖容器從指令的 template 建立動態視圖。

這些嵌入式視圖在列表中的步驟 `#4` 期間檢查：

```typescript
// 刷新透過 ViewContainerRef.createEmbeddedView() 加入的視圖
refreshEmbeddedViews(lView);
```

對於如下的 template：

```html
<span>My component</span>
<ng-container
  [ngTemplateOutlet]="template"
  [ngTemplateOutletContext]="{$implicit: greeting}"
>
</ng-container>
<a-comp></a-comp>
<ng-template>
  <span>I am an embedded view</span>
  <ng-template></ng-template
></ng-template>
```

`LView` 內部的視圖節點可以表示為這樣：

![Image 14: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/1-6.png)

**有一種稱為 transplanted view 的嵌入式視圖的特殊情況。**

transplanted view 是一個嵌入式視圖，其 template 在託管嵌入式視圖的 component 的 template 之外宣告。
使用 `<ng-template>` 宣告 template 的 component 與使用視圖容器插入使用此 template 建立的嵌入式視圖的 component 不同。

在以下範例中，template 在 `AppComp` 內宣告，但在 `LibComp` component 內渲染，因此使用此 template 建立的嵌入式視圖被視為 transplanted：

```typescript
@Component({
  selector: 'lib-comp',
  template: `
    LibComp: {{ greeting }}!
    <ng-container
      [ngTemplateOutlet]="template"
      [ngTemplateOutletContext]="{ $implicit: greeting }"
    >
    </ng-container>
  `,
})
class LibComp {
  @Input()
  template: TemplateRef;
  greeting: string = 'Hello';
}

@Component({
  template: `
    AppComp: {{ name }}!
    <ng-template #myTmpl let-greeting> {{ greeting }} {{ name }}! </ng-template>
    <lib-comp [template]="myTmpl"></lib-comp>
  `,
})
class AppComp {
  name: string = 'world';
}
```

運算 #3 `markTransplantedViewsForRefresh` 處理此類視圖的更新。
