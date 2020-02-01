# Section 4: API Introductions

**Note:**
If you would like to fast forward to the beginning of this section you can `git checkout section4 --force`. If you would like to start this section over at any time, you can run `git reset --hard`. If you would like to see what the files at the end of this section look like, you can check out the [Github here](https://github.com/bryan-at-looker/embed-sdk-sko/tree/section4end)

Remove the options from your dropdown in the HTML (in `index.html`). You're going to want to remove several lines of code starting after `<select id="select-dropdown">` and before `</select>`; you will be removing all the options. Paste the below into that spot you just removed.

```
<option value="">Select...</option>
```


We're going to be running a query from within the frontend.  Let's create a few variables that we'll use to make the API call.  Place this at the bottom of `demo_config.ts`:

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

Place this with the other imports near the top of `demo.ts`:

```
import {  query_object, query_field_name } from './demo_config'
```

At the bottom of demo.ts we will add a function that creates new `<option/>` HTML tags for us. We will send the function the list of states, and it will return the options for us.

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

We have to get the list of states so let's run an API call that grabs all states and passes it to our new function.  We want to run this after the dashboard loads, so place it on a new line after `const me = await sdk.ok(sdk.me())` within the function `setupDashboard(...)`

```js
  const states = await sdk.ok(sdk.run_inline_query(
    {
      body: query_object,
      result_format: 'json'
    }
  ))
  addStateOptions(states)
```

Now check the dropdown - we've got a nice long list of states and locales in other countries; let's narrow that down to U.S. states. Move to `demo_user.json` and give the user a country user attribute similar to what you see coded below. This is another server side change, so in this case you need to go to your terminal and Control+C to exit the server and then `npm start` after updating the user attribute.

```
  "user_attributes": {
    "locale": "en_US",
    "country": "USA"
  }
```

Check your dropdown, you should see just the U.S. states. Remember, we've setup the Embed SDK to make API calls only as that user - we can now keep everything in sync from model to iframe to API calls.

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

Let's add a ranking in front of the states (eg. 1) California). In your `demo.ts` file, replace the entire  `addStateOptions` function with the block below:

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

### Dropdown list powered by dashboard filter

30 days drpdown
![30 days](https://bryan-at-looker.s3.amazonaws.com/images/embed-sdk-sso/section5-state-top10-30days.png)

7 days drpdown
![7 days](https://bryan-at-looker.s3.amazonaws.com/images/embed-sdk-sso/section5-state-top10-7days.png)

We're now showing "Trending States" in the dropdown and giving the end user context to the order (no magnitude... yet). But it's static to our initial query, top states of all time by gross margin. An idea to make it more dynamic is to have the dropdown respond to the changes from within the iframe. When a user changes the date filter, maybe we change the dropdown too? Here is how you would do that.

Create new constants in `demo_config.ts` (paste this at the end of the file):

```
// map the dashboard date filter to the query date filter
export const dashboard_date_filter = 'Date'
export const query_date_filter = 'order_items.created_date'
```


Import the constants into `demo.ts` near the top with the other imports:

```
import { dashboard_date_filter, query_date_filter } from './demo_config'
```


Create a new function for when the dashboard filters update for date in `demo.ts`. You can put this at the very bottom of the file.

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

Then let's add the listener from the Embed SDK (in your `demo.ts` file) to call this function after it gets a filter changed (make sure you paste it before the `.build()`):

```
.on('dashboard:filters:changed', filtersUpdates)
```

Change the date filter on the dashboard and check the dropdown list (no need to actually run the dashbaord with the updated date filter).  Usually you'll see some movements in states 8-10.
