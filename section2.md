


# Section 2: Product Manager and Designer Basics

Default filters on load:

Add this to demo_config.ts

```

// the name of the filter on the dashboard for state
export const dashboardStateFilter = 'State'


```
In `demo.ts`:

Update Line XX to import the new variable from demo_config.ts
```
import { lookerHost, dashboardId, dashboardStateFilter } from './demo_config'
```
Add this to LookerEmbedSDK above `.build()`

```

.withFilters({[dashboardStateFilter]: 'California'})


```


Changing the theme: Can I change the CSS, colors, font?

Add this to the LookerEmbedSDK above `.build()`
```
.withTheme('currency_white')
```

Bonus: Add in a special theme for this user's external group_id. Change `withTheme()` to

```
.withTheme( (user.external_group_id == 'customerABC') ? 'dialpad' : 'currency_white' )
```
The logic above says: 
> If external_group_id == customerABC then display dialpad theme, else use currency_white theme

### Adding basic custom filters
We need a select dropdown and state options to change the state. Add this righ after `<h3>SKO 2020 Embed SDK Walkthrough</h3>`

```
  <select id="select-dropdown">
    <option value="">Select...</option>
    <option value="California" selected>California</option>
    <option value="Illinois">Illinois</option>
    <option value="Minnesota">Minnesota</option>
  </select>
```
This puts the dropdown with California selected (our default filter from above). But when we change it, nothing happens. What we want to do is listen for the change in the dropdown and apply it to the iframe using [Javascript Events](https://docs.looker.com/reference/embedding/embed-javascript-events). The Embed SDK facilitates the communication by providing build in functions.

In the series of functions(?) within `LookerEmbedSDK` in `demo.ts` we have a `.then(setupDashboard)`. What this says is after we've connected the iframe to the page, we want to run the `setupDashboard` function. In this function, we'll want to listen to a change of the dropdown and apply the new value to the iframe. replace the current blank `const setupDashboard = (...)...{...}` block with this

```
const setupDashboard = (dashboard: LookerEmbedDashboard) => {
  const dropdown = document.getElementById('select-dropdown')
  if (dropdown) {
	dropdown.addEventListener('change', (event) => { dashboard.updateFilters({ [dashboardStateFilter]: (event.target as HTMLSelectElement).value }) })
  }
}
```

Your dashboard is now responding to your custom filter!

Lets break down whats happening:

1. When your dashboard loads, we fun the `setupDashboard` function.
2. In this function, we find the element responsible for changing the dropdown, `select-dropdown`, and save it as a variable, `dropdown`
3. When we find it, we add a listener to it; every time the value changes, we then perform an action.
4. The action we perform is to run `dashboard.updateFilters()`, the Embed SDK function provided to us to facilitate communication to the iframe. Instead of relying on writing your own [postMessage() event](https://docs.looker.com/reference/embedding/embed-javascript-events#posting_the_request_to_the_iframes_contentwindow_property)
5. We send a JSON object, and the iframe responds only to the values we've placed in the JSON object (notice how our other filters don't change.

Let's embellish this and run the dashboard everytime the value changes. On line XX, within the `dropdown.addEvenListener(...{...})`, after the `dashbpard.updateFilters`insert a `dashboard.run()`

You full setupDashboard will look like this now.

```
const setupDashboard = (dashboard: LookerEmbedDashboard) => {
  const dropdown = document.getElementById('select-dropdown')
  if (dropdown) {
    dropdown.addEventListener('change', (event) => { 
      dashboard.updateFilters({ [dashboardStateFilter]: (event.target as HTMLSelectElement).value }) 
      dashboard.run()
    })
  }
}
```

This will now change the filter and run the dashboard everytime the value is changed.
