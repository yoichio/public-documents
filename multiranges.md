# Minimal multi ranges explained

Chrome implements user selection as one [Range](https://www.w3.org/TR/dom/#range), which represents a range
 on a document.  
However, there are cases where user want to select contents which can not
be represented with one Range.  
Chrome wants to offer such selection for user and/or web author balancing between capability and stability.  
It is opt-in, StaticRange and few execCommands.

## Cases we need multiple ranges
### Ctrl-click
On Firefox, user can select discontigous DOM Range with ctrl-click/drag.
![img](resources/ctrl-click.png)  
#### Table
![table](resources/table.png)  

### Grid Layout
Following code demostrate grid layout reorder.
```html
<style>
.root { display: grid;}
.root > div { outline: solid 1px #aaa;}
</style>
<div class=root>
<div style="grid-row:1">content1 foo1 bar1</div>
<div style="grid-row:3">content2 foo2 bar2</div>
<div style="grid-row:2">content3 foo3 bar3</div>
</div>
```
In that case, user drag from first grid to second one to select those elements but
“content2” of third grid is also selected.
![grid](https://github.com/yoichio/public-documents/blob/master/resources/grid.png)

### Shadow DOM
Following code demostrate Shadow DOM layout nodes reorder.
```html
out
<span id=host>
 <span slot=s1>foo1</span>
 <span slot=s2>bar2</span>
</span>
<script>
 host.attachShadow({mode:"open"}).innerHTML =
  "<slot name=s2></slot><slot name=s1></slot>";
</script>
```
If user select from 'out' to 'bar2', chrome select them excluding 'foo1'.  
![img](resources/shadow2.png)  
However, ```getSelection().getRangeAt(0)``` returns {'out',1, 'bar2', 2}.

## Problems to implement full multiple Ranges 
If we simply implement multiple Ranges on ```addRange()```,```rangeCount``` and ```getRangeAt()```,
there are many issues:
- Backward compatibility: Many cites assume user selection is a Range and use only ```getRangeAt(0)```.
- Performance: Range should mutate syncronousely for DOM mutation(
[spec](https://www.w3.org/TR/2000/PR-DOM-Level-2-Traversal-Range-20000927/ranges.html#Level-2-Range-Mutation)).
It means if there are more Ranges, DOM mutation performance gets worse.
- Complexity: Overwrapped ranges, ```insertOrderedList``` or other DOM mutation commands.

## Proposition
### Opt-in selection mode
We have few entry points for multiple Range.
```javascript
window.getSelection().modes = ['multiple-user-ctrl', 'multiple-user-layout', 'multiple-addstaticrange'];
```
Default ```modes``` are empty array, in which U.A behaves as-is.  
Setting ```modes``` enables multiple ranges:
- ```multiple-user-ctrl``` enables user to create multiple ranges with ctrl-click/drag.
- ```multiple-user-layout``` enables user to create multiple ranges with drag/shift-arrowkey on layout order.
- ```multiple-addstaticrange``` enables webauthor to create multiple ranges with ```addStaticRange``` method.

### No overwrapping
Any mode doesn't create/allow overwrapping Ranges.

### Editing functionality for user
We offer user only 
- copy  
So that user can get contents they are selecting.  

- delete, cut(copy + delete)  
If ```modes``` include ```multiple-user-ctrl```, that is also useful.

I'm considering the operation inserting text onto multiple ranges simultaniously.
Web author can implement it with
[Input Events](https://www.w3.org/TR/input-events-2/) if we pass multiple ranges
through ```getTargetRanges()```. Ditto to delete and cut.

### Editing API for web author
#### Invalidating existing Range API.
```rangeCount``` returns 0, ```addRange()``` does nothing and ```getRangeAt()``` always throws exception.
That's because I want web author to avoid performance footgun of Range.

### StaticRange API.
```javascript
document.getSelection().addStaticRange(nodeA, 0, nodeB, 3);
```
That's all, but it throws exception if added StaticRange overwrapps existing ```getStaticRanges()```.  
You get all ranges with ```getStaticRanges()```:
```javascript
for (let range of getSelection().getStaticRanges()) {
  // Do "static" opration like
  myDb.bookmarkUserSelection(range);
}
```

If web author want to edit content and have live Ranges, they might
create Range from the StaticRange.
```javascript
let ranges = [];
// Collect all ranges before editing.
for (let range of getSelection().getStaticRanges()) {
  let domrange = document.createRange();
  domrange.setStart(range.startContainer, range.startOffset);
  domrange.setEnd(range.endContainer, range.endOffset);
  ranges.push(domrange);
}
for (let domrange of ranges) {
  // Do "dynamic" opration like
  unbold(domrange);
}
```

### Limited execCommand
Only 'copy', 'undo', 'redo' are allowed.
