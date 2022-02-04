# PWA widgets

Playing with some ideas around widget definitions for PWAs.

## The idea

For the last few years, I’ve been thinking about the many ways native applications can expose information and/or focused tasks within operating systems. Examples of this include Android Home Screen Widgets, macOS Dashboard and Today Panel Widgets, the Apple Touch Bar, Samsung Daily Cards, Mini App Widgets, and so on. When building app experiences as Progressive Web Apps, it could be very useful to be able to project aspects of the web app onto these surfaces (originally I’d considered these "projections" but most systems do call them widgets, so I opted to go with the flow).

Here are a few use cases:

* A streaming video service could offer access to all of the shows or movies you have in your queue that is distinct from the actual player. It might live in a widget on one device, but, with access to all of the plumbing of the PWA itself, could enable users to control the services’ PWA running on the user’s smart TV.
* A stock tracking app could offer a widget for viewing current stock prices for stocks you are watching.
* A calendar service could provide a daily agenda at a glance.
* A music identification service could have a button widget that, when clicked, would access the microphone and attempt to ID the currently playing song.

Given that the web is responsive, the actual widget surface’s dimensions should not matter much. The projection would merely adapt to the available real estate. Some might even be resizable based on user preference.

These are very similar to additional windows and `iframe`s in the traditional browser context, but would probably need to be plumbed quite differently to keep their overhead low. But more on that below.

## The container

Widgets would be defined within a `widgets` member in the Web App Manifest. The `widgets` member would be an array of `WidgetDefinition` objects.

## Sample `WidgetDefinition` Object

```json
{
  "name": "Agenda",
  "tag": "agenda",
  "url": "/widgets/agenda/",
  "type": "text/calendar",
  "template": "agenda",
  "data": "/widgets/data/agenda.ical",
  "update": "900",
  "icons": [ ],
  "backgrounds": [ ],
  "actions": [ ]
}
```

## Defining a widget

We expect that each implementation of widgets may necessarily vary, taking one of two forms depending on a variety of factors including memory use and power management. As such, we have outlined a means of defining two alternative approaches within the same widget definition (`WidgetDefinition`) that would accommodate both scenarios. Developers would be free to define widgets using only one approach, but should do so with the understanding that choosing a single path may limit the distribution/install-ability of their widget.

### Caveats

Widgets must exist within [the scope of the Manifest](https://www.w3.org/TR/appmanifest/#scope-member) in which they are defined.

A widget’s network connection and update cycle is governed by a [Service Worker that applies to its scope](https://www.w3.org/TR/service-workers/#service-worker-registration-scope) and executes at the host application’s discretion.

### Required properties

The `name` value is a `DOMString` that will serve as the title of the widget presented to users.

The `tag` value is a `DOMString` that will serve as a way to reference the widget within the Service Worker as a `WidgetClient` and is analogous to a [Notification `tag`](https://notifications.spec.whatwg.org/#tag). `WidgetClient` still needs to be defined, but will be similar to [`WindowClient`](https://www.w3.org/TR/service-workers/#ref-for-dfn-window-client).

A widget’s [language](https://www.w3.org/TR/appmanifest/#lang-member) and [text direction](https://www.w3.org/TR/appmanifest/#dir-member) are inherited from the Web App Manifest in which it is defined.

### Templated Widgets

In resource-limited scenarios, a widget host may choose to provide a set of stock widget templates that are customizable by developers (similar to [the Notifications API](https://notifications.spec.whatwg.org/#lifetime-and-ui-integrations)). Examples include an agenda, calendar, mailbox, and a task list. The full list of available templates may vary by widget host. This proposal proposes [a list of widget templates, below](#Suggested-widget-templates).

To define a templated widget, a developer must include the following properties:

* `data` - the URL where the data for the widget can be found; if unsupported, the widget would not be offered.
* `type` - the MIME type of the data feed for the widget; if unsupported, the widget would not be offered.
* `template` - the template the developer would like the widget host to use; if unsupported, the host may offer an analogous widget experience (determined using the `type` value) or the widget would not be offered.

A developer may also define the following:

* `update` - the frequency (in seconds) a developer requests that the widget be updated; the actual update schedule will be at the discretion of the widget host, but will use the Service Worker’s Periodic Sync infrastructure
* `icons` - an array of alternative icons to use in the context of this widget; the default value is the app’s icon (from the Manifest’s `icons` member)
* `backgrounds` - an array of alternative background images that could be used in the template (if the host template supports background images)
* `actions` - An array of [`WidgetAction` objects](#Defining-a-WidgetAction) that trigger an event within the origin’s Service Worker

A widget host’s templates may also, at their discretion, make use of a Manifest’s `theme_color` and `background_color`, if they are defined.

#### Defining a `WidgetAction`

A `WidgetAction` uses the same structure as a [Notification Action](https://notifications.spec.whatwg.org/#dictdef-notificationaction):

```json
{
  "action": "create-event",
  "title": "New Event",
  "icons": [ ]
}
```

The `action` and `title` properties are required. The `icons` array is optional but the icon may be used in space-limited presentations with the `title` providing its accessible label.

When activated, a `WidgetAction` will dispatch a `WidgetEvent` (modeled on [`NotificationEvent`](https://notifications.spec.whatwg.org/#example-50e7c86c)) within its Service Worker. Within the Service Worker, the event will contain a payload that includes a reference to the widget itself and the `action`value.

#### Suggested widget templates

For social and productivity apps:

* calendar-agenda
* calendar-day
* calendar-week
* calendar-month

For address books, directories, and social apps:

* contacts-list
* contacts-item

For general purposes (e.g., news, promotions, media, social):

* content-carousel - Generic
* content-feed
* content-item

For productivity apps:

* email-list
* task-item
* task-list

### Rich Widgets

There are instances in which a templated widget is incapable of accomplishing the goal of a widget (e.g., turning on the microphone to identify the song that’s playing). For those instances, developers may use the `url` property to point to a webpage containing the widget. 

Rich widgets are fully-rendered web pages (analogous to an `iframe`, though the actual implementation is not the same) with some restrictions. In order to protect user privacy and limit the stress on system resources, the widget rendering context will have the following restrictions:

1. Limited RAM footprint - max footprint TBD
2. Limited JavaScript execution time - max execution time TBD
3. Any permissions-governed API usage (e.g., location, microphone) requires user interaction to activate
4. Initiating any multimedia playback requires user interaction (though "piggybacking" on pre-existing multimedia playback initiated from within the same origin would not be similarly restricted)
5. Data updates must be performed by the Service Worker

Additionally, the host application may suspend a widget at any time, creating a bitmap representation of the widget in its last-known state. The origin’s Service Worker may periodically request to update the widget (and create a new snapshot) when it’s not in use[^1].

## Extensibility

We recognize that some widget platforms may allow developers to further refine their widget’s appearance and/or functionality within their system. We recommend that those platforms use [the extensibility of the Manifest](https://www.w3.org/TR/appmanifest/#extensibility) to allow developers to encode their widgets with this additional information, if they so choose.

For example, [Microsoft’s Adaptive Card system](https://docs.microsoft.com/en-us/adaptive-cards/templating/) might consider adding something like the following to the `WidgetDefinition`:

```json
"ms-acn": {
  "prefer": "medium",
  "small": "/widgets/data/agenda-acn-small.json",
  "medium": "/widgets/data/agenda-acn-medium.json",
  "large": "/widgets/data/agenda-acn-large.json"
}
```


[^1]: "In use" defined as user interaction + X minutes.
