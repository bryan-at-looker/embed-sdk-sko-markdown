# Section 4: API Introductions

Remove the options from your dropdown in the HTML (in `index.html`). You're going to want to remove several lines of code starting after `<select id="select-dropdown">` and before `</select>`; you will be removing all the options. Paste the below into that spot you just removed.

```
<option value="">Select...</option>
```


We're going to be running a query from within the front end. Lets create a couple variables that we wil use to make the API call. Place this at the bottom of the `demo_config.ts`

```js
export const query_object = {
  "model": "thelook",
  "view": "order_items",
  "fields": ["users.state"],
  "sorts": ["users.state asc"],
  "filters": {}
}
// field name we will want to extract for dropdown
export const query_field_name = 'users.state'
```

Place this with the other imports near the top of `demo.ts`

```
import {  query_object, query_field_name } from './demo_config'
```

At the bottom of demo.ts we will create a function that creates new `<option/>` HTML tags for us. We will send the function the list of states, and it will return the options for us.

```js

function addStateOptions(data: any) {
  const dropdown = document.getElementById('select-dropdown')
  data.forEach(function (row: any, i: number) {
    if (query_field_name && row[query_field_name] && dropdown) {
      const new_option = document.createElement('option')
      new_option.value = row[query_field_name]
      new_option.innerHTML = row[query_field_name]

      // replace or create the dropdown item
      if (dropdown.children[i+1]) {
        dropdown.children[i+1].replaceWith(new_option)
      } else {
        dropdown.appendChild(new_option)
      }
    }
  })
}
```

We have to get the list of states so lets run an API call that grabs all states and passes it to our new function. We will want to run this after the dashboard loads, so place it after our first API call `sdk.me()` within the function `setupDashboard(...)`

```js
  const states = await sdk.ok(sdk.run_inline_query(
    {
      body: query_object,
      result_format: 'json'
    }
  ))
  addStateOptions(states)
```

We've got a nice long list of states / counties in there, lets narrow that down to 50. Move to `demo_user.json` and give the user a country user attribute like this:

```
  "user_attributes": {
    "locale": "en_US",
    "country": "USA"
  }
```

Remember anything you are changing about your user is only done server side, so in this case you need to go to your terminal and Control+C to exit the server and then `npm start` to start it up again.

Check your dropdown, you should just see the US states. Remember, we've setup the embed SDK to only make API calls as that user, we can now keep everything in sync from model to iframe to API calls.

Limit to the top 10 states by total gross margin. In`demo_config.ts` swap out the `query_object` for below.

```
export const query_object = {
  "model": "thelook",
  "view": "order_items",
  "fields": ["users.state", "order_items.total_gross_margin"],
  "sorts": ["order_items.total_gross_margin desc"],
  "limit": "10",
  "filters": {}
}
```

`demo.ts` include row number `i` and place it inside the HTML. Replace the entire  `addStateOptions` function with the block below.

```js
function addStateOptions(data: any) {
  const dropdown = document.getElementById('select-dropdown')
  data.forEach(function (row: any, i: number) {
    if (query_field_name && row[query_field_name] && dropdown) {
      const new_option = document.createElement('option')
      new_option.value = row[query_field_name]
      new_option.innerHTML = `${i+1}) ${row[query_field_name]}`
      dropdown.appendChild(new_option)

      // replace or create the dropdown item
      if (dropdown.children[i+1]) {
        dropdown.children[i+1].replaceWith(new_option)
      } else {
        dropdown.appendChild(new_option)
      }
    }
  })
}
```

###Keep dashboard filters in sync with API calls

We're now showing "Trending States" in the dropdown and giving the end user context to the order (no magnitude... yet). But its static to our initial query, top states of all time by gross margin. An idea to make it more dynamic is to have the dropdown respond to the changes from within the iframe. When a user changes the date filter, maybe we change the dropdown too? Here is how you would do that.

Create new constants in `demo_config.ts`

```
// map the dashboard date filter to the query date filter
export const dashboard_date_filter = 'Date'
export const query_date_filter = 'order_items.created_date'
```


Import the constants into `demo.ts` near the top with the other imports

```
import { dashboard_date_filter, query_date_filter } from './demo_config'
```


Create new function for when the dashboard filters update for date in `demo.ts`. You can put this at the very bottom of the file.

```js
async function filtersUpdates( event: any ) {
  console.log('dashboard:filters:changed', event)
  // instantiate elements, filters, and query objects
  const dashboard_filters: any = (event && event.dashboard && event.dashboard.dashboard_filters) ? event.dashboard && event.dashboard.dashboard_filters : undefined
  let dropdown = document.getElementById('select-dropdown')
  let new_filters = query_object.filters

  // update query object and run query

  if (dashboard_filters && ( dashboard_date_filter in dashboard_filters ) ) { // check to make sure our filter is in the changed
    if (dropdown) { // check to make sure we found our elements to update/keep
      new_filters = Object.assign(new_filters, { [query_date_filter]: dashboard_filters[dashboard_date_filter] })
      const states = await sdk.ok(sdk.run_inline_query(
        {
          body: Object.assign(query_object, { filters: new_filters }),
          result_format: 'json'
        }
      ))
      addStateOptions(states)
    }
  }
}
```

Then lets add the listener from the Embed SDK to call this function after it gets a filter changed

```
.on('dashboard:filters:changed', filtersUpdates)
```

Change the date filter on the dashboard and check the dropdown list, usually you'll see some movements
