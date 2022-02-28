# Broadcasting

## How it broadcasts

If you have the module ``nabu.misc.broadcast``, your CRUD artifacts will have the option to broadcast creates and/or updates.

[:.resources/broadcast-check.png]

If you have checked the create broadcast option and someone calls the create service as exposed by your CRUD artifact, the following will happen:

- insert into database (as per usual)
- check if the list _output_ document of your CRUD has the same fields (or a subset) of the create output
	- if so, an instance of the list output document is built in memory based on the create output
	- if not (e.g. due to imports and/or different blacklisting), a get is performed to build the list output document
- the output instance is broadcast
- all CRUD artifacts that are built on the same core type _and_ have the same broadcast toggle checked, will also perform these steps (apart from the original insert of course)

This last step is done to ensure that you are not limited to using a single CRUD artifact which may not be possible in some scenarios.

**Important**: CRUD artifacts can have multiple list views, _each_ list view has its own set of checkboxes and each will be checked and broadcast if necessary in its format.

For the update, we do the exact same steps if broadcast is toggled on, this means that in the end we have a continuous stream of data in the "output" format of the CRUD after create and/or update.

## How to subscribe

You can of course use custom logic to subscribe to this broadcast if you want to handle these events in your own way, but _if_ your list service is directly exposed as a REST service, additional options open up.

In the swagger it will include extensions to let the frontend know that streaming is possible at design time:

[:.resources/swagger-extensions.png]

### JWT

If a CRUD artifact has any broadcasting turned on _and_ the web application it is in has a JWT keystore and alias configured, it will include a JWT token called "Stream-Token" into the HTTP headers of its list and get output (in the get this is only available if it did not result in "null"). This JWT token can be used by the frontend to subscribe to the data stream, the ``nabu.web.page.data`` component has built-in support for this as long as you also install the ``nabu.web.page.websocket`` module.

The reason this is done through a JWT token is security, you want to make sure the frontend can only subscribe to data that it should be able to access in the first place. Because the token is only available after a successful REST call, it assumes any security check has been performed and you obviously passed. Note that contextual security might be enforced by the REST call but no longer applies to the data stream, so some consideration is still required.

The initial REST call is done to build up the history in your screen, after that the stream can make sure your screen stays up to date.

### Filters

If you have filters set up in your CRUD _and_ you fill them in in your REST call, they are applied to the data stream.

Once you have active filters, two queries will be added to the JWT token (q & j). One is a backend query (in nabu query syntax) which is applied to the data subscription in the broadcast library. The other is a frontend javascript query which can be applied by frontend logic to see if a given data entry matches the filter.

If you do nothing special, these two filters will match (apart from syntax differences). This also means the frontend javascript filter is sort of useless as you will only get data from the backend that already matches it.

So why is this?

Suppose you have a filtered data stream in the frontend _and_ you are broadcasting updates. It is entirely possible that an entry is updated in a way that it no longer matches your filters. In that case, the subscription filter will already ignore the updated document and this update will never reach the frontend, which means from the user perspective, nothing changed.

To this end, once you indicate that you want to broadcast updates as well, an additional checkbox appears per filter. A filter will always be applied in the javascript query, but if you check the new checkbox, it will no longer be applied in the subscription filter.
This allows your updated entry to pass through all the way to the frontend. The frontend can then use its javascript filter to deduce whether the updated entry still belongs in your view or not and take appropriate action.

### Why?

Why would you use the streaming mode? Performance. In a non-streaming setup if you want to open a web page and keep it up to date with external changes, you need to set up long polling. This creates a tradeoff between how often you want to refresh your screen and how heavy a load you want to create on your backend. Especially for data-heavy screens the performance hit of long polling across multiple users should not be underestimated.

Streaming allows for a much lighter weight option which provides near-instantaneous synchronization without generating as heavy a load. This is especially true in a multi-user setup, because users looking at the same data will be sharing the same data stream.