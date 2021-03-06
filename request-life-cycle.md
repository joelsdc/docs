Title: aah Request Lifecycle
Desc: Incoming requests are handled by the server via routes. Each incoming request passes through a pre-defined list of steps, user-defined steps and optional server extensions. Each route describes an HTTP endpoint with a path, method, controller, and action.
Keywords: aah request lifecycle, request lifecycle, incoming request, lifecycle
---
# aah Request Lifecycle

Incoming requests are handled by the server via routes. Each route describes an HTTP endpoint with a `path`, `method`, `controller`, and `action`. Each incoming request passes through a pre-defined list of steps, user-defined steps and optional server extensions.

It would be nice to have diagram to explain the Lifecycle, will do my best in the upcoming release.

## Lifecycle

  * `aah.Context` is prepared for the request. i.e. parsing and creating `ahttp.Request` instance and `ahttp.ResponseWriter` instance.
  * Sets the Request ID header if enabled
  * `OnRequest` server extension point: Always called, the `aah.Context` object instance is passed via event data. `aah.Context` is decorated with `SetURL` and `SetMethod` methods. Calls to these methods will impact how the request is routed and can be used for rewrite rules.
      - Note: route is not evaluated at this point.
  * Route Lookup: Based on request path.
      - If it's static file route then server process that request via `http.ServeContent`.
      - If no route found then below one of the action performed based on request.
          - If `Redirect Trailing Slash` opportunity is found then server redirects that request to the new URL. It can be controlled via [`routes.conf`](routes-config.html).
          - It does HTTP auto `OPTIONS`. User defined `OPTIONS` take precedence over auto. It can be controlled via [`routes.conf`](routes-config.html).
          - It does HTTP auto `405 Method Not Allowed`. It can be controlled via [`routes.conf`](routes-config.html).
          - If custom not found controller is defined, server calls that controller action.
      - Finally it skips to `OnPreReply` server extension point and writing response on the wire.
  * Route Found: `aah.Context` is updated with targeted controller and action information. If controller or action is not found in the registry. Flow skips to `OnPreReply` server extension point with `404 Not Found` and writing response on the wire.
      - Path Variables are parsed and available at this point via `ctx.Req`.
  * Parse Session Cookie if the session mode is `stateful`
  * `OnPreAuth` server extension point **`upcoming`**.
  * Authenticate request **`upcoming`**.
  * Performs Authorization **`upcoming`**.
  * `OnPostAuth` server extension point **`upcoming`**.
  * Read and Parse Request
      - For `GET` method request parse Query parameters
      - For not `GET` method. Query parameters and Payload, Form, Multi-part based on content-type.
      - Sanitize request values to prevent XSS [know more](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet) **`upcoming`**
  * Validate request values **`upcoming`**
  * User-defined middleware(s) execution (basically before `m.Next(ctx)` call).
  * Controller interceptor `Before` is called if exists.
  * Controller-Action interceptor `Before<ActionName>` is called if exists.
  * Targeted controller `Action` is called.
  * Controller-Action interceptor `After<ActionName>` is called if exists.
  * Controller interceptor `After` is called if exists.
  * Controller interceptor `Finally` is called if exists.
  * ***Note:*** If any `panic` happens around controller action interceptor `Panic` is called on that controller.
  * User-defined middleware(s) execution (basically after `m.Next(ctx)` call).
  * If the Response is already sent via `ctx.Res` and `ctx.Reply().Done()` is called then request completes here.
  * If it's a Redirect reply then framework redirects it.
  * Determines response content-type if it's not set in the reply builder.
  * If it's HTML content-type then
      - Finds a view based on controller & action and master layout if it's not overridden in the reply builder.
      - Populates view args with framework provided values.
  * Renders reply body based on content-type, if any errors it logs and sends meaningful error message.
  * If HTTP client supports the Gzip compression and it's enabled in the config then Gzip response writer is used.
  * `OnPreReply` server extension point: Always called, you're allowed to modified `ctx.Reply()` it will reflect on response. Except when-
      - `ctx.Reply().Done()` is called, refer godoc for more info.
      - `ctx.Reply().Redirect(...)` is called.
  * Writing reply on the wire-
      - Response Header(s).
      - Cookies
      - Response Status - default is 200 OK, if not provided.
      - Response body bytes.
  * `OnAfterReply` server extension point: Always called. Response is already written on the wire. Nothing we can do about the response, however context has a valuable information such as response bytes size, response status code, etc. Except when-
      - `ctx.Reply().Done()` is called, refer godoc for more info.
      - `ctx.Reply().Redirect(...)` is called.
