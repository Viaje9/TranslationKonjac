---
title: 執行變更偵測 – 元件樹
date: 2025-01-19 15:40:20
tags:
---

原文：<https://angular.love/running-change-detection-components-tree>

# 在 Angular 中執行變更偵測 - 元件樹 - 它是什麼？

## 元件樹 (Components tree)

在 Web 應用程式中，基於元件的方法通過在模板中包含子元件來實現組合。
因此，我們可以將 Angular 應用程式視為元件樹 (Components tree)。
然而，在底層，對於元件，Angular 使用一個稱為 `View` 的低階抽象。
`View` 是最小的元素分組，可以一起建立和銷毀。
所有操作（如屬性檢查和 DOM 更新）都在 `View` 上執行，
**因此更準確地說，Angular 是一個 `View` 的樹**，
而元件可以描述為 `View` 的更高層級概念。

<!-- more -->

`View` 的結構由 `LView` 介面定義。
[LView](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L79)
儲存了在模板中呼叫指令時處理指令所需的所有資訊。
每個元件和嵌入式 `View` 都有其對應的 `LView`。
我們可以將為元件建立的 `View` 稱為元件 `View` (Component view)，以便與通過
[ViewContainerRef](https://github.com/angular/angular/blob/a6d5fe202cafb419f3beb8d09711132124b6aa9a/packages/core/src/linker/view_container_ref.ts#L52) 使用 [template references](https://github.com/angular/angular/blob/5509e3512f9542896dbbc267752a5034be6df9f6/packages/core/src/linker/template_ref.ts#L39)
（即 `ng-template` 元素）建立的嵌入式 `View` (Embedded view) 區分開來。

`View` 的層次結構使用 `LView` 上的指定屬性進行追蹤：

```typescript
export const PARENT = 3;
export const NEXT = 4;
export const CHILD_HEAD = 13;
export const CHILD_TAIL = 14;

export interface LView {
  [CHILD_HEAD]: LView | LContainer | null;
  [CHILD_TAIL]: LView | LContainer | null;
  [PARENT]: LView | LContainer | null;
  [NEXT]: LView | LContainer | null;
}
```

為了遍歷 `View` 的樹，Angular 使用了這些
[traversal utils](https://github.com/angular/angular/blob/a98fa489fb498273bdfde2fa4598893d1d53582e/packages/core/src/render3/util/view_traversal_utils.ts)。

Angular 還實現了
[TView](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L519)
資料結構，用於保存 `LView` 的靜態資料。此 `TView` 在給定型別的所有 `LView` 之間共享。
這意味著特定元件的每個實例都有其自己的 `LView` 實例，
但它們都引用相同的 `TView` 實例。

我們需要知道的最後一點是，Angular 有幾種不同的 `View` 型別
[定義](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L492)
如下：

```typescript
export const enum TViewType {
  Root = 0,
  Component = 1,
  Embedded = 2,
}
```

`Component` 和 `Embedded` `View` 型別應該是不言自明的。
`Root` `View` 是一種特殊型別的 `View`，Angular 用於引導頂層元件。
它與 `LView` 結合使用，`LView` 接受一個不屬於 Angular 的現有 DOM 節點並將其封裝在 `LView` 中，
以便可以將其他元件載入到其中。

`View` 和元件之間存在直接關係 - 一個 `View` 與一個元件相關聯，反之亦然。
`View` 在 [CONTEXT](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L36) 屬性中保存對相關元件類別實例的[參考](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L172)。
所有操作（如屬性檢查和 DOM 更新）都在 `View` 上執行。

對於包含 `A` 元件兩次的模板，資料結構將如下所示：

![Image 18: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/1-10.png)

## 變更偵測樹 (Change detection tree)

在大多數應用程式中，您有一個主要的元件 `View` 樹，它從您在 `index.html` 中引用的元件開始。
還有其他通過 [portals](https://material.angular.io/cdk/portal/overview) 建立的 `Root` `View`，
主要用於模式對話框、工具提示等。
這些是需要呈現在主樹層次結構之外的 UI 元素，主要是出於樣式目的，
例如，使它們不受 `overflow:hidden` 的影響。

Angular 將此類樹的頂層元素保存在
[ApplicationRef](https://github.com/angular/angular/blob/ff84c7360334792a9a422bd0fa0353e0cf670525/packages/core/src/application_ref.ts#L720) 的 `_views` 屬性中。
這些樹被稱為變更偵測樹，因為當 Angular 全域執行變更偵測時，會遍歷這些樹。
執行變更偵測的 [tick](https://github.com/angular/angular/blob/ff84c7360334792a9a422bd0fa0353e0cf670525/packages/core/src/application_ref.ts#L1001)
方法會迭代 `_views` 中的每個樹，
並通過呼叫 `detectChanges` 方法對每個 `View` 執行檢查：

```typescript
@Injectable({ providedIn: 'root' })
export class ApplicationRef {
  tick(): void {
    try {
      this._runningTick = true;
      for (let view of this._views) {
        view.detectChanges();
      }
      if (typeof ngDevMode === 'undefined' || ngDevMode) {
        for (let view of this._views) {
          view.checkNoChanges();
        }
      }
    } catch (e) { ... } finally { ... }
  }
}
```

您還可以看到 `tick` 在同一組 `View` 上執行 `checkNoChanges` 方法。

## 將動態 `View` 附加到 `ApplicationRef`

Angular 允許將元件呈現在 Angular 變更偵測樹之外的獨立 DOM 元素中。
但是由於這些 `View` 也需要進行檢查，因此 `ApplicationRef` 實現了 `attachView()` 和 `detachView()` 方法
來向變更偵測樹中新增/移除獨立 `View`。
這實際上將這些 `View` 新增到變更偵測期間遍歷的 `_views` 陣列中。

讓我們看看這個例子。我們有一個 `M` 元件，我們想動態實例化它，
然後將其呈現在主 Angular 樹之外的 DOM 中。以下是我們的做法：

```typescript
@Component({
  selector: 'l-cmp',
  template: 'L',
})
export class L {
  constructor(moduleRef: NgModuleRef<any>, appRef: ApplicationRef) {
    const factory =
      moduleRef.componentFactoryResolver.resolveComponentFactory(M);

    let newNode = document.createElement('div');
    newNode.id = 'placeholder';
    document.body.prepend(newNode);

    const ref = factory.create(moduleRef.injector, [], newNode);
    appRef.attachView(ref.hostView);
  }
}

@Component({
  selector: 'm-cmp',
  template: '{{title}}',
})
export class M {
  title = 'I am the component that was created dynamically';
}
```

如果我們檢查應用程式中的 DOM 結構，我們將看到以下內容：

![Image 19: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/2-12.png)

如果我們現在檢視 `_views` 屬性，我們將看到以下內容：

![Image 20: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/3-9.png)

我們可以使用控制台來找出這些 `RootViewRef` 實例代表什麼：

```typescript
const TVIEW = 1;
const CONTEXT = 8;
const CHILD_HEAD = 13;

const view_1 = appRef._views[0];
const view_2 = appRef._views[1];

view_1._lView[TVIEW].type // 0 - HostView
view_1._lView[CONTEXT].constructor.name // M

view_1._lView[CHILD_HEAD][TVIEW].type // 0 - HostView
view_1._lView[CHILD_HEAD][CONTEXT].constructor.name // M

view_2._lView[CONTEXT].constructor.name // AppComponent (RootView)
view_2._lView[TVIEW].type // 0 - HostView

view_2._lView[CHILD_HEAD][CONTEXT].constructor.name // AppComponent (ComponentView)
view_2._lView[CHILD_HEAD][TVIEW].type // 1 - ComponentView

view_2._lView[CHILD_HEAD][CHILD_HEAD][CONTEXT].constructor.name // L
```

該圖表將更清楚地顯示這些關係：

![Image 21: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/4-10.png)

## 引導多個根元件

可以像這樣引導多個根元件：

```typescript
@NgModule({
  declarations: [AppComponent, AppRootAnother],
  imports: [BrowserModule],
  bootstrap: [AppComponent, AppRootAnother],
})
export class AppModule {}
```

這將建立兩個 `Root` `View` 和相應的 html 標籤：

![Image 22: Image alt](https://wp.angular.love/wp-content/uploads/2024/08/5-9.png)

唯一要記住的是 `index.html` 應該包含兩個選擇器的標籤：

```HTML
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>LearnAngular</title>
  </head>
  <body>
    <app-root></app-root>
    <app-root-another></app-root-another>
  </body>
</html>
```

通過此設定，Angular 將建立兩個獨立的變更偵測樹。
它們將在 `ApplicationRef._views` 下註冊，當 `ApplicationRef.tick()` 函數被呼叫時，
Angular 將對兩個樹執行變更偵測。這實際上類似於使用
[attachView](https://github.com/angular/angular/blob/ff84c7360334792a9a422bd0fa0353e0cf670525/packages/core/src/application_ref.ts#L1032)。
但是，它們仍然是單個 `ApplicationRef` 的一部分，因此它們將共享為 `AppModule` 定義的注入器。


