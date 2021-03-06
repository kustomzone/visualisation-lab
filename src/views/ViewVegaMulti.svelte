<!-- A generic Vega view for multiple Vega/Vega-Lite Specs using Vega Embed 

TODO: - add multiple specs, at least one for each Visual Model format
TODO: - allow selection of the spec from a drop down of pre-defined specs
TODO: - ability to load additional specs from file system
TODO: - think about useful spec tweaks to support (should UI for this be here or outside?)
-->

<script>
import {default as embed}  from 'vega-embed';

import {resultDataStore} from "../stores.js";
import {modelFormats} from '../modelFormats.js';
export let activeModelsByFormat;
export let viewModelProxyStore;
$: viewModel = $viewModelProxyStore && $viewModelProxyStore.viewModel;

$: updateReactiveChart(chartSpecNoFlatten, viewModel)

function updateReactiveChart (templateSpec, viewModel) {
  console.log('updateReactiveChart()...');
  if (!viewModel) return;

  const jsonModel = viewModel.getJsonModel();
  console.log('jsonModel');console.dir(jsonModel);
  const fullSpec = Object.assign({}, templateSpec);
  // fullSpec.data.values = jsonModel.values;

  const dataset = getJsonModelAsVegaDataset(jsonModel);
  fullSpec.data.name = 'test';
  fullSpec.data.values = dataset;

  const tableFashion = viewModel.getFashion();
  const fields = tableFashion.getFieldsWithProperty('visible', true, false);
  const xAxisFields = tableFashion.getFieldsWithProperty('x-axis', true, false);
  const xAxis = xAxisFields[0] ? xAxisFields[0] : 'date';

  console.log('fields:');console.log(fields);
  console.log('xAxis: ' + xAxis);
  fullSpec.transform = [
      {"calculate": "toDate(datum." + xAxis + ")", "as": xAxis},
      {"fold": fields, "as": ["Country", "Deaths"]}
    ];

  fullSpec.mark = "line";

  fullSpec.encoding = {
    "x": {"field": xAxis, "type": "temporal"},
    "y": {"field": "Deaths", "type": "quantitative"},
    "color": {"field": "Country", "type": "nominal"}
  };

  console.log('DATASET:');console.dir(fullSpec.data.values);
  console.log('fullSpec');console.dir(fullSpec);

  embed("#vega-reactive-chart", fullSpec, { actions: false }).catch(error =>
    console.log(error)).then(vega => {
      console.log('VIEW:');
      // if(!vega){console.log('ERROR - vega embed failed');return;}
      console.dir(vega.view);
      // console.log('DATA:');
      // console.dir(vega.view._runtime.data.source_0.input.value[0]);
      console.log('TRANSFORMED DATA:')
      console.dir(vega.view._runtime.data.data_0.output.value);
    })
}

// TODO: load built in specs from JS files
const spec = `{
    "$schema": "https://vega.github.io/schema/vega-lite/v4.json",
    "description": "Simple bar chart",
    "data": {
        "values": [
            {"x": "A", "y": 28}, {"x": "y", "B": 55}, {"x": "C", "y": 43},
            {"x": "D", "y": 91}, {"x": "E", "y": 81}, {"x": "F", "y": 53},
            {"x": "G", "y": 19}, {"x": "H", "y": 87}, {"x": "I", "y": 52}
        ]
    },
    "mark": "bar",
    "encoding": {
        "x": {"field": "x", "type": "ordinal"},
        "y": {"field": "y", "type": "quantitative"}
    }
  }`;

const treeSpecTemplate = `{
  "$schema": "https://vega.github.io/schema/vega/v5.json",
  "description": "An example of Cartesian layouts for a node-link diagram of hierarchical data.",
  "width": 600,
  "height": 1600,
  "padding": 5,

  "signals": [
    {
      "name": "labels", "value": true,
      "bind": {"input": "checkbox"}
    },
    {
      "name": "layout", "value": "tidy",
      "bind": {"input": "radio", "options": ["tidy", "cluster"]}
    },
    {
      "name": "links", "value": "diagonal",
      "bind": {
        "input": "select",
        "options": ["line", "curve", "diagonal", "orthogonal"]
      }
    },
    {
      "name": "separation", "value": false,
      "bind": {"input": "checkbox"}
    }
  ],

  "data": [
    {
      "name": "tree",
      "values": {},
      "transform": [
        {
          "type": "stratify",
          "key": "index",
          "parentKey": "parent"
        },
        {
          "type": "tree",
          "method": {"signal": "layout"},
          "size": [{"signal": "height"}, {"signal": "width - 100"}],
          "separation": {"signal": "separation"},
          "as": ["y", "x", "depth", "children"]
        }
      ]
    },
    {
      "name": "links",
      "source": "tree",
      "transform": [
        { "type": "treelinks" },
        {
          "type": "linkpath",
          "orient": "horizontal",
          "shape": {"signal": "links"}
        }
      ]
    }
  ],

  "scales": [
    {
      "name": "color",
      "type": "linear",
      "range": {"scheme": "magma"},
      "domain": {"data": "tree", "field": "depth"},
      "zero": true
    }
  ],

  "marks": [
    {
      "type": "path",
      "from": {"data": "links"},
      "encode": {
        "update": {
          "path": {"field": "path"},
          "stroke": {"value": "#ccc"}
        }
      }
    },
    {
      "type": "symbol",
      "from": {"data": "tree"},
      "encode": {
        "enter": {
          "size": {"value": 100},
          "stroke": {"value": "#fff"}
        },
        "update": {
          "x": {"field": "x"},
          "y": {"field": "y"},
          "fill": {"scale": "color", "field": "depth"}
        }
      }
    },
    {
      "type": "text",
      "from": {"data": "tree"},
      "encode": {
        "enter": {
          "text": {"field": "name"},
          "fontSize": {"value": 9},
          "baseline": {"value": "middle"}
        },
        "update": {
          "x": {"field": "x"},
          "y": {"field": "y"},
          "dx": {"signal": "datum.children ? -7 : 7"},
          "align": {"signal": "datum.children ? 'right' : 'left'"},
          "opacity": {"signal": "labels ? 1 : 0"}
        }
      }
    }
  ]
}`;

const chartSpecTest = {
  "$schema": "https://vega.github.io/schema/vega-lite/v4.json",
  "description": "ViewVegaMulti Line Chart (example)",
  "data": {
    "values": {
      "date": [1,2,3],
      "China": [ 1.2, 1.8, 3.3]
    }
  },
  "transform": [
      {"flatten": ["date", "China"]}
      ],
  "mark": "line",
  "encoding": {
    "x": {"field": "date", "type": "quantitative"},
    "y": {"field": "China", "type": "quantitative"}
  }
};

// For data flattened by a Vega helper lib
//
// This works with:
// - VMTable loading ~/visualisation/datasets/covid19/total_deaths.csv
// - the temporary helper function getJsonModelAsVegaDataset()
//
// TODO: create a ViewModel to Vega library along these lines, data chosen, visualisation tweaked
// TODO: implement UI which allows the visualisation to be selected, data chosen etc.
const chartSpecNoFlatten = {
  "$schema": "https://vega.github.io/schema/vega-lite/v4.json",
  "description": "ViewVegaMulti Line Chart (example)",
  "data": {
      "values": [
        {"date": "2020-01-01","South Korea": "0","China": "13"},
        {"date": "2020-01-02","South Korea": "2","China": "26"},
        {"date": "2020-01-03","South Korea": "5","China": "67"}
    ]
  },
  "selection": {
    "grid": {
      "type": "interval", "bind": "scales"
    }
  },
  "transform": [
      {"calculate": "toDate(datum.date)", "as": "date"},
      {"fold": ["China", "South Korea", "France", "Germany", "United States", "United Kingdom"], "as": ["Country", "Deaths"]}
    ],
  "mark": "line",
  "encoding": {
    "x": {"field": "date", "type": "temporal"},
    "y": {"field": "Deaths", "type": "quantitative"},
    "color": {"field": "Country", "type": "nominal"}
  }
}

// For VMTable data (not flattened)
//
// This works with column-major data as used in VMTable (after header row has been 
// removed), but has the disadvantage that the spec needs customising to work with
// the column labels (date, country).
//
// TODO: think if can create this using parameterised 'makeVegaPlot()' lib and helpers
const chartSpecFlatten = {
  "$schema": "https://vega.github.io/schema/vega-lite/v4.json",
  "description": "ViewVegaMulti Line Chart (example)",
  "data": {
    "values": {
      "date": ["2020-01-01","2020-01-02","2020-01-03"],
      "South Korea": [0, 2, 5],
      "China": [13, 26, 67]
    }
  },
    "transform": [
      {"flatten": ["date", "China", "South Korea"]},
      {"calculate": "toDate(datum.date)", "as": "date"},
      {"fold": ["China", "South Korea"], "as": ["Country", "Deaths"]}
    ],
  "mark": "line",
  "encoding": {
    "x": {"field": "date", "type": "temporal"},
    "y": {"field": "Deaths", "type": "quantitative"},
    "color": {"field": "Country", "type": "nominal"}
  }
};

const treeSpec = JSON.parse(treeSpecTemplate);
// const chartSpec = JSON.parse(lineChartSpec);

let viewModel;
$: updateTree($activeModelsByFormat);
$: updateChart($activeModelsByFormat);

function updateTree (activeModelsByFormat) {
  console.log('updateTree()...');
  let allModels = activeModelsByFormat.get(modelFormats.VM_TREE_JSON);
  if (true||allModels === undefined) return [];

  // TODO how to handle multiple compatible models? (We visualise only the first)
  // TODO merge the model into the spec - don't just set values
  viewModel = allModels[0];      
  const fullSpec = viewModel.makeMegedJsonModel(testSpec);
  console.log('treeSpec');console.dir(treeSpec);
  console.log('fullSpec');console.dir(fullSpec);
  embed("#vega-tree", testSpec, { actions: false }).catch(error =>
    console.log(error));
}

/** Flatten VMTable model for Vega-Lite spec: chartSpecNoFlatten
*/
function getJsonModelAsVegaDataset (model, options) {
  if (options === undefined ) options = {};
  const dataset = {};
  if (options.name) dataset.name = options.name;

  // Headings from from the first row unless model.header is defined
  const values = [...model.values];
  let header = model.header;
  if (header === undefined) header = values.splice(0,1)[0];
  console.log('HEADER:');
  console.dir(header);
  
  const headings = {};
  header.forEach(heading => {headings[String(heading)] = String(heading);});

  // Flatten the data
  const newValues = [];
  values.forEach((row) => {
    const rowValues = {};
    row.forEach((value,i) => {rowValues[header[i]] = value;});
    // console.dir(rowValues);
    newValues.push(rowValues);
  });
  console.log('RESULT:');console.dir(newValues);
  return newValues;
}

function updateChart (activeModelsByFormat) {
  console.log('updateChart()...');
  let allModels = activeModelsByFormat.get(modelFormats.VM_TABULAR_JSON);
  if (allModels === undefined) return [];

  // TODO how to handle multiple compatible models? (We visualise only the first)
  // TODO merge the model into the spec - don't just set values
  viewModel = allModels[0];
  const jsonModel = viewModel.getJsonModel();
  console.log('jsonModel');console.dir(jsonModel);
  const fullSpec = Object.assign({}, chartSpecNoFlatten);
  // fullSpec.data.values = jsonModel.values;

  const dataset = getJsonModelAsVegaDataset(jsonModel);
  fullSpec.data.name = 'test';
  fullSpec.data.values = dataset;
  console.log('DATASET:');console.dir(fullSpec.data.values);
  console.log('fullSpec');console.dir(fullSpec);

  embed("#vega-chart", fullSpec, { actions: false }).catch(error =>
    console.log(error)).then(vega => {
      console.log('VIEW:');
      console.dir(vega.view);
      // console.log('DATA:');
      // console.dir(vega.view._runtime.data.source_0.input.value[0]);
      console.log('TRANSFORMED DATA:')
      console.dir(vega.view._runtime.data.data_0.output.value);
    })
}

const vegaViews = [
  {active: true, tag: 'reactive-chart', description: "Vega Reactive Chart", options: {}},
  {active: true, tag: 'chart', description: "Vega Chart", options: {}},
  {active: false, tag: 'tree', description: "Vega Tree", options: {}},
];

</script>
<style>
.main { 
  background: rgba(174, 202, 127, 0.959);
  border: 1px solid;
  border-radius: 1cm;
  padding-left: 1cm;
  padding-right: 1cm;
  padding-bottom: 1cm;
}

.vega-multi {
  border: 1px solid;
  padding: 0.2cm;
  margin: 0.1cm;
}

.hidden {
  visibility: hidden;
  width: 0;
  height: 0;
}

</style>

<div class="main">
  <p>&lt;ViewVegaMulti&gt; is a prototype for multiple Vega-Lite based view components.
  </p>
  {#if $resultDataStore && $resultDataStore.getRdfDataset()}
    <p><b>Triples:</b> {$resultDataStore && $resultDataStore.getRdfDataset() ? $resultDataStore.getRdfDataset().size : ''}</p>
  {/if}

  <b>Views to show:</b><br/><br/>
  {#each vegaViews as view}
    <label><input type=checkbox bind:checked={view.active}>{view.description}</label>
  {/each}

  {#each vegaViews as view}
    {#if view.active}
      {#if view.tag==='tree'}
        <h2>Tree</h2><p>Not yet implemented.</p>
      {/if}
      {#if view.tag==='chart'}
        <h2>Chart</h2><p>This chart works if you select the "Test CSV file" from data sources above.</p>
      {/if}
      {#if view.tag==='reactive-chart'}
        <h2>Reactive Chart</h2><p>Chart for CSV data sources above. Choose X-Axis and data to plot using the &lt;TableFashionUI&gt; above.</p>
      {/if}
    {/if}
    <div class="vega-multi {view.active ? '' : 'hidden'}" id={'vega-'+view.tag}></div>
  {/each}

</div>