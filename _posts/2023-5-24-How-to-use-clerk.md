---
title: "How to use Clerk for Auth and keep your own user db in sync"
date: 2023-5-24
---

In this post I'll walk through how I'm using Clerk to authenticate users in my web app as well as how I'm keeping a copy of user data in my own db up to date.

This post started off as an ADR for myself, but I thought if I share it as a blog post others can tell me how to do it better, or at worst you can do it the same way as me. This particular project uses nextjs 13+ App Router, React Server Components and is deploying on Vercel.

So before launching into things, I need to say that the solution is like B+ but it's the best I can see right now. I (might) update this post if and when I discover a better one.

## Why bother with an auth provider at all?
I don't want the hassle of setting up or holding onto sensitive or secure data for people.

## Why Clerk?
I initially picked Clerk because Theo used it in the T3 app tutorial video.

I stuck with it because it offered React Server Component (RSC) support where alternatives like AuthO don't (yet).

Authjs (formerly next-auth) does offer RSC support and is an option we might pursue later, especially since it was part of that initial T3 stack. However in Theo's words Clerk is the better option especially if mobile is on the table. We do plan on going to mobile with this so I'm taking his word for it.

And yes a lot of this is "because Theo said" and the reason for that is I'm coming back to web dev after a few years and need to trust these influencers a little bit.

### Support - Thank you!
Clerk's support has been excellent and responsive in Discord with the variety of issues I've had during the setup process

## Sources of Truth

Clerk is where users create, manage and delete their user data. Because of that me must use Clerk as the source of truth for "if a user is authenticated" with our application and "who that user is"

### Effect of Rate Limits on Sources of Truth

> All API requests are subject to a default limit of 50 requests per 10 seconds.

However due to rate limits on Clerk's api we can't rely on it to get that data. We store _just the data we need_ in our own user table and keep it up to date via Clerk's webhooks. Instead of calling directly to Clerk when we need user data we access our own user table.

## How We Keep in Sync with Clerk

**tl;dr We use a redirect on sign up to create the user in our db and then we keep it up to date with webhooks**

If we keep it for a month or two, I'll come back and make a proper sequence diagram but simply:

### Flow A: User Sign Up Redirect

We use a Clerk UI Component redirectUrl prop to redirect the user to our api when they sign up.
Our api endpoint uses Clerk middleware to get the userId. We use the clerkClient to get the full user object and insert it into our db.
We then redirect the user to our app.

We perform an upsert just in case Route B (see below) has somehow beaten us to it.
```
our app                clerk           our api       our db
   |---(afterSignUpUrl)--->|                |            |
   |                       |--------------->|            |
   |                       |<--getUser(id)--|            |
   |                       |--------------->|            |
   |                       |                |----(user)->|
   |<------------(redirect to /)------------|
```

### Flow B: Webhook

If Route A fails to write the user in our DB (e.g. the user closes the tab before redirect) then we'd be in a bad situation.
That bad situation is Clerk has the user in its records, but we don't. Put another way the user is authenticated for our app but we don't know who they are.
It's like showing up to your first day of work with your id card but no one knows who you are, and you don't have a desk, and no one's provisioned your laptop, and I don't think you're on the payroll either.

So as a fallback we're using a webhook that Clerk offers (through svix) that will hit our endpoint with the user object when it's created.

We perform an upsert as it's likely that Route A has succeeded already.

```
clerk         our api       our db
  |              |            |
  |- (user)----->|            |
  |              |---(user)-->|
```

### Flow C: Human

If Route A and B fail then I haven't got an automatic solution. The user could visit the our api directly _as if Clerk had redirected them_ and it will work just fine, but they won't know to do that. We'll need to let them know.

### Alternative Flows

A suggestion that came from Clerk's support is to define an [afterAuth](https://clerk.com/docs/nextjs/middleware#using-after-auth-for-fine-grain-control) function in the Clerk Middleware.

That'd require doing everything that we do in Flow A but on _every page transition_.

That's **one extra blocking request and one extra blocking db call, for every page**. I'm reluctant to add that overhead but will consider it as it's the most reliable way to make sure that a user has their data in both Clerk _and_ our DB.

However one additional caveat of this is that it will require every user to visit the site after signing up in order for them to be added to our DB. So this Flow alone does not guarantee that Clerk and our DB are in sync.

### Flows in Summary

We use multiple ways to keep Clerk and our DB in sync. We prefer Flow A - User Sign Up Redirect as it provides the most synchronous way to onboard new users. We use webhooks as a fallback to keeping our data consistent. We rely entirely on webhooks to keep our user data up to date.
