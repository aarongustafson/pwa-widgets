# PWA widgets

Playing with some ideas around widget definitions for PWAs.

## The idea

For the last few years, I’ve been thinking about the many ways native applications can expose information and/or focused tasks within operating systems. Examples of this include Android Home Screen Widgets, macOS Dashboard and Today Panel Widgets, the Apple Touch Bar, Samsung Daily Cards, Mini App Widgets, smart watch app companions, and so on. When building Progressive Web Apps, it would useful to be able to project aspects of the web app onto these surfaces (originally I’d considered these "projections" but most systems do call them "widgets," so I opted to go with the flow).

Here are a few use cases:

* A streaming video service could offer access to all of the shows or movies you have in your queue that is distinct from the actual player. It might live in a widget on one device, but, with access to all of the plumbing of the PWA itself, could enable users to control the services’ PWA running on the user’s smart TV.
* A stock tracking app could offer a widget for viewing current stock prices for stocks you are watching.
* A calendar service could provide a daily agenda at a glance.
* A music identification service could have a button widget that, when clicked, would access the microphone and attempt to ID the currently playing song.

It’s expected that each Widget Host will provide different opportunities and have different constraints that depend on a variety of factors including memory use and power management. As such, this proposal outlines a means of defining two alternative approaches within the same Widget definition (`WidgetDefinition`) that would accommodate both scenarios:

1. [Templated Widgets](#Templated-Widgets)
2. [Rich Widgets](#Rich-Widgets)

Under this proposal, developers would be free to define Widgets that support both approaches or only one. In supporting only one, they should be aware that choosing a single path may limit the distribution/install-ability of that Widget.

## Templated Widgets

In resource-limited scenarios, a Widget Host may choose to provide a set of built-in widget templates that are minimally-customizable by developers (similar to [the Notifications API](https://notifications.spec.whatwg.org/#lifetime-and-ui-integrations)) through use of the PWA’s `icons`, `theme_color`, `background_color`, and so on. Examples include an agenda, calendar, mailbox, and a task list. The full list of available templates will likely vary by Widget Host. This proposal suggests [list of widgets template types](#Suggested-template-types) as a reasonable starting point.

Templated Widgets support user interaction through one or more [developer-defined `WidgetAction` objects](#Defining-a-WidgetAction)s, which are analogous to a [`NotificationAction`](https://notifications.spec.whatwg.org/#dictdef-notificationaction).

### Suggested template types

For social and productivity apps:

* calendar-agenda
* calendar-day
* calendar-week
* calendar-month

For address books, directories, and social apps:

* contacts-list
* contacts-item

For general purposes (e.g., news, promotions, media, social):

* content-carousel
* content-feed
* content-item

For productivity apps:

* email-list
* task-item
* task-list

### Data flow in a Templated Widget

Data flow in a Templated Widget is largely managed in two ways:

1. Data flows from the Service Worker to a Widget via the `showWidget()` method.
2. Data (in the form of interaction) flows from the Widget to the Service Worker via a `WidgetEvent` `tag`-ed to the specific Widget.

Here is an example of how this might look in the context of a Periodic Sync:

<figure id="periodic-sync">

![](media/sync.gif)

</figure>

This video shows the following steps:

1. As part of a Periodic Sync, the Service Worker makes a Request to the host or some other endpoint.
2. The Response comes back.
3. As the Service Worker is aware of which widgets rely on that data, via the `WidgetDefinition` provided during install, the Service Worker can identify which widgets need updating. (This is internal logic and not shown in the video).
3. The Service Worker takes that data—perhaps packaging it with other instructions—and uses `showWidget()` to update the specific widgets that make use of that data.

To show a more complicated set of examples, consider what should happen if certain Widgets depend on authentication and the user happens to log out in the PWA or a browser tab. The developers would need to track this and ensure the Service Worker is notified so it can replace any auth-requiring Widgets with a prompt back into the app to log in.

Here’s how that might work:

<figure id="user-logout">

![](media/user-logout.gif)

</figure>

This video shows:

1. The user logging out from the context of a Client. When that happens, the Client, sends a `postMessage()` to the Service Worker, alerting it to the state change in the app.
2. The Service Worker maintains a list of active Widgets and is aware of which ones require authentication. Knowing it’s been revoked, it pushes a new template to each auth-requiring Widget with a notice and a button to prompt the user to log in again.

The next ste in this flow would be for the use to log back in. They could do that directly in the Client, but let’s use the `WidgetAction` provided in the previous step:

<figure id="user-login">

![](media/user-login.gif)

</figure>

This video shows:

1. The user clicking the "Login" action in the Widget. This triggers a `WidgetEvent` called "login".
2. The Service Worker is listening for that action and redirects the user directly to the login page of the app, either in an existing Client or a new Client (if one is not open).
3. The user logs in and the app sends a `postMessage()` to the Service Worker letting it know the user is authenticated again.
4. The Service Worker grabs new data from the network.
5. The Service Worker pipes that data back into the auth-requiring Widgets.

You can see more examples in [the `WidgetEvent` section](#Widget-related-Events).

## Rich Widgets

There are instances in which a templated widget is incapable of accomplishing the goal of a widget (e.g., turning on the microphone to identify the song that’s playing, frequently updating to display an accurate clock). For those instances, developers will a lot more control. This is where Rich Widgets come in. Rich Widgets are wholly developer managed at a URL they control.

As fully-rendered web pages (analogous to an `iframe`), Rich Widgets need to come with some restrictions that protect user privacy and limit stress on system resources. Each Widget Host will likely have its own set of restrictions for Rich Widgets (if it even allows them). These will likely include:

1. Limited RAM footprint - *recommendation TBD*,
2. Limited JavaScript execution time - *recommendation TBD*,
3. Permissions-governed API usage (e.g., location, microphone) would require user interaction to activate,
4. Initiating any multimedia playback requires user interaction (though "piggybacking" on pre-existing multimedia playback initiated from within the same origin would not be similarly restricted), and
5. Data updates must be performed by the Service Worker.

Additionally, the Widget Host will likely reserve the right to suspend a widget at any time. Looking at how many renderers handle suspension, it’s likely that many Widget Hosts will create a bitmap representation of a Widget in its last-known state. The origin’s Service Worker may periodically request to update the Widget (and create a new snapshot) when it’s not in use.[^1] For more on this, consult the [events section](#Events).

Given that the web is responsive, a Rich Widget’s dimensions should not matter much. The content would merely adapt to the available real estate. Some Widget Hosts may even support user resizing of a Widget.

### Caveats

Rich Widgets must exist within [the scope of the Manifest](https://www.w3.org/TR/appmanifest/#scope-member) in which they are defined.

A Widget’s network connection and update cycle is governed by a [Service Worker that applies to its scope](https://www.w3.org/TR/service-workers/#service-worker-registration-scope) and executes at the Widget Host’s (and/or OS’) discretion.

## Internationalization

A widget’s [language](https://www.w3.org/TR/appmanifest/#lang-member) and [text direction](https://www.w3.org/TR/appmanifest/#dir-member) are inherited from the Web App Manifest in which it is defined. See also [Web App Manifest Internationalization](https://w3c.github.io/manifest/#internationalization).


## Defining a Widget

One or more Widgets are defined within the `widgets` member of a Web App Manifest. The `widgets` member would be an array of `WidgetDefinition` objects.

### Sample `WidgetDefinition` Object

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

### Required properties

The `name` value is a `DOMString` that will serve as the title of the widget presented to users.

The `tag` value is a `DOMString` that will serve as a way to reference the widget within the Service Worker as a `WidgetClient` and is analogous to a [Notification `tag`](https://notifications.spec.whatwg.org/#tag). `WidgetClient` still needs to be defined, but will be similar to [`WindowClient`](https://www.w3.org/TR/service-workers/#ref-for-dfn-window-client).


### Rich Widget properties

A [Rich Widget](#Rich-Widgets) MUST include a `url` property, which is a valid `URL` string pointing to the Widget’s HTML. That URL MUST exist within [the scope of the Web App Manifest](https://w3c.github.io/manifest/#dfn-scope) and [the scope of the Service Worker](https://w3c.github.io/ServiceWorker/#dom-serviceworkerregistration-scope).

*Note: Rich Widgets will likely be resource-limited, so developers are advised to avoid relying on large client-side libraries to render and update Rich Widgets.*

### Templated Widget properties

A [Templated Widget](#Templated-Widgets) MUST include the following properties:

* `data` - the `URL` where the data for the widget can be found; if the format is unsupported, the widget would not be offered.
* `type` - the MIME type of the data feed for the widget; if unsupported, the widget would not be offered.
* `template` - the template the developer would like the Widget Host to use; if unsupported, the host may offer an analogous widget experience (determined using the `type` value) or the widget would not be offered.

A developer MAY define the following:

* `update` - the frequency (in seconds) a developer requests that the widget be updated; the actual update schedule will be at the discretion of the Widget Host, but will use the Service Worker’s Periodic Sync infrastructure. Effectively, this is a declarative way of defining a Periodic Sync event that makes a Request to the `data` URL and pushes it to the widget. When the widget is installed, the Periodic Sync would be created; when it is uninstalled, the event would be removed.
* `icons` - an array of alternative icons to use in the context of this Widget; if undefined, the Widget icon will be the chosen icon from [the Manifest’s `icons` array](https://w3c.github.io/manifest/#icons-member).
* `backgrounds` - an array of alternative background images (as [`ImageResource` objects](https://www.w3.org/TR/image-resource/)) that could be used in the template (if the Widget Host and template support background images).
* `actions` - An array of [`WidgetAction` objects](#Defining-a-WidgetAction) that trigger an event within the origin’s Service Worker

A Widget Host’s templates may also, at their discretion, make use of a Manifest’s [`theme_color`](https://w3c.github.io/manifest/#theme_color-member) and [`background_color`](https://w3c.github.io/manifest/#background_color-member), if they are defined.

### Defining a `WidgetAction`

A `WidgetAction` uses the same structure as a [Notification Action](https://notifications.spec.whatwg.org/#dictdef-notificationaction):

```json
{
  "action": "create-event",
  "title": "New Event",
  "icons": [ ]
}
```

The `action` and `title` properties are required. The `icons` array is optional but the icon may be used in space-limited presentations with the `title` providing its [accessible name](https://w3c.github.io/aria/#dfn-accessible-name).

When activated, a `WidgetAction` will dispatch a [`WidgetEvent`](#WidgetEvent) (modeled on [`NotificationEvent`](https://notifications.spec.whatwg.org/#example-50e7c86c)) within its Service Worker. Within the Service Worker, the event will contain a payload that includes a reference to the Widget itself and the `action` value.

### Extensibility

We recognize that some widget platforms may allow developers to further refine a Widget’s appearance and/or functionality within their system. We recommend that those platforms use [the extensibility of the Manifest](https://www.w3.org/TR/appmanifest/#extensibility) to allow developers to encode their widgets with this additional information, if they so choose.

For example, if using something like [Microsoft’s Adaptive Cards](https://docs.microsoft.com/en-us/adaptive-cards/templating/) for rendeing, a Widget Host might consider adding something like the following to the `WidgetDefinition`:

```json
"ms-ac": {
  "prefer": "medium",
  "small": "/widgets/data/agenda-ac-small.json",
  "medium": "/widgets/data/agenda-ac-medium.json",
  "large": "/widgets/data/agenda-acnlarge.json"
}
```

## Service Worker APIs

This proposal introduces two new methods to the   `ServiceWorkerRegistration`:

* `showWidget()`
* `closeWidget()`

### Showing a Widget

Developers will use `showWidget()` to both create and update Widgets. The method takes two arguments:

1. <var>tag</var> - This is the tag property and is used for identifying the widget that should be updated.
2. <var>payload</var> - This is an object that provides the Widget Host with the necessary information for rendering the widget. The members are:
    * `widget` - The `WidgetDefinition` from the Manifest and
    * `data` - The data to flow into it.


### Closing a Widget

Developers will use `closeWidget()` to uninstall. The method takes one argument: the Widget’s `tag`.

## Widget-related Events

There are a host of different events that will take place in the context of a Service Worker. For simplicity, all come through the `widgetClick` event listener.

A `WidgetEvent` is an object with the following properties:

* `action`: This is the primary way you will disambiguate events. The names of the events may be part of a standard lifecycle or app-specific, based on any [`WidgetAction` that has been defined](#Defining-a-WidgetAction).
* `widget`: This is a reference to the Widget itself. As with Notifications, this object provides access to details about the Widget, most importantly its `tag`, which would be used to update the widget using `showWidget()`.

```js
{
  "action": "create-event",
  "widget": {
    "id": {{ System_UUID }},
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
}
```

You can see a basic example of this in use in [the user login video, above](user-login). There is a walkthrough of the interaction following that video, but here’s how the actual `WidgetEvent` could be handled:

```js
self.addEventListener('widgetclick', function(event) {

  const action = event.action;

  // If user is being prompted to login 
  if ( action == "login" ) {
    // open a new window to the login page
    clients.openWindow( "/login?from=widget" );
  }

});
```

There are a few special `WidgetEvent` `action` types to consider as well. The first two are pretty self-explanatory:

* "install" - installs a single widget
* "uninstall" - uninstalls a single widget

Here is the flow for install:

<figure id="periodic-sync">

![](media/install.gif)

</figure>

1. An "install" event is received by the Service Worker. This could originate from the Widget Host or directly from the User Agent.
2. The Service Worker captures the Widget definition from the `widget` property and makes a `Request` for its `data` endpoint.
3. The Service Worker then combines the `Response` with the Widget definition and passes that along to the Widget Host via the `showWidget()` method, using the `tag` as the first argument and an object containing the `widget` and the Response `data` as the second argument.

Uninstall is similar:

<figure id="periodic-sync">

![](media/uninstall.gif)

</figure>

1. An "uninstall" event is received by the Service Worker. This could originate from the Widget Host or directly from the User Agent.
2. The Service Worker runs any necessary cleanup steps (such as unregistering a Periodic Sync).
3. The Service Worker then prompts the Widget Host to remove the Widget using `closeWidget()`, passing in the `tag`.


The final special event case is the "resume" event. Many Widget Hosts will suspend the rendering surface when it is not in use (to conserve resources). In order to ensure Widgets are refreshed when the rendering surface is presented, the Widget Host will issue a "resume" event. Unlike "install" and "uninstall," the "resume" event does not include a reference to an individual Widget. The Service Worker will need to enumerate its `WidgetClients` and Fetch new data for each.

<figure id="periodic-sync">

![](media/resume.gif)

</figure>

Here is how each of these events could be handled in the Service Worker:

```js
self.addEventListener('widgetclick', function(event) {

  const action = event.action;
  const widget = event.widget;

  event.waitUntil(
    // If a widget is being installed
    switch action:
      case "install":
        console.log("installing", widget);
        
        // get the data needed
        fetch( widget.data )
          .then( response => {

            // show the widget, passing in 
            // the widget definition and data
            self.showWidget( widget.tag, {
              widget: widget,
              data: response.body
            });

          });
        return;
      
      case "uninstall":
        console.log("uninstall", widget);
        
        // do any cleanup that’s needed
        
        // close the widget
        self.closeWidget( widget.tag );
        return;

      case "resume":
        console.log("resuming all widgets");

        // refresh the data on each widget
        clients
          .matchAll({ type: "widget" })
          .then(function(widgetList) {
            for (let i = 0; i < widgetList.length; i++) {
              var widget = clientList[i];
              fetch( widget.data )
                .then( response => {
                  self.showWidget( widget.tag, {
                    widget: widget,
                    data: response.body
                  });
                });
            }
          });

        return;
      
      // other cases
    }
  );

});
```

[^1]: "In use" defined as user interaction + X minutes.
 