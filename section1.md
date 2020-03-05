# Section 1: Familiarity with the Embed SDK

[Download VS Code here](https://code.visualstudio.com/download) (do it, it is great)

When you open up VS Code, press Command+Shift+P, type in `Shell Command: Install ...` and choose the option `Shell Command: Install 'code' command in PATH`

Move to your terminal and we're going to clone a different repository based off of the embed sdk

```
git clone https://github.com/bryan-at-looker/embed-sdk-sko
cd embed-sdk-sko
git checkout section1
code .
```

Command+Shift+P, type in `create new integrated terminal`

A terminal appears at the bottom, thats where we'll be doing the rest of our commands.

## Configuring your setup
Now that you've got a pared down version of the Embed SDK installed; you will need to configure it to talk to your Looker instance. In this example we are going to use environment variables and configurations in javascript.

### .env: Environment variables

On your command line in the integrated terminal (VS Code); rename a couple files

```
mv .env.example .env
mv ./demo/demo_user.json.example ./demo/demo_user.json
```

The Embed SDK uses `dotenv` a common Node package that will look for a `.env` and load them up for you.  For light reading on environment variables you can read [this article](https://medium.com/chingu/an-introduction-to-environment-variables-and-how-to-use-them-f602f66d15fa).

In short, we use environment variables as a way to configure the server to store necessary information **on startup** like the host name and API credentials.

Head over to the SKO instance, [https://sko2020.dev.looker.com/admin/users](https://sko2020.dev.looker.com/admin/users), and create your API 3 keys.  Navigate to your .env and fill in your API id and secret in `LOOKERSDK_CLIENT_ID`, `LOOKERSDK_CLIENT_SECRET`. (To open a file in VSCode, simply click the filename on the left pane)

Now that the .env is configured, lets start up the server.  Note that you will see error messages when `npm install` runs.  This is expected and will be fixed with the subsequent `awk` command. Run the following commands in your VS Code in the integrated terminal.

```
npm install

## fix the bug for StringDecoder in readable-stream:

awk '{gsub(/: StringDecoder/,": any")}1' ./node_modules/@types/readable-stream/index.d.ts > tmp.txt && mv ./tmp.txt ./node_modules/@types/readable-stream/index.d.ts

echo -e '\n127.0.0.1 embed.demo\n' | sudo tee -a /etc/hosts

## You will need to enter your laptop's password to change the /etc/hosts file

npm start

```

Webpage pops up, see "Your Connection is Not Private".  This is OK - click advanced and continue.

Our webpage is simply a header at this point.  Let's start configuring the frontend.


###  demo_config.ts: Frontend Configs

In VSCode, open the file `demo_config.ts` within the demo folder and change your dashboard_id to 5.

```js
export const dashboard_id = 5
```
Save the file (Cmd S shortcut)

This configuration is used by the Embed SDK to create an SSO URL ([docs](https://docs.looker.com/reference/embedding/sso-embed)) for an application user which will see Looker dashboards, looks and explores.

Note that hot reload is enabled:  if you change a configuration file that affects the frontend, you should see the effects of that change immediately - the browser will auto-refresh. In this case, now that a dashboard id has been specified, our dashboard (albeit a tiny one!) is rendered.

## CSS Intro
An important part of web development is styling the pages and how they interact with your browser. The basics being how page elements are sized, shaped, positioned and styled through Cascading Style Sheets (CSS). We have a tiny dashboard because it has not been told to be bigger. We will use CSS to increase the size of the dashboard - effectively making the iframe bigger.

Open the file `index.html`.  There's a style block that looks like:

```
  <style type="text/css">
    body {
      font-family: "Comic Sans MS", cursive, sans-serif;
      text-align: center;
    }
  </style>
```

Let's replace the above with this:

```
  <style type="text/css">
    body {
      font-family: "Comic Sans MS", cursive, sans-serif;
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

Save the changes and navigate to your dashboard. The above CSS says the element with an ID of `demo-dashboard` will have a height of 90% and width of 90% (relative to its parent) and its margins will be auto-sized. This gives it more space on the page and also centers it. Our dashboard is then being embedded into the element with a class of `looker-embed` (more on this later in the Embed SDK object). We are telling it to take up 100% of its parent, `demo-dashboard`.

And now we have a normal-sized dashboard :party:


Lets walkthrough what exactly is happening to make this work:

1. In `demo/demo.ts` you can see `LookerEmbedSDK.init(lookerHost, '/auth')`. This tells the Embed SDK that I'm going to use the looker_host variable from `demo_config.ts` and the `/auth` API endpoint to generate an SSO embed URL
2. Then `LookerEmbedSDK.createDashboardWithId(dashboardId)` is the start of the Embed SDK where we instantiate a dashboard via the dashboardId variable from `demo/demo_config.ts`.
3. The `.build()` and  `.connect()` methods take the dashboardId and lookerHost and send to the `/auth` endpoint to generate the SSO embed URL.
4. The `/auth` endpoint is on a server that's running on your laptop that simulates a backend service your customers may have in production. We send a request to the backend asking for an SSO embed URL; you can find it in `webpack-devserver.config.js`.
5. This endpoint receives the generated URL from the Embed SDK  (`/embed/dashboards/5?embed_doman=...`) and the host from `demo/demo_config.ts` and the user from `demo/demo_user.json` and passes it to a function called `createSignedUrl()`
6. `createSignedUrl()` can be found in `auth_utils/auth_utils.js` and uses our new Typescript/JS SDK to create an API session and create the signed embed URL to send back to the browser. The new JS SDK will log us in automatically using the environment variables we placed in `.env`; all that's needed after setup is `const sso_obj = await sdk.ok(sdk.create_sso_embed_url(sso_url_params))`. But wait!  We're using the API for this?  More on this in Section 4. No more embed secret resets?!
7. The signed URL will then be put into the `src` property of an iframe. By having the line `.appendTo('#dashboard')`, the iframe will placed in the element with an id of `dashboard`; `<div id="dashboard"></div>`.
8. The dashboard is also being applied with a classname of `looker-embed` through the line `looker-embed` which sizes it appropriately on the page.

There is A LOT happening here with just a few lines of configuration. That is the baseline of what the Embed SDK is great at:  with just a few lines of configuration, you can provide advanced capabilities of Looker embedding without going into the complexities of the underlying code. We will be building on this concept through the remaining sections.  You can see where .build(), .connect(), etc are documented in the EmbedBuilder class
[Embed SDK Reference](https://looker-open-source.github.io/embed-sdk/)

### Change the user in demo/demo_user.json
There is one other configuration piece:  who this user is and what attributes to they have. Open the file `demo_user.json`.  Here we are going to change the user_name and their permissions.

**Create your own external\_user\_id in your code; e.g., yourfirstname-yourlastname-567**

```json
{
  "external_user_id": "replace-this",
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

This file does not live update via hot reload like the others since demo_user.json is picked up only upon `npm start`; kill the terminal session `control+c`, close your existing browser tab that has the embedded dashboard, and `npm start` again.  A new browser tab will open.

Changing the json file gave the user more permissions like the ability to create alerts, plus explore from the dashboard we embedded. Check to make sure you can create alerts and explore.

This user json file acts similar to an environment variable in our context because only our server is using this information; we don't want the user to be able to choose their own external_group_id and user_attributes.

Now everything is configured and ready for us to start playing with configuration properties for the Embed SDK.
