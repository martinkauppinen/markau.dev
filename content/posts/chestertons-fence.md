---
title: "Chesterton's Fence"
date: 2023-01-30T18:27:46+01:00
author: Martin Kauppinen
---

A short post to tell a little cautionary tale about being too zealous with what
you think should or shouldn't exist in a code base.

At work today I ran into a small problem where I was doing some refactoring and
to make a service talk to another service in a more efficient manner. The short
of it is that my team's service -- `service-a` -- tells another team's service
-- `service-b` to perform some action, and then `service-b` responds with an
acknowledgement that the action was performed or rejected for some reason.
`service-a` can also tell `service-b` to undo a previous action.

Well, in my infinite[^infinite] wisdom, I decided in my refactoring that any
request by the application using `service-a` to undo an action that `service-b`
had not acknowledged positively should simply be skipped. After all, if the
action never happened in the first place, it cannot be undone. Previously it
just sent undo-requests regardless.

[^infinite]: Very much finite.

Oh boy, then a bunch of tests started failing. Turns out it is expected to undo
actions anyway if it seems `service-b` has timed out after several retries.

I asked a colleague after some thought and then we realized that it is
absolutely possible that `service-b` _did_ take the action, and even tried to
acknowledge it to `service-a`, but that the message got lost in transmission. So
we _should_ send an undo-request anyway because it's possible that it is simply
the connection from `service-b` to `service-a` that has degraded and everything
else worked perfectly fine. Removing my code to skip and remove undo-requests
from the queue fixed the issue and all tests went back to passing again.

I will leave [Chesterton's
fence](https://en.wikipedia.org/wiki/G._K._Chesterton#Chesterton's_fence) up for
now.
