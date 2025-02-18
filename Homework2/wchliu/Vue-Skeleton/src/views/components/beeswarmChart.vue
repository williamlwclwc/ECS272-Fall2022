<template>
    <a-row type="flex">
        <a-col :flex="2">
            <div id="beeswarm"></div>
        </a-col>
        <a-col :flex="1">
            <div>
                <label>Select Genre: </label>
                <a-select v-model:value="genre" style="width: 120px" @change="handleChange">
                    <a-select-option :value="item" v-for="item in genreList">
                        {{item}}
                    </a-select-option>
                </a-select>
            </div>
        </a-col>
    </a-row>
</template>

<script>
import * as d3 from "d3";

export default {
    name: 'BeeSwarmChart',
    data() {
        return {
            genre: 'european',
            genreList: [
                'action', 'animation', 'comedy', 'crime',
                'documentation', 'drama', 'european', 'family',
                'history', 'horror', 'music', 'reality', 'romance',
                'scifi', 'sport', 'thriller', 'war', 'western', 'fantasy'
            ],
        }
    },
    props:{
        myBeeSwarmChartData: Object,
    },
    mounted(){
        // this.drawBarChart(this.myBarchartData, "#bar")
        console.log("Data Passed down as a Prop  ", this.myBeeSwarmChartData);
        var that = this;
        setTimeout(() => {
            that.drawBeeswarmChart(that.myBeeSwarmChartData[this.genre], '#beeswarm', {
                x: d => d.imdb_score,
                label: "IMDB Score →",
                type: d3.scaleLinear, // try d3.scaleLog
                radius: 2,
                height: 250,
                width: 280
        })}, 500);
    },
    methods: {
        handleChange() {
            console.log(this.genre);
            d3.select('#beeswarm').select('svg').remove();
            var radius = 1;
            var height = 200;
            var width = 200;
            var len = this.myBeeSwarmChartData[this.genre].length;
            console.log(this.myBeeSwarmChartData);
            if (len > 1000) {
                radius = 0.4;
                height = 200;
                width = 200;
            } else if (len > 500) {
                radius = 1.2;
                height = 280;
                width = 250;
            } else {
                radius = 2;
                height = 250;
                width = 280;
            }
            this.drawBeeswarmChart(this.myBeeSwarmChartData[this.genre], '#beeswarm', {
                x: d => d.imdb_score,
                label: "IMDB Score →",
                type: d3.scaleLinear, // try d3.scaleLog
                radius: radius,
                height: height,
                width: width
            }); 
        },
        drawBeeswarmChart(data, id, {
                value = d => d, // convenience alias for x
                label, // convenience alias for xLabel
                type = d3.scaleLinear, // convenience alias for xType
                domain, // convenience alias for xDomain
                x = value, // given d in data, returns the quantitative x value
                title = null, // given d in data, returns the title
                group, // given d in data, returns an (ordinal) value for color
                groups, // an array of ordinal values representing the data groups
                colors = d3.schemeTableau10, // an array of color strings, for the dots
                radius = 1, // (fixed) radius of the circles
                padding = 1.5, // (fixed) padding between the circles
                marginTop = 10, // top margin, in pixels
                marginRight = 20, // right margin, in pixels
                marginBottom = 30, // bottom margin, in pixels
                marginLeft = 20, // left margin, in pixels
                width = 500, // outer width, in pixels
                height, // outer height, in pixels
                xType = type, // type of x-scale, e.g. d3.scaleLinear
                xLabel = label, // a label for the x-axis
                xDomain = domain, // [xmin, xmax]
                xRange = [marginLeft, width - marginRight] // [left, right]
            } = {}) {
            // Compute values.
            const X = d3.map(data, x).map(x => x == null ? NaN : +x);
            const T = title == null ? null : d3.map(data, title);
            const G = group == null ? null : d3.map(data, group);

            // Compute which data points are considered defined.
            const I = d3.range(X.length).filter(i => !isNaN(X[i]));

            // Compute default domains.
            if (xDomain === undefined) xDomain = d3.extent(X);
            if (G && groups === undefined) groups = d3.sort(G);

            // Construct scales and axes.
            const xScale = xType(xDomain, xRange);
            const xAxis = d3.axisBottom(xScale).tickSizeOuter(0);
            const color = group == null ? null : d3.scaleOrdinal(groups, colors);

            // Compute the y-positions.
            const Y = dodge(I.map(i => xScale(X[i])), radius * 2 + padding);

            // Compute the default height;
            if (height === undefined) height = d3.max(Y) + (radius + padding) * 2 + marginTop + marginBottom;

            // Given an array of x-values and a separation radius, returns an array of y-values.
            function dodge(X, radius) {
                const Y = new Float64Array(X.length);
                const radius2 = radius ** 2;
                const epsilon = 1e-3;
                let head = null, tail = null;

                // Returns true if circle ⟨x,y⟩ intersects with any circle in the queue.
                function intersects(x, y) {
                let a = head;
                while (a) {
                    const ai = a.index;
                    if (radius2 - epsilon > (X[ai] - x) ** 2 + (Y[ai] - y) ** 2) return true;
                    a = a.next;
                }
                return false;
                }

                // Place each circle sequentially.
                for (const bi of d3.range(X.length).sort((i, j) => X[i] - X[j])) {

                // Remove circles from the queue that can’t intersect the new circle b.
                while (head && X[head.index] < X[bi] - radius2) head = head.next;

                // Choose the minimum non-intersecting tangent.
                if (intersects(X[bi], Y[bi] = 0)) {
                    let a = head;
                    Y[bi] = Infinity;
                    do {
                    const ai = a.index;
                    let y = Y[ai] + Math.sqrt(radius2 - (X[ai] - X[bi]) ** 2);
                    if (y < Y[bi] && !intersects(X[bi], y)) Y[bi] = y;
                    a = a.next;
                    } while (a);
                }
            
                // Add b to the queue.
                const b = {index: bi, next: null};
                if (head === null) head = tail = b;
                else tail = tail.next = b;
                }
            
                return Y;
            }

            const svg = d3.select(id).append("svg")
                .attr("width", width)
                .attr("height", height)
                .attr("viewBox", [0, 0, width, height])
                .attr("style", "max-width: 100%; height: auto; height: intrinsic;");

            svg.append("g")
                .attr("transform", `translate(0,${height - marginBottom})`)
                .call(xAxis)
                .call(g => g.append("text")
                    .attr("x", width)
                    .attr("y", marginBottom - 4)
                    .attr("fill", "currentColor")
                    .attr("text-anchor", "end")
                    .text(xLabel));

            const dot = svg.append("g")
                .selectAll("circle")
                .data(I)
                .join("circle")
                .attr("cx", i => xScale(X[i]))
                .attr("cy", i => height - marginBottom - radius - padding - Y[i])
                .attr("r", radius);

            if (G) dot.attr("fill", i => color(G[i]));

            if (T) dot.append("title")
                .text(i => T[i]);
        }
    }
}
</script>

<style>
a {
    color:black
}
</style>
