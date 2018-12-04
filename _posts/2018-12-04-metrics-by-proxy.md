---
layout: post
title: "Metrics by proxy"
---

It's usually hard to measure the things we care about. So we compromise.

<!-- more -->

What makes a metric good? In an ideal world, you could pick the thing you want to make better (say, "How productive are my users?") and then measure it precisely. But since it's rare to be able to take realistic measurements in reply to such a question, we need to come up with something else that we can actually measure. If you look around, you'll find that most metrics we use are only an approximation of the real thing we care about. So I've been calling these *proxy metrics*.

I've noticed this pattern in several places, so in this post I'll walk through how this works from multiple angles.

**The technical side:** Let's take web performance. We care about how long things take on real users' devices. But it's hard to track that. Even if you set up the telemetry to collect performance numbers, they're usually noisy.

* If you're trying to decide if a particular change regresses page load time, production data is very accurate but it likely won't detect small changes.

* So [maybe you set up a controlled environment](https://code.fb.com/web/browserlab-automated-regression-detection-for-the-web/) where there are fewer sources of noise. This means your numbers are more repeatable but they're less accurate (since the lab machines may be unrepresentative). Reducing the noise like this helps, though in my experience it's still hard to measure small regressions.

* So [maybe you use a tool](http://sqlite.1065341.n5.nabble.com/50-faster-than-3-7-17-td78082.html) like [Valgrind's Cachegrind](http://valgrind.org/docs/manual/cg-manual.html) to make your benchmark runs even more repeatable, so that you can detect 0.05% changes or smaller. This also isn't a free lunch: Valgrind simulates a CPU but it won't behave identically to the real ones you care about – so it is yet more precise but still perhaps less representative.

* I tend to think that using Valgrind is a pretty good place to end up. It's a good compromise between accuracy and precision. But there's also another direction you can go. Sometimes it's nice to just have a really understandable, predictable, precise metric that people can drive down. Like file size.

  If we send fewer bytes of JS to the client, our pages will generally be faster. And this is an easier goal to rally people around – people can look at a piece of code and think about what they need to do to cut down on its size. It's can be a lot easier than predicting if something will run faster. If you get 20 people to cut down 5kB each, you've removed 100kB. And that's almost certainly a practical improvement. Even though it's not directly correlated, it's easy to count and it's usually *directionally accurate*. That is, improvements in file size tend to improve performance, and regressions in file size tend to regress performance. So it turns out to be useful.

**The management side:** Let's say you're running a company and you're trying to figure out how to evaluate your employees. Your ideal metric might be, "How much value did this employee generate for the company during this time?"

* You can't measure that, so you try to look at what you _can_ measure. You reward people for improving performance metrics and for improving product metrics. (That's one reason that you want to be able to measure performance improvements precisely. These metrics themselves are a proxy for "How well is the company doing?") You reward employees for the number of interviews they did, and the number of new people they mentored. It's not an actual formula, but it's an attempt to be fair and relatively repeatable.

* Many managers know the struggle of saying, "Spending time on that would be useful, but it probably won't count for much at performance review time". That represents a mismatch between the actual value to the company and measured value to the company. If that happens a lot, it probably means your proxy metric is bad and you should find something better to evaluate people on.

* When designing a performance review system, the tragedy of the commons can bite you. If no individual person feels that a particular type of work is rewarded, it likely won't get done. For example, take testing. If creating automated tests isn't valued by The System, people won't write them. And then you have no tests. If you need those tests to keep the machine well-oiled – well, you're gonna have a bad time. So you, the performance management overlord, notice this and revise your system. You realize that the menial work of writing tests, refactoring, and fixing bugs is undervalued. So you amend your measurements to count this sort of work explicitly and avoid the negative externality. If you do it right, your problem with bad engineering practices might start to go away.

**The engagement side:** About those product metrics I mentioned. Let's say it's 2004, and you're building Facebook. How do you know if you're doing a good job? Maybe your first instinct is: if people use the site more, they're probably getting value out of it. (Note: This might not be how things actually happened. I work at Facebook, but not on this.)

* You want to know whether your changes are making the site better or worse, so you set up your A/B testing tool to measure how much people are using the site. In practical terms, you're looking at metrics like the number of likes, the number of comments, the amount of time spent on the site. You figure that this isn't a perfect representation of "Are people getting value out of this?", but it's a step in the right direction. It's directionally accurate.

* One day, you reflect and realize that those aren't the best metrics. People are spending a lot of time on your site and they don't feel that time is well spent. You run research studies, and you find that people who spend a lot of time passively consuming content aren't as happy. So [you decide that you want more "time well spent" and more "meaningful social interactions", that the way to do that is to emphasize "friends, family, and groups"](https://www.facebook.com/zuck/posts/10104413015393571).

* Well, what does that actually mean? Your research tells you that people value receiving comments more than likes. (Say, about twice as much.) And that people value interactions with friends more than with non-friends. So you come up with a formula that looks something like **3 &times; (2 &times; friend comments + friend likes) + (2 &times; other comments + other likes)**. And then you find out that people think reactions are more meaningful than likes, so you [weigh reactions higher in your formula too](https://mashable.com/2017/02/28/facebook-reactions-news-feed/).

At the end of the day, you've created some formula that purports to measure how meaningful people are finding their interactions. The formula looks silly. Dry.

Could it possibly know what people are feeling? Well, it actually kinda does! Because your user research told you that people value comments twice as much as likes. And so your formula is your best attempt at actually codifying that into science. It's not perfect, but it's based in human interaction.

It'll have to do. At least until your next big realization, when you'll pick a better metric.

---

*(Thanks to [Ada Powers](https://twitter.com/mspowahs) for reviewing.)*
