
# Section 3: Software Engineers and iframe interactions

In Section 1 we talked about not using the embed secret, well why not? Instead of strugglingWait a minute, we are embedding without the embed secret, what gives? We don't need it anymore! We're using the API; see [Create SSO Embed URL](https://docs.looker.com/reference/api-and-integration/api-reference/v3.1/auth#create_sso_embed_url)

COMMENT: maybe above is better in the presentation

Typically in the deal cycle we would mention the API in three main areas

1. Enhancing the capabilities of Looker while its embedded
2. Providing consistency of metrics between dashboard and custom experiences
3. Providing a scalable governance layer with row level security

Let us set up the demo environment to get us ready for making API calls in the browser. In the `demo.ts` starting at line XX import our new SDK.


```
import { LookerSDK, IApiSettings, AuthToken, IError, CorsSession } from '@looker/sdk'
```


Then lets instantiate it by placing this at line XX (right underneath the import lines).

```
let sdk: LookerSDK
class EmbedSession extends CorsSession {
  async getToken() {
    const token = await sdk.ok(sdk.authSession.transport.request<AuthToken,IError>('GET', `${document.location.origin}/token`  ))
    return token
  }
}

const session = new EmbedSession({
  base_url: `https://${looker_host}:19999`,
  api_version: '3.1'
} as IApiSettings)
sdk = new LookerSDK(session)
```

The above code

1. Extends the Typescript/JS SDK and creates a class for us to use called `EmbedSession`.
2. We then tell the code how to get an authorization token for this user by creating a `getToken` function. We request this information from the dev server endpoint `/token`.

 We don't have an endpoint yet to grab an authorization token from the API;

 Lets move to `webpack-devserver.config.js` and include this new `/token` endpoint after the first `.get()` is closed:

 ```
app.get('/token', async function(req, res) {
  const token = await accessToken(user.external_user_id);
  res.json( token );
});
```

 Then import the function we need, `accessToken`, into `webpack-devserver.config.js` as well. Place this line near the top:

 ```
var { accessToken } = require('./server_utils/auth_utils')
```

 Now lets create the endpoint in `server_utils/auth_utils.ts`, at the end of the file create the `accessToken` function:

```
export async function accessToken (external_user_id: string) {
  var user = await sdk.ok(sdk.user_for_credential('embed',external_user_id))
  if (user && user.id) {
    return await sdk.ok(sdk.login_user(user['id']))
  } else {
    return {}
  }
}
```

 These were all backend server changes, so we need to restart the server. Go to your terminal and press Command+C; then `npm start` to start it again.

 Continuing on from the flow above, once the `/token` endpoint is hit, the backend server

1. We take take the user's `external_user_id` and pass it to the accessToken function that runs on the server
2. The `accessToken` function uses our Admin credentials to use the API again. First it finds the user based on the `external_user_id` but calling `user_for_credential` API.
3. Once we found the user, we can ask for the an authorization token from `login_user`. We return this back to the browser to make new requests.

Our user now has an `access_token` to make API calls as themelves.

### Verifying and Debugging

Let's verify that the access token is working by making an API call from within the front end and checking the results of it.

After our dashboard loads, we will want to make an API call and check the user's credentials. Within `setupDashboard()` lets make our first API call by inserting this within the function but at the bottom.

```
const me = await sdk.ok(sdk.me())
console.log('me', me)
```

This is the first time we're making an API call in the application so this is flow when the page reloads

1. The iframe connects and loads, then `setupDashbaord()` is run
2. We make an API call with the SDK to grab the current user's info `me()`
3. The SDK realizes it doesn't have an accessToken yet, so it will auto-login via the `getToken()` function we created above.
4. `getToken()` goes to the backend, logs into the API, retrieves an access_token for that user and returns to the browser
5. The SDK now has an access_token and will try to run the `me()`.
6. We use `console.log(me)` to output it to the console in your browser; open up your developer tools `View > Developer > Javascript Console` or `Cmd+Option+J`

You should see the name of your user in the Javascript console. console.log is an easy way to check the value of a variable while debugging our trying to understand whats happening in your code. There are other, more sophisticated ways like [breakpoints](https://developers.google.com/web/tools/chrome-devtools/javascript/breakpoints), but `console.log()` is more than adequate for now.
