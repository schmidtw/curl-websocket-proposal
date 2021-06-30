## curl multi interface

With this approach the idea is that libcurl handles the entire websocket experience through a collection of callbacks and function calls.

## API Changes
1. Add support for `ws://` and `wss://` schemes.
1. Add `CURLOPT_WS_PRESERVE_SCHEME` option.
1. Add `CURLOPT_WS_WRITEFUNCTION` option.
1. Add `CURLOPT_WS_WRITEDATA` option.
1. Add `CURLOPT_WS_READFUNCTION` option.
1. Add `CURLOPT_WS_READDATA` option.

## Usage
The flow is along the lines of:

```
curl_global_init(CURL_GLOBAL_DEFAULT);
easy = curl_easy_init();
curl_set_easyopt(easy, CURLOPT_URL, "wss://example.com");
curl_easy_setopt(easy, CURLOPT_WS_WRITEFUNCTION writecb);
curl_easy_setopt(easy, CURLOPT_WS_READFUNCTION, readcb);

m = curl_multi_init();
curl_multi_add_handle(easy, multi);

... normal multi loop ...

```

To send data or control the caller needs to queue the data up to the sender, then make a call to `curl_easy_pause` to get the `CURLOPT_WS_READFUNCTION` called repeatedly until it is paused again.

## `CURLOPT_WS_WRITEFUNCTION`


```
#define CURL_WS_CONT   0
#define CURL_WS_BINARY 1
#define CURL_WS_TEXT   2

#define CURL_WS_CLOSE  8
#define CURL_WS_PING   9
#define CURL_WS_PONG   10

#define CURL_WS_FIRST  (1 << 4)
#define CURL_WS_LAST   (1 << 5)

/* 'user' is the user specified data from CURLOPT_WS_WRITEDATA
 *
 * 'flags' is the bitmask of the above CURL_WS_[CONT...LAST] that
 * describe where in the stream the buffer.
 *
 * 'code' is the close code when the frame is a close frame, NULL
 * otherwise.
 *
 * 'data' is the data buffer.
 *
 * 'len' is the length of the data buffer.
 *
 * Expected return value is equal to the length to continue,
 * error otherwise & the connection is closed.
 */
size_t (*write_cb)(void *user,
                   int flags,
                   const int *code,
                   const unsigned char *data,
                   size_t len)
```

## `CURLOPT_WS_READFUNCTION`

```
/* 'user' is the user specified data from CURLOPT_WS_READDATA
 *
 * 'flags' is the bitmask of the above CURL_WS_[CONT...LAST] that
 * describe where in the stream the buffer.
 *
 * 'code' is the close code when the frame is a close frame, do
 * not change otherwise.
 *
 * 'data' is the data buffer to fill in.
 *
 * 'len' is the length of the data buffer.
 *
 * Return value is the number of valid bytes passed into the data
 * buffer.
 */
size_t (*read_cb)(void *user,
                  int *flags,
                  int *code,
                  unsigned char *data,
                  size_t len);
```


## Usage
In order to use an interface this basic means you really need to understand the websocket spec fairly thoroughly.

One example is that the user can mix in control codes in between binary and text frames, but you can't mix non-control codes.  The user will need to keep track of this.

Another example is that control messages are limited to 125 bytes including any response codes.  The user must know to not violate that.

### Extensions
They are order dependent and may interfere with normal expectations/operations.  Specifying the controls for them will be several additional `CURLOPT_*` flags (at least a couple per extension).

I'm not sure how to address general user extensions in a flexible way.  They may happen before or after the default extensions.  Adding additional callbacks via some form of chain may be possible, but I'm not sure the API complexity is worth it.

RFC7692 appears to be exceptional in that while not a standard appears close & reasonably useful for many folks.

### Subprotocols
From my spot checking of the assorted [subprotocols](https://www.iana.org/assignments/websocket/websocket.xhtml) most are either limits to the way the websocket spec can be used or simply identifiers passed around.