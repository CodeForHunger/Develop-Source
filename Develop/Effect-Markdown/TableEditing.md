<h1 align="center">Table Editing</h1>

[toc]

## Tables in Markdown

Typora supports table syntax of [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/). For example, it will parse following text in a markdown file into a table.

> ```gfm
> |First Header | Second Header|
> |------------ | -------------|
> |Content from cell 1 | Content from cell 2|
> |Content in the first column | Content in the second column|
> ```

## Table Editing in Typora

In Typora, you can write or edit the markdown text to add or modify tables.

We also provides menu entry, context menu, shortcut keys, drag & move, tooltips to make table editing easier to use under related GUI.

![img](media/Screen%20Shot%202019-07-26%20at%2001.03.21.png)Table Menu in Typora

### Create Table in Typora

To create a table in Typora, you can simply write out a table header in markdown.

```
|First Header | Second Header|
```

Or, you can also insert table from the menu bar.

A table must have table headers and at least one row and one column.

### Add Row in Table

Press `Command/Ctrl+Enter` to quickly insert an empty row under the current table row, or use the Context menu (right click).

### Delete Row in Table

The delete line command or delete table row (Shift+Ctrl+L or Shift+Command+L on macOS) will delete current table row in a table, or use the Context menu (right click).

### Add/Delete Column in Table

Right click on a table cell, and in the Context menu, there are menu items for add/remove table columns.

### Resize Table

Put the cursor inside a table and a table tooltip will show above the table header. Click the most left icon, and you will be able to resize the table.

If you want to make the table larger than 6 columns or 10 rows, you can click the row/column number and enter a number.

![Snip20170227_2](media/Snip20170227_2.png)

### Text Alignment in Column

In [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/), column alignment can be specified as follows:

```
| Default | Left  | Right | Center |
| ------- | :---- | ----: | :----: |
| cell1   | cell2 | cell3 | cell4  |
```

| Default | Left  | Right | Center |
| ------- | :---- | ----: | :----: |
| cell1   | cell2 | cell3 | cell4  |

In Typora, you can simply change text alignment under a column by selecting the related alignment icon from table tooltip.

With alignment set, Typora will add attribute like `style="text-align: left"` to the current column (`<td>`), but the final alignment can still be changed by CSS rules in current theme or custom CSS.

### Move Row/Column

It’s easy to reorder rows/columns through Typora’s WYSIWYG feature. Just click on the left/top border of a row/column, and use drag and drop to move it:



### Touch Bar Support

Tables can also be altered using the Touch Bar on certain MacBook laptops.

![TouchBar](media/TouchBarShot.png)