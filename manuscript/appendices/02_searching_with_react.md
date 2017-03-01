# Searching with React

Let's say we want to implement a rough little search for our application without a proper backend. We might want to use something like [lunr](http://lunrjs.com/) for generating an index to search.

The problem is that the index can be sizable depending on the amount of the content. The dumb way to implement this kind of search would be to include the index required to the application bundle itself and then perform a search against that.

The good thing is that we don't need the search index straight from the start. We can do something cleverer. We can start loading the index when the user selects our search field.

Doing this defers the loading and moves it to a place where it's more acceptable for performance. Given the initial search might be slower than the subsequent ones we could display a loading indicator. But that's fine from the user point of view.

Webpack's **code splitting** feature allows us to do this. See the *Code Splitting* chapter for more detailed discussion and the exact setup required.

## Implementing Search with Lazy Loading

To implement lazy loading, you will need to decide where to put the split point, put it there, and then handle the `Promise`. The basic `import` looks like `import('./asset').then(asset => ...).catch(err => ...)`.

The nice thing is that this gives us error handling in case something goes wrong (network is down etc.) and gives us a chance to recover. We can also use `Promise` based utilities like `Promise.all` for composing more complicated queries.

In this case, we need to detect when the user selects the search element, load the data unless it has been loaded already, and then execute our search logic against it. Using React, we could end up with something like this:

**App.jsx**

```javascript
import React from 'react';

export default class App extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      index: null,
      value: '',
      lines: [],
      results: [],
    };

    this.onChange = this.onChange.bind(this);
  }
  render() {
    const results = this.state.results;

    return (
      <div className="app-container">
        <div className="search-container">
          <label>Search against README:</label>
          <input
            type="text"
            value={this.state.value}
            onChange={this.onChange} />
        </div>
        <div className="results-container">
          <Results results={results} />
        </div>
      </div>
    );
  }
  onChange(e) {
    const value = e.target.value;
    const index = this.state.index;
    const lines = this.state.lines;

    // Set captured value to input
    this.setState({
      value
    });

    // Search against lines and index if they exist
    if(lines && index) {
      this.setState({
        results: this.search(lines, index, value),
      });

      return;
    }

    // If the index doesn't exist, we need to set it up.
    // Unfortunately we cannot pass the path so we need to
    // hardcode it (webpack uses static analysis).
    //
    // You could show loading indicator here as loading might
    // take a while depending on the size of the index.
    loadIndex().then(lunr => {
      // Search against the index now.
      this.setState({
        index: lunr.index,
        lines: lunr.lines,
        results: this.search(lunr.lines, lunr.index, value),
      });
    }).catch(err => {
      // Something unexpected happened (connection lost
      // for example).
      console.error(err);
    });
  }
  search(lines, index, query) {
    // Search against index and match README lines
    // against the results.
    return index.search(
      query.trim()
    ).map(
      match => lines[match.ref]
    );
  }
};

const Results = ({results}) => {
  if(results.length) {
    return (<ul>{
      results.map((result, i) => <li key={i}>{result}</li>)
    }</ul>);
  }

  return <span>No results</span>;
};


function loadIndex() {
  // Here's the magic. Set up `import` to tell webpack
  // to split here and load our search index dynamically.
  //
  // Note that you will need to shim Promise.all for
  // older browsers and Internet Explorer!
  return Promise.all([
    import('lunr'),
    import('../search_index.json'),
  ]).then(([lunr, search]) => {
    return {
      index: lunr.Index.load(search.index),
      lines: search.lines,
    };
  });
}
```

In the example, webpack detects the `import` statically. It can generate a separate bundle based on this split point. Given it relies on static analysis, you cannot generalize `loadIndex` in this case and pass the search index path as a parameter.

## Conclusion

Beyond search, the approach is useful with routers too. As the user enters some route, you can load the dependencies the resulting view needs. Alternately, you can start loading dependencies as the user scrolls a page and gets adjacent parts with actual functionality. `import` provides a lot of power and allows you to keep your application lean.

You can find a [full example](https://github.com/survivejs-demos/lunr-demo) showing how it all goes together with lunr, React, and webpack. The basic idea is the same, but there's more setup in place.
