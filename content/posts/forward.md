+++
date = '2026-04-09T12:58:54-07:00'
draft = true
title = 'Building an LMS with Basically 0 Experience'
+++
![](attachments/Pasted%20image%2020260409130133.png)

# But Why Make Your Own?
In January of 2025, multiple universities across the west coast received an email from Dr. Annee Grayson asking if they could share a job posting to their Computer Science students. The task at hand was to design a fully-fledged Learning Management System with enough data collection to defend a thesis on, as well as run with the minimal funding provided to conduct the research itself.

The thesis centered on a 'comprehensive, accessible transition curriculum for youth with disabilities in juvenile justice.' That imposed some unusual constraints: facilities often had very restrictive whitelists, so the system needed to avoid extra integrations, third-party assets, and anything else that increased the odds of being blocked. A general-purpose LMS like Moodle or Canvas might have worked in one facility and be blocked in another, and even then would not have given us the instrumentation and data collection needed for the research.

I was one of a lucky few that made it through the interview process (despite having, at the time, a very obviously canned resume), and showed up bright eyed to the first meeting with my peers where we would discuss our plan of attack to finish the job.

At the time, my *modern* web development experience was very minimal, with my only finished projects being my [homepage](https://varphi.online), a [graphing calculator](https://graph.varphi.online/), and a [course search](https://navarch.varphi.online/), all of which were nowhere near complexity of a large-scale CRUD app expected to handle many regular users, and not look like a backend developer's dream functional frontend.

We quickly came to a tech stack that fit the strengths of our team of 5, and would keep maintenance costs down:
- React - Served Statically
- Django/DRF
- PostgreSQL

All running on AWS, as they have generous low-usage tiers, and a member of the team had previous experience that made it preferable to hosting on a VPS somewhere.

Aside from the basic goals, we also needed to provide extensive accessibility features due to the nature of the research, as well as surface some kind of interface for Annee to manage both users and lessons herself to solve the [bus factor](https://en.wikipedia.org/wiki/Bus_factor) of such a small team comprised of students with uncertain futures.
# Actually Building It
Initially the team had 5 members, two backend, two frontend, and myself, who would interface between the two to make sure no mix-ups happened and help where it was needed.

That did not last long.

One of the frontend developers had extenuating circumstances and had to step away from the project within the first two weeks. The other had extremely minimal experience outside of an "Intro to Web" class and needed time to learn about Node, Vite, REST APIs and other basics. So, just like that, I found myself as the sole frontend developer for the time being.

After a month of frantic research, reading docs, and juggling my university courseload, I found myself comfortable enough with React, Redux, and  {{< side "Docker" right >}} I cannot stress how useful Docker/Compose was after a certain point of development when working with 3 people on 3 different platforms. Adding mock services like [Minio](https://www.min.io/) for S3 and Postgres, all in a way that live reloaded with file changes was a game-changer.{{< /side >}}to develop at a pace that would let us finish by the deadline we had been given. Or so I thought...

## What does it take?
In short, if you cut out all of the winding paths I and my teammates wandered down throughout the journey, I'd say this project could have been completed in 6 months. It took us just about a year, which does include a 3 month break during the summer, and bringing on a new member, [Lorran Alves Galdino](https://low-go.github.io/portfolio/), halfway through.

At some point early*ish* on, both of the backend guys needed to step away, and so it was just Lorran and I, and I became more central to the whole thing. Django provides an administration interface called their "admin panel" that gives direct access to modify fields of their ORM-Defined class instances. We ended up using this, plus some helpful plugins extensively to make the final product as hands-off as we could.
![The admin panel interface, editing the contents of a Slideshow activity.](attachments/Pasted%20image%2020260409190325.png)
*The admin panel interface, editing the contents of a Slideshow activity.*

This also led to developing a heavily ORM-based approach to how we handled the activities for rapid prototyping. We needed a way to add new lesson activity types without rewriting the whole sections of the backend/frontend every time.

We created an abstract `BaseActivity` class and `BaseResponse` class that all activities would inherit from, then, registered with a global singleton `ActivityManager` which was then the {{< side "common interface to deal with regarding activities." left>}}
For example, in multiple places, we would need to aggregate over all activity/response objects related to a specific user, so you could loop through the registered activities to run your queries on. <br><br> In other places, like the REST endpoints, we would just take the `__cls__.name.lower()` and check the URL parameter against that. <br><br>Overall though, many places handled all activities and adding each one every time would have been a pain.
{{< /side >}} It ended up looking like this:

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

The class definitions on the Python side were then mirrored with associated Typescript interface definitions, as activity and response data were sent via a common JSON format.

After designing the system of defining new activities, came the actual implementation in the client. On the backend, most activities (excluding activities like quizzes where questions must be "checked"), were just yanking response data into an ORM instance and `.save()`ing it. The frontend is where most of the "creative" functionality took place. Here is one of my favorite examples.
### The Identification Activity
![The admin panel page for the Identification type, with a preview of selectable areas and a mouse-position indicator for making accurate selections.](attachments/Pasted%20image%2020260409200806.png)
*The admin panel page for the Identification type, with a preview of selectable areas and a mouse-position indicator for making accurate selections. In this case, a W-2 form selecting total wages.*

The Identification activity type was used less than most others, but solved a unique problem with learning virtually: confirm that the student understands the concept by actually pointing it out. As opposed to just circling something and calling it a day.

Internally, the object stores an image, and a list of rectangle vertices stored as percentages from an origin in the top left, as well as any of the fields it inherits from `BaseActivity`. On render, the image is positioned absolutely, and all rectangles are instantiated with their own onclick `EventListener`s that highlight and record the interaction.

This was made more difficult by the fact that the entire LMS is meant to support mobile devices, and with the images being quite large, had to be conditionally displayed through a full-screen scrollable modal that had similar functionality.

### Managing Accessibility
Due to the nature of the thesis' subject, and by extension the expected user base of the app, designing an accessible interface was one of the priorities outlined as a must from the start. This meant we needed to implement
- Global free-of-cost text-to-speech, with speed and voice options
- Custom markup parsing to provide inline definitions for less common words
- App themes, including a usable high-contrast mode
- Variable font size options
- Universal color/use semantics in design for clarity

while keeping the normal functionality of the app intact and unobtrusive.

Luckily, because this was a core requirement and one of the main reasons I joined the team, we baked this into the design of the app early on rather than tacking it on at the end.

In its finalized version, the text-to-speech and markdown parsing were consolidated into one wrapper component using `react-text-to-speech`, Remark, Rehype, and Hast. This allowed us to parse our modified [flavor of markdown](https://github.com/FORWARD-Curriculum/Forward-app/tree/main/Forward-client#markdown), allowing the UI to be TTS-compatible.

For example:
```html {.wrap-code}
There are many types of colleges and ways to receive <def def='Education or training that takes place after high school, like college, trade school, or apprenticeships.'>post-secondary</def> education. There are also different types of degrees that you can get, providing various levels of <def def='Special skills or knowledge in a subject.'>expertise</def>. Sometimes you won't know what a college offers until you ask, so it's all a matter of finding the perfect fit for you!
```
Ends up as:
![](attachments/Pasted%20image%2020260409232312.png)
## Mistakes Were Made
One of the design choices I made very early on into the project, was going for a brittle "seamless" saving approach that relied entirely on the component actually displaying the activity, sending response data when it became unmounted, with no error handling and component-local storage of the data itself.

This led to bugs where, data would be deleted as the user would return to an activity before the server responded, and in doing so, the client would not have their previous response, effectively wiping it from the server once the client navigated away again.

When initially designing this hook, about a month into React, I thought I was a genius... You would use it just like a `setState()`, and all the internals were abstracted for developer to focus on functionality and nothing more. Eventually, I ripped this out in favor of a more robust, yet "visible" option where any attempts to navigate without saving were given a warning, and progression only occurred on a 200 OK from the server.
## ...Others Were Not
As Lorran and I were building the functionality of the app, we were given the first lesson in the curriculum with all media included to properly prototype and test. This was great, but I noticed while debugging some networking code that even on a {{< side "cached refresh," right >}}
Due to the S3 backend we eventually fell upon for our media backend, and using [presigned urls](https://docs.aws.amazon.com/boto3/latest/guide/s3-presigned-urls.html) for security, browsers would not cache the image as it has a unique url each request.
{{< /side >}} the total data transfer would be somewhere in the *several megabytes*. After excluding dev dependencies like the icon library, and unminified source mapped code, the lesson media itself was obviously the culprit.

Annee would often use stock photos that were extremely high resolution, and other times would give us the raster version of a complex SVG, which was all in the name of supporting the content of the lessons as best as possible.

One of the last large tasks I took on prior to the finalization of the app was implementing a media delivery optimization system, taking a similar approach to how many platforms trancode their user-generated media into multiple fidelities for bandwidth saving purposes.

What I ended up with was a set of wrappers and utility functions to help interface with the [django-imagefield](https://pypi.org/project/django-imagefield/) library, which allows one to specify a set of formats, transformation functions, and aspect ratios that images are processed through upon upload to the media storage backend.

<details>
<summary>What that code ended up looking like:</summary>

```python
DEFAULT_IMAGE_FORMATS = {
    "mobile":  ("default", ("thumbnail", (480,  480))),
    "tablet":  ("default", ("thumbnail", (800,  800))),
    "desktop": ("default", ("thumbnail", (1500, 1500))),
} if settings.OPTIMIZE_MEDIA else {}

class FwdImage():
    _formats: dict[str, tuple[str, tuple[str, tuple[int, int]]]] = {}
    def __init__(self, formats: dict[str, tuple[str, tuple[str, tuple[int, int]]]] = None):
        base = formats if formats is not None else DEFAULT_IMAGE_FORMATS
        if settings.OPTIMIZE_MEDIA:
            self._formats = {
                "internal_default_thumbnail": ("default", ("thumbnail", (240, 240))),
                **base,
            }
    
    @property
    def formats(self):
        return {key: list(value) for key, value in self._formats.items()}
    
    def stringify(self, image: ImageFieldFile) -> dict[str, str | dict[str, int]]:
        if image is None or not isinstance(image, ImageFieldFile):
            raise TypeError("Passed image was not an ImageFieldFile")
        
        optimized: dict[str, int] = {}
        for key, value in self._formats.items():
            optimized[getattr(image, key)] = value[1][1][0]
            
        return {
            "thumbnail": image.internal_default_thumbnail if image.internal_default_thumbnail else image.url,
            "original": image.url,
            "optimized": optimized
        }
```

Which was used during serialization similar to:
```python
def to_dict(self):
        return {
            "id": self.id,
            "lesson_id": self.lesson_id,
            "type": self.activity_type,
            "title": self.title,
            "instructions_image": GENERIC_FORWARD_IMAGE.stringify(self.instructions_image) if self.instructions_image else None,
            "instructions": self.instructions,
            "order": self.order
        }
```

And finally on the frontend:
```ts
export function srcsetOf(image: Image) {
  let out: string[] = [];
  for (const [url, width] of Object.entries(image.optimized)) {
    out.push(`${url} ${width}w`);
  }
  return out.join(", ");
}
```

```tsx
// the image is expected to be 82vw width on desktop and 31vw othewise
<img
	src={activity.image}
	srcSet={srcsetOf(activity.image)}
	sizes="(max-width: 1020px) 82vw, 31vw"
/>
```
> In reality, this is part of a component we had that allowed maximizing the image into its full-resolution version, which was extremely useful when screen space was at a premium and needed to fit multiple small images together
</details>

In the most extreme cases, this cut our media source bandwidth usage 20x or more, at the cost of ~2 seconds extra on image uploads within the admin panel, and storing ~2x the data, which through AWS's S3 is not the primary cost concern.

# In Hindsight
I am extremely grateful to have been given the opportunity to work on the FORWARD LMS, with every engineer that helped, and especially Annee for entrusting us with part of the future of her doctoral thesis.

My takeaway from this was that the drive to do something will eventually triumph over any blockades you might come across. From learning the nested pages and quirks of the AWS console, to realizing that the Admin Panels system I thought I could easily mod for our needs was very difficult to mess with, it all blurs together into something I'm proud to have worked on!

So as to not cause extra usage that may charge Annee in some way or another, I am omitting the site link itself, though one could find it if they were persistent enough.