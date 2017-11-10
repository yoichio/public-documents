# [Selection API](https://www.w3.org/TR/selection-api/) prospotion for [Shadow DOM](https://www.w3.org/TR/shadow-dom/)
In this document, I explain Selection API issues on Shadow DOM and propose spec modification.  

Selection API defines selection as it is unique on a ducument which consists of one node tree.  
However, Shadow DOM inserts other node trees into a document recursively and we don't expose contents in the trees with javascript API.  
That has made Selection API not working for Shadow DOM.([reported issue](https://github.com/w3c/webcomponents/issues/79))  
Also there are interop issues between user agents' implementation.  

## Support table
|                           |   Chrome  | Safari | Firefox | Edge |
|------------               |:---------:|:------:|:------:|:------:|
| document.getSelection()   |    ✔️     |   ✔️   |✔️|✔️|
| Shadow DOM                |  ✔️       | ✔️     | (in development) | (under consideration) | 
| User selection for Shadow | ❗(see example) | ❗(see example)  | N/A| N/A |
| shadowRoot.getSelection() |  ❗(see example)      |  undefined  | N/A| N/A |

## Examples
Following code illustrates very simple Shadow DOM:
```html
outer<span id=host></span>
<script>
host.attachShadow({mode:'open'}).innerHTML = 'inner';
</script>
```
![image](resources/shadow.png)  

Let's see what happens if the user drag mouse in various ways.

### #1. outer->inner  
The user drags mouse from ```'outer'``` to ```'inner'```.  
|                           |   Chrome  | Safari |
|------------               |:---------:|:------:|
| User selection   |    a       |   ✔️   |
| shadowRoot.getSelection() |  ❗(see example)      |  undefined  | 

#### Chrome
![image](resources/outerinner-chrome.png)   
```javascript
document.getSelection() = {‘outer’,2, ‘outer’, 5};
host.shadowRoot.getSelection() = {null, 0, null, 0};
Safari

document.getSelection() = {‘outer’,2, ‘outer’, 5};

### #2. inner->outer  
Chrome

document.getSelection() = {document.body, 1, document.body, 1};
host.shadowRoot.getSelection() = {‘inner’, 3, ‘inner’, 0};
Safari

document.getSelection() = {document.body, 1, document.body, 1};

### #3. Only inner  
Chrome

document.getSelection() = {document.body, 1, document.body, 1};
host.shadowRoot.getSelection() = {‘inner’, 1, ‘inner’, 4};
Safari

document.getSelection() = {document.body, 1, document.body, 1};

Except javascript API, Safari implementation follows spec that prohibits the user crossing shadow but permits inner shadow.
Though there are many other cases like outer->innter->moreinner, inner->outer->anotherinner, I want to get a consensus about this simplest case first. 


Proposition

For user interactions, the user can select contents crossing shadow boundary.
If web author want to control user selection, recommend using CSS user-select property.
As javascipt API,
document.getSelection() = {‘outer’,2, document.body, 2};
                (document.body, 2) means after host.
it indicates shadowRoot is selected but still not expose shadow contents.
host.shadowRoot.getSelection() = {host.shadowRoot, 0, ‘inner’, 3};
selection starts from shadowroot and ends at the middle of text.
