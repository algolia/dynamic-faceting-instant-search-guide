# Dynamic Faceting Using Instant Search

###### By: Sepehr Fakour, Solutions Engineer @ Algolia

_This guide is intended to demonstrate how one could go about building a dynamic faceting user experience using instantsearch.js_

## Requirements

- [Instantsearch.js](https://github.com/algolia/instantsearch.js)

## Install

- Add instantsearch.js to your project via `npm`, `yarn`, or via `<script>` tag

## Format Data

- For each record, add an extra attribute (in this guide, we'll name it `dynamic_attributes`) which will contain an array of the names of attributes that can / should be used for faceting for that specific record... (e.g. `dynamic_attributes: ['Size', 'Brand', 'Color']`).

- For each record, add two attributes, `numeric_attributes` and `non_numeric_attributes`, each of which should contain an object with properties that are the names of all the attributes to facet on for that record. The values corresponding to each property should be the actual facet value for that facet. (e.g. `numeric_attributes: {Size: 9}` and `non_numeric_attributes: {Brand: 'Algolia', Color: 'Blue'}` ).

## Configuration Variables

- Create / include / import an object in the JS file where you are implementing instantsearch.js with property keys that are all the possible names of the attributes you wish to ultimately make facet-able. The property values should be the name of the parent attribute. For example:

```js
FACET_CONFIG = {
  "Size": "numeric_attributes",
  "Brand": "non_numeric_attributes",
  "Color": "non_numeric_attributes"
}
```

- Create a config variable and assign it an integer value representing the maximum number of facet attributes we want to use to build refinement lists with. For example:

```js
MAX_FACET_DISPLAYED = 10;
```

- Create a variable to store a reference to the DOM node we will use as the container for all our future refinement lists. For example:

```js
const refinementListContainer = document.body.querySelector('#refinement-lists');
```

## Implementation

#### (1) - Custom search function

Inside the instantsearch.js instance's param object, declare a `searchParameters` property which indicates that `dynamic_attributes` should be a disjunctiveFacet from the beginning:

```js
const search = instantsearch({

  // ...

  searchParameters: {
    disjunctiveFacets: ['dynamic_attributes']
  }

  // ...

});
```

Inside the instantsearch.js instance's param object, we can override the search function by simply declaring a property called `searchFunction`. Its value will be a callback function to be executed when a search is performed. We want to customize this to perform 2 back-to-back searches. The first will simply retrieve the facet values for the `dynamic_attributes` attribute we defined on each record. For efficiency, it will not be used to retrieve any actual records. To do this, we will employ the [searchOnce](https://community.algolia.com/algoliasearch-helper-js/reference.html#AlgoliaSearchHelper#searchOnce) method, setting the param object `{hitsPerPage:0}` to ensure we don't needlessly get any hits in the response. We can then take our newly retrieved list of facets, sort them by count, grab as many as we want (determined by the value we assign to the `MAX_FACET_DISPLAYED` config variable), and add them to the helper state. Finally, we trigger a 2nd search with our new facets, this time to actually get real results for display. When finished, our custom search function will look something like this:

```js
const search = instantsearch({

  // ...

  searchFunction: function(h) {
    const helper = this.helper;

    // After changing the query, reset active facets
    helper.setState(helper.state.setFacets(['']));

    helper.searchOnce({hitsPerPage:0}).then(function(params) {
      const content = params.content;
      if(content.disjunctiveFacets) {
        const newFacetsArr = content.getFacetValues('dynamic_attributes');

        // Sort facets array by count DESC LIMIT 10 to use for refinementLists
        let facetsForRefinement = newFacetsArr
                                    .sort((a,b) => b.count - a.count)
                                    .slice(0,MAX_FACET_DISPLAYED)
                                    .map((facetObj) => FACET_CONFIG[facetObj.name] + '.' + facetObj.name);

        // Update helper state to use newly retrieved facets
        helper.setState(helper.state.setFacets(facetsForRefinement));
      }

      h.search();
    });
  }

 // ...

 });
```

#### (2) - Custom widget

[Instantsearch custom widgets](https://community.algolia.com/instantsearch.js/documentation/#custom-widgets) provide two instance methods we are going to use to hook cleanly into the library lifecycle: the `init` method, and the `render` method. The `init` method executes once when the widget is initialized, while the `render` method executes each time new results are retrieved from an index.

##### Custom widget -> `init` method

Inside `init` we can register event listeners that may be used to handle clicks on facet values to perform a specific refinement on our search. There are a number of reasonable ways to do this, depending on the desired goal of the implementation, but in this case, we will plan to only register one single listener for all `click` events on the container housing our refinement lists. We will include an exit-early clause in the listener callback function, such that we only continue if the click occurs on a valid element (one associated with a facet value we want to filter on). The rest of the logic in our event handler will revolve around performing conditional checks to determine what type of refinement to make, then updating our query with the correct refinement. The specifics of this may differ depending on the desired structure of your markup (including `classNames`, etc). When finished our custom widget's `init` method will look something like this:

```js
init: function (options) { // This method executes once, when the dynamicFacetsWidget is initialized

  // ...

  const refineElement = function (el) {
    el.setAttribute('data-refined', "refined");
  }
  const unrefineElement = function (el) {
    el.setAttribute('data-refined', "unrefined");
  }

  const _facetClickHandler = function (e) {
    let element = e.target;
    // Only continue if the target of the click is a valid selection
    if (e.target.className !== 'refinement-list-unordered-list-item') {
      return false;
    }
    let type               = element.getAttribute('data-facet-type'),
        name               = element.getAttribute('data-facet-name'),
        value              = element.getAttribute('data-facet-value'),
        facetAttributeName = type + '.' + name;

    if (type === 'non_numeric_attributes') {
      options.helper.toggleRefinement(facetAttributeName,value).search();
      (element.getAttribute('data-refined') === "refined") ? unrefineElement(element) : refineElement(element);
    }

    if (type === 'numeric_attributes') {
      if (element.getAttribute('data-refined') === "refined") {
        // Already refined:
        // (1) Determine type of numeric refinement and removeNumericRefinement
        if (element.textContent.indexOf('-') === -1) {
          // Simple greater-than integer refinement
          options.helper.removeNumericRefinement(facetAttributeName,'>=', parseInt(value,10)).search();
        } else {
          // Mock range-based refinement (actually filters on string)
          options.helper.toggleRefinement(facetAttributeName,value).search();
        }
        // (2) setAttribute to unrefined...this can be used for CSS too
        element.setAttribute('data-refined', "unrefined");
      } else {
        // Not yet refined:
        // (1) Determine type of numeric refinement and addNumericRefinement
        if (element.textContent.indexOf('-') === -1) {
          // Simple greater-than integer refinement
          options.helper.addNumericRefinement(facetAttributeName,'>=', parseInt(value,10)).search();
        } else {
          // Mock range-based refinement (actually filters on string)
          options.helper.toggleRefinement(facetAttributeName,value).search();
        }
        // (2) setAttribute to refined...this can be used for CSS too
        element.setAttribute('data-refined', "refined");
      }
    }
  }

  refinementListContainer.addEventListener('click', _facetClickHandler);

  // ...

}
```

##### Custom widget -> `render` method

Inside `render`, we can define the logic to remove previous refinement lists and build / append new ones to the DOM. Here we will again take our facets, order them by the sum of the counts of their respective values, then build refinement lists out of them. One key consideration here is to add adequately specific markup for the click event handler (from the custom widget's `init` method) to determine what refinement to make. Finally we append them to the DOM. When finished, our custom widget's `render` method will look something like this:

```js
render: function (options) {

  // ...

  const content = options.results;
  let facetValues = content.facets.map((facet) => {
    return {
      name: facet.name,
      values: content.getFacetValues(facet.name)
    };
  });

  // Sort facet values by count
  let sortedFacetValues = facetValues.sort((a,b) => (b.values.reduce((p,c) => p + c.count, 0)) - (a.values.reduce((p,c) => p + c.count, 0)));

  // Create array of current refinements as strings
  let currentRefinements = content.getRefinements().map((refinementObj) => {
    return refinementObj.attributeName + '.' + refinementObj.name;
  });

  refinementListContainer.innerHTML = '';

  sortedFacetValues.map((facet,index,array) => {
    let facetParent = facet.name.split('.')[0],
        facetName   = facet.name.split('.')[1];

    let newRefinementList = document.createElement('div'),
        header            = document.createElement('h3'),
        unorderedList     = document.createElement('ul');

    header.textContent = facetName;

    newRefinementList.appendChild(header);
    newRefinementList.appendChild(unorderedList);
    refinementListContainer.appendChild(newRefinementList);

    facet.values.map((value) => {
      let listItem = document.createElement('li');

      listItem.className   = "refinement-list-unordered-list-item";
      listItem.textContent = value.name + ' (' + value.count + ')';

      listItem.setAttribute('data-facet-type',facetParent);
      listItem.setAttribute('data-facet-name',facetName);
      listItem.setAttribute('data-facet-value',value.name);
      listItem.setAttribute('data-facet-count',value.count);

      if (currentRefinements.indexOf(facetParent+'.'+facetName+'.'+value.name) !== -1) {
        // Set data-attribute to indicate that this value is currently refined upon
        listItem.setAttribute('data-refined','refined');
      }

      unorderedList.appendChild(listItem);
    });
  });

  // ...

}
```

#### (3) - Finish up

Next, let's make sure we've added 2 default instantsearch widgets - one for a search box and another to render hits:

```js
// ...

// Main search input
search.addWidget(
  instantsearch.widgets.searchBox({
    container: '#input-container',
    placeholder: 'Search for projects...'
  })
);

// Results list
search.addWidget(
  instantsearch.widgets.hits({
    container: '#results-container',
    templates: {
      item:
        `<div>
          <span class="title">{{{_highlightResult.title.value}}}</span>
          <span class="description">{{{_highlightResult.description.value}}}</span>
        </div>`
    }
  })
);

// ...
```

Finally, let's add our custom widget and start the search:

```js
// ...

search.addWidget(dynamicFacetsWidget);
search.start();

// ...
```

That's it! We've successfully implemented dynamic faceting with [instantsearch.js](https://github.com/algolia/instantsearch.js)!

### Notes:

This method of implementing dynamic faceting with instant search currently uses disjunctive facets to look up the `dynamic_attributes`, and uses regular conjunctive facets for the refinement lists (for more on conjunctive vs disjunctive faceting [see here](https://www.algolia.com/doc/guides/searching/faceting/#conjunctive-and-disjunctive-faceting) ). If you'd like the refinement lists to use disjunctive faceting, a few small modifications can be made to reverse how the facets are used:

#### List of changes to make to use disjunctive faceting for the refinementLists:

##### 1) Inside our `searchParameters` property during initialization, we have this:
```
searchParameters: {
  disjunctiveFacets: ['dynamic_attributes']
},
```
It should be changed to:
```
searchParameters: {
  facets: ['dynamic_attributes']
},
```
---------------------------------------------------------------------------
##### 2) At the top of our custom `searchFunction()`, we have this:
```
// After changing the query, reset active facets
  helper.setState(helper.state.setFacets(['']));
```
It should be changed to this:
```
// After changing the query, reset active facets
  helper.setState(helper.state.setDisjunctiveFacets(['']));
```
---------------------------------------------------------------------------
##### 3) Inside our custom `searchFunction()`, we have this:
```
helper.searchOnce({hitsPerPage:0}).then(function(params) {
  const content = params.content;
  if(content.disjunctiveFacets) {
```
It should be changed to:
```
helper.searchOnce({hitsPerPage:0}).then(function(params) {
  const content = params.content;
  if(content.facets) {
```
---------------------------------------------------------------------------
##### 4) Inside our custom `searchFunction()`, we have this:
```
// Update helper state to use newly retrieved facets
helper.setState(helper.state.setFacets(facetsForRefinement));
```
It should be changed to:
```
// Update helper state to use newly retrieved facets
helper.setState(helper.state.setDisjunctiveFacets(facetsForRefinement));
```
---------------------------------------------------------------------------
##### 5) At the top of our custom widget render method, we have this:
```
const content = options.results;
let facetValues = content.facets.map((facet) => {
```
It should be changed to:
```
const content = options.results;
let facetValues = content.disjunctiveFacets.map((facet) => {
```

There are of course numerous ways to implement this - InstantSearch.js is highly customizable. Happy hacking!
