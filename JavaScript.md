#### Event Delegation
Consider the following:

```
<ul id="parent-list">
	<li id="post-1">Item 1</li>
	<li id="post-2">Item 2</li>
	<li id="post-3">Item 3</li>
	<li id="post-4">Item 4</li>
	<li id="post-5">Item 5</li>
	<li id="post-6">Item 6</li>
</ul>
```

Adding an event handler to each LI is tedious and potentially causes performance issues. Additionally, if your page is configured dynamically, it would be very problematic to constantly be adding and removing handlers.

Instead, by using event delegation, you can add a single handler to the parent element by using the target element. The event will bubble to the parent, and then you can use the reference to the clicked node to compare.

```javascript
document.getElementById("parent-list").addEventListener("click", function(e) {
	if(e.target && e.target.nodeName == "LI") {
		console.log("List item ", e.target.id.replace("post-", ""), " was clicked!");
	}
});
```
