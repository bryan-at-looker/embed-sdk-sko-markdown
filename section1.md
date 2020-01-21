# Section 1: Familiarity with the Embed SDK

create your .env

```
mv .env.example .env
```

Navigate to your .env and fill in your API Key and Secret


On your command line install all the packages and start the server

```
npm install
npm start

```
Webpage pops up, but there is nothing on it.


###  demo_config.ts

change your demo_config.ts

```js
// The address of your Looker instance. Required.
export const lookerHost = 'saleseng.dev.looker.com'
// A dashboard that the user can see. Set to 0 to disable dashboard.
export const dashboardId = 715

```

Changed the default dashboard on load and we're in business. But its tiny.

XX: Here is an intro to CSS, go to line XX

In `index.html` the style block

```
  <style type="text/css">
    body {
      text-align: center;
    }
  </style>
```

```
  <style type="text/css">
    body {
      text-align: center;
    }
    #demo-dashboard {
      height: 90%;
      width: 90%;
      margin: auto;
    }
    .looker-embed {
      height: 100%;
      width: 100%;
    }
  </style>
```

And we have a normal sized dashboard :party:



**Note:**

Lets walkthrough how this happens:

1. In `demo.ts` on line XX, you can see `LookerEmbedSDK.init(lookerHost, '/auth')`. This tells the Embed SDK that I'm going to use the lookerHost variable from `demo_config.ts` and the `/auth` API endpoint to generate an SSO embed URL
2. Then on line XX, `LookerEmbedSDK.createDashboardWithId(dashboardId)` is the start of the Embed SDK where we instantiate a dashboard via the dashboardId variable from `demo_config.ts`.
3. The `.build()` and  `.connect()` then use the dashboardId and lookerHost and send it to the `/auth` endpoint to generate the SSO embed URL.
4. The `/auth` endpoint is a server thats running on your laptop that simulates a backend service yoru customers may have in production. We send a request to the backend asking for an SSO embed URL; you can find it in `webpack-devserver.config.js` on line XX.
5. This takes API receives from the generated URL from the Embed SDK (`/embed/dashboards/715?embed_doman=...`) and the host from `demo_config.ts` and the user from `demo_user.json` and passes it to a function called `createSignedUrl()`
6. `createSignedUrl()` can be found in `auth_utils/auth_utils.js` and uses our new Typescript/JS SDK to create an API session and create the signed embed url to send back to the browser. The new API SDK will log us in automatically using our environment variables we placed in `.env`; all thats needed after setup is `const sso_obj = await sdk.ok(sdk.create_sso_embed_url(sso_url_params))`. Wait we're using the API for this? More on this in section 4.
7. The signed url will then be put into the `src` property of an iframe. By having the line `.appendTo('#dashboard')`, the iframe will placed in the element that has an id of `dashboard`; `<div id="dashboard"></div>`.



### Change the user in demo/demo_user.json
Navigate to the `demo_user.json` file, here we are going to change the user_name and their permissions.

**Add your own external\_user\_id in your code; e.g., yourfirstname-yourlastname-567** 

```json
{
  "external_user_id": "",
  "first_name": "First Name",
  "last_name": "Last Name",
  "session_length": 3600,
  "force_logout_login": true,
  "external_group_id": "group1",
  "group_ids": [],
  "permissions": [
    "access_data",
    "see_looks",
    "see_user_dashboards",
    "see_lookml_dashboards",
    "explore",
    "create_table_calculations",
    "download_with_limit",
    "download_without_limit",
    "see_drill_overlay",
    "save_content",
    "embed_browse_spaces",
    "schedule_look_emails",
    "schedule_external_look_emails",
    "send_to_sftp",
    "send_to_s3",
    "send_outgoing_webhook",
    "see_sql",
    "send_to_integration",
    "create_alerts"
  ],
  "models": ["thelook"],
  "user_attributes": { "locale": "en_US" }
}

```

This file does not live update like the others since demo_user.json is picked up only on `npm start`; kill the terminal session `control+c` and `npm start` again.

Changing the json file gave the user more permissions like the ability to explore from the dashboard we embedded. Check to make sure you can explore.



