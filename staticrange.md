# DOM StaticRange

DRAFT

This proposal is for a lightweight `StaticRange` that can be used in place of a
Range when the complexity of a full `Range` is not necessary.

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

## Proposal

A simple, lightweight `StaticRange` that contains only a start and an end boundary
(node + offset). A `StaticRange` does not update when the DOM is mutated.

The `toRange` method on `StaticRange` allows it to be up-converted to a full-featured
`Range` if the application requires the additional functionality provided by a `Range`.

## IDL

    [Constructor, Exposed=Window]
    interface StaticRange {
      readonly attribute Node startContainer;
      readonly attribute unsigned long startOffset;
      readonly attribute Node endContainer;
      readonly attribute unsigned long endOffset;
      readonly attribute boolean collapsed;
    
      void setStart(Node node, unsigned long offset);
      void setEnd(Node node, unsigned long offset);
    
      [NewObject] Range toRange();
    };

#### *node* = *staticrange* . `startContainer`

Returns staticrange’s start node.

#### *offset* = *staticrange* . `startOffset`

Returns staticrange’s start offset.

#### *node* = *staticrange* . `endContainer`

Returns staticrange’s end node.

#### *offset* = *staticrange* . `endOffset`

Returns staticrange’s end offset.

#### *collapsed* = *staticrange* . `collapsed`

Returns `true` if *staticrange*’s start and end are the same, and `false` otherwise.

#### *staticrange* . `setStart(node, offset)`

Set the start node and offset. If start > end, then swap boundaries. Update collapsed.

#### *staticrange* . `setEnd(node, offset)`

Set the end node and offset. If start > end, then swap boundaries. Update collapsed.

#### *range* = *staticrange* . `toRange()`

Returns a new `Range` with the same start and end as the context object. This is a
convenience method that is equivalent to:

    var newRange = document.createRange()
    newRange.selStart(staticRange.startContainer, staticRange.startOffset);
    newRange.selEnd(staticRange.endContainer, staticRange.endOffset);

## Ranges and Event Handlers

Event handlers cause a particular problem for ranges because the DOM can be
modified during the event handler, potentially invaliding the range. A `Range`
will keep being updated in face of these DOM mutations but a `StaticRange` will
not. Since the same event needs to be sent to each handler in the event handler
chain, using a `StaticRange` would (unfortunately) force the user agent to send
an invalid `StaticRange` to subsequent event handlers in that case.

Thus, when a selection range is required on an `Event`, it is recommended that a method
(e.g., `getRanges()`) be added to that `Event` that can be used when the user
needs the range. This method will take a snapshot of the current range and return
it to the user. Note that the alternative of using a `Range` is not recommended because it
must keep the range updated whenever the DOM is mutated, even if the range is
never actually used. 

Because of the problems listed above, under no circumstances should a
`StaticRange` be used as an `Event` attribute.

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
