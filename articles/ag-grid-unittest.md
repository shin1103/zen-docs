---
title: "単体テストでag-gridのセルに値をセットする"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aggrid", "jasmine", "angular"]
published: true
---
# 概要
Angularバージョンアップに際し、ag-gridのバージョンをv22.0からv25.3.0にアップデートした。その際に単体テストでag-gridのセルに値を書き込むテストで軒並みエラーとなっていたので、その対処方法を記載する。

# 対処方法

ag-gridには以下のリンクの通り https://www.ag-grid.com/javascript-data-grid/provided-cell-editors/ デフォルトのcellEditorが準備されている。今回の移行では３つのcellEditorを使っていたので、それぞれ対処方法を記載する。

前提として本単体テストはangularのcomponentのテストで、componentではag-gridのgridOptionsが定義されている。

## agTextCellEditor
このcellEditorはシンプルであるため、HTML要素から直接値をセットできた。後述の２つは設定する箇所を特定できなかったので、ag-gridのクラスから値をセットしている。このcellEditorでも同様のことができるかもしれない。
```javascript
component.gridOptions.api.startEditingCell({
    rowIndex: updateRowIndex,
    colKey: updateColKey
})
const textbox = component.gridOptions.api.getCellEditorInstances()[0].getGui().querySelector('.ag-text-field-input') as HTMLInputElement
textbox.value = updateText
textbox.dispatchEvent(new Event('input'))
component.gridOptions.api.stopEditing()
fixture.detectChanges()
```

## agLargeTextCellEditor

```javascript
component.gridOptions.api.startEditingCell({
    rowIndex: updateRowIndex,
    colKey: updateColKey
})
const textbox: AgInputTextArea = (component.gridOptions.api.getCellEditorInstances()[0] as any).cellEditor.eTextArea
textbox.setValue(updateText)
component.gridOptions.api.stopEditing()
fixture.detectChanges()
```

## agSelectCellEditor
本当はSelect要素のIndexを指定したかったが、指定する方法が見つからず。そのため、値を直接指定する
```javascript   
component.gridOptions.api.startEditingCell({
    rowIndex: updateRowIndex,
    colKey: updateColKey
})
const selectbox: AgSelect = (component.gridOptions.api.getCellEditorInstances()[0] as any).eSelect
selectbox.setValue(updateText)
component.gridOptions.api.stopEditing()
fixture.detectChanges()
```
