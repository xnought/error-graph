<script>
	import * as d3 from "d3";
	import { onMount } from "svelte";

	let svgEl, svg, sim;
	export let width = 500;
	export let height = 500;

	export let nodes = [
		{ id: "a", group: 1 },
		{ id: "b", group: 1 },
		{ id: "c", group: 1 },
		{ id: "d", group: 1 },
		{ id: "e", group: 1 },
	];
	export let links = [
		{ source: "a", target: "b", value: 4 },
		{ source: "b", target: "c", value: 1 },
		{ source: "e", target: "d", value: 1 },
		{ source: "c", target: "a", value: 4 },
	];

	function graph(svg, nodes, links) {
		sim = d3
			.forceSimulation(nodes)
			.force(
				"link",
				d3.forceLink(links).id((d) => d.id)
			)
			.force("charge", d3.forceManyBody())
			.force("center", d3.forceCenter(width / 2, height / 2))
			.on("tick", update);

		// Add a line for each link, and a circle for each node.
		const link = svg
			.append("g")
			.attr("stroke", "#999")
			.attr("stroke-opacity", 0.6)
			.selectAll("line")
			.data(links)
			.join("line")
			.attr("stroke-width", (d) => Math.sqrt(d.value));

		const node = svg
			.append("g")
			.attr("stroke", "#fff")
			.attr("stroke-width", 1.5)
			.selectAll("circle")
			.data(nodes)
			.join("circle")
			.attr("r", 5)
			.attr("fill", "red");

		function update() {
			link.attr("x1", function (d) {
				return d.source.x;
			})
				.attr("y1", function (d) {
					return d.source.y;
				})
				.attr("x2", function (d) {
					return d.target.x;
				})
				.attr("y2", function (d) {
					return d.target.y;
				});

			node.attr("cx", function (d) {
				return d.x;
			}).attr("cy", function (d) {
				return d.y;
			});
		}

		update();
	}

	onMount(() => {
		svg = d3.select(svgEl);
		graph(svg, nodes, links);
	});
</script>

<div>
	<svg bind:this={svgEl} {width} {height} />
</div>

<style>
	svg {
		outline: red 1px solid;
		overflow: visible;
	}
</style>
