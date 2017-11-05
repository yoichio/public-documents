# Multi ranges explained

Chrome wants user to be able to select elements intuitively by mouse/touch/keyboard but   
we need new APIs on [Selection API](https://www.w3.org/TR/selection-api/) to represent such selection for web author.

## Scenarios
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

### BiDi
Let's see following Bidi text:
```
<p dir="ltr">bahrain مصر kuwait</p>
```
(“مصر “ means Egypt pronoused ‘misr’ from right to left.) <br/>
 rsim
  
User might want to select text in reading order neither DOM order nor visibly.

![bidi_reading](https://github.com/yoichio/public-documents/blob/master/resources/bidi_reading.png)

This is the result when I drag from left to right in the middle of ‘Egypt’.

![bidi_select](https://github.com/yoichio/public-documents/blob/master/resources/bidi_select.png)
User can select in reading order on Chrome no in DOM order.

Chrome needs multiple ranges representation internally for highliting/copy/paste such selected content.
If web author needs the range too(for example, news paper/ebook web app that can bookmark user selection like kindle.), we should expose the ranges.

As much discussed, we don’t want to increase Range(let me say DOM Range for correctness) instances for performance[1] and Range mutation sometimes doesn’t work for web authors expectation[2].<br/>
[1] Replace StaticRange with a dictionary or figure out if normal Ranges could be used ( (2016-09-22, w3c)
[2] Support multi range selection (2017-05-29, w3c)

Chrome plans to represent multiple ranges w/o DOM Range internally and expose the ranges as not DOM Range.

## Proposition
I propose:
```
partial interface Selection {
    sequence<StaticRange> getRanges();
};
```
This is very simple API and enough to capture user selection for none DOM changing operation(bookmark, change style,,).
Pros
	Simple.
Cons
	No live Range.
Even If web author calls addRange(range), getRanges() returns different StaticRanges. 

If web author want to edit content and have live Ranges, they might
create Range from the StaticRange.
However, Range mutation doesn’t already work as web aurhor expects:
https://github.com/w3c/selection-api/issues/41#issuecomment-289924788 
I’m thinking another API using Promise chain:
```
partial interface Selection {
    Promise<RangeIterator> getNextRangeIterator(
optional RangeIterator iterator);
};
interface RangeIterator {
 readonly attribute boolean HasRange;
 readonly attribute StaticRange;
}
```
The following code illustrates editing with live StaticRanges:
```javascript
async editAsync() {
	var iterator= await window.getSelection().getNextRangeIterator();
	if (!iterator.HasRange) {
		console.log(“no selection”).
		return;
	}	
	do {
	const range = iterator.range;
	console.log(range.startContainer);
// some DOM mutation code
iterator = await window.getSelection().getNextRangeIterator(iterator);
} while (iterator.HasRange);
}
```
Point is web aurhor only can get StaticRange one by one through Promise.
If some mutation changes remaining ranges, getNextRange return such
updated range. Number of iteration also can change in the middle of the loop.

Pros
Web author accesses fresh ranges w/o Range. U.A. can implement another range mutation.

Cons
Complex.
What If user change selection while editing?
Even If web author calls addRange(range), getNextRangeIterator returns different
ranges.
