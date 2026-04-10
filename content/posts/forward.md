+++
date = '2026-04-09T12:58:54-07:00'
draft = true
title = 'Building an LMS with 0 Experience'
+++
![](attachments/Pasted%20image%2020260409130133.png)

# But Why?
In January of 2025, multiple universities across the west coast received an email from Dr. Annee Grayson asking if they could share a job posting to their Computer Science students. The task at hand was to design a fully-fledged Learning Management System with enough data collection to defend a thesis on, as well as run with the minimal funding provided to conduct the research itself.

I was one of a lucky few that made it through the interview process (despite, at the time, having very obviously canned resume), and showed up bright eyed to the first meeting with my peers where we would discuss our plan of attack to finish the job.

At the time, my *modern* web development experience was very minimal, with my only finished projects being a pure HTML/CSS/JS [homepage](https://varphi.online), a JS/WASM [graphing calculator](https://graph.varphi.online/), and a Svelte-based [course search](https://navarch.varphi.online/), all of which were nowhere near "complexity" of a large scale CRUD app expected to handle many regular users, and not look like a backend developer's dream functional frontend.

We quickly came to a tech stack that fit the strengths of our team of 5, and would keep maintenance cost down:
- Static React
- Django/DRF
- PostgreSQL

All running on AWS, as they have generous low-usage tiers that are historically reliable, and technically scalable.

Aside from the basic goals, we also needed to provide extensive accessibility features due to the nature of the research, as well as surface some kind of interface for Annee to manage both users and lessons herself to solve the [bus factor](https://en.wikipedia.org/wiki/Bus_factor) of such a small team comprised of students with uncertain futures.
# But How?
Initially the team had 5 members, two backend, two frontend, and myself, who would interface between the two to make sure no mix-ups happened and help where it was needed.

That did not last long.

One of the frontend developers had extenuating circumstances in, and had to step away from the project within the first two weeks, and the other had extremely minimal experience outside anything that would be covered in an "Intro to Web" class, and so needed time to look into what Node/Vite was, as well as working with a REST API, and so, I found myself as the sole frontend developer for the time being.

After a month of frantic research, reading docs, and discussing design with the backend team, I found myself comfortable enough with React, Redux, and  {{< side "Docker" right >}} I cannot stress how useful Docker/Compose was after a certain point of development when working with 3 people on 3 different platforms. Adding mock services like [Minio](https://www.min.io/) for S3 and Postgres, all in a way that live reloaded with file changes was a game-changer.{{< /side >}} to develop at a pace that would let us finish by the end time we had been given.

## What does it take?
In short, if you cut out all of the winding paths I and my teammates wandered down throughout the journey, I'd say this project could have been completed in 6 months. It took us just about a year, which does include a 3 month break during the summer, and bringing on a new member, [Lorran Alves Galdino](https://low-go.github.io/portfolio/), halfway through.

At some point early*ish* on, both of the backend guys needed to step away, and so it was just Lorran and I, and I became more central to the whole thing. Django provides an administration interface called their "admin panel" that gives direct access to modify fields of their ORM-Defined class instances. We ended up using this, plus some helpful plugins extensively to make the final product as hands-off as we could.
![The admin panel interface, editing the contents of a Slideshow activity.](attachments/Pasted%20image%2020260409190325.png)
*The admin panel interface, editing the contents of a Slideshow activity.*

This also led to developing heavily ORM-object based approach to how we handled the activities for rapid prototyping.

We created an abstract `BaseActivity` class and `BaseResponse` class that all activities would inherit from, then, registered with a global singleton `ActivityManager` which was then the {{< side "common interface to deal with regarding activities" left>}}
For example, in multiple places, we would need to aggregate over all activity/response objects related to a specific user, so you could loop through the registered activities to run your queries on. <br><br> In other places, like the REST endpoints, we would just take the `__cls__.name.lower()` and check the URL parameter against that. <br><br>Overall though, many places handled all activities and adding each one every time would have been a pain.
{{< /side >}}. It ended up looking like this:
```python
class ActivityManager():
    """A centralized management class meant to streamline
    the process of creating and using a activities within the backend.
    """
    
    def __init__(self):
        if self._initialized:
            return
        self._initialized = True
        self.registerActivity(TextContent, TextContentResponse)
	    ...
```

The class definitions on the python side were then mirrored with associated typescript interface definitions, as activity and response data were sent via a common JSON format.

After designing the system of defining new activities, came the actual implementation in the client. On the backend, most activities (excluding quizzes where questions must be "checked"), were just yanking response data into an ORM instance and `.save()`ing it. The frontend is where most of the "creative" functionality took place.


## Mistakes Were Made
One of the design choices I made very early on into the project, was going for a brittle "seamless" saving approach that relied entirely on the component actually displaying the activity, sending response data when it became unmounted, with no error handling and component-local storage of the data itself.

This led to bugs where, data would be deleted as the user would return to an activity before the server responded, and in doing so, the client would not have their previous response, effectively wiping it from the server once the client navigated away again.

When initially designing this hook, about a month into React, I thought I was genius... You would use it just like a setState(), and all the internals were abstracted for developer to focus on functionality and nothing more. Eventually, I ripped this out in favor of a more robust, yet "visible" option where any attempts to navigate without saving were given a warning, and progression only occurred on a 200 OK from the server.

## Others Were Not
Now th