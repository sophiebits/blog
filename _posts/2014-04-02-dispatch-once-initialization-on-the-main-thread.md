---
layout: post
title: Initializing on the main thread using dispatch_once
---

We recently refactored the [Khan Academy](http://www.khanacademy.org/) iOS app to use Core Data instead of the haphazard system of JSON files which we were previously using. In this post I'll explain how to run initialization code that has to happen on the main thread without worrying about race conditions.

When anything in the app needs to access the database, it needs a reference to a [Core Data context][context]. Our app shares one main thread context used by all of the UI code, which runs on the main thread. The first time this context is accessed, we need to initialize the persistent store coordinator and the context. Our core data stack class looks something like this:

[context]: https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSManagedObjectContext_Class/NSManagedObjectContext.html

{% highlight objc %}
// KACoreDataStack.m

@implementation KACoreDataStack

+ (instancetype)defaultStack {
  static KACoreDataStack *stack;
  static dispatch_once_t onceToken;

  dispatch_once(&onceToken, ^{
    stack = [[KACoreDataStack alloc] init];
  });

  return stack;
}

- (instancetype)init {
  if (!(self = [super init])) {
    return nil;
  }

  NSAssert([NSThread isMainThread], @"init must happen on main thread");
  _mainThreadContext = ...;
  return self;
}

@end
{% endhighlight %}

This is the standard way to initialize an object that may be accessed on multiple threads to ensure that the initialization code runs only once; if two threads reach the `dispatch_once` call at the same time, the initialization will happen only once and the second thread will block until the initialization completes.

However, initialization for a Core Data main-queue context has to happen on the main thread or else deadlock can occur. Here, if `[KADefaultStack defaultStack]` is called on a background thread before it's called on the main thread, the initialization will happen on the wrong thread and consequently fail.

## First attempt

My first attempt to fix this was to check within the dispatch block whether we're on the main thread and if not, dispatch to the main thread to do the initialization:

{% highlight objc %}
+ (instancetype)defaultStack {
  static KACoreDataStack *stack;
  static dispatch_once_t onceToken;

  dispatch_once(&onceToken, ^{
    if ([NSThread isMainThread]) {
      stack = [[KACoreDataStack alloc] init];
    } else {
      dispatch_sync(dispatch_get_main_queue(), ^{
        stack = [[KACoreDataStack alloc] init];
      });
    }
  });

  return stack;
}
{% endhighlight %}

Unfortunately, there's a race condition here. If `+ defaultStack` is called on a background thread then is called on the main thread immediately after, the main thread's `dispatch_once` will wait for the background thread's `dispatch_once` to finish, but the `dispatch_sync` call will wait for the main thread to finish its current run loop, causing deadlock.

## Second attempt

I then tried refactoring so that `dispatch_once` is always called from the main thread:

{% highlight objc %}
+ (instancetype)defaultStack {
  static KACoreDataStack *stack;
  static dispatch_once_t onceToken;

  dispatch_sync(dispatch_get_main_queue(), ^{
    dispatch_once(&onceToken, ^{
      stack = [[KACoreDataStack alloc] init];
    });
  });

  return stack;
}
{% endhighlight %}

Unfortunately, this deadlocks consistently if ever called from the main thread since `dispatch_sync` again waits for the main thread to finish, not realizing that we're already on the main thread.

## A working solution

Our last solution was completely broken but gives us the pieces we need to build a working one. When we're on the main thread, we want to perform the initialization immediately; when we're on a background thread we want to dispatch to the main thread then initialize. We can write this logic as follows:

{% highlight objc %}
+ (instancetype)defaultStack {
  static KACoreDataStack *stack;
  static dispatch_once_t onceToken;

  if ([NSThread isMainThread]) {
    dispatch_once(&onceToken, ^{
      stack = [[KACoreDataStack alloc] init];
    });
  } else {
    dispatch_sync(dispatch_get_main_queue(), ^{
      dispatch_once(&onceToken, ^{
        stack = [[KACoreDataStack alloc] init];
      });
    });
  }

  return stack;
}
{% endhighlight %}

This works, though is inefficient in the common case that we're fetching the already-initialized stack from a background thread; we'll end up dispatching to the main thread even if it's completely unnecessary.

We'd prefer to skip the dispatch_sync call if the initialization has happened already. The static variable `onceToken` is set by default to `0` and if we look at the [libdispatch source](http://www.opensource.apple.com/source/libdispatch/libdispatch-339.1.9/dispatch/once.h), we can see that it is set to `~0` once the block has run. (Note that `DISPATCH_EXPECT` expands to `__builtin_expect`, a [compiler primitive](http://llvm.org/docs/BranchWeightMetadata.html#builtin-expect) provided to help with branch prediction.)

## A reusable helper

We can take advantage of this and write a reusable helper for solving this problem:

{% highlight objc %}
void dispatch_once_on_main_thread(dispatch_once_t *predicate,
                                  dispatch_block_t block) {
  if ([NSThread isMainThread]) {
    dispatch_once(predicate, block);
  } else {
    if (DISPATCH_EXPECT(*predicate == 0L, NO)) {
      dispatch_sync(dispatch_get_main_queue(), ^{
        dispatch_once(predicate, block);
      });
    }
  }
}
{% endhighlight %}

With this helper, we can simply call `dispatch_once_on_main_thread` instead of `dispatch_once` in `+ defaultStack`, and the initialization will always happen on the main thread.

Arguably, setting up Core Data shouldn't require being on the main thread, but it does as of iOS 7. This helper is useful for this case and others where thread-unsafe initializers need to be run.

---

**Update (2018/02/27):** [Stephan Tolksdorf warns](https://github.com/facebook/react-native/issues/18096) that this technique is risky because it can deadlock if the main thread is already blocked on the thread running this code and notes that newer versions of libdispatch have slightly different implementations which may be more correct.
