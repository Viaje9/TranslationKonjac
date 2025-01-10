---
title: Everything you need to know about change detection in Angular
date: 2025-01-04 15:50:14
tags:
---

原文：<https://angular.love/everything-you-need-to-know-about-change-detection-in-angular>

# 關於 Angular 中 change detection 你需要知道的一切


如果你和我一樣，想要全面了解 Angular 中的變更偵測機制，你必須探索原始碼，因為網路上沒有太多可用的資訊。大多數文章都提到每個 component 都有自己的變更偵測器，負責檢查 component，但它們並未深入探討，大多只關注 immutables 和變更偵測策略的使用案例。本文為你提供了解 **為什麼** 使用 immutables 的案例有效，以及變更偵測策略 **如何** 影響檢查所需的資訊。此外，你從本文學到的內容將使你能夠自行提出各種效能優化的情境。

> 本文相當深入。如需了解變更偵測的簡介，請參閱 [Angular 中變更偵測的溫和介紹](https://angular.love/en/a-gentle-introduction-into-change-detection-in-angular/)

本文包含兩個部分。第一部分技術性較強，包含許多指向原始碼的連結。它詳細解釋了變更偵測機制在底層的運作方式。其內容基於最新的 Angular 版本 — 4.0.1。此版本中變更偵測機制的底層實作方式與之前的 2.4.1 不同。如有興趣，你可以閱讀 [此 stackoverflow 回答](http://stackoverflow.com/a/42807309/2545680) 中關於它在之前的版本是如何運作的資訊。

第二部分展示了如何在應用程式中使用變更偵測，其內容適用於較早的 2.4.1 和最新的 4.0.1 版本的 Angular，因為公共 API 沒有改變。

<!-- more -->

View 作為核心概念
----------------------

在教學課程中提到，Angular 應用程式是一個 component 樹。然而，在底層，Angular 使用名為 [view](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/types.ts#L301) 的低階抽象概念**。**view 和 component 之間存在直接關係 — 一個 view 與一個 component 相關聯，反之亦然。view 在 `component` 屬性中保留了對相關 component 類別實例的[參考](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/types.ts#L309)。所有操作（如屬性檢查和 DOM 更新）都在 view 上執行，因此，更技術性地來說，Angular 是一個 view 樹，而 component 可以被描述為 view 的高階概念。以下是你可以從[原始碼](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/linker/view_ref.ts#L31)中讀到的關於 view 的資訊：

> View 是應用程式 UI 的基本建構區塊。它是最小的一組同時建立和銷毀的 Element。
> View 中 element 的屬性可以變更，但 View 中 element 的結構（數量和順序）不能變更。只能透過 ViewContainerRef 插入、移動或移除巢狀 View 來變更 Element 的結構。每個 View 可以包含許多 View Container。

在本文中，我將交替使用 component view 和 component 的概念。

> 這裡需要注意的是，網路上所有關於變更偵測的文章和 StackOverflow 上的解答都將我在此描述的 View 稱為變更偵測器物件或 ChangeDetectorRef。事實上，沒有單獨的物件用於變更偵測，而 View 才是變更偵測運行的對象。

每個 view 都有一個透過 [nodes](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/types.ts#L316) 屬性連接到其子 view 的連結，因此可以對子 view 執行操作。

View 狀態
----------

每個 view 都有一個[狀態](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/types.ts#L317)，這個狀態扮演非常重要的角色，因為 Angular 會根據其值來決定是否為 view 和 ****其所有子 view**** 執行變更偵測，或跳過變更偵測。有許多[可能的狀態](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/types.ts#L325)，但在本文的上下文中，以下狀態是相關的：

1.  FirstCheck
2.  ChecksEnabled
3.  Errored
4.  Destroyed

如果 `ChecksEnabled` 為 `false`，或 view 處於 `Errored` 或 `Destroyed` 狀態，則會跳過 view 及其子 view 的變更偵測。預設情況下，所有 view 都會使用 `ChecksEnabled` 初始化，除非使用 `ChangeDetectionStrategy.OnPush`。稍後會詳細說明。這些狀態可以組合，例如，一個 view 可以同時設定 `FirstCheck` 和 `ChecksEnabled` 旗標。

Angular 有一堆高階概念來操作 view。我[在這裡](https://hackernoon.com/exploring-angular-dom-abstractions-80b3ebcfc02)寫了一些關於它們的文章。其中一個概念是 [ViewRef](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/refs.ts#L219)。它封裝了[底層 component view](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/refs.ts#L221)，並具有一個名為 [detectChanges](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/refs.ts#L239) 的方法。當非同步事件發生時，Angular 會在其最上層的 ViewRef 上[觸發變更偵測](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/application_ref.ts#L552.)，該 ViewRef 在為自身執行變更偵測後，****會為其子 view 執行變更偵測****。

這個 `viewRef` 就是你可以使用 `ChangeDetectorRef` token 注入到 component 建構函式中的內容：

```typescript
export class AppComponent {
    constructor(cd: ChangeDetectorRef) { ... }
```

從這個類別的定義中可以看出：

```typescript
export declare abstract class ChangeDetectorRef {
    abstract checkNoChanges(): void;
    abstract detach(): void;
    abstract detectChanges(): void;
    abstract markForCheck(): void;
    abstract reattach(): void;
}
export abstract class ViewRef extends ChangeDetectorRef {
   ...
}
```

變更偵測操作
---------------------------

> 與 Ivy 中變更偵測相關的操作在[這篇更新的文章](https://angular.love/en/overview-of-angulars-change-detection-operations-in-ivy/)中有說明。

負責執行 view 變更偵測的主要邏輯位於 [checkAndUpdateView](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/view.ts#L325) 函式中。其大部分功能都對 ****子**** component view 執行操作。這個函式 ****會針對每個 component 從 host component 開始以遞迴方式呼叫****。這表示子 component 在下一次呼叫時會成為父 component，因為遞迴樹會展開。

當針對特定 view 觸發此函式時，它會按照指定的順序執行以下操作：

1.  如果 view 是第一次被檢查，則將 `ViewState.firstCheck` 設定為 `true`；如果之前已經檢查過，則設定為 `false`。
2.  檢查並[更新](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L154)子 component/directive 實例上的輸入屬性。
3.  [更新](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L436)子 view 的變更偵測狀態（變更偵測策略實作的一部分）。
4.  為嵌入式 view [執行變更偵測](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/view.ts#L327)（重複列表中的步驟）。
5.  如果繫結已變更，則在子 component 上[呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L202) `OnChanges` 生命週期 Hook。
6.  在子 component 上[呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L202) `OnInit` 和 `ngDoCheck`（`OnInit` 僅在第一次檢查期間呼叫）。
7.  [更新](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/query.ts#L91) 子 view component 實例上的 `ContentChildren` 查詢列表。
8.  [呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L503)子 component 實例上的 `AfterContentInit` 和 `AfterContentChecked` 生命週期 Hook（`AfterContentInit` 僅在第一次檢查期間呼叫）。
9.  如果 ****目前 view**** component 實例上的屬性已變更，則[更新](https://hackernoon.com/the-mechanics-of-dom-updates-in-angular-3b2970d5c03d) ****目前 view**** 的 DOM 插值。
10. 為子 view [執行變更偵測](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/view.ts#L541)（重複此列表中的步驟）。
11. [更新](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/query.ts#L91) 目前 view component 實例上的 `ViewChildren` 查詢列表。
12. [呼叫](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/provider.ts#L503)子 component 實例上的 `AfterViewInit` 和 `AfterViewChecked` 生命週期 Hook（`AfterViewInit` 僅在第一次檢查期間呼叫）。
13.  [停用](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/view.ts#L346)目前 view 的檢查（變更偵測策略實作的一部分）。

根據上面列出的操作，有幾件事需要強調。

第一件事是，`onChanges` 生命周期 Hook 會在檢查子 component 之前在子 component 上觸發，即使將跳過子 view 的變更偵測，它也會觸發。這是重要的資訊，我們將在本文的第二部分中看到如何利用這些知識。

第二件事是，view 的 DOM 是作為變更偵測機制的一部分更新的，同時正在檢查 view。這表示，如果未檢查 component，則即使在範本中使用的 component 屬性發生變更，DOM 也不會更新。範本會在第一次檢查之前呈現。我所指的 DOM 更新實際上是插值更新。因此，如果你有 `<span>some {{name}}</span>`，DOM 元素 `span` 將在第一次檢查之前呈現。在檢查期間，只會呈現 `{{name}}` 部分。

另一個有趣的觀察是，子 component view 的狀態可以在變更偵測期間變更。我之前提到，所有 component view 預設都會使用 `ChecksEnabled` 初始化，但是對於所有使用 `OnPush` 策略的 component，變更偵測會在第一次檢查後停用（列表中的操作 9）：

```typescript
if (view.def.flags & ViewFlags.OnPush) {
  view.state &= ~ViewState.ChecksEnabled;
}
```

這表示在後續的變更偵測執行期間，將會跳過此 component view 及其所有子 view 的檢查。關於 `OnPush` 策略的文件指出，只有在其繫結變更時才會檢查 component。因此，為了執行此操作，必須透過設定 `ChecksEnabled` 位元來啟用檢查。這就是以下程式碼所做的事情（操作 2）：

```typescript
if (compView.def.flags & ViewFlags.OnPush) {
  compView.state |= ViewState.ChecksEnabled;
}
```

只有在父 view 繫結已變更且子 component view 已使用 `ChangeDetectionStrategy.OnPush` 初始化時，才會更新狀態。

最後，目前 view 的變更偵測負責啟動子 view 的變更偵測（操作 8）。這是檢查子 component view 狀態的地方，如果它是 `ChecksEnabled`，則會對此 view 執行變更偵測。以下是相關的程式碼：

```typescript
viewState = view.state;
...
case ViewAction.CheckAndUpdate:
  if ((viewState & ViewState.ChecksEnabled) &&
    (viewState & (ViewState.Errored | ViewState.Destroyed)) === 0) {
    checkAndUpdateView(view);
  }
}
```

現在你了解到 view 狀態控制是否對此 view 及其子 view 執行變更偵測。因此，問題就出現了 — 我們可以控制該狀態嗎？事實證明，我們可以，這就是本文第二部分要討論的內容。

一些生命週期 Hook 在 DOM 更新之前呼叫（3、4、5），而一些則在之後呼叫（9）。因此，如果你有以下 component 階層：`A -> B -> C`，以下是 Hook 呼叫和繫結更新的順序：

```
A: AfterContentInit
A: AfterContentChecked
A: 更新繫結
    B: AfterContentInit
    B: AfterContentChecked
    B: 更新繫結
        C: AfterContentInit
        C: AfterContentChecked
        C: 更新繫結
        C: AfterViewInit
        C: AfterViewChecked
    B: AfterViewInit
    B: AfterViewChecked
A: AfterViewInit
A: AfterViewChecked
```

探索影響
--------------------------

假設我們有以下 component 樹：

![圖片 24](https://wp.angular.love/wp-content/uploads/2024/07/content-image1-629.png)

正如我們在上面了解到的，每個 component 都與一個 component view 相關聯。每個 view 都會使用 `ViewState.ChecksEnabled` 初始化，這表示當 Angular 執行變更偵測時，樹中的每個 component 都會被檢查。

假設我們要停用 `AComponent` 及其子 view 的變更偵測。這很容易做到 — 我們只需要將 `ViewState.ChecksEnabled` 設定為 `false`。變更狀態是一種低階操作，因此 Angular 為我們提供了 view 上可用的許多公開方法。每個 component 都可以透過 `ChangeDetectorRef` token 取得其相關 view 的控制權。對於這個類別，Angular 文件定義了以下公開介面：

```typescript
class ChangeDetectorRef {
  markForCheck() : void
  detach() : void
  reattach() : void
  
  detectChanges() : void
  checkNoChanges() : void
}
```

讓我們看看如何利用它來為我們帶來好處。

### detach

第一個允許我們操作狀態的方法是 `detach`，它只是停用目前 view 的檢查：detach()：

```typescript
detach(): void { this._view.state &= ~ViewState.ChecksEnabled; }
```

讓我們看看如何在程式碼中使用它：

```typescript
export class AComponent {
  constructor(public cd: ChangeDetectorRef) {
    this.cd.detach();
  }
```

這確保在後續的變更偵測執行期間，將會跳過從 `AComponent` 開始的左分支（不會檢查橘色 component）：

![圖片 25](https://wp.angular.love/wp-content/uploads/2024/07/content-image2-531.png)

這裡有兩件事要注意 — 第一件事是，即使我們變更了 `AComponent` 的狀態，它的所有子 component 也都不會被檢查。第二件事是，由於不會為左分支 component 執行任何變更偵測，因此其範本中的 DOM 也將不會更新。以下是一個小範例來說明：

```typescript
@Component({
  selector: 'a-comp',
  template: `<span>看看我是否改變：{{changed}}</span>`
})
export class AComponent {
  constructor(public cd: ChangeDetectorRef) {
    this.changed = 'false';

    setTimeout(() => {
      this.cd.detach();
      this.changed = 'true';
    }, 2000);
  }
```

第一次檢查 component 時，span 將會使用文字 `看看我是否改變：false` 呈現。在兩秒內，當 `changed` 屬性更新為 `true` 時，span 中的文字將不會變更。但是，如果我們移除這行 `this.cd.detach()`，一切都會如預期般運作。

### reattach

如本文第一部分所示，如果 `AppComponent` 上輸入繫結 `aProp` 發生變更，則仍然會為 `AComponent` 觸發 `OnChanges` 生命周期 Hook。這表示，一旦我們收到輸入屬性發生變更的通知，就可以啟用目前 component 的變更偵測以執行變更偵測，並在下一個 tick 時分離它。以下是示範這一點的程式碼片段：

```typescript
export class AComponent {
  @Input() inputAProp;

  constructor(public cd: ChangeDetectorRef) {
    this.cd.detach();
  }

  ngOnChanges(values) {
    this.cd.reattach();
    setTimeout(() => {
      this.cd.detach();
    })
  }
```

由於 `reattach` 只是[設定](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/refs.ts#L242) `ViewState.ChecksEnabled` 位元：

```typescript
reattach(): void { this._view.state |= ViewState.ChecksEnabled; }
```

這幾乎等同於將 `ChangeDetectionStrategy` 設定為 `OnPush` 時所執行的操作：在第一次變更偵測執行後停用檢查，當父 component 繫結屬性變更時啟用它，並在執行後停用。

請注意，`OnChanges` Hook 僅會為停用分支中最上層的 component 觸發，而不是為停用分支中的每個 component 觸發。

### markForCheck

`reattach` 方法僅會為目前 component 啟用檢查，但是如果其父 component 未啟用變更偵測，它將不會有任何作用。這表示 `reattach` 方法僅對停用分支中最上層的 component 有用。

我們需要一種方法來為所有父 component 啟用檢查，直到根 component。並且[有一個方法](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/util.ts#L110)為此 `markForCheck`：

```typescript
let currView: ViewData|null = view;
while (currView) {
  if (currView.def.flags & ViewFlags.OnPush) {
    currView.state |= ViewState.ChecksEnabled;
  }
  currView = currView.viewContainerParent || currView.parent;
}
```

正如你可以從實作中看到的，它只是向上迭代，並為每個父 component 啟用檢查，直到根。

這什麼時候有用？就像 `ngOnChanges` 一樣，即使 component 使用 `OnPush` 策略，也會觸發 `ngDoCheck` 生命周期 Hook。同樣，它僅會為停用分支中最上層的 component 觸發，而不是為停用分支中的每個 component 觸發。但是，我們可以透過這個 Hook 執行自訂邏輯，並將我們的 component 標示為符合一次變更偵測週期的執行條件。由於 Angular 僅檢查物件參考，我們可能會實作某些物件屬性的髒檢查：

```typescript
Component({
   ...,
   changeDetection: ChangeDetectionStrategy.OnPush
})
MyComponent {
   @Input() items;
   prevLength;
   constructor(cd: ChangeDetectorRef) {}

   ngOnInit() {
      this.prevLength = this.items.length;
   }

   ngDoCheck() {
      if (this.items.length !== this.prevLength) {
         this.cd.markForCheck(); 
         this.prevLenght = this.items.length;
      }
   }
```

### detectChanges

有一種方法可以為目前 component 及其所有子 component [執行變更偵測] ****一次****。這是使用 `detectChanges` [方法](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/refs.ts#L239) 完成的。此方法會針對目前 component view 執行變更偵測，無論其狀態為何，這表示對於目前 view，檢查可能會保持停用狀態，並且在後續的常規變更偵測執行期間不會檢查 component。以下是一個範例：

```typescript
export class AComponent {
  @Input() inputAProp;

  constructor(public cd: ChangeDetectorRef) {
    this.cd.detach();
  }

  ngOnChanges(values) {
    this.cd.detectChanges();
  }
```

當輸入屬性變更時，會更新 DOM，即使變更偵測器參考保持分離狀態。

### checkNoChanges

變更偵測器上提供的最後一個方法確保在目前變更偵測執行中不會進行任何變更。基本上，它會執行第一篇文章中列表中的 1、7、8 操作，如果發現變更的繫結或判斷 DOM 應該更新，則會擲回例外狀況。
