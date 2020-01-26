# Section 6

### Saving dashboard filters using localstorage

User changes a filter, can we save it and bring up the last values?

Set at the end of filtersUpdates

```js
  if (dashboard_filters) {
    localStorage.setItem('dashboard_filters', JSON.stringify(dashboard_filters)); 
  }
```

set this right before the EmbedSDK instantiation on line XX

```
const last_filters = JSON.parse(localStorage.getItem('dashboard_filters') || '{}');
```

Then call last_filters using `withFilters()`; replace the current `withFilters()` with this

```
.withFilters(last_filters)
```

Set this on line XX where we are using listening to dropdown clicks and updating values. We will want to update the same values.

```js
      const last_filters = JSON.parse(localStorage.getItem('dashboard_filters') || '{}');
      localStorage.setItem('dashboard_filters', JSON.stringify(Object.assign(last_filters, new_filter)))
```

Refresh the page or open the current URL in a new tab to see the localStorage in action. Your last filters will be populated on load.


## Click Events
Measure, dimension, and link clicks are tracked on click. With Javacript event we can capture the event, with the Embed SDK, we can capture and CANCEL the click.
### Capture Drill Menu Clicks

Create a function at the end of demo.ts

**NOTE:**

Currently drill menu clicks don't fire with Dashboards Next (but will soon), so comment out the `.withNext()` line.

```js
async function drillClick ( event: any) {
  console.log('drillmenu:click', event)
  return {cancel: false}
}
```

listen to drillmenu:click

```
.on('drillmenu:click', drillClick)
```

When we click around on the dashboard we start to see the event come through. Here's an example:


```js
{
  "type": "drillmenu:click",
  "label": "Show All $12,992.10",
  "link_type": "measure_default",
  "modal": true,
  "url": "/embed/explore/thelook/order_items?fields=order_items.id,order_items.order_id,order_items.status,order_items.created_date,order_items.sale_price,products.brand,products.item_name,users.name,users.email&f[order_items.day_in_period]=5&f[order_items.previous_period_filter]=30+days&f[order_items.previous_period]=Previous+Period&f[users.state]=&limit=500"
}
```



### Cancelling Clicks

Notice the return.. click events require you to return an object with whether or not to cancel the click. We left it off

Cancel all drill menu clicks

```js
return {cancel: true}
```

Notice this doesn't stop the dropdown menu from appearing, but will prevent the automatic drills + clicks from within the menu.

### Advanced click interactions

Open an explore page instead of drill modal underneith the dashboard. Replace the `drillMenu()` function with the following:

```js
async function drillClick ( event: any) {
  console.log('drillmenu:click', event)
  if (event && event.modal) {
    const dashboard_div = document.getElementById('dashboard')
    if ( dashboard_div && dashboard_div.children.length > 1 && dashboard_div.lastChild) {
      dashboard_div.lastChild.remove()
    }
    LookerEmbedSDK
    .createExploreWithUrl(`https://${looker_host}${event.url}`)
    .appendTo('#dashboard')
    .withId('cool')
    .build()
    .connect()
    .then()
    .catch((error: Error) => {
      console.error('Connection error', error)
    })
    return {cancel: true}
  } else {
    return {cancel: false}
  }
}
```
Instead of a drillmenu:click opening in theparent page; we have the explore page open instead.

If we want to not do the explore but just a `/query`


```
.createExploreWithUrl(`https://${looker_host}${event.url.replace('/explore/','/query/')}`)
```

Instead of using an iframe, can we make make an API call and populate a table instead?

```js
function drillClick ( event: any) {
  console.log('drillmenu:click', event)
  if (event && event.modal) {
    const dashboard_div = document.getElementById('dashboard')
    if ( dashboard_div ) {
      if (dashboard_div.children.length > 1 && dashboard_div.lastChild) {
        dashboard_div.lastChild.remove()
      }

      const url = new URL(`https://${looker_host}${event.url}`)
      const path_split = url.pathname.split('/')
      const query_params: any = {}
      url.search.slice(1).split('&').forEach(h=>{
        const hash = h.split('=')
        query_params[hash[0]] = url.searchParams.get(hash[0])
      })
      console.log(query_params)
      console.log('Click URL', url)
      sdk.run_url_encoded_query(path_split[3], path_split[4], 'csv', query_params)
      .then((data: any) => {
        var rows = data.value.split('\n'),
        table = document.createElement('table'),
        tr = null, td = null,
        tds = null;
      
        table.className = "ui celled table"

        for ( var i = 0; i < rows.length; i++ ) {
            tr = document.createElement('tr');
            tds = rows[i].split(',');
            for ( var j = 0; j < tds.length; j++ ) {
              td = document.createElement('td');
              td.innerHTML = tds[j];
              tr.appendChild(td);
            }
            table.appendChild(tr);
        }
        dashboard_div.appendChild(table)
    })
  }
    return {cancel: true}
  } else {
    return {cancel: false}
  }
}
```

How would you do unlimited rows in this table?

```
query_params['limit'] = "-1"
```

You could try to other things like move this into a pop-up modal...

### Create as dashboard dropdown

insert a new dropdown in `index.html`. Starting on line 49 insert the new dropdown divs

```html
      <div id="select-dashboard" class="ui selection violet dropdown">
        <input type="hidden" name="dashboard">
        <i class="chart line icon violet" style="display: none;"></i>
        <div class="default text">Dashboards</div>
        <div id="dashboard-dropdown-menu" class="menu">
          <div class="item" data-value="">Select a dashboard...</div>
        </div>
      </div>
```

Starting on line 77 insert a new function to handle the dropdown clicks

```js
    $('#select-dashboard')
      .dropdown()
    ;
```

Create a function that listens to the changes of the dropdowns, makes an api call to get all the dashboards avialable to the user, and will reset the iframe when it is selected.

Place this function at the bottom of `demo.ts`


```js
async function addDashboardOptions () {
  const dashboard_dropdown_parent = document.getElementById('select-dashboard')
  if (dashboard_dropdown_parent) {
    dashboard_dropdown_parent.addEventListener('change', (event: any) => { 
      const dashboard_div = document.getElementById('dashboard')
      if (dashboard_div) {
        dashboard_div.firstChild!.remove()
        embedSdkInit( (event.target as HTMLSelectElement).value )
      }
    })
  }
  const dropdown = document.getElementById('dashboard-dropdown-menu')
  let dashboards = await sdk.ok(sdk.all_dashboards())
  dashboards = dashboards.sort((a,b) => {
    if (a.title!.toLowerCase() > b.title!.toLowerCase() ) { return 1 }
    if (a.title!.toLowerCase() < b.title!.toLowerCase() ) { return -1 }
    return 0
  })
  if (dropdown && dashboards && dashboards.length > 1) {
    dashboards.forEach(function (db: any, i: number) {
      dropdown.appendChild(dashboardItem(db))
    })
  }
}

function dashboardItem (db: any) {
  const dropdown_item = document.createElement('div')
  dropdown_item.classList.add('item')
  dropdown_item.setAttribute('data-value', db.id)
  dropdown_item.innerHTML = `${db.title}`
  return dropdown_item
}
```

You'll also need to update the first line of the `embedSdkInit` function. Replace line XX with below

```js
function embedSdkInit ( dashboard_id: any ) {
```

At the top of the file, replace line XX with 

```js
document.addEventListener('DOMContentLoaded', ()=>embedSdkInit(dashboard_id))
```

### use local storage for remembering dashboard

store the dashboard selection, on line XX within the `change` function

```
localStorage.setItem('dashboard', (event.target as HTMLSelectElement).value )
```

update the `DOMContentLoaded` function on line XX to include the `getAttribute`

```js
document.addEventListener('DOMContentLoaded', ()=> {
  const dashboard = localStorage.getItem('dashboard') || dashboard_id
  embedSdkInit(dashboard)
})
```