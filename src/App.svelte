<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import textures from "textures";
  import data from "./data/data.json";
  import summaryData from "./data/summary.json";

  // all variables related to dimensions
  const marginTop = 50;

  //scale to map original width to pixel width
  const lengthScale = d3
    .scaleLinear()
    .domain([0, d3.max(summaryData, (d) => Math.sqrt(d.total))])
    .range([0, 620]);

  //add extra width variable to summary stats
  let summary = summaryData
    .map((d) => {
      return {
        ...d,
        length: lengthScale(Math.sqrt(d.total)),
      };
    })
    .sort((a, b) => b.total - a.total);

  // sort data
  data.sort((a, b) => a.total - b.total);

  // all related to scales and styles
  const orders = [...new Set(summary.map((d) => d.order))];
  const strokeColorScale = d3
    .scaleOrdinal()
    .domain(orders)
    .range([
      "#9f80d1",
      "#f7a663",
      "#7fc97f",
      "#80b1d3",
      "#f4a6d1",
      "#72baac",
      "#fc9272",
    ]);
  const fillColorScale = d3
    .scaleOrdinal()
    .domain(orders)
    .range([
      "#e4d4fc",
      "#f9dec7",
      "#cbf4cb",
      "#a9d3ea",
      "#fce3f1",
      "#aae5d9",
      "#f7b1a1",
    ]);

  //convert data to hierarchical format
  const convertDataHierarchy = (data) => {
    const groupingFn = [(d) => d.category];
    const rollupData = d3.rollup(
      data,
      (v) => d3.sum(v, (d) => d.total),
      ...groupingFn
    );
    const childrenAccessorFn = ([key, value]) =>
      value.size && Array.from(value);
    return d3
      .hierarchy(new Map(rollupData), childrenAccessorFn)
      .sum(([key, value]) => value);
  };

  // Layout + data prep
  const setupTreemap = (length) => {
    return d3
      .treemap()
      .paddingInner(3.5)
      .paddingOuter(2)
      .paddingTop(1)
      .round(true)
      .size([length, length])
      .tile(d3.treemapBinary);
  };

  const getLeaves = (shape) => {
    const dataFiltered = data.filter((d) => d.order === shape.order);
    const hierarchyData = convertDataHierarchy(dataFiltered);
    const treemap = setupTreemap(shape.length, shape.length);
    const root = treemap(hierarchyData);
    return root.leaves();
  };

  //texture related

  const textureArray = orders.map((d) => {
    return textures.lines().size(6).strokeWidth(2).stroke(fillColorScale(d));
  });

  let svgElements = orders.map((d, i) => i);

  onMount(() => {
    svgElements.map((d, i) => {
      d3.select(svgElements[i]).call(textureArray[i]);
    });
  });
</script>

<div id="chart">
  {#each summary as d, i}
    <svg
      id={`order${i}`}
      width={d.length}
      height={d.length + marginTop}
      bind:this={svgElements[i]}
    >
      <text x="0" y={marginTop - 5} font-weight="700">{d.order}</text>
      <g transform="translate(0,{marginTop})">
        {#each getLeaves(d) as d2}
          <rect
            transform={`translate(${d2.x0},${d2.y0})`}
            width={d2.x1 - d2.x0}
            height={d2.y1 - d2.y0}
            rx="2"
            ry="2"
            stroke-width="2"
            stroke={strokeColorScale(d.order)}
            fill={d2.data[0] === "EX" ||
            d2.data[0] === "EW" ||
            d2.data[0] === "CR" ||
            d2.data[0] === "EN" ||
            d2.data[0] === "VU"
              ? textureArray[i].url()
              : fillColorScale(d.order)}
          />
          <text x={d2.x0 + 3} y={d2.y0 + 15}>{d2.data[0]}</text>
        {/each}
      </g>
    </svg>
  {/each}
</div>

<style>
  #chart {
    font-family: "Source Sans Pro", sans-serif;
    text-align: center;
  }

  text {
    font-size: 12px;
    font-weight: 700;
    paint-order: stroke;
    stroke: white;
    stroke-width: 2px;
    stroke-linecap: butt;
    stroke-linejoin: miter;
  }
</style>
