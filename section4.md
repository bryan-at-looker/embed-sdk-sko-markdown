# Section 4: API Introductions

Start with no options in your dropdown 

 ```
   <select id="select-dropdown">
    <option value="">Select...</option>
  </select>
 ```
 
Create an API object that we will run and place in demo_config.js:

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

In demo.ts import query_object:

```
import { lookerHost, dashboardId, dashboardStateFilter, query_object, query_field_name } from './demo_config'
```

At the bottom of demo.ts create a new function that will take a list of json responses

```js

function addStateOptions(data: any) {
  const dropdown = document.getElementById('select-dropdown')
  data.forEach(function (row: any) {
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

Run an API call that grabs all states and passes it to our new function. We will want to run this after the dashboard loads, so place it after our first API call

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

Limit to the top 10 states by total gross margin

`demo_config.ts` add in new fields, change sorts, limit to 10.

```
export const query_object = {
  "model": "thelook",
  "view": "order_items",
  "fields": ["users.state", "order_items.total_gross_margin"],
  "sorts": ["order_items.total_gross_margin desc"],
  "limit": "10"
}
```

`demo.ts` include row number `i` and place it inside the HTML.

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

Keep dashboard filters in sync with API calls

function that clears options, reruns query

```json
{
  "type": "dashboard:filters:changed",
  "dashboard": {
    "id": 2,
    "title": "Business Pulse",
    "dashboard_filters": {
      "Date": "90 days",
      "State": "New York"
    },
    "absoluteUrl": "https://sko2020.dev.looker.com/embed/dashboards/2?embed_domain=https:%2F%2Fembed.demo:8080&sdk=2&State=New%20York&theme=sko&Date=90%20days&filter_config=%7B%22Date%22:%5B%7B%22type%22:%22past%22,%22values%22:%5B%7B%22constant%22:%2290%22,%22unit%22:%22day%22%7D,%7B%7D%5D,%22id%22:0%7D%5D,%22State%22:%5B%7B%22type%22:%22%3D%22,%22values%22:%5B%7B%22constant%22:%22New%20York%22%7D,%7B%7D%5D,%22id%22:1%7D%5D%7D",
    "url": "/embed/dashboards/2?embed_domain=https:%2F%2Fembed.demo:8080&sdk=2&State=New%20York&theme=sko&Date=90%20days&filter_config=%7B%22Date%22:%5B%7B%22type%22:%22past%22,%22values%22:%5B%7B%22constant%22:%2290%22,%22unit%22:%22day%22%7D,%7B%7D%5D,%22id%22:0%7D%5D,%22State%22:%5B%7B%22type%22:%22%3D%22,%22values%22:%5B%7B%22constant%22:%22New%20York%22%7D,%7B%7D%5D,%22id%22:1%7D%5D%7D"
  }
}
```

Create new constants

```
// map the dashboard date filter to the query date filter
export const dashboard_date_filter = 'Date'
export const query_date_filter = 'order_items.created_date'
```


Import the constants

```
import { lookerHost, dashboardId, dashboardStateFilter, query_object, query_field_name, dashboard_date_filter, query_date_filter } from './demo_config'
```


Create new function for when the dashboard filters update for date

```js
async function filtersUpdates( event: any ) {
  console.log(event)
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

add the filter listener in the embed sdk

```
.on('dashboard:filters:changed', filtersUpdates)
```
