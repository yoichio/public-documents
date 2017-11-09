UA Implements
Chrome
document.getSelection();
host.shadowRoot.getSelection();
Safari
document.getSelection();
host.shadowRoot.getSelection() is undefined.
Firefox, Edge
no shadow dom yet(coming!)
Cases
outer<span id=host></span>
<script>
host.attachShadow({mode:'open'}).innerHTML = 'inner';
</script>
#1. outer->inner
Chrome

document.getSelection() = {‘outer’,2, ‘outer’, 5};
host.shadowRoot.getSelection() = {null, 0, null, 0};
Safari

document.getSelection() = {‘outer’,2, ‘outer’, 5};

#2. inner->outer
Chrome

document.getSelection() = {document.body, 1, document.body, 1};
host.shadowRoot.getSelection() = {‘inner’, 3, ‘inner’, 0};
Safari

document.getSelection() = {document.body, 1, document.body, 1};
#3. inner
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
