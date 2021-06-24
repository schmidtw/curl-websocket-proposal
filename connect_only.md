## Websocket Connect Only

With this approach the idea is to establish a socket to the right host and enable a simple hand off to code that want's to manage it's own websocket, but benefit from using curl to connect.

### API Changes

1. Add a `CURLOPT_WS_CONNECT_ONLY` option.
2. Add support for `ws://` and `wss://` schemes.
3. Add a `CURLOPT_PRESERVE_SCHEME` option.

### Internal Changes

The `ws://`or `wss://` schemes will indicate to use the websocket semantics for the connection.

If the `CURLOPT_WS_CONNECT_ONLY` option is specified then the socket can be extracted with `CURLINFO_ACTIVESOCKET` like normal & the rest is up to the calling app.

If the `CURLOPT_WS_CONNECT_ONLY` option is **NOT** specified then the call to `curl_easy_perform()` will return an error instead of making a call.  The logic behind this choice is that if curl needs to perform the sequence needed to close the socket at the websocket level, it will need to do a fair bit more to support the websocket spec  (close frames, reasons and the like).

Normally the `ws://` or `wss://` scheme will be translated to `http://` or `https://` respectively.  However, some servers expect `ws://` or `wss://`.  If `CURLOPT_PRESERVE_SCHEME` is set, then the scheme used in the requesting URL is preserved as the specified `ws://` or `wss://`.  I'm not sure how prevalent this is, but I'm aware of a few instances in production today.

Curl would provide the websocket headers needed to negotiate a "normal" connection.  If an application wanted to support specific `Sec-WebSocket-Protocol` or `Sec-WebSocket-Accept` values, it would need to set the headers compliant with the websocket spec.  Curl would see the existence of the header & not alter the application specified version.

### Thoughs

Both as a starting point & long term I like this approach as an option.  It adds the least amount of websocket logic to curl, and provides a way to future-proof curl.  Even if other options are pursued, I think this should be pursued as well.