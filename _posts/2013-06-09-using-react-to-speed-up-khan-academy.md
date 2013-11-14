---
layout: post
title: Using React to speed up the Khan Academy question editor
---

*This post was written in response to &ldquo;[How is Facebook's React JavaScript library?](http://www.quora.com/React-JS-Library/How-is-Facebooks-React-JavaScript-library)&rdquo; on Quora.*

I'd looked briefly at Ember and Angular and was intending to write something in each to test them out but [React](http://facebook.github.io/react/) caught my eye last week.

I just rewrote a 2000-line project in React and have now made a handful of pull requests to React. Everything about React I've seen so far seems really well thought-out and I'm proud to be the first non-FB/IG production user of React.

The project that I rewrote in React (and am continuing to improve) is the Khan Academy question editor which content creators can use to enter questions and hints that will be presented to students:

<a href="/images/2013-06-09/perseus.png" target="_blank"><img src="/images/2013-06-09/perseus.png" width="560" height="360" alt="screenshot of Khan Academy question editor"></a>

The left side has the actual editor, while the right side of the page shows a live preview of the question you're writing. Originally, the editor would serialize the entire question and give it to the previewer, which rendered the question to the page.

--

Performance was a concern with the old editor so I had it keep track of whether the question, the answer area, or the solution was changing and only rerender the appropriate part. Unfortunately, the interface was still annoyingly slow because we would still often rerender more than the specific changed pieces; putting more dirty-checking logic into the editor–renderer interface seemed like it would only make the code more convoluted and harder to maintain.

(An aside: why was rendering so slow? The primary culprit is MathJax, which we use to render math expressions within the text. Even typesetting a few simple expressions (like *y = 3x – 5* in the screenshot above) takes on the order of 200ms on my computer, and much longer on slower machines.)

React's declarative interface seemed to be perfect for my needs because it allows simply writing how the document tree should look and it will intelligently determine which changes need to be made; it won't touch the DOM when it doesn't have to. I rewrote the renderer completely declaratively and React takes care of figuring out what to update. The source shrunk by a few hundred lines and updating the preview takes an order of magnitude less time than it did previously.

--

I think my favorite thing about React is that it's simple. As an example, the Angular documentation for how to write a [directive](http://docs.angularjs.org/guide/directive) (a reusable component) needs to explain directive priorities, the difference between compiling and linking, the creation of new scopes, and transclusion, all of which are Angular-specific concepts. In contrast, React recommends that you learn [JSX](http://facebook.github.io/react/docs/jsx-in-depth.html), but after that it's just JavaScript, so everything you know about JavaScript still applies. As far as I can tell, React's approach here is just as powerful as other frameworks but won't be any harder to maintain.

React's code feels reasonably mature to me – since it's been used by both Facebook and Instagram in production for a while now, it looks like most of the bugs have been ironed out. It doesn't do everything for you yet (e.g., automatic async promise support or two-way data binding like [Angular's ng-model](http://docs.angularjs.org/api/ng.directive:ngModel) or [Ember's TextField](http://emberjs.com/api/classes/Ember.TextField.html)) but what's there seems to work really well.

The biggest complaint I've had is that the docs are sometimes unclear and the best practices aren't clearly documented yet. However, this shouldn't prevent anyone from using React. In addition, I've found that the developers are really helpful on IRC and I have no doubt that the documentation will only get better over time.

I love React so far, and I have high hopes for its future.
