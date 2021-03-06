.. Copyright 2016 OpenMarket Ltd
..
.. Licensed under the Apache License, Version 2.0 (the "License");
.. you may not use this file except in compliance with the License.
.. You may obtain a copy of the License at
..
..     http://www.apache.org/licenses/LICENSE-2.0
..
.. Unless required by applicable law or agreed to in writing, software
.. distributed under the License is distributed on an "AS IS" BASIS,
.. WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
.. See the License for the specific language governing permissions and
.. limitations under the License.

Receipts
========

.. _module:receipts:

This module adds in support for receipts. These receipts are a form of
acknowledgement of an event. This module defines a single acknowledgement:
``m.read`` which indicates that the user has read up to a given event.

Sending a receipt for each event can result in sending large amounts of traffic
to a homeserver. To prevent this from becoming a problem, receipts are implemented
using "up to" markers. This marker indicates that the acknowledgement applies
to all events "up to and including" the event specified. For example, marking
an event as "read" would indicate that the user had read all events *up to* the
referenced event.

Events
------
Each ``user_id``, ``receipt_type`` pair must be associated with only a
single ``event_id``.

{{m_receipt_event}}

Client behaviour
----------------

In ``/initialSync``, receipts are listed in a separate top level ``receipts``
key. In ``/sync``, receipts are contained in the ``ephemeral`` block for a
room. New receipts that come down the event streams are deltas which update
existing mappings. Clients should replace older receipt acknowledgements based
on ``user_id`` and ``receipt_type`` pairs. For example::

  Client receives m.receipt:
    user = @alice:example.com
    receipt_type = m.read
    event_id = $aaa:example.com

  Client receives another m.receipt:
    user = @alice:example.com
    receipt_type = m.read
    event_id = $bbb:example.com

  The client should replace the older acknowledgement for $aaa:example.com with
  this one for $bbb:example.com

Clients should send read receipts when there is some certainty that the event in
question has been **displayed** to the user. Simply receiving an event does not
provide enough certainty that the user has seen the event. The user SHOULD need
to *take some action* such as viewing the room that the event was sent to or
dismissing a notification in order for the event to count as "read".

A client can update the markers for its user by interacting with the following
HTTP APIs.

{{receipts_cs_http_api}}

Server behaviour
----------------

For efficiency, receipts SHOULD be batched into one event per room before
delivering them to clients.

Receipts are sent across federation as EDUs with type ``m.receipt``. The
format of the EDUs are::

    {
        <room_id>: {
            <receipt_type>: {
                <user_id>: { <content> }
            },
            ...
        },
        ...
    }

These are always sent as deltas to previously sent receipts. Currently only a
single ``<receipt_type>`` should be used: ``m.read``.

Security considerations
-----------------------

As receipts are sent outside the context of the event graph, there are no
integrity checks performed on the contents of ``m.receipt`` events.

