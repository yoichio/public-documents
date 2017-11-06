# Limited multi ranges explained

Chrome implements user selection as one [Range](https://www.w3.org/TR/dom/#range), which represents a range
 on a document.  
However, there are cases where user want to select contents which can not
be represented with one Range.
Chrome wants to offer such selection with less functionality thant what chrome offers with one Range.

we need new APIs on [Selection API](https://www.w3.org/TR/selection-api/) to expose such selection for web author.

## Cases
### Grid Layout
Following code demostrate grid layout layouting reorder.
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
### Table
### Ctrl-click
### Shadow DOM

## Proposition
We offer
- for user
Copy, Delete
- for web author
getRanges(), execCommand copy, delete

Chrome needs multiple ranges representation internally for highliting/copy/paste such selected content.
If web author needs the range too(for example, news paper/ebook web app that can bookmark user selection like kindle.), we should expose the ranges.

However, as we discussed, we don’t want to increase Range instances for performance because Range should
mutate syncronousely if depending DOM tree is changed[[1](https://github.com/w3c/input-events/issues/38#issuecomment-252309333)], and Range mutation sometimes doesn’t work for web authors expectation[[2](https://github.com/w3c/selection-api/issues/41#issuecomment-289924788)].

Chrome plans to represent multiple ranges w/o DOM Range internally and expose the ranges as not DOM Range.

### #1 StaticRanges.
I propose:
```webidl
// Web IDL
partial interface Selection {
  sequence<StaticRange> getRanges();
};
```
This is very simple API and enough to capture user selection for none DOM changing operation(bookmark, change style,,):
```javascript
// javascript
for (let range of getSelection().getRanges()) {
  // Do "static" opration like
  myDb.bookmarkUserSelection(range);
}
```

#### Pros
- Simple.
#### Cons
- No live Range.
- Even If web author calls ```addRange(range)```, ```getRanges()``` might return different
number/range of StaticRanges because U.A highlight selection splitting the passed ```range```
as the scenarios show. 

If web author want to edit content and have live Ranges, they might
create Range from the StaticRange.
```javascript
// javascript
let ranges = [];
for (let range of getSelection().getRanges()) {
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
