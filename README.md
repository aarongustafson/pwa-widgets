# PWA widgets

Playing with some ideas around widget definitions for PWAs

## The container

Widgets would be defined within a `widgets` member in the Web App Manifest. The `widgets` member would be an array of `WidgetDefinition` objects.

## Sample `WidgetDefinition` Object

```json
{
  "name": "Agenda",
  "tag": "widget-agenda",
  "url": "/widgets/agenda/",
  "type": "text/calendar",
  "widget": "agenda",
  "data": "/widgets/data/agenda.ical",
  "update": "900",
  "icons": [ ],
  "backgrounds": [ ],
  "actions": [ ]
}
```

## Defining a widget

We expect that each implementation of widgets may necessarily take one of two forms depending on a variety of factors including memory and power management. As such, we have outlined a means of defining two alternative approaches within the same `WidgetDefinition` that would accommodate both scenarios. Developers would be free to define widgets that only offer one approach, but should do so with the understanding that choosing a limited definition will likely limit the distribution of their widget.

### Required properties

The `name` value is a `DOMString` that will serve as the title of the widget as presented to users.

The `tag` value is a `DOMString` that will serve as a `client.id` that refers to the widget within the origin’s Service Worker via the `WidgetClient` (and `Clients`) interface. `WidgetClient` will be defined in the future and inherits from the parent interface `Client`.

A widget’s language and text direction are inferred from the Web App Manifest in which they are defined.

### Templated Widgets

In resource-limited scenarios, it may be advantageous for the widget host to provide a set of stock widget templates that may be customized by the developer, similarly to how notifications are managed. Examples include an agenda, calendar, mailbox, and a task list. The full list of available templates may vary by widget host.

To define a templated widget, a developer must include the following properties:

* `data` - the URL where the data for the widget can be found; if unsupported, the widget would not be offered
* `type` - the MIME type of the data feed for the widget; if unsupported, the widget would not be offered
* `template` - the template the developer would like the widget host to use; if unsupported, the widget would not be offered

A developer may also define the following:

* `update` - the frequency (in seconds) a developer requests that the widget be updated; the actual update schedule will be at the discretion of the widget host, but will use the Service Worker’s Periodic Sync infrastructure
* `icons` - an array of alternative icons to use in the context of this widget; the default value is the app’s icon (from the Manifest’s `icons` member)
* `backgrounds` - an array of alternative background images that could be used in the template (if the template supports background images)
* `actions` - An array of WidgetAction objects that trigger an event within the origin’s Service Worker

Templates may also make use of the `theme_color` and `background_color` if they are defined in the Manifest.

#### Defining a `WidgetAction`

A `WidgetAction` uses the same structure as a Notification Action:

```json
{
  "action": "create-event",
  "title": "New Event",
  "icons": [ ]
}
```

The `action` and `title` properties are required. The `icons` array is optional but the icon may be used in space-limited presentations with the `title` providing its accessible label.

When activated, a `WidgetAction` will dispatch a `WidgetEvent` (modeled on `NotificationEvent`, which inherits from the `ExtendableEvent` interface) within its Service Worker. Within the Service Worker, the event will contain a payload that includes a reference to the widget itself and the `action`value.

### Rich Widgets

There are instances in which a templated widget is incapable of accomplishing the goal of a widget (e.g., turning on the microphone to identify the song that’s playing). For those instances, developers may use the `url` property to point to a webpage containing the widget.

In order to protect user privacy and limit the stress on system resources, the widget rendering context will have the following restrictions:

1. Limited RAM footprint - TBD
2. Limited JavaScript execution time - TBD
3. Permissions-governed API usage (e.g., location, microphone) requires user interaction
4. Multimedia playback requires user interaction
5. Data updates must be performed by the Service Worker

Additionally, the host application may suspend a widget at any time, creating a bitmap snapshot representation of the widget in its last-known state. The origin’s Service Worker may periodically request to update the widget (and create a new snapshot) when its not in use[^1].

## Extensibility

We recognize that some widget platforms may allow developers to further refine their widget’s appearance within their system. We recommend that those platforms use the extensibility of the manifest to allow developers to encode their widgets with this additional information, if they so choose.

For example, Microsoft’s Adaptive Card platform might consider adding something like the following to the `WidgetDefinition`:

```json
"ms-acn": {
  "prefer": "medium",
  "small": "/widgets/data/agenda-acn-small.json",
  "medium": "/widgets/data/agenda-acn-medium.json",
  "large": "/widgets/data/agenda-acn-large.json"
}
```


[^1]: "In use" defined as user interaction + X minutes.