


# Section 2: Product Manager and Designer Basics

**Note:**
If you would like to fast forward to the beginning of this section you can `git checkout section2 --force`. If you would like to start this section over at any time, you can run `git reset --hard`. If you would like to see what the files at the end of this section look like, you can check out the [Github here](https://github.com/bryan-at-looker/embed-sdk-sko/tree/section3)

### Default filters on load:

Add this to `demo_config.ts` at the bottom:

```
// the name of the filter on the dashboard for state
export const dashboard_state_filter = 'State'
```

In `demo.ts`, add a new line at the top to import the new variable from demo_config.ts:

```
import {  dashboard_state_filter } from './demo_config'
```

Add this to LookerEmbedSDK above `.build()`:


```
.withFilters({[dashboard_state_filter]: 'California'})

```


Changing the theme: Can I change the CSS, colors, font?

Add this above `.build()`:

```
.withTheme('sko')
```

Bonus: Add in a special theme for this user's external group_id. Change `withTheme()` to

```
.withTheme( (user.external_group_id == 'group2') ? 'sko' : 'ugly' )
```
The logic above says:
> If external\_group_id is equal to group2, then display the sko theme, else use ugly theme

Feel free to revert it :)

### Adding basic custom filters
We need a select dropdown with state options to change the state filter. In `index.html`, at the bottom, add this right after `<h3>SKO 2020 Embed SDK Walkthrough</h3>`:

```
  <select id="select-dropdown">
    <option value="">Select...</option>
    <option value="California" selected>California</option>
    <option value="Colorado">Colorado</option>
    <option value="Illinois">Illinois</option>
    <option value="New York">New York</option>
    <option value="Texas">Texas</option>
 </select>
```

This puts the dropdown with California selected (our default filter from above). But when we change it, nothing happens. What we want to do is listen for the change in the dropdown and apply it to the iframe using [Javascript Events](https://docs.looker.com/reference/embedding/embed-javascript-events) and the Embed SDK. The Embed SDK facilitates this communication by providing built-in functions to listen and respond to these events.

Looking at the code block in `demo.ts` after  `LookerEmbedSDK.createDashboardWithId(dashboard_id)`, we have `.then(setupDashboard)`. What this says is after we've connected the iframe to the page, we want to run the `setupDashboard` function. In this function, we'll want to listen to a change of the dropdown and apply the new value to the iframe. replace the current blank `const setupDashboard = (...)...{...}` block with this

```
const setupDashboard = async (dashboard: LookerEmbedDashboard) => {
  const dropdown = document.getElementById('select-dropdown')
  if (dropdown) {
  dropdown.addEventListener('change', (event) => { dashboard.updateFilters({ [dashboard_state_filter]: (event.target as HTMLSelectElement).value }) })
  }
}
```

Play around with the filter; your dashboard should be responding to the new dropdown.

**Note:** There may be some delay between the dropdown and the filter updating thats normal.

Let's break down whats happening:

1. When your dashboard loads, the `setupDashboard` function runs.
2. In this function, we find the element responsible for changing the dropdown, `select-dropdown`, and save it as a variable `dropdown`
3. When we find it, we add a listener to it; every time the value changes, we then perform an action.
4. The action we perform is to run `dashboard.updateFilters()`, the Embed SDK function provided to us to facilitate communication to the iframe.  Much easier than writing your own [postMessage() event](https://docs.looker.com/reference/embedding/embed-javascript-events#posting_the_request_to_the_iframes_contentwindow_property).
5. We send a JSON object of the filters we want to apply; we don't need to give all of them, just the ones we want to change. The iframe responds only to the values we've placed in the JSON object; notice how our other filters don't change

It would be nice if we didn't have to hit the run button so much - for a dropdown like this, we can run right after we've selected it. Within the `dropdown.addEventListener(...{...})`, after the `dashboard.updateFilters` line, insert a new line and paste `dashboard.run()`

Your full setupDashboard will now look like this:

```
const setupDashboard = async (dashboard: LookerEmbedDashboard) => {
  const dropdown = document.getElementById('select-dropdown')
  if (dropdown) {
    dropdown.addEventListener('change', (event) => {
      dashboard.updateFilters({ [dashboard_state_filter]: (event.target as HTMLSelectElement).value })
      dashboard.run()
    })
  }
}
```

This will now change the filter and run the dashboard everytime the dropdown value is changed.

Another common ask is to make sure that multiple scrollbars don't appear:  one within the iframe and one on the parent page (you may not see this in your example right now depending on the size of your browser window relative to the size of the iframe)

![HTML height](https://bryan-at-looker.s3.amazonaws.com/images/embed-sdk-sso/section2-height-scroll-before.png)

In order to do this, we need to listen to metadata that Looker is sending out to the parent page through Javascript events. The `page:properties:changed` event ([docs](https://docs.looker.com/reference/embedding/embed-javascript-events#page:properties:changed)) gives us the ability to listen to the height of the Looker iframe and then we can dynamically change the style of the parent `div, #dashboard`.

The `page:properties:changed` event looks like this:

```
{
  "type": "page:properties:changed",
  "height": 2444,
  "dashboard": {
    "id": 2,
    "title": "Business Pulse",
    "dashboard_filters": {
      "Date": "30 days",
      "State": "California"
    },
    "absoluteUrl": "https://sko2020.dev.looker.com/embed/dashboards/2?embed_domain=https:%2F%2Fembed.demo:8080&sdk=2&State=California&theme=sko&Date=30%20days&filter_config=%7B%22Date%22:%5B%7B%22type%22:%22past%22,%22values%22:%5B%7B%22constant%22:%2230%22,%22unit%22:%22day%22%7D,%7B%7D%5D,%22id%22:0%7D%5D,%22State%22:%5B%7B%22type%22:%22%3D%22,%22values%22:%5B%7B%22constant%22:%22California%22%7D,%7B%7D%5D,%22id%22:1%7D%5D%7D",
    "url": "/embed/dashboards/2?embed_domain=https:%2F%2Fembed.demo:8080&sdk=2&State=California&theme=sko&Date=30%20days&filter_config=%7B%22Date%22:%5B%7B%22type%22:%22past%22,%22values%22:%5B%7B%22constant%22:%2230%22,%22unit%22:%22day%22%7D,%7B%7D%5D,%22id%22:0%7D%5D,%22State%22:%5B%7B%22type%22:%22%3D%22,%22values%22:%5B%7B%22constant%22:%22California%22%7D,%7B%7D%5D,%22id%22:1%7D%5D%7D"
  }
}
```

First let's add a new function at the bottom of `demo.ts` that will listen to the event and apply the change to the height of the element that contains the iframe:

```
function changeHeight( event: any ) {
  console.log(event)
  const div = document.getElementById('dashboard')
  if (event && event.height && div) {
    div.style.height = `${event.height+15}px`
  }
}
```

We'll get a little bit of a buffer by adding 15 pixels.

Then within the Embed SDK code (still in our `demo.ts` file) we need to tell it to perform the function below when it receives the `page:properties:changed` event. Copy and paste this function on its own line above `.build()`:

```
.on('page:properties:changed', changeHeight )
```



![HTML height](https://bryan-at-looker.s3.amazonaws.com/images/embed-sdk-sso/section2-height-html.png?raw=true)

The scroll bar is now on the outer page and not in the iframe, it feels more native and prevents the "multiple scroll bar problem".
![HTML height](https://bryan-at-looker.s3.amazonaws.com/images/embed-sdk-sso/section2-height-scroll-after.png)
