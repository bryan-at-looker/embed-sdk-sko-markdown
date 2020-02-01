# Section 5: Dynamic Dashboard Control

**Note:**
If you would like to fast forward to the beginning of this section you can `git checkout section5 --force`.If you would like to start this section over at any time, you can run `git reset --hard`. If you would like to see what the files at the end of this section look like, you can check out the [Github here](https://github.com/bryan-at-looker/embed-sdk-sko/tree/section6)

Pull in new skin: This will blow away anything you haven't saved.  From your Terminal window, Control+C, then type:

```
git checkout section5 --force
```

Now restart your server with 'npm start'

Wow!  New UI!  Dashboard Next!  But it will feel very familiar; we've just added prepackaged styling from [Semantic UI](https://semantic-ui.com/). This is very similar to how you might use Looker components in the future, just using React.

Dynamic dashboard control starts with understanding what options we have to play with. We are going to start listening to the `dashboard:load` event to see the options that are available to us.

Create a function at the bottom of `demo.ts` to track the load event

```js
function loadEvent (event: any) {
  console.log('dashboard:loaded', event)
  if (event && event.dashboard && event.dashboard.options ) {
    gOptions = event.dashboard.options
    console.log('OPTIONS', gOptions)
  }
}
```

Call the function when you see the load, in Embed SDK, above `.build()`:

```js
.on('dashboard:loaded', loadEvent)
```

If you look at your JavaScript console (Command+Option+J) you will see the OPTIONS that are available to do dynamic dashboards. It will look something like this:


```js
{
  "layouts": [
    {
      "id": 5,
      "dashboard_layout_components": [...],
      ...
    }
  ],
  "elements": {
    "36": {
      "title": "Total Gross Margin",
    "vis_config": {...}
    },
    ...
  }
}
```

Head over to the Looker documentation on [dashboard:loaded](https://docs.looker.com/reference/embedding/embed-javascript-events#dashboard:loaded) for further explanation and a full set of options.

In short: these are the properties of the dashboard we are allowed to change for dynamic dashboard control.

The first thing we will try to change is every title on all the tiles.  When we're selecting a dropdown, an idea that a designer might have is to reflect the state that's filtered on the tiles.

So let's create a function that will update the charts and graph tiles

![title change](https://bryan-at-looker.s3.amazonaws.com/images/embed-sdk-sso/section5-title-changer.gif)

Add this at the bottom of `demo.ts`


```js
function changeTitles(elements: any, state: string) {
  const add_state = (state && state !== '') ? ` (${state})` : ''
  const new_elements = JSON.parse(JSON.stringify(elements))
  Object.keys(new_elements).forEach(el=>{
    console.log(new_elements[el].vis_config.title )
    if (new_elements[el].vis_config.type == "single_value") {
      new_elements[el].vis_config.title = new_elements[el].vis_config.title + add_state
      new_elements[el].title = ""
    } else {
      new_elements[el].title = new_elements[el].vis_config.title + add_state
    }
  })
  gDashboard.setOptions({elements: new_elements})
}
```

This function will loop through all the element keys and update the titles in each element. Single tile visualizations and other chart types have different title structures for display, so we have an `if` statement that updates the correct title for us. Once we've updated them all, we use our Embed SDK variable, gDashboard, to set the options to tell Looker to update; alternatively you could do this manually the old way with [Javascript Events](https://docs.looker.com/reference/embedding/embed-javascript-events#dashboard:options:set).

Now let's call this function where it makes most sense, the place where we listened for the dropdown change and updated the dashboard filters. Within the `setupDashboard` function, right after `dashboard.run()` add the following:

```js
    changeTitles(gOptions.elements,(event.target as HTMLSelectElement).value)
```
Check the dashboard - you should see updated titles containing the state filtered on.

In the case above, we're both updating the visualization configuration and running at the dashboard so there's a reload. But the dashboard doesn't have to reload to set options. Take an example where we will change all of our visualizations to tables by clicking a button.

First create a function that accepts an input of the element we clicked on. Place this at the bottom of `demo.ts`:

```js
function tableChange(table_icon: HTMLElement) {
  let new_elements = JSON.parse(JSON.stringify(gOptions.elements))
  const to_table = ( table_icon.getAttribute('data-value') == '0' ) ? true : false
  table_icon.classList.remove((to_table) ? 'black' : 'violet')
  table_icon.classList.add((to_table) ? 'violet' : 'black')
  table_icon.setAttribute('data-value', (to_table) ? '1': '0')
  if (to_table) {
    Object.keys(new_elements).forEach(element => {
      new_elements[element].vis_config.type = 'looker_grid'
    })
    gDashboard.setOptions({ elements: new_elements})
  } else {
    gDashboard.setOptions({ elements: gOptions.elements })
  }
}
```
This function does the following:

1. Flips the color of the icon
2. Flips the value associated to the icon
3. Uses the value to determine if we are swapping to tables or from tables
4. If we're going to tables, it loops through each element and changes the vis_config.type to `looker_grid` then uses `.setOptions()` to set the visualization configuration
5. If we're moving from tables, it takes the original element configurations and applies them.

Now we need to call this function and pass it an HTML element for the button we're clicking. We can add this within the `setupDashboard` function underneath where we make our API calls for the first dropdown. You can put this after `loadingIcon(false)`:


```js
  const table_icon = document.getElementById('table-swap')
  if (table_icon) {
    table_icon.addEventListener('click', () => {
      tableChange(table_icon)
    })
  }
```

Click the table swap button to see the the dynamic dashboard control in action

You'll notice that when you mouse over the table icon in upper right, the cursor doesn't change to indicate an action is availble for the table icon.  Simple fix to change the cursor from an arrow to a pointer - edit `index.html` and replace the line below:

```js
        <i id="table-swap" data-value="0" style="cursor: pointer;" class="big table icon violet"></i>
```

![Table Swap](https://bryan-at-looker.s3.amazonaws.com/images/embed-sdk-sso/section5-table-swap-icon.png)


**Bonus:** Don't change the single value visualizations; wrap a condition to check for `looker_grid`. At the bottom of your `demo.ts` file replace the entire `function tableChange` with this:

```
function tableChange(table_icon: HTMLElement) {
  let new_elements = JSON.parse(JSON.stringify(gOptions.elements))
  const to_table = ( table_icon.getAttribute('data-value') == '0' ) ? true : false
  table_icon.classList.remove((to_table) ? 'black' : 'violet')
  table_icon.classList.add((to_table) ? 'violet' : 'black')
  table_icon.setAttribute('data-value', (to_table) ? '1': '0')
  if (to_table) {
    Object.keys(new_elements).forEach(element => {
      if (new_elements[element].vis_config.type !== 'single_value' ) {
        new_elements[element].vis_config.type = 'looker_grid'
      }
    })
    gDashboard.setOptions({ elements: new_elements})
  } else {
    gDashboard.setOptions({ elements: gOptions.elements })
  }
}
```

### Vis Swap

![Vis Swap](https://bryan-at-looker.s3.amazonaws.com/images/embed-sdk-sso/section5-donut-swap.gif)

Lets find a very specific tile and have a control that only updates that one tile. There may be situations where your embedding customer would like to be able to change configuration options on the fly. For example, changing the visualization type, hiding certain fields, or re-calculating a goal line. In this example, we will create a button that lets you toggle a donut on and off, but any visualization configuration is available to change.

We need to figure out which element we want to swap the visualization configuration on. We've been logging the options for dynamic dashboard control, so lets poke around in our console (Command+Option+J) in there to find the Active Users tile that is of visualization type `looker_donut_multiples`.

```js
{
  ...,
  "42": {
  "title": "Active Users",
  "title_hidden": false,
  "vis_config": {
    "type": "looker_donut_multiples",
    ...,
    "title": "Active Users"
  },
  ...,
}
```

The dashboard_element_id we want to swap is 42, lets create a couple variables in in demo_config.ts

```js
export const swap_element = "42"
export const new_vis_config = {
  "type": "looker_bar"
}
```

Import in `demo.ts` at the top with the other imports:


```js
import { swap_element, new_vis_config } from './demo_config'
```

Place at the end of `demo.ts` to perform interactions on the


```js
function swapVisConfig( icon: HTMLElement ) {
  if ( swap_element && gOptions && gOptions.elements && gOptions.elements[swap_element] ) {
    const elements = JSON.parse(JSON.stringify( gOptions.elements ))
    const to_original = (icon.getAttribute('data-value') === '1') ? true : false
    icon.classList.remove((to_original) ? 'black' : 'violet')
    icon.classList.add((to_original) ? 'violet' : 'black')
    icon.setAttribute('data-value', (to_original) ? '0': '1')

    if (to_original) {
      let new_element = { [swap_element]: elements[swap_element] }
      new_element[swap_element]['vis_config'] = Object.assign( new_element[swap_element]['vis_config'],  new_vis_config  )
      gDashboard.setOptions({ elements: new_element})
    } else {
      gDashboard.setOptions({ elements: gOptions.elements })
    }
  }
}
```
The function above does the following.
1. Flips the color of the icon after click
2. Flips the value associated to the icon (we are using this to track state)
3. Uses the value to determine if we are swapping from or to the original vis config
4. After applying the visualization options, we use `.setOptions()` through the `element` object.


In `demo.ts` at the end of setupDashboard function, place this

```js
  const donut_icon = document.getElementById('vis-swap')
  if (donut_icon) {
    donut_icon.addEventListener('click', () => {
      swapVisConfig(donut_icon)
    })
  }
```

This listens for when the donut icon is clicked, then runs the `swapVisConfig()` function.
![Donut Swap](https://bryan-at-looker.s3.amazonaws.com/images/embed-sdk-sso/section5-donut-swap-icon.png)

### Layout

In `demo_config.ts`, switch to a dashboard with a new set of filters

```
export const dashboard_id = 6
```

*Note:* We're switching dashboards; if you want to keep your vis swap you have to change `swap_element` variable to `51`.

In `demo_config.ts` create an object that listens to the KPIs filter object, this is the Filter Name you see on the dashboard. It currently doensn't control any queries at all, its just a dummy filter.

```js
export const dashbord_layout_filter = 'KPIs'
```

in `demo.ts` import the variable at the top near the other imports

```js
import { dashbord_layout_filter } from './demo_config'
```

create a function to hide tiles and place it at the bottom of `demo.ts`

```js
function layoutFilter(filter: any) {
  const copy_options = JSON.parse(JSON.stringify(gOptions))
  const elements = copy_options.elements || {}
  const layout = copy_options.layouts[0]
  let components = (layout.dashboard_layout_components) ? layout.dashboard_layout_components : []

  const new_components: any = []
  filter = filter.split(',')

  components.forEach((c: any )=>{
    const found = elements[c.dashboard_element_id]
    if (filter.indexOf(found.title) > -1 ) {
      new_components.push(c)
    }
  })
  layout.dashboard_layout_components = new_components
  gDashboard.setOptions({ layouts: [layout] })
}
```

The above function
1. Creates copies of the elements and layout/components so it can update it seperately from the default
2. Loops through each component on the dashboard
3. If the title of the tile matches whats been selected in the KPI filters, it keeps the layout the same, if the tile's title does not match a filter selection, then it removes the layout component.

In `demo.ts` at the end of filtersUpdate function, after `loadingIcon(false)`, we want to add this call.

```js
  if (dashboard_filters && dashboard_filters[dashbord_layout_filter] && dashboard_filters[dashbord_layout_filter]) {
    layoutFilter(dashboard_filters[dashbord_layout_filter])
  }
```



### Have one KPI on load

In `demo.ts`, at the end of loadEvent function place these lines of code, it will send the KPIs filter to the function so it can hide them.

```js
  if (event && event.dashboard && event.dashboard.dashboard_filters) {
    const dashboard_filters = event.dashboard.dashboard_filters
    if (dashboard_filters && dashboard_filters[dashbord_layout_filter] && dashboard_filters[dashbord_layout_filter]) {
      layoutFilter(dashboard_filters[dashbord_layout_filter])
    }
  }
```

Then place a default filter for the KPI field where it instantiate the Embed SDK in the `embedSdkInit()` function. Replace the current `// .withFilters()` with this:

```js
  .withFilters({
    [dashbord_layout_filter]: 'Active Users',
    [dashboard_date_filter]: '30 days'
  })
```

Your dashboard should collapse to just the Active Users tiles on load!
