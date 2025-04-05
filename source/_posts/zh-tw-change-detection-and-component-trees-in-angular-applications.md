---
title: Change detection and component trees in Angular applications
date: 2025-02-17 16:26:07
tags:
---

原文：<https://angular.love/change-detection-and-component-trees-in-angular-applications>

# [Angular 應用程式中的變更檢測和元件樹]

> **_本文摘錄自我的_** [**_Angular Deep Dive_**](https://angular.love/en/change-detection-big-picture-overview) **_課程_**

在 Web 應用程式中，透過基於元件的方法，組合是通過在模板中包含子`Component`來實現的。因此，我們可以將 Angular 應用程式視為`Component`樹。然而，在底層，對於`Component`，Angular 使用了一個稱為`View`的低階抽象概念。`View`是最小的元素分組，這些元素會一起被建立和銷毀。所有操作（如屬性檢查和 DOM 更新）都是在`View`上執行的，因此，更準確地說，Angular 是一個`View`的樹，而`Component`可以被描述為`View`的更高層次的概念。

<!-- more -->

`View`的結構由[LView](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L79)介面定義。`LView`儲存了在從模板調用指令時處理指令所需的所有資訊。每個`Component`和嵌入式`View`都有其對應的`LView`。我們可以將為`Component`建立的`View`稱為`Component views`，以區別於使用[template references](https://github.com/angular/angular/blob/5509e3512f9542896dbbc267752a5034be6df9f6/packages/core/src/linker/template_ref.ts#L39)（即`ng-template`元素）透過[ViewContainerRef](https://github.com/angular/angular/blob/a6d5fe202cafb419f3beb8d09711132124b6aa9a/packages/core/src/linker/view_container_ref.ts#L52)建立的嵌入式`View`。

`View`的層次結構是使用`LView`上的指定屬性來追蹤的：

```typescript
export const PARENT = 3;
export const NEXT = 4;
export const CHILD_HEAD = 13;
export const CHILD_TAIL = 14;
 
export interface LView {
  [CHILD_HEAD]: LView|LContainer|null;
  [CHILD_TAIL]: LView|LContainer|null;
  [PARENT]: LView|LContainer|null;
  [NEXT]: LView|LContainer|null;
}
```

為了遍歷`View`的樹，Angular 使用了這些[遍歷工具](https://github.com/angular/angular/blob/a98fa489fb498273bdfde2fa4598893d1d53582e/packages/core/src/render3/util/view_traversal_utils.ts)。

Angular 還實作了[TView](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L519)資料結構，該結構保存`LView`的靜態資料。此`TView`在給定類型的所有`LView`之間共享。這意味著特定`Component`的每個實例都有自己的`LView`實例，但它們都引用`TView`的相同實例。

我們需要知道的最後一點是，Angular 有一些不同的`View`類型[定義](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L492)如下：

```typescript
export const enum TViewType {
  Root = 0,
  Component = 1,
  Embedded = 2,
}
```

`Component`和`Embedded`視圖類型應該是不言自明的。`Root`視圖是一種特殊類型的`View`，Angular 使用它來引導頂層`Component`。它與`LView`結合使用，`LView`接受一個不屬於 Angular 的現有 DOM 節點，並將其包裝在`LView`中，以便可以將其他`Component`加載到其中。

`View`和`Component`之間存在直接關係 – 一個`View`與一個`Component`相關聯，反之亦然。`View`在[CONTEXT](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L36)屬性中保存對關聯`Component`類別實例的[參考](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L172)。所有操作（如屬性檢查和 DOM 更新）都在視圖上執行。

對於包含兩次`A` `Component`的模板，資料結構將如下所示：

![Image 22](https://wp.angular.love/wp-content/uploads/2024/07/content-image1-540.png)

### 變更檢測樹

在大多數應用程式中，您有一個主要的`Component views`樹，它從您在`index.html`中引用的`Component`開始。還有其他透過[portals](https://material.angular.io/cdk/portal/overview)建立的`Root views`，主要用於對話方塊、工具提示等。這些是需要呈現的 UI 元素在主樹的層次結構之外，主要是出於樣式目的，例如，使它們不受`overflow:hidden`的影響。

Angular 將此類樹的頂層元素保存在[ApplicationRef](https://github.com/angular/angular/blob/ff84c7360334792a9a422bd0fa0353e0cf670525/packages/core/src/application_ref.ts#L720)的`_views`屬性中。這些樹被稱為變更檢測樹，因為當 Angular 全域執行變更檢測時，會遍歷它們。[tick](https://github.com/angular/angular/blob/ff84c7360334792a9a422bd0fa0353e0cf670525/packages/core/src/application_ref.ts#L1001)方法執行變更檢測，它會迭代`_views`中的每個樹，並通過調用`detectChanges`方法對每個`View`執行檢查：

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

您還可以看到`tick`在同一組`View`上執行`checkNoChanges`方法。

### 將動態視圖附加到 ApplicationRef

Angular 允許將`Component`呈現為 Angular 變更檢測樹之外的獨立 DOM 元素。但是由於這些`View`也需要被檢查，`ApplicationRef`實作了`attachView()`和`detachView()`方法來向變更檢測樹新增/移除獨立`View`。這有效地將這些`View`新增到在變更檢測期間遍歷的`_views`陣列中。

讓我們看看這個例子。我們有一個`M` `Component`，我們想要動態實例化它，然後呈現到主 Angular 樹之外的 DOM 中。以下是我們的做法：

```typescript
@Component({
  selector: 'l-cmp',
  template: 'L'
})
export class L {
  constructor(moduleRef: NgModuleRef<any>, appRef: ApplicationRef) {
    const factory = moduleRef.componentFactoryResolver.resolveComponentFactory(M);
 
    let newNode = document.createElement('div');
    newNode.id = 'placeholder';
    document.body.prepend(newNode);
 
    const ref = factory.create(moduleRef.injector, [], newNode);
    appRef.attachView(ref.hostView);
  }
}
 
@Component({
  selector: 'm-cmp',
  template: '{{title}}'
})
export class M {
  title = 'I am the component that was created dynamically';
}
```

如果我們檢查應用程式中的 DOM 結構，這就是我們將看到的：

![Image 23](https://wp.angular.love/wp-content/uploads/2024/07/content-image2-461.png)

如果我們現在看看`_views`屬性，這就是我們將看到的：

![Image 24](https://wp.angular.love/wp-content/uploads/2024/07/content-image3-423.png)

我們可以使用控制台來找出這些`RootViewRef`實例代表什麼：

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

圖表將更清楚地顯示這些關係：

![Image 25](https://wp.angular.love/wp-content/uploads/2024/07/content-image4-347.png)

### 引導多個根元件

可以像這樣引導多個根`Component`：

```typescript
@NgModule({
  declarations: [ AppComponent, AppRootAnother ],
  imports: [  BrowserModule ],
  bootstrap: [ AppComponent, AppRootAnother ]
})
export class AppModule {}
```

這將建立兩個`Root views`和相應的 html 標籤：

![Image 26](https://wp.angular.love/wp-content/uploads/2024/07/content-image5-305.png)

唯一要記住的是`index.html`應該包含兩個選擇器的標籤：

```typescript
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>LearnAngular</title>
</head>
<body>
  <app-root></app-root>
  <app-root-another></app-root-another>
</body>
</html>
```

通過此設置，Angular 將建立兩個獨立的變更檢測樹。它們將在`ApplicationRef._views`下註冊，當`ApplicationRef.tick()`函數被調用時，Angular 將對兩個樹執行變更檢測。這實際上類似於使用[attachView](https://github.com/angular/angular/blob/ff84c7360334792a9a422bd0fa0353e0cf670525/packages/core/src/application_ref.ts#L1032)。但是，它們仍將是單個`ApplicationRef`的一部分，因此它們將共享為`AppModule`定義的`Injector`。

有關您在上面閱讀的更深入的內容，[請查看課程](https://angular.love/en/change-detection-big-picture-overview)

![Image 27: Content image](https://wp.angular.love/wp-content/uploads/2024/07/content-image6-236.png)

如果您認為這裡缺少某些重要的內容，請在評論中告訴我！

![Image 28: Content image](https://wp.angular.love/wp-content/uploads/2024/07/content-image7-215.png)
