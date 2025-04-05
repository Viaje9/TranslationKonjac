---
title: 變更偵測大觀 – 總覽
date: 2024-12-31 17:08:14
tags:
---


原文：<https://angular.love/change-detection-big-picture-overview>

# 變更偵測概觀 - 總覽

## 概觀

在任何應用程式中，更新螢幕的過程都需要程式的內部狀態，並將其投影到使用者可以在螢幕上看到的東西。在網頁開發中，我們會使用諸如物件和陣列之類的資料結構，並最終以影像、按鈕和其他視覺元素的形式呈現該資料的 DOM 表示法。框架負責同步應用程式的內部狀態和底層平台。但是，它們不僅減輕了我們的負擔，而且還非常有效地進行狀態追蹤和 DOM 更新。

這種同步機制稱為渲染。
它有不同的名稱 – Angular 中的變更偵測或 React 中的協調 – 但它執行的核心操作
在不同的實作中是相似的。
變更偵測是架構中最重要的部分之一，因為
它負責更新負責可見部分的平台模型 (DOM)。
它也是顯著影響應用程式效能的領域。
接下來，我將交替使用渲染和變更偵測這兩個術語。

<!-- more -->

為了定義 UI 和應用程式狀態之間的關係，我們在範本中使用表達式。
在下面的範本中，我們表示 DOM 屬性 `className` 取決於元件的屬性 `rating`。
因此，每當評分變更時，都應重新評估表達式。
如果偵測到變更，則應更新 `className` 屬性。

```html
<div [className]="'fa-star ' + (rating > value ? 'fas' : 'far')"></div>
```

在 Angular 中，當編譯器分析範本時，它會識別與 DOM 元素相關聯的元件屬性。
對於每個此類關聯，編譯器都會建立一個繫結。
繫結定義了元件的屬性（通常包裝在某些表達式中）和 DOM 元素屬性之間的關係。
**變更偵測機制會執行處理繫結的指令。**
這些指令的工作是檢查具有元件屬性的表達式的值是否已變更
並在必要時執行 DOM 更新。

處理執行髒檢查並更新 DOM 相關部分的繫結是 Angular 中變更偵測的核心操作。
我們將在 [操作](https://angular.love/en/change-detection-big-picture-operations/) 章節中詳細介紹其他操作。
現在讓我們快速看看 Angular 如何更新 DOM。

## DOM 更新總覽

這個互動式小工具允許使用者按一下任何星星並設定新的評分：

![圖片 19：圖片替代文字](https://wp.angular.love/wp-content/uploads/2024/08/6.gif)

以下是如何實作它：

```typescript
@Component({
	selector: "rating-widget-cmp",
	template: `
		<ul class="rating">
			<li
				*ngFor="let value of values"
				[className]="'fa-star ' + (rating > value ? 'fas' : 'far')"
				(click)="onRatingClick(value)"
			>
				{{ value }}
			</li>
		</ul>
	`,
})
export class RatingWidgetComponent {
	values = [0, 1, 2, 3, 4];
	rating = 2;

	onRatingClick(v: number) {
		this.rating = v + 1;
	}
}
```

範本中的 `rating` 屬性透過以下表達式繫結到 `className` 屬性：

```html
[className]="'fa-star ' + (rating > value ? 'fas' : 'far')"
```

對於範本的這部分，編譯器會產生設定繫結、執行髒檢查和更新 DOM 的指令。
以下是為我們的範本產生的程式碼：

```typescript
if (changeDetectionPhase) {
    property("className", "fa-star " + (ctx.rating > 0 ? "fas" : "far"));
    ...
}
```

在這裡，我們可以看到指令 `property`（在來源中以 `ɵɵ` 作為字首），它會檢查表達式的值是否已變更，
如果是，則將繫結標記為已變更並更新值。
Angular 為 `className` 建立繫結，並且繫結的目前值為 `'fa-star far'`。
一旦元件中的評分屬性更新，Angular 就會執行變更偵測並處理這些指令。

[property](https://github.com/angular/angular/blob/40c138c13d17b638908999faafc9eb4cca0202fb/packages/core/src/render3/instructions/property.ts#L35-L35)
指令在簡化的程式碼中如下所示：

```typescript
export function property(propName, value, ...) {
  const lView = getLView();
  const bindingIndex = nextBindingIndex();

  if (bindingUpdated(lView, bindingIndex, value)) {
    const tView = getTView();
    const tNode = getSelectedTNode();
    elementPropertyInternal(tView, tNode, lView, propName, value, ...);
  }

  return property;
}
```

首先，我們取得 [LView](https://github.com/angular/angular/blob/da7318e2fa9d35af46387965f92b6871ca7946a4/packages/core/src/render3/interfaces/view.ts#L79) 的參考，
它是 Angular 元件和所有相關資料的容器。
然後，使用 `nextBindingIndex`，我們擷取 `LView` 中的索引，該索引保留有關 `propName` 繫結的資訊。
在我們的例子中，它是 `className` 屬性。

[bindingUpdated](https://github.com/angular/angular/blob/663d477cb04da4960a9a6552c556abe73417b95b/packages/core/src/render3/bindings.ts#L46)
函式會評估表達式，並使用它來與繫結記住的先前值進行比較。
這就是「髒檢查」名稱的由來。如果值已變更，它會更新目前值並回傳 `true`：

![圖片 20：圖片替代文字](https://wp.angular.love/wp-content/uploads/2024/08/2-1.png)

如果表達式已變更，Angular 會執行 `elementPropertyInternal` 函式，該函式會使用新值來更新 DOM。
在我們的例子中，它會更新清單項目的 `className` 屬性。

![圖片 21：圖片替代文字](https://wp.angular.love/wp-content/uploads/2024/08/4-2.png)

`property` 函式會回傳自身，以便可以鏈接：

```typescript
property("name", ctx.name)("title", ctx.title);
```

就這樣。我們將在「渲染引擎內部」一節中詳細了解其他指令。

## 觸發變更偵測

若要全面了解變更偵測，我們需要知道 Angular 何時執行處理繫結的指令。

有兩種方法可以啟動變更偵測。第一種是**明確告知框架**
某些內容已變更或有可能發生變更，因此它應執行變更偵測。
基本上，透過這種方式，我們手動啟動變更偵測。
第二種方法是**依賴外部機制**來知道何時有可能
發生變更，並自動執行變更偵測。

在 Angular 中，我們有這兩種選項。我們可以使用變更偵測器服務來
[手動執行變更偵測](https://github.com/angular/angular/blob/a98fa489fb498273bdfde2fa4598893d1d53582e/packages/core/src/render3/view_ref.ts#L273)：

```typescript
export class RatingWidgetComponent {
	values = [0, 1, 2, 3, 4];
	rating = 2;

	constructor(private changeDetector: ChangeDetectorRef) {}

	onRatingClick(v: number) {
		this.rating = v + 1;
		this.changeDetector.detectChanges();
	}
}
```

但是，我們也可以依賴框架自動觸發變更偵測。
透過這種方式，我們只需更新元件上的屬性：

```typescript
export class RatingWidgetComponent {
	values = [0, 1, 2, 3, 4];
	rating = 2;

	onRatingClick(v: number) {
		this.rating = v + 1;
	}
}
```

Angular 需要以某種方式知道屬性已更新，並且它應執行變更偵測。
若要解決這個問題，Angular 會使用名為
[zone.js](https://angular.love/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found/) 的函式庫。
它會修補瀏覽器中的所有非同步事件，然後可以在發生特定事件時通知 Angular。
與 UI 事件類似，Angular 然後可以等待應用程式碼完成執行
並自動啟動變更偵測。我們將在 [zones 部分](https://angular.love/en/change-detection-big-picture-overview) 中詳細探討 `zone.js`。

## 檢查順序

Angular 會以深度優先的順序對每個元件執行變更偵測。這表示對於下面描述的元件樹狀結構，檢查將會以下列順序發生 `A、K、L、J、O、C` 等等：

![圖片 22：圖片替代文字](https://wp.angular.love/wp-content/uploads/2024/08/7-1.gif)

由於元件檢查包含多個操作，因此執行某些步驟的順序可能會產生一些不同的結果。例如，導致 DOM 和繫結更新的操作會以正確的深度優先順序執行。我使用函式 `logRender` 來記錄 Angular 何時更新範本並執行函式來評估範本表達式：

```typescript
@Component({
	selector: "a-cmp",
	template: `{{ logRender() }}`,
})
export class A {
	logRender() {
		console.log("A");
	}
}
```

請查看 [這裡](https://stackblitz.com/edit/angular-ivy-pfemyn?file=src%2Fapp%2Fcomponents.ts) 的即時執行範例。

但是，如果我將記錄新增到 [`ngDoCheck`](https://angular.love/if-you-think-ngdocheck-means-your-component-is-being-checked-read-this-article/) 鉤子中，如下所示：

```typescript
@Component({
	selector: "a-cmp",
	template: `A`,
})
export class A {
	ngDoCheck() {
		console.log("A");
	}
}
```

這是你將看到的順序：

![圖片 23：圖片替代文字](https://wp.angular.love/wp-content/uploads/2024/08/8-1.gif)

Angular 會檢查 `A`、`K`，然後檢查 `V`、`L`，然後檢查 `C`，依此類推。
請查看 [這裡](https://stackblitz.com/edit/angular-ivy-cvy5ze?file=src%2Fapp%2Fapp.component.ts) 的即時執行範例。

這既不是深度優先演算法，也不是廣度優先演算法。
廣度優先演算法的傳統實作會檢查同一層級上的所有同級項目，
而如你在上面的圖表中看到的，該演算法確實會檢查 `L` 和 `C` 同級元件，
但它不是檢查 `X` 和 `F`，而是向下到 `J` 和 `O`。
