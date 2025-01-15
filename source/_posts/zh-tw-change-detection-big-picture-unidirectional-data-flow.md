---
title: Change Detection Big Picture – Unidirectional data flow
date: 2025-01-13 21:43:04
tags:
---


原文：[https://angular.love/change-detection-big-picture-unidirectional-data-flow](https://angular.love/change-detection-big-picture-unidirectional-data-flow)

# 在 Angular 中運行變更偵測 - 單向數據流的概念是什麼？

單向數據流
------------------------

Angular 強制執行所謂的**由上而下的單向數據流**。
這個慣例的本質是數據從父組件流向子組件，但反之則不然。
如果父組件的狀態發生變化，並且有對子組件的輸入綁定，
這些變化將在渲染（變更偵測）過程中被推送至子組件。

組件之間相互通信最常見的方式是透過綁定。
如果你在 Angular 中定義了一個父組件 `A`，如下所示，
你會看到它有一個子組件 `B`，並且父組件 `A` 通過 `obj` 輸入綁定將 `value` 傳遞給子組件 `B`：

```
// 父組件
@Component({
    template: `<b-component [prop]="value"></b-component>`
})
export class A {
    value = {name: 'initial'};
}

// 子組件
@Component({ ... })
export class B {
    @Input() prop;
}
```

<!-- more -->

理解 **Angular 在變更偵測期間更新綁定**這一點非常重要。
因此，當框架為父組件 `A` 執行變更偵測時，它將更新子組件 `B` 上的 `prop` 輸入綁定。
這表示變更偵測也總是從根組件開始，為每個組件，每次都由上而下執行。

如果子組件的狀態發生變化，而父組件以某種方式依賴這些變化，
子組件需要明確地將包含更改數據的事件發送回父組件。
沒有內建機制可以追蹤子組件中的這些變化，並自動將它們傳播回父組件。

這就是所謂的單向數據流。應用程式狀態在單次變更偵測後變得穩定。
這比循環更有效率且更可預測，如果子組件可以更新其父組件，就會出現循環。
我們始終知道視圖中使用的數據來自哪裡，因為它只能來自其父組件。

為了更好地理解這個限制，想像一下，在當前的變更偵測運行期間，某些已檢查組件的屬性以某種方式被更新了。
因此，模板中的表達式將產生與 Angular 作為這些組件檢查的一部分在螢幕上呈現的值不一致的新值。
Angular 必須做什麼才能使應用程式狀態和螢幕同步？
它當然可以運行另一個變更偵測循環，試圖同步這些。
但是，如果在該過程中某些屬性再次被更新了呢？你現在應該看到這個模式了。
Angular 實際上可能會陷入變更偵測運行的無限循環中，試圖同步應用程式狀態和副作用的結果，例如螢幕上呈現的內容。
因此，總而言之，一旦 Angular 處理完當前組件的綁定，
你就不再能更新組件中用於綁定表達式的屬性。

強制執行單向流
-----------------------------

在開發模式下，Angular 通過在常規變更偵測週期後運行額外的檢查來強制執行單向數據流。
此檢查包括比較組件屬性的當前值和模板中的表達式，與 Angular 在前面的變更偵測週期中使用和記住的值。
如果任何值不同，框架將拋出臭名昭著的 [`ExpressionChangedAfterItHasBeenCheckedError`](https://angular.love/everything-you-need-to-know-about-the-expressionchangedafterithasbeencheckederror-error/) 錯誤。

![圖片 10：圖片替代文字](https://wp.angular.love/wp-content/uploads/2024/08/1-12.png)

可以很容易地觀察到這種效果，方法是從子組件的 `AfterViewChecked` 鉤子內部更新父組件的狀態。
只需確保要更新的祖先組件的屬性導致副作用，即 DOM 更新或子組件輸入綁定更新。

```
// 父組件
@Component({
	selector: "a-cmp",
	template: `<b-cmp [prop]="value"></b-cmp>`,
})
export class A {
	value = { name: "initial" };
}

// 子組件
@Component({
	selector: "b-cmp",
	template: `{{ prop.name }}`,
})
export class B {
	@Input() prop;

	constructor(private parent: A) {}

	ngAfterViewChecked() {
		this.parent.value.name = "updated";
	}
}
```

在此處查看即時運行示例 [這裡](https://stackblitz.com/edit/angular-ivy-ngmpgu?file=src%2Fapp%2Fapp.module.ts)。

在生產模式下，Angular 不會拋出錯誤，但它也不會嘗試穩定應用程式狀態。
這通常會導致組件屬性的值與螢幕上呈現的值不一致。
在下一次變更偵測期間，不一致可能會得到解決，例如，更新的屬性同步到 DOM。
但是，如果在變更偵測檢查完組件後再次更新屬性值，
不一致的狀態可能永遠無法解決。當我們回顧變更偵測操作時，我們將花費大量時間來解釋這個錯誤。

儘管 Angular 中沒有內建機制可以導致變更偵測期間父組件模型更新，
但仍然可以通過各種機制意外地導致這種影響：注入父組件引用、共享服務或同步事件廣播。

子組件與父組件的通信
-----------------------------

在 Angular 中，從子組件到其父組件的通知機制是透過輸出綁定實現的，通常稱為組件事件。
它看起來像這樣：

```
// 父組件
@Component({
    template: `
        <h1>Hello {{value.name}}</h1>
        <a-comp (updateObj)="value = $event"></a-comp>
    `
})
export class AppComponent {
    value = {name: 'initial'};

    constructor() {
        setTimeout(() => {
            console.log(this.value); // logs {name: 'updated'}
        }, 3000);
    }
}

// 子組件
@Component({...})
export class AComponent {
    @Output() updateObj = new EventEmitter();

    constructor() {
        setTimeout(() => {
            this.updateObj.emit({name: 'updated'});
        }, 2000);
    }
}
```

組件事件最常從附加到 UI 事件、網路事件或計時器的事件處理程式中廣播。
由於這些事件在 Angular 運行變更偵測週期**之前**被觸發，
因此發出一個將導致父組件更新的事件是完全沒問題的。
事實上，這些瀏覽器事件是大多數觸發變更偵測的原因。
假設事件處理程式可能會更改應用程式狀態，因此 Angular 需要處理副作用，
例如將組件狀態與 DOM 同步。

但是，如果發出的事件導致父組件的屬性更新，
並且這些屬性在變更偵測期間被處理，即模板表達式，
則事件必須在 Angular 的變更偵測循環之外發出。
否則，這將導致父組件的屬性在變更偵測循環內被更新，從而導致 `ExpressionChangedAfterItHasBeenCheckedError` 錯誤。

有時，事件是從變更偵測循環期間執行的處理程式內部同步發出的。
只要響應事件的邏輯在 Angular 完成父組件的檢查之前更新父組件的屬性即可。
例如，可以從 [DoCheck](https://angular.io/docs/ts/latest/api/core/index/DoCheck-interface.html) 鉤子中執行此操作，
該鉤子在處理父組件變更之前在子組件上執行。

我們將在相應的章節中更仔細地研究這種情況。

全局狀態管理庫中的單向數據流
-------------------------------------------------

當今大多數 Web 應用程式都使用 NgRx 和 Redux 等狀態管理庫。
這些庫實現了與處理和儲存業務相關數據相關的功能。
今天，這個服務層被稱為狀態管理。
在這個層中運作的庫通常與呈現層（帶有通過 DOM 向使用者顯示應用程式相關數據的組件）沒有密切關聯。
因此，區分在變更偵測期間強制執行的單向數據流和用作狀態管理庫的主要架構原則的單向數據流是有意義的。
