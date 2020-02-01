# Section 6: Advanced Interactions

**Note:**
If you would like to fast forward to the beginning of this section you can `git checkout section6 --force`. If you would like to start this section over at any time, you can run `git reset --hard`. If you would like to see what the files at the end of this section look like, you can check out the [Github here](https://github.com/bryan-at-looker/embed-sdk-sko/tree/section6end)

### Saving dashboard filters using localstorage

A user changes a filter, can we save it and bring up the last values?

Set at the end of filtersUpdates in `demo.ts` file:

```js
  if (dashboard_filters) {
    localStorage.setItem('dashboard_filters', JSON.stringify(dashboard_filters));
  }
```

In `demo.ts` place the below code in the EmbedSDK instantiation function after `const logo = document.getElementById('logo')`:

```
const last_filters = JSON.parse(localStorage.getItem('dashboard_filters') || '{}');
```

Then call last_filters using `withFilters()`; replace the current `withFilters()` with this:

```
.withFilters(last_filters)
```

When someone clicks a dropdown we need to set it as well. We're using JavaSript listeners to update these values. In your `demo.ts` file within the setupDashboard add the lines below after `dashboard.run()`:

```js
      const last_filters = JSON.parse(localStorage.getItem('dashboard_filters') || '{}');
      localStorage.setItem('dashboard_filters', JSON.stringify(Object.assign(last_filters, { [dashboard_state_filter]: (event.target as HTMLSelectElement).value })))
```

Refresh the page or open the current URL in a new tab to see the localStorage in action. Your last filters will be populated on load.


## Click Events
Measure, dimension, and link clicks are tracked on click. With Javacript events we can capture the event, with the Embed SDK, we can capture and CANCEL the click.
### Capture Drill Menu Clicks

Create a function at the end of demo.ts

**NOTE:**

Currently drill menu clicks don't fire with Dashboards Next (but will soon), so comment out the `.withNext()` line in the embedSdkInit function. Place the below function at the bottom of demo.ts.

```js
  function drillClick ( event: any) {
  console.log('drillmenu:click', event)
  return {cancel: false}
}
```

listen to drillmenu:click by placing this within the EmbedSDK:

```
.on('drillmenu:click', drillClick)
```

When we click around on the dashboard we start to see the event come through in our console. Here's an example:


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

The drillclick function has a return of `cancel:false`, this means the clicks will always go through. You can change this to cancel all clicks on the dashboard. In the drillclick function change the return to this:

```js
return {cancel: true}
```
After clicking around on the dashboard, when we click on charts the modal never pops up. Notice this doesn't stop the dropdown menu from appearing, but will prevent the automatic drills + clicks from within the menu.

### Advanced click interactions

In this example we will open an Explore page underneath the dashboard. Replace the `drillClick()` function with the following, and click on Total Sale Price (This Period) in the line chart:

```js
  function drillClick ( event: any) {
  console.log('drillmenu:click', event)
  if (event && event.modal) {
    const dashboard_div = document.getElementById('dashboard')
    if ( dashboard_div && dashboard_div.children.length > 1 && dashboard_div.lastChild) {
      dashboard_div.lastChild.remove()
    }
    LookerEmbedSDK
    .createExploreWithUrl(`https://${looker_host}${event.url}`)
    .appendTo('#dashboard')
    .withClassName('looker-embed')
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
Instead of the modal opening in the iframe, we have the explore page open at the bottom of the page instead. We did this by creating a new iframe with the data from the click.

If we want to have a table but not the full Explore page, we can change `/explore` to  `/query`. Find the `.createExploreWithUrl` and replace it with the following:


```
.createExploreWithUrl(`https://${looker_host}${event.url.replace('/explore/','/query/')}`)
```

Instead of using an iframe, can we make make an API call and populate a table instead. Replace the drillClick function with all of this:

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
      sdk.get(`/queries/models/${path_split[3]}/views/${path_split[4]}/run/csv${url.search}`)
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

How would you do unlimited rows in this table? Replace `sdk.get` with the following:

```
sdk.get(`/queries/models/${path_split[3]}/views/${path_split[4]}/run/csv${url.search}&limit=-1`)
```

This is just an example of drillClick putting data into other parts of the application. There are many other use cases you might want to do this with.

### Create as dashboard dropdown

insert a new dropdown in `index.html`. After `<div id="dropdown-selected"></div>` insert the new dropdown divs:

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

Within the script tag at the bottom of the page, insert this after the new function to handle the dropdown clicks:

```js
    $('#select-dashboard')
      .dropdown()
    ;
```

Create a function that responds to the changes of the dropdowns, makes an api call to get all the dashboards avialable to the user, and will reset the iframe when it is selected.

Place this function at the bottom of `demo.ts` (there will be an error, we will fix it in the next few commands):


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

You'll also need to update the first line of the `embedSdkInit` function. Replace `function embedSdkInit` with the below code:

```js
function embedSdkInit ( dashboard_id: any ) {
```

At the top of the file, replace `document.addEventListener` with:

```js
document.addEventListener('DOMContentLoaded', ()=>embedSdkInit(dashboard_id))
```

Now include this function within the `setupDashboard` const (at the end):

```js
addDashboardOptions()
```

### Use local storage for remembering dashboard

When the user selects the dropdown, we want to store the value in their local storage. Add the following within the `addDashboardsOptions` function, after `dashboard_dropdown_parent.addEventListener`:

```
localStorage.setItem('dashboard', (event.target as HTMLSelectElement).value )
```

Replace the `DOMContentLoaded` listener with the following:

```js
document.addEventListener('DOMContentLoaded', ()=> {
  const dashboard = localStorage.getItem('dashboard') || dashboard_id
  embedSdkInit(dashboard)
})
```
