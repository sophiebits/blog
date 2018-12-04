---
layout: post
title: "Yak Shaving and Fixing"
---

I have a niche at work: I like to fix stuff.

<!-- more -->

I'm not as good at big, long projects or at writing lots of code from scratch. But this works well for me. I especially like when there's something Big and Important that's super broken and needs to be fixed Really Soon -- I can really get into it. (The adrenaline helps. I'm a competitive person so I like racing to see how fast I can find the fix.)

My strategy here for being useful is different from what most people do but it works well for me. I think there are a few elements that contribute to making this be a success.

I know everyone always says that [yak shaving](https://seths.blog/2005/03/dont_shave_that/) is bad, but I'm not so sure. If I'm trying to use some tool we've built and then I notice that it's buggy in an annoying way, I often sit down to fix it. Or if the build system for whatever I'm working on is borked, is it a good idea to try to fix it for the next person or just work around it and leave it broken?

Conventional wisdom seems to suggest that I shouldn't dig in and fix it. It's probably true that my immediate goal would happen a little later. But I don't think that anyone else (including future Sophie) will ever feel as motivated to fix the annoying bug as much as I am in the moment that I hit it. And I'm making everyone else's experience just a little bit better when I do these fixes. One way I often like to think about behavioral questions is this: if everyone followed in these footsteps, would it make the whole organization (or community) overall better or worse? (Is it OK to tailgate when driving? No, because if everyone did then driving would be a hellish nightmare, plus traffic would be worse.)

Here, I think the answer to "Would we collectively get more done if we all stopped to fix bugs in infrastructure and tools when we hit them?" is a clear yes. Because our tools would get so much better, practically overnight -- and in turn that would speed us up. I personally fix [lots of](https://mobile.twitter.com/sophiebits/status/984562042852929536) [small stuff](https://mobile.twitter.com/sophiebits/status/1063180141872898048) in our tools and I think even my contributions alone have made a small but significant difference. (I use the word "tools" loosely. At work, we use Facebook for work communication, so even bugs in Facebook can be an excuse for me to do this and fix quirks. Being able to fix stuff yourself instead of waiting for someone else is one thing that is nice about working at a company that builds most of its own internal tools.)

To be able to fix effectively, you gotta be willing to dive into code that you didn't write. And into lower layers of the stack than you might be used to. I've gotten [pretty good](https://mobile.twitter.com/acdlite/status/1057085489591672832) [at this](https://mobile.twitter.com/dan_abramov/status/1058498922551955457). [I wish I knew](https://mobile.twitter.com/sophiebits/status/1058563330535616513) how to teach it. Mostly I think it takes a lot of practice. And it gets easier the more you do it: _so often_ when I'm fixing something, I'll lean on knowledge that I got from a previous fix I made. If you're newer to doing this, I'd just suggest you persevere: there is always a logical explanation for why a computer is behaving in a certain way, and you _can_ find it if you dig enough.

Finally: if you spend some time chasing a bug then give up, that's OK too! You probably learned something in the process, and you can always fix the next one.
