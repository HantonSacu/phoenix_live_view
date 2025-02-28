# Bindings

Phoenix supports DOM element bindings for client-server interaction. For
example, to react to a click on a button, you would render the element:

    <button phx-click="inc_temperature">+</button>

Then on the server, all LiveView bindings are handled with the `handle_event`
callback, for example:

    def handle_event("inc_temperature", _value, socket) do
      {:ok, new_temp} = Thermostat.inc_temperature(socket.assigns.id)
      {:noreply, assign(socket, :temperature, new_temp)}
    end

| Binding                | Attributes |
|------------------------|------------|
| [Params](#click-events) | `phx-value-*` |
| [Click Events](#click-events) | `phx-click`, `phx-click-away` |
| [Form Events](form-bindings.md) | `phx-change`, `phx-submit`, `phx-feedback-for`, `phx-disable-with`, `phx-trigger-action`, `phx-auto-recover` |
| [Focus Events](#focus-and-blur-events) | `phx-blur`, `phx-focus`, `phx-window-blur`, `phx-window-focus` |
| [Key Events](#key-events) | `phx-keydown`, `phx-keyup`, `phx-window-keydown`, `phx-window-keyup`, `phx-key` |
| [DOM Patching](dom-patching.md) | `phx-mounted`, `phx-update`, `phx-remove` |
| [JS Interop](js-interop.md#client-hooks) | `phx-hook` |
| [Lifecycle Events](#lifecycle-events) | `phx-mounted` | `phx-disconnected` | `phx-connected`|
| [Rate Limiting](#rate-limiting-events-with-debounce-and-throttle) | `phx-debounce`, `phx-throttle` |
| [Static tracking](`Phoenix.LiveView.static_changed?/1`) | `phx-track-static` |

## Click Events

The `phx-click` binding is used to send click events to the server.
When any client event, such as a `phx-click` click is pushed, the value
sent to the server will be chosen with the following priority:

  * The `:value` specified in `Phoenix.LiveView.JS.push/3`, such as:

    ```heex
    <div phx-click={JS.push("inc", value: %{myvar1: @val1})}>
    ```

  * Any number of optional `phx-value-` prefixed attributes, such as:

        <div phx-click="inc" phx-value-myvar1="val1" phx-value-myvar2="val2">

    will send the following map of params to the server:

        def handle_event("inc", %{"myvar1" => "val1", "myvar2" => "val2"}, socket) do

    If the `phx-value-` prefix is used, the server payload will also contain a `"value"`
    if the element's value attribute exists.

  * The payload will also include any additional user defined metadata of the client event.
    For example, the following `LiveSocket` client option would send the coordinates and
    `altKey` information for all clicks:

        let liveSocket = new LiveSocket("/live", Socket, {
          params: {_csrf_token: csrfToken},
          metadata: {
            click: (e, el) => {
              return {
                altKey: e.altKey,
                clientX: e.clientX,
                clientY: e.clientY
              }
            }
          }
        })

The `phx-click-away` event is fired when a click event happens outside of the element.
This is useful for hiding toggled containers like drop-downs.

## Focus and Blur Events

Focus and blur events may be bound to DOM elements that emit
such events, using the `phx-blur`, and `phx-focus` bindings, for example:

```heex
<input name="email" phx-focus="myfocus" phx-blur="myblur"/>
```

To detect when the page itself has received focus or blur,
`phx-window-focus` and `phx-window-blur` may be specified. These window
level events may also be necessary if the element in consideration
(most often a `div` with no tabindex) cannot receive focus. Like other
bindings, `phx-value-*` can be provided on the bound element, and those
values will be sent as part of the payload. For example:

```heex
<div class="container"
    phx-window-focus="page-active"
    phx-window-blur="page-inactive"
    phx-value-page="123">
  ...
</div>
```

## Key Events

The `onkeydown`, and `onkeyup` events are supported via the `phx-keydown`,
and `phx-keyup` bindings. Each binding supports a `phx-key` attribute, which triggers
the event for the specific key press. If no `phx-key` is provided, the event is triggered
for any key press. When pushed, the value sent to the server will contain the `"key"`
that was pressed, plus any user-defined metadata. For example, pressing the
Escape key looks like this:

    %{"key" => "Escape"}

To capture additional user-defined metadata, the `metadata` option for keydown events
may be provided to the `LiveSocket` constructor. For example:

    let liveSocket = new LiveSocket("/live", Socket, {
      params: {_csrf_token: csrfToken},
      metadata: {
        keydown: (e, el) => {
          return {
            key: e.key,
            metaKey: e.metaKey,
            repeat: e.repeat
          }
        }
      }
    })

To determine which key has been pressed you should use `key` value. The
available options can be found on
[MDN](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/key/Key_Values)
or via the [Key Event Viewer](https://w3c.github.io/uievents/tools/key-event-viewer.html).

*Note*: it is possible for certain browser features like autofill to trigger key events
with no `"key"` field present in the value map sent to the server. For this reason, we
recommend always having a fallback catch-all event handler for LiveView key bindings.
By default, the bound element will be the event listener, but a
window-level binding may be provided via `phx-window-keydown` or `phx-window-keyup`,
for example:

    def render(assigns) do
      ~H"""
      <div id="thermostat" phx-window-keyup="update_temp">
        Current temperature: <%= @temperature %>
      </div>
      """
    end

    def handle_event("update_temp", %{"key" => "ArrowUp"}, socket) do
      {:ok, new_temp} = Thermostat.inc_temperature(socket.assigns.id)
      {:noreply, assign(socket, :temperature, new_temp)}
    end

    def handle_event("update_temp", %{"key" => "ArrowDown"}, socket) do
      {:ok, new_temp} = Thermostat.dec_temperature(socket.assigns.id)
      {:noreply, assign(socket, :temperature, new_temp)}
    end

    def handle_event("update_temp", _, socket) do
      {:noreply, socket}
    end

## Rate limiting events with Debounce and Throttle

All events can be rate-limited on the client by using the
`phx-debounce` and `phx-throttle` bindings, with the exception of the `phx-blur`
binding, which is fired immediately.

Rate limited and debounced events have the following behavior:

  * `phx-debounce` - Accepts either an integer timeout value (in milliseconds),
    or `"blur"`. When an integer is provided, emitting the event is delayed by
    the specified milliseconds. When `"blur"` is provided, emitting the event is
    delayed until the field is blurred by the user. Debouncing is typically used for
    input elements.

  * `phx-throttle` - Accepts an integer timeout value to throttle the event in milliseconds.
    Unlike debounce, throttle will immediately emit the event, then rate limit it at once
    per provided timeout. Throttling is typically used to rate limit clicks, mouse and
    keyboard actions.

For example, to avoid validating an email until the field is blurred, while validating
the username at most every 2 seconds after a user changes the field:

```heex
<form phx-change="validate" phx-submit="save">
  <input type="text" name="user[email]" phx-debounce="blur"/>
  <input type="text" name="user[username]" phx-debounce="2000"/>
</form>
```

And to rate limit a volume up click to once every second:

    <button phx-click="volume_up" phx-throttle="1000">+</button>

Likewise, you may throttle held-down keydown:

```heex
<div phx-window-keydown="keydown" phx-throttle="500">
  ...
</div>
```

Unless held-down keys are required, a better approach is generally to use
`phx-keyup` bindings which only trigger on key up, thereby being self-limiting.
However, `phx-keydown` is useful for games and other use cases where a constant
press on a key is desired. In such cases, throttle should always be used.

### Debounce and Throttle special behavior

The following specialized behavior is performed for forms and keydown bindings:

  * When a `phx-submit`, or a `phx-change` for a different input is triggered,
    any current debounce or throttle timers are reset for existing inputs.

  * A `phx-keydown` binding is only throttled for key repeats. Unique keypresses
    back-to-back will dispatch the pressed key events.

## JS Commands

LiveView bindings support a JavaScript command interface via the `Phoenix.LiveView.JS` module, which allows you to specify utility operations that execute on the client when firing `phx-` binding events, such as `phx-click`, `phx-change`, etc. Commands compose together to allow you to push events, add classes to elements, transition elements in and out, and more.
See the `Phoenix.LiveView.JS` documentation for full usage.

For a small example of what's possible, imagine you want to show and hide a modal on the page without needing to make the round trip to the server to render the content:

```heex
<div id="modal" class="modal">
  My Modal
</div>

<button phx-click={JS.show(to: "#modal", transition: "fade-in")}>
  show modal
</button>

<button phx-click={JS.hide(to: "#modal", transition: "fade-out")}>
  hide modal
</button>

<button phx-click={JS.toggle(to: "#modal", in: "fade-in", out: "fade-out")}>
  toggle modal
</button>
```

Or if your UI library relies on classes to perform the showing or hiding:

```heex
<div id="modal" class="modal">
  My Modal
</div>

<button phx-click={JS.add_class("show", to: "#modal", transition: "fade-in")}>
  show modal
</button>

<button phx-click={JS.remove_class("show", to: "#modal", transition: "fade-out")}>
  hide modal
</button>
```

Commands compose together. For example, you can push an event to the server and
immediately hide the modal on the client:

```heex
<div id="modal" class="modal">
  My Modal
</div>

<button phx-click={JS.push("modal-closed") |> JS.remove_class("show", to: "#modal", transition: "fade-out")}>
  hide modal
</button>
```

It is also useful to extract commands into their own functions:

```elixir
alias Phoenix.LiveView.JS

def hide_modal(js \\ %JS{}, selector) do
  js
  |> JS.push("modal-closed")
  |> JS.remove_class("show", to: selector, transition: "fade-out")
end
```

```heex
<button phx-click={hide_modal("#modal")}>hide modal</button>
```

The `Phoenix.LiveView.JS.push/3` command is particularly powerful in allowing you to customize the event being pushed to the server. For example, imagine you start with a familiar `phx-click` which pushes a message to the server when clicked:

```heex
<button phx-click="clicked">click</button>
```

Now imagine you want to customize what happens when the `"clicked"` event is pushed, such as which component should be targeted, which element should receive css loading state classes, etc. This can be accomplished with options on the JS push command. For example:

```heex
<button phx-click={JS.push("clicked", target: @myself, loading: ".container")}>click</button>
```

See `Phoenix.LiveView.JS.push/3` for all supported options.

## Lifecycle Events

LiveView supports the `phx-mounted`, `phx-connected`, and `phx-disconnected` events to react to
different lifecycle events with JS commands.

For example, to show an element when the LiveView has lost its connection, and hide it when the
connection recovers, you can combine `phx-disconnected` and `phx-connected`

```heex
<div id="status" class="hidden" phx-disconnected={JS.show()} phx-connected={JS.hide()}>
  Attempting to reconnect...
</div>
```

To execute commands when an element first appears on the page, you can leverage `phx-mounted`,
such as to animate a notice into view:

```heex
<div id="flash" class="hidden" phx-mounted={JS.show(transition: ...)}>
  Welcome back!
</div>

### LiveView vs static view

`phx-connected` and `phx-disconnected` are only executed when operating
inside a LiveView container. For static templates, they will have no effect.

For LiveView, the `phx-mounted` binding is executed as soon as the LiveView is
mounted with a connection. When using `phx-mounted` in static views, it is executed
as soon as the DOM is ready.
```

## LiveView Specific Events

The `lv:` event prefix supports LiveView specific features that are handled
by LiveView without calling the user's `handle_event/3` callbacks. Today,
the following events are supported:

  - `lv:clear-flash` – clears the flash when sent to the server. If a
    `phx-value-key` is provided, the specific key will be removed from the flash.

For example:

```heex
<p class="alert" phx-click="lv:clear-flash" phx-value-key="info">
  <%= live_flash(@flash, :info) %>
</p>
```

## Loading states and errors

All `phx-` event bindings apply their own css classes when pushed. For example
the following markup:

```heex
<button phx-click="clicked" phx-window-keydown="key">...</button>
```

On click, would receive the `phx-click-loading` class, and on keydown would receive
the `phx-keydown-loading` class. The css loading classes are maintained until an
acknowledgement is received on the client for the pushed event.

In the case of forms, when a `phx-change` is sent to the server, the input element
which emitted the change receives the `phx-change-loading` class, along with the
parent form tag. The following events receive css loading classes:

  - `phx-click` - `phx-click-loading`
  - `phx-change` - `phx-change-loading`
  - `phx-submit` - `phx-submit-loading`
  - `phx-focus` - `phx-focus-loading`
  - `phx-blur` - `phx-blur-loading`
  - `phx-window-keydown` - `phx-keydown-loading`
  - `phx-window-keyup` - `phx-keyup-loading`

Additionally, the following classes are applied to the LiveView's parent
container:

  - `"phx-connected"` - applied when the view has connected to the server
  - `"phx-loading"` - applied when the view is not connected to the server
  - `"phx-error"` - applied when an error occurs on the server. Note, this
    class will be applied in conjunction with `"phx-loading"` if connection
    to the server is lost.
