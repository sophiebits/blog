---
layout: post
title: What I did at Khan Academy, 2012 edition
---

This week marks the end of my second summer working at [Khan Academy](http://www.khanacademy.org/). Inspired by [David Hu's internship post-mortem](http://david-hu.com/2012/01/05/khan-academy-internship-post-mortem.html) and [Jamie Wong's end-of-summer braindump](http://jamie-wong.com/2012/08/22/what-i-did-at-khan-academy/), here's a summary of what I worked on this summer.

This summer I worked mainly on the our "people helping people" team (abbreviated PHP, of course) on improving the community and encouraging discussion around the content on our site, with an emphasis on Q&A.

## iPad app

Before starting on my main work for the summer, I had to get out a few bug fixes for our iPad app ([iTunes link](http://itunes.apple.com/us/app/khan-academy/id469863705?mt=8)), which we [launched](http://www.khanacademy.org/about/blog/post/24905749288/ipad-app) earlier this year.

In particular, Facebook deprecated the [offline_access OAuth permission](http://developers.facebook.com/roadmap/offline-access-removal/) that we were using when users log in to Khan Academy with a Facebook account, so after making the appropriate server-side changes to ensure nothing would break, I needed to go through all of the app code to make sure that a 401 Unauthorized response from our API would always be handled appropriately.

## Exercise framework

Last summer, I worked almost entirely on our open-source exercise framework, [khan-exercises](https://github.com/Khan/khan-exercises). Even though I wasn't working directly on exercises this summer, I still ended up making some small improvements to how exercises are generated. Here's one example:

In order to be able to recreate past problems, we use our own pseudo-random number generator so that we can easily save the per-problem random seed when logging a user's progress to the server. However, the `word-problems.js` library I wrote to allow exercise authors to insert random names into word problems shuffled each array of names when the library was initialized, causing as many calls to `random()` as there were elements in the arrays that were being shuffled. That meant that if we ever changed the list of names, we'd end up invalidating all of the seeds that were stored and showing students the wrong things when they look back at completed problems.

<img src="/images/2012-08-24/word-problems.png" width="560" height="200">

To fix this problem, I [committed a change](https://github.com/Khan/khan-exercises/commit/af7ead9a4f2986f5f6fb699f2724b16036c895dd) that causes `random()` to be called only when it's needed. Now if you ask for a random piece of furniture (or anything else that we have in the file), only one call to `random()` happens which allows the random seeds to stay the same over time. At worst, a student will see a different name when she looks back at an old problem, but the numbers will all be the same.

## New incentives for discussion participation

My first real PHP-related project was to create new incentives for participation in discussion. I added several new badges, including two new instances of our heavily-coveted sun badges:

<img src="/images/2012-08-24/badges.png" width="560" height="110">

Adding these badges involved a small refactor of how badges are awarded throughout the site as well as new code to track per-user statistics about discussion participation so we are able to quickly calculate whether a user should be awarded a badge such as Oracle, pictured above: "Post 100 answers that earn 3+ votes".

## EC2 configuration scripts

Though we use [Google App Engine](https://developers.google.com/appengine/) for essentially everything on our main website, we run several virtual machines on [Amazon EC2](http://aws.amazon.com/ec2/) to power things such as:

* an instance of Facebook's [Phabricator](http://phabricator.org/) for internal code reviews,
* [continuous integration code](https://github.com/Khan/gae-continuous-deploy) that runs tests and deploys to a staging server,
* the Khan Academy [API explorer](http://api-explorer.khanacademy.org/),
* [sandcastle](http://sandcastle.khanacademy.org/) for easily reviewing pull requests to khan-exercises
* a caching proxy using [Varnish](https://www.varnish-cache.org/) to support [Smarthistory](http://smarthistory.khanacademy.org/) as well as kasandbox.org, a domain which supports sandboxed execution for our [new interactive programming environment](http://ejohn.org/blog/introducing-khan-cs/).

In addition to open-sourcing most of the above code, we've open-sourced the system configurations in the form of setup scripts in our [aws-config repository](https://github.com/Khan/aws-config). This summer I created the configurations for the API explorer sandcastle, and the Varnish proxy.

## Hackathon

At our first-ever [healthy hackathon](http://bjk5.com/post/26810034751/how-we-ran-the-first-khan-academy-healthy-hackathon), I worked on a few small projects in addition to helping other people out with any questions they had:

### Hackathon project #1

I worked with Jessica Yuen and Matt Wahl to redo our [careers page](http://www.khanacademy.org/careers) to be more colorful and have more information than the bland Jobvite page it replaced. Hopefully it will encourage more awesome people (like you, perhaps) to apply for some of our jobs. Don't miss the fun rollover picture at the bottom of the page!

<a href="http://www.khanacademy.org/careers"><img src="/images/2012-08-24/careers.jpg" width="560" height="200"></a>

### Hackathon project #2

When we deploy a new version of the site to Google App Engine, it's nice to know when it's been set to be the live version. We already had a notification for the initial deploy (in purple), so I wrote a [new script](https://github.com/Khan/gae-default-version-notifier) to send notifications (in green) so we know when all our users are seeing the new deployed code without having to check in the App Engine admin dashboard.

<img src="/images/2012-08-24/mr-gorilla.png" width="560" height="200">

### Hackathon project #3

We haven't yet had the time to deploy this yet, but I wrote an interactive scientific calculator that we'll add to some of the harder exercises so when students are working on solving systems of equations, they won't need to worry as much about the nitty-gritty arithmetic.

<img src="/images/2012-08-24/calculator.png" width="400" height="350">

The calculator uses a [Jison](http://zaach.github.com/jison/) parser -- in the future, we might adapt it to allow the entry of arbitrary algebraic expressions for exercise answers.

## Topic discussion pages

My largest project this summer was the creation of per-topic discussion pages -- previously, if you wanted to answer community questions about (for example) algebra, you'd need to go to each algebra video and look for unanswered questions.

I worked on integrating discussions into our topic pages -- now it's much easier to find questions to answer as well as learn from the answers of others. In addition to being able to answer without leaving the page, you can click on the name of a video to bring up a modal player with that video.

<a href="http://www.khanacademy.org/cs/d"><img src="/images/2012-08-24/cs-d.png" width="560" height="200"></a>

Thanks to Jamie Wong's great work, discussions are available for the new interactive programming environment in addition to the videos -- pictured above is some of the discussion for the Computer Science topic page. We've already started to see discussion activity increase and we're hopeful that this is the start of a great community.

## Miscellanea

Though they aren't big enough to deserve their own sections, I made speed improvements and bug fixes to many components including exercises, the deploy script, the homepage, the blog, the OAuth login flow for API clients, and our A/B testing framework.

I also gave a tech talk about [CSS](http://www.w3.org/Style/CSS/Overview.en.html) -- I focused on the basics of the box model and how to create simple layouts using CSS. They probably won't be very useful without audio or the demo that took up most of the time, but I've [posted the slides](https://speakerdeck.com/u/spicyj/p/css) anyway.

## Mentoring and organizing team events

Finally, I stepped into a [mentorship role](http://bjk5.com/post/23266999170/how-intern-mentorship-works-at-khan-academy); for the entire summer, I mentored one of the other interns, taking responsibility for his code reviews, figuring out which projects he should work on, and answering any questions he had about the code.

I also organized a couple of team events, including our second-ever [FIXIT day](http://bjk5.com/post/21317611897/step-one-fix-step-two-it) where the whole dev team took a day to convert our [Jinja2](http://jinja.pocoo.org/) templates to use autoescape, fix gaps in our exercise coverage and bugs in our existing exercises, add documentation to existing code, and anything else that falls in the category of cleaning up our codebase.

## Conclusion

Khan Academy is a phenomenal place to work -- we have plenty of interesting problems to work on and countless amazing team members to tackle them with. I'm really glad that I returned for a second summer, and I'd encourage anyone interested to [apply for a job](http://www.khanacademy.org/careers) -- we're hiring both full-timers and interns year-round.
