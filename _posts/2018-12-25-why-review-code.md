---
layout: post
title: "Why review code?"
---

A friend asked me recently why it's valuable to do code review. Most Silicon Valley tech companies, at least, do code review on every change to get at least two sets of eyes on it. At one of my earlier jobs we did opt-in code review (rarely) for a while, then a new employee coming from Google joined us and encouraged us to review all our code -- which we did. It was a great decision.

<!-- more -->

If you do it right, code review shouldn't feel onerous. You and your reviewer aren't adversaries; you're working together to build the best software you can together. (It's important to not take feedback personally -- even if your code needs to be changed, that doesn't mean *you* are problematic. It's normal to get feedback, and that's what helps you grow!)

Some companies have complicated rules about how many people have to review each piece of code, with strict "owners" for who is responsible for each piece of code. I've never felt that to be necessary; I prefer a simpler system where the only rule is that each piece of code must be reviewed by one person. In practice you'll still send reviews to the people responsible for maintaining the particular code you changed, but it's nice to not have the hard requirement.

Here are the biggest reasons I thought of for why code review is valuable. There's a lot of them!

1. **The code itself.** The most obvious value in code review is often framed in "catching bugs". Or if you look a little further, catching cases where there are best practices or unspoken rules that the author didn't know about, where a reviewer can help make the actual concrete code better by responding to it.
2. **Macro-level knowledge sharing.** When you review other people's code, you tend to learn new techniques that can benefit you later -- and vice versa, if someone suggests a better way to do something when your code is under review. If you can take what you learned and use it later, you'll grow as an engineer.
3. **Micro-level knowledge sharing.** Or, mitigating the "[bus factor](https://en.wikipedia.org/wiki/Bus_factor)" by increasing the number of people who are familiar with any given piece of code.
4. **Direction sharing.** Relatedly, code review forces you to communicate to teammates what you're doing, which helps ensure that you don't end up going in a wrong direction for days or weeks by giving them a chance to push back.
5. **Communication practice.** Being able to communicate clearly, both within a team and beyond it, is one of the most important skills for succeeding at work! Code review gives you a chance to practice writing clearly, both in describing the purpose of a change and when giving feedback on one. With any luck, you'll be more prepared for the next time you need to write something that "really matters".
6. **Historical record.** In my experience, people tend to write _much_ better commit messages when they know someone's reading them. That's often useful later when looking back at old changes!
7. **Something to discuss.** When you're trying to agree on what change to make, it can sometimes be hard to verbally describe and agree on the specifics of, say, a particular algorithm. Communicating over a piece of code can be more precise, since code tends to be unambiguous.
8. **Team cohesion.** When you do regular code reviews, work feels a little more like a single team working together instead of each person operating "in their own lane". 
9. **Reading practice.** Getting practice reading other people's code helps you remember how to make your own code more readable (and thus, maintainable). Which leads to better code forever after!

If I had to pick, reasons 2, 5, and 6 probably stand out to me as the most valuable.