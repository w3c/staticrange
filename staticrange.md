# DOM StaticRange

DRAFT

This proposal is for a lightweight `StaticRange` that can be used
instead of a `Range` when the complexity of a full `Range` is not necessary.

It is intended that all current uses of `Range` would remain unchanged, but
new APIs or objects would make use of `StaticRange`.

## Background

DOM4 defines a `Range` object (originally from DOM2 Traversal/Range) which can
be used to represent a sequence of content in the DOM tree. A `Range` consists
of a start boundary (a node and an offset) and an end boundary (also a node
and an offset). A key benefit of using a `Range` is that once it is created, it
will maintain the integrity of the range as best it can even in the face
of DOM mutations.

## Motivation

A problem with `Range` is that whenever a DOM mutation occurs, all of the active
`Range` objects affected by the mutation need to be updated. This can be an expensive
operation, especially if there are a large number of active `Range` objects. This cost
may be acceptable if all of these `Range` objects are actually needed by the application,
but `Range` is used whenever we need to record start- and end-positions.
Because of this, many of the `Range` objects that are created are not actually used by
the application, and many of the `Range` objects that are being used don’t actually need
their range start and end to be resilient to DOM mutations.

This problem is exacerbated when an application caches an object that happens to
contain a `Range` along with other data that the application needs. As long as this
object is active, the `Range` will be active and it will need to be updated for
every DOM mutation. In this manner, an application can end up with many active
`Range` objects even if none of them are being used.

Because of this performance cost, there is a strong desire from browser vendors that
new objects or APIs should not require a `Range`. The purpose of this proposal is to
create a variant of `Range` (one without the performance cost) that can be used in 
new API, event, and object proposals.

## Proposal

A simple, lightweight `StaticRange` that contains only a start and an end boundary
(node + offset). A `StaticRange` does not update when the DOM is mutated.

A `collapsed` attribute would also be provided as a convenient way to determine if the
start and end boundaries were at the same location.

A `toRange` method on `StaticRange` would be used to create a full-featured `Range`
object that is up-converted from the `StaticRange`. This would only be used if the
application required the additional functionality provided by a `Range`.

## Note on Ranges and Event Handlers

Event handlers cause a particular problem for ranges because the DOM can be
modified during the event handler, potentially invaliding the range.
Using a `Range` on the event would ensure that the range is kept up-to-date with
the DOM changes, but, as noted earlier, a `Range` is (often wastefully) expensive
to maintain.

Unfortunately, using a `StaticRange` in this situation is not an option because if
the DOM is updated in the event handler, then the range snapshot passed to the first
event handler might now be invalid. We can't take a new snapshot and send that to
subsequent event handlers because we need to make sure that we send the same event
to each handler.
That leaves us with the option of passing an invalid range to the subsequent events,
which is undesirable.

To summarize these problems with ranges on events:

* `StaticRange` is not appropriate because DOM mutations during the event handler
can result in an invalid range being sent.
* `Range` is not appropriate because it is expensive to maintain.

To work around these problems, it is recommended that, when an `Event` requires a
selection range, a new method (e.g., `getRanges()`) should be added to the `Event`
that can be used when the user needs the range.
This method will take a snapshot of the current range and return
it to the user as a `StaticRange`.

## Example Usage

One example of where a `StaticRange` would be immediately useful is with the
`beforeinput` event. This event has a need to specify a selection range however we don't
want the user agent to have to maintain a full `Range` for the lifetime of the event object.

Here is example code showing how a user might access the current selection range during
`beforeinput`:

    var beforeinputHandler = function(e) {
        // Get current selection ranges.
        var ranges = e.getRanges();
        if (ranges.length == 1) {
            // Handle single selection.
            var range = ranges[0];
            if (range.collapsed) {
                // Empty selection. Input will happen at insertion point.
                ...
            } else {
                // Selected text will be replaced with input.
                ...
            }
        } else {
            // Handle multi-selection.
            ...
        }
    }
    
    element.addEventListener('beforeinput', beforeinputHandler, false);

## Acknowledgements

Thanks to the following people for the discussions that lead to the creation
of this proposal:

Enrica Casucci (Apple),
Bo Cupp (Microsoft),
Emil Eklund (Google),
Gary Kacmarcik (Google),
Ian Kilpatrick (Google),
Grisha Lyukshin (Microsoft),
Miles Maxfield (Apple),
Ryosuke Niwa (Apple),
Dave Tapuska (Google),
Ojan Vafai (Google),
Johannes Wilm (Fidus)

## References

https://www.w3.org/TR/DOM-Level-2-Traversal-Range/

https://dom.spec.whatwg.org/#ranges
