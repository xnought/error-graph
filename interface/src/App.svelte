<script>
	import * as d3 from "d3";
	import { onMount } from "svelte";

	let svgEl, svg, sim;
	export let width = 1600;
	export let height = 1000;

	export let r = 5;
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
	export let filename = "imagenette.json";

	async function fileToNodes(filename, limit = Infinity) {
		const data = await fetch(`${filename}`);
		const json = await data.json();
		json.splice(limit);
		const xScale = (x) =>
			d3
				.scaleLinear()
				.domain(d3.extent(nodes, (d) => d.x))
				.range([0, width]);
		const yScale = (y) =>
			d3
				.scaleLinear()
				.domain(d3.extent(nodes, (d) => d.y))
				.range([0, height]);

		// put it in format so I have ID and group
		// TODO add groups
		for (let i = 0; i < json.length; i++) {
			const item = json[i];
			json[i] = {
				...json[i],
				radius: r,
			};
		}
		return json;
	}

	/**
	 * most of this is from [ObservableHQ](https://observablehq.com/@d3/disjoint-force-directed-graph/2?intent=fork)
	 */
	function graph(svg, nodes, links) {
		const xScale = (x) => x;
		const yScale = (y) => y;
		sim = d3
			.forceSimulation(nodes)
			.force(
				"link",
				d3.forceLink(links).id((d) => d.id)
			)
			.force("charge", d3.forceManyBody())
			.force("x", d3.forceX())
			.force("y", d3.forceY())
			// .force("center", d3.forceCenter(width / 2, height / 2))
			.on("tick", update);

		// Add a line for each link, and a circle for each node.
		const link = svg
			.append("g")
			.attr("stroke", "#999")
			.attr("stroke-opacity", 0.1)
			.selectAll("line")
			.data(links)
			.join("line")
			.attr("stroke-width", (d) => Math.sqrt(1 / d.value));

		const node = svg
			.append("g")
			.attr("stroke", "#fff")
			.attr("stroke-width", 1.5)
			.selectAll("circle")
			.data(nodes)
			.join("circle")
			.attr("r", (d) => d.radius)
			.attr("fill", (d) => (d.correct ? "steelblue" : "red"));

		function update() {
			const x = "x";
			const y = "y";
			// const x = "cx";
			// const y = "cy";
			link.attr("x1", function (d) {
				return xScale(d.source[x]);
			})
				.attr("y1", function (d) {
					return yScale(d.source[y]);
				})
				.attr("x2", function (d) {
					return xScale(d.target[x]);
				})
				.attr("y2", function (d) {
					return yScale(d.target[y]);
				});

			node.attr("cx", function (d) {
				return xScale(d[x]);
			}).attr("cy", function (d) {
				return yScale(d[y]);
			});
		}

		node.call(
			d3
				.drag()
				.on("start", dragstarted)
				.on("drag", dragged)
				.on("end", dragended)
		);

		// Reheat the simulation when drag starts, and fix the subject position.
		function dragstarted(event) {
			if (!event.active) sim.alphaTarget(0.3).restart();
			event.subject.fx = event.subject.x;
			event.subject.fy = event.subject.y;
		}

		// Update the subject (dragged node) position during drag.
		function dragged(event) {
			event.subject.fx = event.x;
			event.subject.fy = event.y;
		}

		// Restore the target alpha so the simulation cools after dragging ends.
		// Unfix the subject position now that it’s no longer being dragged.
		function dragended(event) {
			if (!event.active) sim.alphaTarget(0);
			event.subject.fx = null;
			event.subject.fy = null;
		}

		update();
	}

	/**
	 * Connect incorrect to all the nearest neighbors (– -> + or –)
	 * @param {{id: string, x: number, y: number, correct: boolean}[]} data
	 */
	function wrong(data, k = 5) {
		let links = [];
		for (let i = 0; i < data.length; i++) {
			const source = data[i];
			const incorrect = source.correct === false;
			if (incorrect) {
				// determine the top closest neighbors
				for (let j = 0; j < data.length; j++) {
					const target = data[j];
					if (target === source) continue;

					// 2-norm and subtraction = distance
					const dist = Math.sqrt(
						Math.pow(source.x - target.x, 2) +
							Math.pow(source.y - target.y, 2)
					);
					// add to the memory distance
					target["distance"] = dist;
				}
				// wildly inefficient
				const topK = data
					.sort((a, b) => a["distance"] - b["distance"])
					.slice(0, k);

				source["radius"] = r + 1;
				for (const t of topK) {
					if (t.correct) continue;
					source["radius"] += 1;
				}
				source["radius"] = Math.max(
					-20 * Math.log((1 / 20) * source["radius"]),
					r
				);

				// go from source to all targets found in topK
				for (const target of topK) {
					links.push({
						source: source.id,
						target: target.id,
						value: target["distance"],
					});
				}
			}
		}
		return { nodes: data, links: links };
	}

	/**
	 * In the exact order of the array, connect em!
	 * @param {{id: string, x: number, y: number}[]} data
	 */
	function inOrder(data) {
		let links = [];
		for (let i = 1; i < data.length - 1; i++) {
			const node = data[i];
			const source = data[i - 1];
			const target = node;
			links.push({ source: source.id, target: target.id, value: 1 });
		}
		return { nodes: data, links: links };
	}

	function none(data) {
		return { nodes: data, links: [] };
	}

	onMount(async () => {
		const data = await fileToNodes(filename, 3000);
		const { nodes, links } = wrong(data, 20);
		svg = d3.select(svgEl);
		graph(svg, nodes, links);
	});
</script>

<div>
	<svg
		bind:this={svgEl}
		{width}
		{height}
		viewBox="{(-width * 2.5) / 2} {(-height * 2.5) / 2} {width *
			2.5} {height * 2.5}"
	/>
</div>

<style>
	svg {
		/* outline: red 1px solid; */
		overflow: visible;
		margin: 25px;
	}
</style>
