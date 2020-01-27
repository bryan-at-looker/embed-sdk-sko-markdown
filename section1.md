# Section 1: Familiarity with the Embed SDK

Time: 10 minutes

## Configuring your setup
Now that you've got a pared down version of the EmbedSDK installed; you will need to configure it to talk to your Looker instance. In this example we are going to use environment variables and configurations in javascript.

### .env: Environment variables

On your command line in the integrated terminal (VS Code); rename a couple files

```
mv .env.example .env
mv ./demo/demo_user.json.example ./demo/demo_user.json
```

The Embed SDK uses `dotenv` a common Node package that will look for a `.env` and load them up for you.  For light reading on environment variables you can read [this article](https://medium.com/chingu/an-introduction-to-environment-variables-and-how-to-use-them-f602f66d15fa).

In short, we use environment variables as a way to configure the server to store necessary information **on startup** like the host name and API credentials.

Navigate to your .env and fill in your API id and secret in `LOOKERSDK_CLIENT_ID`, `LOOKESDK_CLIENT_SECRET`. If you don't have API credentials yet, log into your Looker instance and follow the directions [here](https://docs.looker.com/admin-options/settings/users#api3_keys).

Now that the .env is configured, lets start up the server.

```
npm install

## fix the bug for StringDecoder in readable-stream:

awk '{gsub(/: StringDecoder/,": any")}1' ./node_modules/@types/readable-stream/index.d.ts > tmp.txt && mv ./tmp.txt ./node_modules/@types/readable-stream/index.d.ts

echo '127.0.0.1 embed.demo' | sudo tee -a /etc/hosts

## You will need to enter your laptops password to change the /etc/hosts file

npm start

```

Webpage pops up, but there is nothing on it besides the header. Lets start configuring the frontend configurations.


###  demo_config.ts: Frontend Configs

Open up the `demo_config.ts` within the demo folder and change your dashboard_id to dashboard 5.

```js
export const dashboard_id = 5
```
This configuration is used by the EmbedSDK to create an SSO URL ([docs](https://docs.looker.com/reference/embedding/sso-embed)) for an application user which will see Looker dashboards, looks and explores.

Hot reload is turned on, if you change a configuration that affects the frontend, you should see that change made immediately. In this case, the dashboard id is used in your browser, so navigating back should show you a tiny dashboard.

## CSS Intro
An important part of web development is styling the pages and how they interact with your browser. The base of it being how elements are sized, shaped, positioned and styled through CSS. We have a very tiny dashboard because it has not been told to be bigger. We will use CSS for that

In `index.html` there is a style block that looks like

```
  <style type="text/css">
    body {
      text-align: center;
    }
  </style>
```

Lets replace that with this.


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

Save the changes and navigate to your dashboard. The above CSS says the element with an ID of `demo-dashboard` will have a height of 90% and width of 90% (relative to its parent) and its margins will be auto sized. This gives it more space on the page and centered. Our dashboard is then being embedded into the element with a class of `looker-embed` (more on this later in the EmbedSDK object). We are telling it to take up 100% of its parent, `demo-dashboard`.

And we have a normal sized dashboard :party:


Lets walkthrough what exactly is happening to make this work:

1. In `demo/demo.ts` on line XX, you can see `LookerEmbedSDK.init(lookerHost, '/auth')`. This tells the Embed SDK that I'm going to use the looker_host variable from `demo_config.ts` and the `/auth` API endpoint to generate an SSO embed URL
2. Then on line XX, `LookerEmbedSDK.createDashboardWithId(dashboardId)` is the start of the Embed SDK where we instantiate a dashboard via the dashboardId variable from `demo/demo_config.ts`.
3. The `.build()` and  `.connect()` then use the dashboardId and lookerHost and send it to the `/auth` endpoint to generate the SSO embed URL.
4. The `/auth` endpoint is a server thats running on your laptop that simulates a backend service yoru customers may have in production. We send a request to the backend asking for an SSO embed URL; you can find it in `webpack-devserver.config.js` on line XX.
5. This takes API receives from the generated URL from the Embed SDK (`/embed/dashboards/5?embed_doman=...`) and the host from `demo/demo_config.ts` and the user from `demo/demo_user.json` and passes it to a function called `createSignedUrl()`
6. `createSignedUrl()` can be found in `auth_utils/auth_utils.js` and uses our new Typescript/JS SDK to create an API session and create the signed embed url to send back to the browser. The new API SDK will log us in automatically using our environment variables we placed in `.env`; all thats needed after setup is `const sso_obj = await sdk.ok(sdk.create_sso_embed_url(sso_url_params))`. Wait we're using the API for this? More on this in Section 4. No more embed secret resets?!
7. The signed url will then be put into the `src` property of an iframe. By having the line `.appendTo('#dashboard')`, the iframe will placed in the element that has an id of `dashboard`; `<div id="dashboard"></div>`.
8. The dashboard is also being applied with a classname of `looker-embed` through the line `looker-embed` which sizes it appropriately on the page.

There is A LOT happening here with just a few lines of configuration. That is the baseline of what the EmbedSDK is great at, with just a few lines of configuration, you can provide advanced capabilities of the Looker embed without going into the complexities of the code underneath. We will be building on this concept through the sections.


### Change the user in demo/demo_user.json
There is one other configuration piece, who this user is and what attributes to they have. Navigate to the `demo_user.json` file, here we are going to change the user_name and their permissions.

**Create your own external\_user\_id in your code; e.g., yourfirstname-yourlastname-567**

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

This is similar to an environment variable in our context because only our server is using this information; we don't want the user to be able to choose their own external_group_id and user_attributes.

Now everything is configured and ready for us to start playing with configuring properties sfor the EmbedSDK.
