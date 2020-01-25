# Section 5

pull in new skin


```
git ...
```


It will feel very familiar, we've just added prepackaged styling from [Semantic UI](https://semantic-ui.com/). You will see a bit of Looker components in the extension framework, but we didn't want to have you all switch to React.



Dynamic dashboard control starts with listening to understanding what options we have. We are going to start listening to the `dashboard:load` event to see the options that are available to us.


Create a function at the bottom of `demo.ts`

```js
function loadEvent (event: any) {
  if (event && event.dashboard && event.dashboard.options ) {
    gOptions = event.dashboard.options
    console.log('OPTIONS', gOptions)
  }
}
```

Call the function when you see the load

```js
.on('dashboard:loaded', loadEvent)
```

If you look at your console (Command+Option+j) you will see the options that are available to do dynamic dashboards. It will look something like this:


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

Please read the documentation on [dashboard:loaded](https://docs.looker.com/reference/embedding/embed-javascript-events#dashboard:loaded) to see more explanation and a full set of options.

In short: these are the properties of the dashboard we are allowed to change for dynamic dashboard control.

The first thing we will try to change is every title on all the tiles. When we're selecting a dropdown, an idea that a designer might have is to reflect the state thats filtered on the tiles.

So lets create a function that will update the charts and graph tiles

![title change](./images/section5-title-changer.gif)


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

This function will loop through all the element keys and update the titles in each element. Single tile visualizations and other chart types have different title structures for display, so we have an if statement that updates the correct title for us. Once we've updated them all, we use our EmbedSDK variable, gDashboard to set the options to tell Looker to update; alternatively you could do this manually the old way with [Javascript Events](https://docs.looker.com/reference/embedding/embed-javascript-events#dashboard:options:set).

Now lets call this function where it makes most sense, the place where we listened for the dropdown change and updated the dashboard filters. On line XX right after `dashboard.run()` you can add the below to call the function.

```js
    changeTitles(gOptions.elements,(event.target as HTMLSelectElement).value)
```

In the case above we're both updating the visualization configuration and running at the dashboard so theres a reload. But the dashboard doesn't have to reload to set options. Lets take an example where we will change all of our visualizations to tables by clicking a button.

First lets create a function that accepts an input of the element we clicked on and

1. Flips the color of the icon
2. Flips the value associated to the icon
3. Uses the value to determine if we are swapping to tables or from tables
4. If we're going to tables, it loops through each element and changes the vis_config.type to `looker_grid` then uses `.setOptions()` to set the visualization configuration
5. If we're moving from tables, it takes the default dashboard options

```

```
