+++
date = '2026-04-09T12:58:54-07:00'
draft = false
title = 'Building a Custom LMS for Juvenile Justice Facilities'
+++
![](attachments/Pasted%20image%2020260410130848.png)
# But Why Make Your Own?
In early 2025, I joined a small student team building a custom Learning Management System for a doctoral research project on “a comprehensive, accessible transition curriculum for youth with disabilities in juvenile justice.”

That environment imposed some unusual constraints: facilities often had *very restrictive whitelists*, so our system would need to avoid extra integrations, third-party assets, and anything else that increased the odds of being blocked. A general-purpose LMS like Moodle or Canvas might have worked in one facility and be blocked in another, and even then would not have given us the necessary instrumentation and research data collection.

To keep the maintenance cost down and play to our team's strengths, we settled upon a simple stack:
- React, Served Statically
- Django/DRF
- PostgreSQL

All running on AWS, as they have generous low-usage tiers, a member of the team had previous experience that made it preferable to hosting on a VPS somewhere, and reliability with automatic backups was crucial to keeping the data safe for later.

Aside from the basic goals, we also needed to provide extensive accessibility features, as well as surface some kind of interface for Dr. Grayson to manage both users and lessons herself to solve the [bus factor](https://en.wikipedia.org/wiki/Bus_factor) of such a small team comprised of students with uncertain futures.
# Actually Building It
Initially the team had 5 members, due to life and schooling taking necessary precedence, that quickly became 3, and then 1 about halfway through the project. Dr. Grayson and I then spent some time to find another full stack developer, who ended up being the wonderful [Lorran Alves Galdino](https://low-go.github.io/portfolio/), and we set out to work whenever we could to see the project to completion.

After months of frantic research, reading docs, and juggling my university course load, I found myself comfortable enough with React, Redux, and {{< side "Docker" right >}} I cannot stress how useful Docker/Compose was after a certain point of development when working with any amount of people on different platforms. Adding mock services like [Minio](https://www.min.io/) for S3 and Postgres, all in a way that live reloaded with file changes was a game-changer.{{< /side >}}to develop at a pace that would let us finish by the deadline we had been given.
## Managing Accessibility
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

One downside to our approach in using the browser's built-in Speech API, was that different platforms offer a wide variety of voices, ranging from "undeniably a robot" to "did they hire a voice-over?", however, using a browser-agnostic TTS-on-demand service like Amazon Polly or Microsoft Foundry Tools was in no way realistic for our budget.
## The Admin Panel
![The admin panel interface, editing the contents of a Slideshow activity.](attachments/Pasted%20image%2020260409190325.png)
*The admin panel interface, editing the contents of a Slideshow activity.*

Django has a wonderful out-of-the-box administrative website for projects using its ORM and app system. When I heard it existed, all of my naive plans to create a Google Forms-esque lesson creation tool that would somehow interface with Django and update in real time were thrown out of the window.

The admin panel is composed of template based widgets used to display fields of an ORM class in some (usually) interactive way. It is also the focus of many Django plugin libraries, as the basic interfaces provided by Django itself were often lacking in ease of use, or simply did not make sense for a certain datatype.

To that end, we ended up with 3 main admin panel libraries that significantly increased the QOL for its users. Those were: [django-json-widget](https://github.com/jmrivas86/django-json-widget), [martor](https://django-markdown-editor.readthedocs.io/en/latest/), and [django-admin-sortable2](https://github.com/jrief/django-admin-sortable2), which all had enough configuration options to fit our needs. Unfortunately, some things aren't easily achievable using the built-in tools given, and we ended up using little "hacks" everywhere to make the admin interface behave just right.

One such example was grouping related interfaces together, as seen in the sidebar of the above image. Which by default is only split by Django "app", and otherwise alphabetically as a whole. Our entire repository was one large app (which would have been a pain to split up), and so user management was squeezed in between random implemented activities. Our solution was to introduce a grouping field to each interface, and then [tell Django they were different apps](https://github.com/FORWARD-Curriculum/Forward-app/blob/a407b957c1c1cb1c02f1fd08727b68abbc3f71ff/Forward-server/core/admin/admin.py#L14-L41) for display.

Other "hacks" included [wrapping self-contained html](https://github.com/FORWARD-Curriculum/Forward-app/blob/a407b957c1c1cb1c02f1fd08727b68abbc3f71ff/Forward-server/core/admin/activity_admin.py#L294-L375) as "widgets" for fields that didn't exist in the class, and [sub views of models](https://github.com/FORWARD-Curriculum/Forward-app/blob/a407b957c1c1cb1c02f1fd08727b68abbc3f71ff/Forward-server/core/admin/core_admin.py#L20) for users in the "instructors" group so facility admin themselves could reset passwords, and change usernames as needed to reduce the reliance on Dr. Grayson or the dev team in production.
## Mistakes Were Made
One of the design choices I made very early on into the project, was going for a brittle "seamless" saving approach that relied entirely on the component actually displaying the activity, sending response data when it became unmounted, with no error handling and component-local storage of the data itself.

This led to bugs where, data would be deleted as the user would return to an activity before the server responded, and in doing so, the client would not have their previous response, effectively wiping it from the server once the client navigated away again.

When initially designing this hook, about a month into React, I thought I was a genius... You would use it just like a `setState()`, and all the internals were abstracted for developer to focus on functionality and nothing more. Eventually, I ripped this out in favor of a more robust, yet "visible" option where any attempts to navigate without saving were given a warning, and progression only occurred on a 200 OK from the server.
## ...Others Were Not
As Lorran and I were building the functionality of the app, we were given the first lesson in the curriculum with all media included to properly prototype and test. This was great, but I noticed while debugging some networking code that even on a {{< side "cached refresh," left >}}
Due to the S3 service we eventually fell upon for our media backend, and using [presigned urls](https://docs.aws.amazon.com/boto3/latest/guide/s3-presigned-urls.html) for security, browsers would not cache the image as it has a unique url each request.
{{< /side >}} the total data transfer would be somewhere in the *several megabytes*. After excluding dev dependencies like the icon library, and unminified source mapped code, the lesson media itself was obviously the culprit.

Dr. Grayson would often use stock photos that were extremely high resolution, and other times would give us the raster version of a complex SVG, which was all in the name of supporting the content of the lessons as best as possible.

One of the last large tasks I took on prior to the finalization of the app was implementing a media delivery optimization system, taking a similar approach to how many platforms transcode their user-generated media into multiple variants for bandwidth saving purposes.

What I ended up with was a set of wrappers and utility functions to help interface with the [django-imagefield](https://pypi.org/project/django-imagefield/) library, which allows one to specify a set of formats, transformation functions, and aspect ratios that images are processed through upon upload to the media storage backend.

<details>
<summary>What that code ended up looking like:</summary>

```python
# Define a set of sizes to transcode into on upload
DEFAULT_IMAGE_FORMATS = {
    "mobile":  ("default", ("thumbnail", (480,  480))),
    "tablet":  ("default", ("thumbnail", (800,  800))),
    "desktop": ("default", ("thumbnail", (1500, 1500))),
} if settings.OPTIMIZE_MEDIA else {}
```
```python
# Serialize the different variants as their URLs
# and sizes for the client
class BaseActivity(models.Model):
...
@abstractmethod
def to_dict(self):
        return {
			...
            "x_image": GENERIC_FORWARD_IMAGE.stringify(self.x_image) \ 
	            if self.x_image else None,
			...
        }
```
And finally on the frontend, a utility function to format the JSON into an HTML srcset attribute which is used for every image:
```tsx
// the image is expected to be 82vw width on desktop and 31vw othewise
<img
	src={activity.image}
	srcSet={srcsetOf(activity.image)}
	sizes="(max-width: 1020px) 82vw, 31vw"
/>
```
> In reality, this is just part of a component we had that allowed maximizing the image into its full-resolution version, which was extremely useful when screen space was at a premium and needed to fit multiple small images together
</details>

In the most extreme cases, this cut our media source bandwidth usage 20x or more, at the cost of ~2 seconds extra on image uploads within the admin panel, and storing ~2x the data, which through AWS's S3 is not the primary cost concern.

# In Hindsight
For such a small team, the batteries-included admin panel on top of a simple to use ORM was a godsend that cut out almost half the work of making the app's functionality, even if some of the things we did were hacky. It and React Router were more than enough to only require dev dependencies for solving more esoteric problems like Markdown parsing and PDF display, which gave us less to worry about becoming an issue with facilities down the road.

This is expected to be posted following Dr. Grayson's defense in May, and while the data itself hasn't been fully processed, she seems very satisfied with the tools we provided to manage the site while we're away.

I am extremely grateful to have been given the opportunity to work on the FORWARD LMS, with every engineer that helped, and especially Dr. Grayson for entrusting us with part of the future of her doctoral thesis.

The site *is* live but I'm not linking it to avoid unexpected traffic costs.