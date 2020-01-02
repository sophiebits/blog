---
layout: post
title: "Fast and maintainable patterns for fetching from a database"
redirect_from: ["/2019/01/01/fast-maintainable-db-patterns.html"]
---

In this post, I want to talk about how we can build better application frameworks and how we can structure code in our applications more clearly.

Both of these are just means of working towards our real goal: finding ways to make our application code faster and more maintainable.

When I say "application code", this time I'm mostly thinking of server-side (web backend) development, where you have a database that you perform many fetches from. Technically speaking, nothing here is database-specific, but the _idea_ of a database is the right mental model here, so it's the terminology I'll use.

I'm going to focus on the sort of application frameworks where you fetch individual pieces of stored data (maybe using an [ORM](https://en.wikipedia.org/wiki/Object-relational_mapping) library). In particular, this post will tell you little about how to improve code that uses, say, complex SQL queries with joins.

These are techniques that I've seen work at scale – in codebases developed by thousands of engineers, containing many millions of lines of code, and used by billions of users.

I'm confident they make sense on small teams too.

## Introduction

For the code examples in this post, I'll give my database functions names like `fetchPosts()`. It could have just as easily been something like `Post.fetchAll()` or `DB.queryAll(Post)`. The exact naming and syntax here isn't what I'm focused on.

As an example, let's say the app we're building has a blog component, and we'd like to render a summary of published posts, with the post title, author, number of times it's been viewed, and a fun fact about that view count number. The list we're rendering will look like this:

* **"What makes a sofa a sofa?" by Sophie A**<br>
  View count: 496 (_the third [perfect number](https://en.wikipedia.org/wiki/Perfect_number)!_)
* **"Salami for cats (as a treat)" by Joseph A**<br>
  View count: 17 (_the number of distinct [wallpaper groups](https://en.wikipedia.org/wiki/Wallpaper_group)_)
* **"Why is the sky blue?" by Sophie A**<br>
  View count: 0

To fetch the data needed to render this, the most straightforward code we might write here could look something like this:

{% highlight javascript %}
// [{
//   title: 'What makes a sofa a sofa?',
//   authorName: 'Sophie A',
//   viewCount: 496,
//   funFact: 'the third perfect number!',
// }, ...]
async function buildPostsSummary() {
  let posts = await fetchPosts();
  let summary = [];
  for (let post of posts) {
    let author = await post.fetchAuthor();
    let stats = await post.fetchStats();
    let funFact =
      stats.viewCount > 0
      ? await fetchFunFact(stats.viewCount)
      : undefined;
    summary.push({
      title: post.title,
      authorName: author.name,
      viewCount: stats.viewCount,
      funFact,
    });
  }
  return summary;
}
{% endhighlight %}

This code is some of the simplest code we can write to solve this task – we fetch the posts, then for each post we fetch the author, the view stats, and the fun fact. I've added a conditional where we fetch the fun fact (suppose `fetchFunFact` only takes non-zero numbers) to make it a little more representative of most real-world code.

However, this code likely doesn't run very fast. In particular, we talk to our database three separate times for most posts in the list. Worse, we loop over the posts serially; that is, we process them one by one, waiting for each post's data to finish before we even look at the next post in the list.

## Batching

One major improvement would be to make these requests in batch, where we ask our database to fetch multiple things at once. Imagining we have functions such as `fetchManyAuthors(posts)` which takes a list of posts and returns a list of authors corresponding 1:1 with the list of posts, we could rewrite our code to process the whole list of posts at once:

{% highlight javascript %}
async function buildPostsSummary() {
  let posts = await fetchPosts();
  let authors = await fetchManyAuthors(posts);
  let stats = await fetchManyStats(posts);

  // We only want to fetch facts for non-zero view counts.
  // It gets more complicated than a simple .filter() because we need
  // to track which view count came from which index.
  // For example, we need to:
  // ...turn [496, 17, 0, 2] into [496, 17, 2],
  // ...pass that to fetchManyFunFacts(),
  // ...get back three fun facts [factA, factB, factC],
  // ...then space it into [factA, factB, undefined, factC] so that
  //    it matches up with [post0, post1, post2,     post3].
  let nonZeroViewCounts = [];
  let nonZeroViewCountIndices = [];
  stats.forEach((stat, idx) => {
    if (stat.viewCount > 0) {
      nonZeroViewCounts.push(stat.viewCount);
      nonZeroViewCountIndices.push(idx);
    }
  });
  let nonZeroFunFacts = await fetchManyFunFacts(nonZeroViewCounts);
  let funFacts = Array(posts.length).fill(undefined);
  for (let i = 0; i < nonZeroFunFacts.length; i++) {
    let originalIndex = nonZeroViewCountIndices[i];
    funFacts[originalIndex] = nonZeroFunFacts[i];
  }

  // Now our variables posts, authors, stats, and funFacts are all the
  // same length and have the data we need. We can now stitch them back
  // together:
  let summary = [];
  for (let i = 0; i < posts.length; i++) {
    summary.push({
      title: posts[i].title,
      authorName: authors[i].name,
      viewCount: stats[i].viewCount,
      funFact: funFacts[i],
    });
  }
  return summary;
}
{% endhighlight %}

Whew.

The good news is this now talks to the database only 4 times total.

The bad news is that the code is a small nightmare. The first few lines started out OK – but as soon as you run into anything more complicated, even the one-line conditional we had in our original function, the code turns into a disaster.

**Switching from processing one item at a time to processing many at once makes our code _dramatically_ more complex.** This is bad news both because it's annoying to write and because it's much easier to introduce bugs. (Do _you_ know if my code snippet has a bug in it? Me neither.)

Note that we're also now coupled to the particular array-of-posts structure we've chosen. For example, what if we had multiple separate blogs, and we wanted to fetch this summary for each blog's respective posts? Unless we restructure this entire function, we'd end up fetching the data for each blog separately, rather than having it fully batched together.

The fact that this is annoying to write also means that this optimized version is less likely to get written in the first place. Devs won't want to write this, and so they'll usually gravitate towards simpler alternatives, even if that means having a slower app.

So we need to find another approach.

## Parallelism

Let's think back to our original example. Our biggest problem is that we were processing the posts array completely serially. With a small tweak, we can make it process each post in parallel, by initiating processing for every post then waiting for them all to finish with a `Promise.all`.

{% highlight javascript %}
async function buildPostsSummary() {
  let posts = await fetchPosts();
  let summary = Promise.all(posts.map(post => buildPostSummary(post)));
  return summary;
}

async function buildPostSummary(post) {
  let author = await post.fetchAuthor();
  let stats = await post.fetchStats();
  let funFact =
    stats.viewCount > 0
    ? await fetchFunFact(stats.viewCount)
    : undefined;
  return {
    title: post.title,
    authorName: author.name,
    viewCount: stats.viewCount,
    funFact,
  };
}
{% endhighlight %}

This code is pretty manageable. Our extracted helper function only needs to worry about one post, which makes coding easier. We also now use a `.map(...)` and don't need mutation to build up the `summary` list now, which feels a little cleaner to me.

What are the performance characteristics of this? It depends on how these `fetch*()` functions are implemented.

The most straightforward implementation of these would send a separate request to the database for each of the 3 things we're fetching per post, plus the one request to fetch the posts in the first place – if we had 100 posts, then we'd talk to our database around 301 times. Many of these requests would happen concurrently, so it will be much faster than what we started with.

However, there's a typically a sizable fixed cost for each database request we make. Since we're incurring that cost many times, this is still slower than the optimized, batched (nightmare) solution we wrote above.

## Automatic batching

It would be ideal if we could reduce the number of requests while keeping this clean coding style. What we want is to leave the individual calls to `fetchAuthor` unchanged (for example) but have that result in a single batched database request behind the scenes. Luckily, this is possible.

The trick is that `fetchAuthor` should wait – even for just a fraction of a millisecond – before sending the request to the database. If anyone else calls `fetchAuthor` in that moment, we can combine them into a single request containing both fetches. When we map over our `posts` array to process them in parallel, all of the `fetchAuthor` calls are initiated at approximately the same time, which means they can all be batched together.

Our `fetchStats` and `fetchFunFact` functions should batch in the same way. Typically, you wouldn't implement this batching separately for each type of data that can be fetched; if they share an underlying database, you would implement the batching at that layer.

That way, if you want to fetch two different kinds of data at the same time, those can also be batched into a single database request. (Ideally, our database's underlying batch API would allow streaming responses, so that if some pieces of data are ready before the rest, we can immediately begin work on their next steps.)

## Optimal parallelism

At this point, we're in a pretty decent state. But let's look at another optimization opportunity. Let's reread this code from above:

{% highlight javascript %}
async function buildPostSummary(post) {
  let author = await post.fetchAuthor();
  let stats = await post.fetchStats();
  let funFact =
    stats.viewCount > 0
    ? await fetchFunFact(stats.viewCount)
    : undefined;
  return ...;
}
{% endhighlight %}

Note that we wait until the author has been fetched before we begin fetching the post's stats. We can parallelize these too. You might try to fix it by parallelizing the author and stats fetches:

{% highlight javascript %}
async function buildPostSummary(post) {
  let [
    author,
    stats,
  ] = await Promise.all([
    post.fetchAuthor(),
    post.fetchStats(),
  ]);
  let funFact =
    stats.viewCount > 0
    ? await fetchFunFact(stats.viewCount)
    : undefined;
  return ...;
}
{% endhighlight %}

This is better but isn't optimal: if fetching the author takes longer than fetching the stats, we unnecessarily wait for the author before we fetch the fun fact.

There are a few ways we can structure this for optimal parallelism. You only want to fetch the stats once, so maybe you'd try having one "thread" fetch the stats and fun fact, in parallel with the author fetch:

{% highlight javascript %}
async function buildPostSummary(post) {
  let [
    author,
    [stats, funFact],
  ] = await Promise.all([
    post.fetchAuthor(),
    fetchStatsAndFunFact(),
  ]);
  return ...;

  async function fetchStatsAndFunFact() {
    let stats = await post.fetchStats();
    let funFact =
      stats.viewCount > 0
      ? await fetchFunFact(stats.viewCount)
      : undefined;
    return [stats, funFact];
  }
}
{% endhighlight %}

This does run with optimal parallellism.

However, take a moment to notice that our code structure is now tied to the specific structure of the data dependencies we're using.

For example, imagine that a change to our app now means that we _do_ need to fetch the author before fetching the stats. We'd probably end up rewriting back to the 3-serial-fetches structure we had before. What would ideally be a one-line change now means we need to refactor our whole function. In this small example with 3 fetches, it's doable. In a real-world example that might have 10+ fetches with complicated interdependencies, it's hard to see what the optimal structure is, and it's harder to do the refactor too.

How else can we structure this?

Let's look back at what our function returns (eliding the actual data fetching code, which we'll soon rewrite):

{% highlight javascript %}
async function buildPostSummary(post) {
  ...
  return {
    title: post.title,
    authorName: ...,
    viewCount: ...,
    funFact: ...,
  };
}
{% endhighlight %}

We know we need to fetch three pieces of information – the author name, the view count, and the fun fact. Ideally, we'd like to think about these separately and not have to exert human brainpower on understanding the interdependencies between them.

Let's write code that considers these three independently:

{% highlight javascript %}
async function buildPostSummary(post) {
  async function fetchAuthorName() {
    let author = await post.fetchAuthor();
    return author.name;
  }

  async function fetchViewCount() {
    let stats = await post.fetchStats();
    return stats.viewCount;
  }

  async function fetchViewCountFunFact() {
    let stats = await post.fetchStats();
    return
      stats.viewCount > 0
      ? await fetchFunFact(stats.viewCount)
      : undefined;
  }

  let [
    authorName,
    viewCount,
    funFact,
  ] = await Promise.all([
    fetchAuthorName(),
    fetchViewCount(),
    fetchViewCountFunFact(),
  ]);

  return { title: post.title, authorName, viewCount, funFact };
}
{% endhighlight %}

Now, we first specify how to fetch the three pieces of info we need; then, we initiate and wait for the trio together. This will runs with the same optimal parallelism structure, but it happens automatically for us; we didn't need to write it out by hand.

I love this code: it is more lines of code than before, but the code is less "clever" – which is a good thing. (You could also write this in an object-oriented fashion with a class if you prefer.)

I wrote `post.fetchStats()` in my `fetchViewCountFunFact()` function; it would also work to do `let viewCount = fetchViewCount();` if you wanted to reuse some code. These inner functions are pretty simple in this example, but they would often be more complex in the real world.

The only issue with this code is that we may now be fetching the stats twice. In fact, we almost never want to fetch one thing twice in a single web request. To make sure this doesn't happen, the best solution here (that I know of) is…

## Caching (within a request)

Add caching. A lot of it. Here's how:

I'll recommend that most caches shouldn't last longer than the web request. This is for a few reasons: first, because the memory is freed at the end of the request, it's easier to cache more without worrying about memory usage (it's rare that a user-facing request will fetch or create more data than fits in available RAM); second, users don't need to worry about getting a stale result when loading the page; third, scoping to one user's request reduces the chance of accidental privacy leaks due to "cross-contamination" between different users' data.

But with that in mind, we can get a huge win by caching essentially all reads from our database. Just like the auto-batching, this is typically best implemented at a lower level of your application frameworks so that all of your app code that fetches from your database can benefit.

Even in an application built by people trying to be judicious about data fetching and make sure each piece of data is fetched once and at the right time (a cumbersome task, as we saw above), people make mistakes and it's likely you would see wins from implementing this. But the bigger advantage here is that it allows us to write clearer, more maintainable code.

We can also add caching in higher levels of our application. Whenever there's a complex computation you wouldn't want to redo, just cache it – again, for the length of the request.

For example, we might put a `@memoize` decorator on the Post object's `.fetchStats()` function, which will make a cache that lives only as long as that Post is in memory. Or on the `fetchViewCount()` function we just wrote – so that it only runs once even if we change `fetchViewCountFunFact()` to use it.

With the small tweak of adding memoization, the last code snippet above – and code like it – is efficient, easy to write, and maintainable.

## Conclusion

This is an outline of how to build infrastructure that allows application code to be simple while still being fast and clean.

All of the techniques I've mentioned are ultimately motivated by one thing:

**Facilitating local reasoning.**

What I mean by that is: you should be able to worry about each part of your code in isolation, without holding the entire system in your head. In my experience, this is _the_ key to making complex systems scale, especially (but not only) in a large organization.

For example, you should be able to edit the three helper functions `fetchAuthorName`, `fetchViewCount`, and `fetchViewCountFunFact` independently without worrying about affecting the others. When you call a function (like `buildPostSummary` itself), you ideally shouldn't need to worry about whether what it fetches overlaps with other functions you're calling.

Some of these techniques are built into [dataloader](https://github.com/graphql/dataloader), an open-source project that is [based directly on](https://medium.com/@leeb/dataloader-v2-0-925b4dccf8d6) the same systems I gleaned these techniques from.

But it's more about the principles – when you understand them, it's not hard to write your own implementation.
