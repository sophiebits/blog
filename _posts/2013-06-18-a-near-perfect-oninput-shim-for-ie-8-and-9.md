---
layout: post
title: A near-perfect oninput shim for IE 8 and 9
---

I recently began using [React](http://facebook.github.io/react/), which, like many libraries, has an events system designed so event properties can be normalized between browsers and custom events can be created. I contributed a synthetic event called [textchange](https://github.com/facebook/react/pull/75) which is essentially a cross-browser shim for the input event (which is natively supported in modern browsers but not in older versions of IE).

In this post I'll explain how the input event works and how I'm simulating it in IE 8 and 9. I'll also point you to a jQuery plugin I made so you can take advantage of my textchange event regardless of whether you're using React yet or not.

## Understanding the input event

First off, what is the [input](https://developer.mozilla.org/en-US/docs/Web/Reference/Events/input) event? If you have an `<input>` element and want to receive events whenever the value changes, the most obvious thing to do is to listen to the change event. Unfortunately, change fires only after the text field is defocused, rather than on each keystroke. The next obvious choice is the keyup event, which is triggered whenever a key is released. Unfortunately, keyup doesn't catch input that doesn't involve the keyboard (e.g., pasting from the clipboard using the mouse) and only fires once if a key is held down, rather than once per inserted character.

Both keydown and keypress do fire repeatedly when a key is held down, but both fire immediately before the value changes, so to read the new value you have to defer the handler to the next event loop using `setTimeout(fn, 0)` or similar, which slows down your app. Of course, like keyup, neither keydown nor keypress fires for non-keyboard input events, and all three can fire in cases where the value doesn't change at all (such as when pressing the arrow keys).

The input event was created to solve all of these problems. It fires immediately whenever the value of a text box changes, whether due to the keyboard or another input device. (Technically, the [spec](http://www.whatwg.org/specs/web-apps/current-work/multipage/common-input-element-attributes.html#event-input-input) says that browsers can delay the event if they so choose, but all modern browsers seem to fire it immediately.) It's supported in most common browsers, with the notable exceptions of IE 8 (which doesn't support it at all) and IE 9 (which has a buggy implementation).

## Simulating the input event in IE 8

Let's see how to simulate the input event in IE 8. Conveniently, IE supports a proprietary event called [propertychange](http://msdn.microsoft.com/en-us/library/ie/ms536956%28v=vs.85%29.aspx) which fires whenever any property of an element changes, including `value`. Thus, you can get 80% of the way there by doing:

{% highlight javascript %}
el.attachEvent("onpropertychange", function(e) {
    if (e.propertyName === "value") {
        // Fire textchange event on el
    }
});
{% endhighlight %}

However, we don't want to fire the textchange event if the value has been changed from JavaScript code because such a notification is inconsistent with how the input event works (and almost always unhelpful). In addition, the propertychange event doesn't bubble like most events do, so you have to bind to each input whose value you want to track (rather than binding on `document` and relying on [event delegation](http://davidwalsh.name/event-delegate)).

IE 8 allows you to use `Object.defineProperty` on DOM elements, so to ignore `value` changes caused by JavaScript, we can override the property setter for `value` to intercept the change and silence our event. To avoid the overhead of listening to propertychange, we can listen to the focus and blur events and only listen to changes on the currently-focused element. Now our shim looks something like this:

{% highlight javascript %}
var activeElement = null;
var activeElementValue = null;

// On focus, start watching the element
document.addEventListener("focusin", function(e) {
    var target = e.srcElement;
    if (target.nodeName !== "INPUT") return;

    // Store a reference to the focused element and its current value
    activeElement = target;
    activeElementValue = target.value;

    // Listen to the propertychange event
    activeElement.attachEvent("onpropertychange", handlePropertyChange);

    // Override .value to track changes from JavaScript
    var valueProp = Object.getOwnPropertyDescriptor(
            HTMLInputElement.prototype, 'value');
    Object.defineProperty(activeElement, {
        get: function() { return valueProp.get.call(this); },
        set: function(val) {
            activeElementValue = val;
            valueProp.set.call(this, val);
        }
    });
});

// And on blur, stop watching
document.addEventListener("focusout", function(e) {
    if (!activeElement) return;

    // Stop listening to propertychange and restore the original .value prop
    activeElement.detachEvent("onpropertychange", handlePropertyChange);
    delete activeElement.value;

    activeElement = null;
    activeElementValue = null;
});

function handlePropertyChange(e) {
    if (e.propertyName === "value" &&
            activeElementValue !== activeElement.value) {
        activeElementValue = activeElement.value;
        // Fire textchange event on activeElement
    }
};
{% endhighlight %}

Now we're about 90% of the way there. Our textchange event is fired when typing normally, deleting text (with backspace, forward-delete, or the context menu), cutting, pasting, and dragging to reorder characters.

Turns out that IE 8 has a bug where it fails to fire propertychange on the very first keystroke following a `value` change from JavaScript code -- it fires only keydown, keypress, and keyup. To fix this, we can bind to the keyup event and trigger textchange if the value's changed. Of course, there's one final case: if the user _holds down_ a key immediately after some JavaScript code changes `value`, then keyup won't fire until the key is released. Since we want an event for every character, we can also bind to keydown in order to catch the second keydown (which is fired just before propertychange for the second character).

Phew! Now we have an event that works 99% of the time in IE 8.

## Simulating the input event in IE 9

Surprisingly (or maybe unsurprisingly), IE 9 has some problems of its own.

IE 9 claims to support the input event, but it fails to fire the input event upon deleting text. Worse yet, text deletions don't trigger propertychange either.

Fortunately, the IE developers were merciful and decided to fire the selectionchange event on every deletion, presumably because the cursor is moving and that counts as the selection changing. If we bind to selectionchange as well as propertychange, keyup, and keydown, we can catch all input events.

(IE 9 seems to have a bug where if you set the value of a `<textarea>` during a selectionchange event, the cursor gets moved to the start of the text box, and it refuses to move the cursor using createTextRange. I haven't figured out how to get around this problem -- please let me know if you find a way.)

## Putting it all together

It's amazing how many cross-browser quirks there are to deal with, but if we combine all of these techniques together, we can finally create an event that works consistently across all browsers. If anyone knows how to improve any of the solutions laid out in this post, please let me know.

If you're using React, there's no plugin needed; simply use `onTextChange={...}` and you're done. If you're not using React yet, I've created a jQuery plugin, [jquery.splendid.textchange.js](https://github.com/spicyj/jquery-splendid-textchange), that abstracts away cross-browser differences and dispatches textchange events via jQuery that behave almost identically in IE 8, IE9, and modern browsers.
