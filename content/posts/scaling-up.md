---
title: "Scaling up"
date: 2015-10-20
---

You've heard about it, thought about it and probably executed it. You've scaled your application to deal with a new amount of users. Nowadays, with services like Heroku (which we'll focus on here), this becomes relatively easy. Either by using a bigger dyno or adding more, you're "fine".

This works when this is a steady level of extra traffic. When you have linear growth and no spikes. Often, this is not the case though.

### Unexpected scaling

You might have had to deal with unexpected scaling. An article being posted on Hacker News for example, giving you a huge spike in visitors, something you didn't foresee.

Scaling this is hard, you don't know when this will happen. You don't know what the impact will be. There are addons on Heroku for this which will look at your application response time. If this starts to fall under a certain value, it will add more dynos.

### Timed scaling

Now imagine that you know when you will have an additional amount of traffic — like TV adverts — and you know roughly how much traffic you will have. For example, next week, you will have a peak in visitors at 3pm every day, which lasts about 30 minutes.

Scaling this is relatively easy. You calculate how much capacity you need, go to your Heroku dashboard and scale up. That's it, time to rest now.

The problem with this is that for 23.5 hours each day you are paying for capacity you don't need. The solution to this is straightforward. At 2:30pm every day, you go to your dashboard and scale up. At 4pm you go to your dashboard again and scale down. Great, works! Money saved, everyone happy!

Now, what if this happens 3 times a day — every day. One of them at 2am which lasts 1h. This would mean waking up in the middle of the night, scaling up and not forgetting to scale down again. You could argue to do this upfront, but if you have this going on for 2 months, this might come in costly.

## Introducing Heroku Timed Scaler

I've had to do the previous before. A lot.

Heroku has an API, a pretty good API. This API provides a way to scale up. That's why I've built a tool — spoiler, there are similar tools out there — that takes a range of time slots with settings and a time frame to scale your application.

![scaling-up](/images/scaling-up-000.png)

You create a new slot which will then be queued to scale. You define the type of process you want to scale, the size you want to scale to and the quantity of dynos you want. Once the time slot is over, the application will scale back to the settings of your application just before it scaled up.

The great thing about this compared to other services that do this? It's [open source](https://github.com/jelmersnoeck/heroku-timed-scaler).
