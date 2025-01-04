---
title: The mechanics of DOM updates in Angular
date: 2025-01-04 17:10:03
tags:
---

原文：[https://angular.love/the-mechanics-of-dom-updates-in-angular](https://angular.love/the-mechanics-of-dom-updates-in-angular)

# Angular 中 DOM 更新的機制

由模型變更觸發的 DOM 更新是所有現代前端框架的關鍵功能，Angular 也不例外。 我們只需指定像這樣的表達式：

```
<span>Hello {{name}}</span>
```

或像這樣的綁定：

```
<span [textContent]="'Hello ' + name"></span>
```

每當 `name` 屬性變更時，Angular 就會神奇地更新 DOM。 從表面上看來這似乎很容易，但實際上內部是一個相當複雜的過程。 DOM 更新是 Angular [變更偵測機制](https://angular.love/en/everything-you-need-to-know-about-change-detection-in-angular/)的一部分，主要由三個主要操作組成：

*   DOM 更新
*   子元件 `Input` 綁定更新
*   查詢列表更新

本文探討了變更偵測的渲染部分是如何運作的。 如果你曾經想知道這是如何實現的，請繼續閱讀並獲得啟發。 在參考來源時，我假設應用程式正在生產模式下執行。 讓我們開始吧。

<!-- more -->

應用程式內部表示
-------------------

在我們可以開始探索 Angular DOM 更新功能之前，我們需要先了解 Angular 應用程式在底層是如何表示的。 讓我們簡單回顧一下。

### 視圖 (View)

你可能[從我的其他文章](https://angular.love/en/here-is-what-you-need-to-know-about-dynamic-components-in-angular/)中了解到，對於應用程式中使用的每個元件，Angular 編譯器都會產生一個 ****工廠****。 當 Angular 從工廠建立元件時，例如像這樣：

```
const factory = r.resolveComponentFactory(AComponent);
factory.create(injector);
```

Angular 使用此工廠來實例化 [視圖定義 (View Definition)](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/types.ts#L48)，而視圖定義又用於[建立元件視圖 (View)](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/view.ts#L217)。 ****在底層，Angular 將應用程式表示為視圖樹****。 每個元件類型只有一個視圖定義的實例，它作為所有視圖的模板。 但是對於每個元件實例，Angular 都會建立一個單獨的視圖。

### 工廠 (Factory)

元件工廠主要由編譯器在解析範本後產生的視圖節點組成。 假設你像這樣定義元件的範本：

```
<span>I am {{name}}</span>
```

使用此資料，編譯器會產生以下元件工廠：

```
function View_AComponent_0(l) {
    return jit_viewDef1(0,
        [
          jit_elementDef2(0,null,null,1,'span',...),
          jit_textDef3(null,['I am ',...])
        ], 
        null,
        function(_ck,_v) {
            var _co = _v.component;
            var currVal_0 = _co.name;
            _ck(_v,1,0,currVal_0);
```

它描述了元件視圖的結構，並在實例化元件時使用。 `jit_viewDef1` 是對 [viewDef](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/view.ts#L23) 函數的引用，該函數會建立視圖定義。

視圖定義接收 [視圖定義節點 (View definition nodes)](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/types.ts#L105) 作為參數，這些參數類似於 HTML 的結構，但也包含許多 Angular 特定的詳細資訊。 在上面的範例中，第一個節點 `jit_elementDef2` 是元素定義，第二個 `jit_textDef3` 是文字定義。 Angular 編譯器會產生許多不同的節點定義，並且與節點相關聯的類型會在 [NodeFlags](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/types.ts#L153) 中設定。 我們稍後將看到 Angular 如何使用有關節點類型的資訊來決定如何處理更新。

就本文而言，我們僅對元素和文字節點感興趣：

```
export const enum NodeFlags {
    TypeElement = 1 << 0, 
    TypeText = 1 << 1
```

讓我們簡單回顧一下它們。

### 元素定義 (Element definition)

[元素定義](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/types.ts#L235) 是 Angular 為每個 HTML 元素產生的節點。 這種類型的元素也[為元件產生](https://angular.love/en/here-is-why-you-will-not-find-components-inside-angular/)。 元素節點可以包含其他元素節點和文字定義節點作為子節點，這反映在 `childCount` 屬性中。

所有元素定義都由 [elementDef](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/element.ts#L56) 函數產生，因此工廠中使用的 `jit_elementDef2` 引用了此函數。 元素定義採用一些通用參數：

```
+------------------+-----------------------------------+
|       名稱        |          描述            
+------------------+-----------------------------------+
| childCount       | 指定目前元素有多少個子節點      
| namespaceAndName | HTML 元素的名稱              
| fixedAttrs       | 在元素上定義的屬性            
+------------------+-----------------------------------+
```

以及其他特定於特定 Angular 功能的參數：

```
+----------------------+--------------------------------+
|         名稱         |          描述               
+----------------------+--------------------------------+
| matchedQueriesDsl    | 在查詢子節點時使用            
| ngContentIndex       | 用於節點投射                      
| bindings             | 用於 DOM 和綁定屬性更新       
| outputs, handleEvent | 用於事件傳播                    
+----------------------+--------------------------------+
```

就本文而言，我們只對 bindings 參數感興趣。

### 文字定義 (Text definition)

[文字定義](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/types.ts#L290) 是 Angular 編譯器為每個 [文字節點](https://developer.mozilla.org/en/docs/Web/API/Node/nodeType#Constants) 產生的節點定義。 通常，這些是元素定義節點的子節點，就像我們範例中的情況一樣。 這是一個非常簡單的節點定義，由 [textDef](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/text.ts#L12) 函數產生。 它以常數形式接收已剖析的表達式作為第二個參數。 例如，以下文字：

```
<h1>Hello {{name}} and another {{prop}}</h1>
```

將剖析為一個陣列：

```
["Hello ", " and another ", ""]
```

然後使用它來產生正確的綁定：

```
{
  text: 'Hello',
  bindings: [
    {
      name: 'name',
      suffix: ' and another '
    },
    {
      name: 'prop',
      suffix: ''
    }
  ]
}
```

並在髒檢查期間這樣評估：

```
text
+ context[bindings[0][property]] + context[bindings[0][suffix]]
+ context[bindings[1][property]] + context[bindings[1][suffix]]
```

### 節點定義綁定 (Node definition bindings)

Angular 使用 [綁定](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/types.ts#L196) 來定義每個節點對元件類別屬性的相依性。 在變更偵測期間，每個綁定都會決定 Angular 應用於更新節點的操作類型，並提供內容資訊。 操作類型由 [綁定旗標](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/types.ts#L205) 決定，對於 DOM 特定的操作，它構成了以下清單：

```
+-----------------------+--------------------------+
|         名稱          | 範本中的建構      |
+-----------------------+--------------------------+
| TypeElementAttribute  | attr.name                |
| TypeElementClass      | class.name               |
| TypeElementStyle      | style.name               |
+-----------------------+--------------------------+
```

元素和文字定義會根據編譯器識別的綁定旗標在內部建立這些綁定。 每種節點類型都有特定於綁定產生的不同邏輯。

更新渲染器
---------------

我們最感興趣的是編譯器產生的工廠 `View_AComponent_0` 末尾列出的函數：

```
function(_ck,_v) {
    var _co = _v.component;
    var currVal_0 = _co.name;
    _ck(_v,1,0,currVal_0);
```

此函數稱為 `updateRenderer`。 它接收兩個參數 `_ck` 和 `v`。 `_ck` 是 `check` 的縮寫，並引用函數 [prodCheckAndUpdate](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/services.ts#L262)。 另一個參數是具有節點的元件視圖。 每次 [當 Angular 對元件執行變更偵測時](https://angular.love/en/everything-you-need-to-know-about-change-detection-in-angular/)，都會執行 `updateRenderer` 函數，並且該函數的參數由變更偵測機制提供。

`updateRenderer` 函數的主要任務是從元件實例中檢索綁定屬性的目前值，並呼叫 `_ck` 函數，傳遞視圖、節點索引和檢索到的值。 重要的是要了解，Angular 會單獨為每個視圖節點執行 DOM 更新 — 這就是為什麼需要節點索引的原因。 當檢查 `_ck` 引用的函數的參數清單時，可以清楚地看到它：

```
function prodCheckAndUpdateNode(
    view: ViewData, 
    nodeIndex: number, 
    argStyle: ArgumentType, 
    v0?: any, 
    v1?: any, 
    v2?: any,
```

`nodeIndex` 是應執行變更偵測的視圖節點的索引。 如果你的範本中有多個表達式：

```
<h1>Hello {{name}}</h1>
<h1>Hello {{age}}</h1>
```

編譯器將為 `updateRenderer` 函數產生以下主體：

```
var _co = _v.component;

// 此處節點索引為 1，屬性為 `name`
var currVal_0 = _co.name;
_ck(_v,1,0,currVal_0);

// 此處節點索引為 4，綁定的屬性為 `age`
var currVal_1 = _co.age;
_ck(_v,4,0,currVal_1);
```

更新 DOM
----------------

現在我們了解了 Angular 編譯器產生的所有特定物件，我們可以探索如何使用這些物件執行實際的 DOM 更新。

我們在上面了解到，在變更偵測期間，`updateRenderer` 函數會傳遞 `_ck` 函數，而此參數引用 [prodCheckAndUpdate](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/services.ts#L262)。 這是一個簡短的通用函數，它會進行一系列呼叫，最終執行 [checkAndUpdateNodeInline](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/view.ts#L400) 函數。 當表達式計數超過 10 時，[該函數有一個變體](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/view.ts#L386)。

`checkAndUpdateNode` 函數只是一個路由器，用於區分以下類型的視圖節點，並將檢查和更新委派給相應的函數：

```
case NodeFlags.TypeElement   -> checkAndUpdateElementInline
case NodeFlags.TypeText      -> checkAndUpdateTextInline
case NodeFlags.TypeDirective -> checkAndUpdateDirectiveInline
```

現在讓我們看看這些函數的作用。 有關 `NodeFlags.TypeDirective`，請參閱[Angular 中屬性綁定更新的機制](https://angular.love/en/the-mechanics-of-property-bindings-update-in-angular/)。

### 元素類型 (Type Element)

它使用函數 [CheckAndUpdateElement](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/element.ts#L229)。 該函數基本上會檢查綁定是否為 Angular 特殊形式 `[attr.name, class.name, style.some]` 或某些節點特定的屬性。

```
case BindingFlags.TypeElementAttribute -> setElementAttribute
case BindingFlags.TypeElementClass     -> setElementClass
case BindingFlags.TypeElementStyle     -> setElementStyle
case BindingFlags.TypeProperty         -> setElementProperty;
```

相應的函數只是使用渲染器的相應方法來對節點執行所需的操作。

### 文字類型 (Type Text)

它在兩種變體中使用函數 [CheckAndUpdateText](https://github.com/angular/angular/blob/15090a8ad4a23dbe947ec48b581f1bf6a2da411e/packages/core/src/view/text.ts#L62)。 以下是該函數的要旨：

```
if (checkAndUpdateBinding(view, nodeDef, bindingIndex, newValue)) {
    value = text + _addInterpolationPart(...);
    view.renderer.setValue(DOMNode, value);
}
```

它基本上會取得從 `updateRenderer` 函數傳遞的目前值，並將其與上次變更偵測的值進行比較。 視圖會在 [oldValues](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/types.ts#L318) 屬性中保留舊值。 如果值已變更，Angular 會使用更新的值來組成字串，並使用渲染器更新 DOM。

結論
----------

我確信這是一個難以承受的大量資訊。 但是透過了解它，你在設計應用程式或偵錯與 DOM 更新相關的問題時會更好。 我建議你也使用偵錯工具並遵循本文中說明的執行邏輯。
