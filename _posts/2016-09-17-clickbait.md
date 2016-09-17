---
layout: post
title: I gave commit rights to someone I didn't know, I could never have guessed what happened next!
date: 2016-09-17
type: post
published: true
status: publish
comments: true
categories:
- Technology
tags:
- open-source
meta:
  _edit_last: '1'
author:
  login: jake
  email: jakewins@gmail.com
  display_name: jake
  first_name: ''
  last_name: ''
---

(Spoiler: trusting your contributors works)

Some years ago, I polished up and released an abandoned project for storing financial data in Django.
It let you declare "Money" fields on your models, dealing with proper storage and currencies for you.

My use case for the library, `django-money` eventually faded, but it ended up teaching me a useful lesson in trust and OSS abandonware.

<!--more-->

Some time after the project I was using `django-money` for was binned, I read a blog post about trusting OSS contributors. 

I don't recall the author now, but the gist of the argument made was that we're too protective of our code - if you give someone responsibility,
show that you trust them, more often than not, your intuition about people abusing their freedom is way off. 
On the contrary, giving a diligent contributor commit rights will often have the opposite effect from what you might expect, making people take 
their contributions even more seriously.

Obviously this has caveats and depends on what your project is - but the gist of the post really struck a chord. 

A few weeks later, I was going over issues and contributions for the projects I maintained on Github, when a massive PR emerged for `django-money`. 
Someone had fixed tons of bugs and orchestrated a much larger group of contributors around a fork they'd made of my project, and now they were asking me 
to merge their fixes back upstream.

The [PR](https://github.com/django-money/django-money/pull/2) was bigger than what I felt I could sensibly review and, in honesty, my desire 
to go through the hours of work I could tell this would take for a project I no longer used was not stellar. 
But I remembered that blog post. 
So, instead of doing what I'd done previously when someone sent hard-to-review PRs to other projects and close them with a short note on how 
to break it into reviewable parts, I did something different.

I opened the project settings, and I gave this person I'd never heard of commit rights to the repo.

And then I wrote a comment on the PR saying as much:

![Comment]({{ site.baseurl }}/public/assets/2016-09/commit-rights.png)

And then I forgot all about it.

## A few years later

I'm having lunch with some friends, and one of them mentions that they were reviewing django addons to handle money.
I say, that's funny - I've got a project on Github for doing that, `django-money`.

To which my friend says "YOU built django-money!?? That was one of the libraries we were reviewing!"

I'm entirely startled - how had they found my obscure little library? I browse over to my github profile, and realize
something I had not before - `django-money` was, by far, the most popular repository I have. As in, all the 
other repos with one-off things had a few stars, `django-money` had several hundred!

So, I open it up, and it was like walking into a factory filled with people I'd never met, PRs flying. It was a very odd
feeling, since it was all happening on my Github page, right under my nose!

After digging some, it turns out that [Greg Reinbach](https://github.com/reinbach), the guy who opened the PR, had eventually
done basically the same thing I did, and given a third person commit rights, [Benjamin Bach](https://github.com/benjaoming).

Benjamin had since then, for several years at this point, quitely been maintaining the project, reviewing PRs that people sent,
making sure CI hummed along and so on. And as he had, the project had slowly but surely gained more and more adoption, to the 
point where it was now being considered to be used for a project at the crowdfunding platform where my friend worked.

## Wrap-up

We eventually moved the project into it's own org, Benjamin is still maintaining the project, and the world is a tiny bit 
better at building web-apps because of it.

So, I guess this is my way of saying thanks, whoever you are, who wrote that blog post back in 2012; you were right.

