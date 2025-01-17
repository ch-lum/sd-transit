<script>
    import { onMount } from "svelte";
    import * as d3 from "d3";

    let minutes = [];
    let maxAreaByCategory;
    let tooltipText = "";
    let selectedPath = null;

    const colors = ['#e6194b', '#3cb44b', '#ffe119', '#4363d8', '#f58231', '#911eb4', '#46f0f0', '#f032e6', '#bcf60c', '#fabebe', '#008080', '#e6beff', '#9a6324', '#a9a9a9', '#800000', '#aaffc3', '#808000', '#ffd8b1', '#000075', '#808080']
    const colorScale = {
        "Bankers Hill": colors[0],
        "Barrio Logan": colors[1],
        "College": colors[2],
        "Columbia": colors[3],
        "Core": colors[4],
        "Cortez": colors[5],
        "Cortez Hill": colors[6],
        "East Village": colors[7],
        "Fize Points": colors[8],
        "Gaslamp": colors[9],
        "Golden Hill": colors[10],
        "Hillcrest": colors[11],
        "Little Italy": colors[12],
        "Marina": colors[13],
        "Midtown": colors[14],
        "Mission Hills": colors[15],
        "North Park": colors[16],
        "Point Loma": colors[17],
        "Talmadge": colors[18],
        "University Heights": colors[19]
    }

    onMount(async () => {
        const height = 3000;
        const width = 1200;
        const margin = 120;

        const res = await fetch('by_minute.csv'); 
        const csv = await res.text();
        minutes = d3.csvParse(csv, d3.autoType)

        maxAreaByCategory = Array.from(d3.group(minutes, d => d.area), ([key, values]) => ({key, values: d3.max(values, d => d.count)}))
        maxAreaByCategory.sort((a, b) => d3.ascending(a.values, b.values));
        // console.log(maxAreaByCategory);

        const maxAreaMap = new window.Map(maxAreaByCategory.map(d => [d.key, d.values]));
        minutes.sort((a, b) => {
            // Primary sort by the "area" column
            const areaDiff = d3.descending(maxAreaMap.get(a.area), maxAreaMap.get(b.area));
            if (areaDiff !== 0) return areaDiff;

            // Secondary sort by the "minute" column
            return d3.ascending(a.minute, b.minute);
        });

        // console.log(minutes);

        const svg = d3.select("#timeline")
            .append("svg")
            .attr("width", width)
            .attr("height", height)
            .style("background-color", "rgba(255, 255, 255, 0)");
        

        const xScaleDiscrete = d3.scalePoint()
            .domain(maxAreaByCategory.map(d => d.key))
            .range([0 + margin, width * 0.35])
            .padding(1);

        const xScaleLinear = d3.scaleLinear()
            .domain(d3.extent(minutes.map(d => d.count)))
            .range([0 + margin, 0 + width * 0.65 - margin]);

        const yScale = d3.scaleTime()
            .domain(d3.extent(minutes.map(d => d3.timeParse("%Y-%m-%d %H:%M:%S")(d.time))))
            .range([0 + margin, height - margin]);

        // const colorScale = d3.scaleOrdinal(d3.schemeCategory10);
        
        const timeRange = d3.extent(minutes.map(d => d3.timeParse("%Y-%m-%d %H:%M:%S")(d.time)));
        const numHours = (timeRange[1] - timeRange[0]) / 1000 / 60 / 60;

        const xAxisTop = d3.axisTop(xScaleDiscrete);
        const xAxisBottom = d3.axisBottom(xScaleDiscrete);
        const yAxis = d3.axisLeft(yScale)
            .ticks(numHours * 2, "%H:%M")
            .tickFormat(d3.timeFormat("%H:%M"));

        svg.append("g")
            .attr("transform", `translate(0, ${margin})`)
            .call(xAxisTop)
            .selectAll("text")
            .style("text-anchor", "end")
            .attr("dx", "-.8em")
            .attr("dy", ".15em")
            .attr("transform", "rotate(65)")
            .style("fill", function(d) { return colorScale[d]; })
            .style("font-weight", "bold")
            .style("text-transform", "uppercase");

        svg.append("g")
            .attr("transform", `translate(0, ${height - margin})`)
            .call(xAxisBottom)
            .selectAll("text")
            .style("text-anchor", "end")
            .attr("dx", "-.8em")
            .attr("dy", ".15em")
            .attr("transform", "rotate(-65)")
            .style("fill", function(d) { return colorScale[d]; })
            .style("font-weight", "bold")
            .style("text-transform", "uppercase");

        
        svg.append("g")
            .attr("transform", `translate(${margin}, 0)`)
            .call(yAxis);

        const line = d3.line()
            .x(d => xScaleDiscrete(d.area) + xScaleLinear(d.count) - margin)
            .y(d => yScale(d3.timeParse("%Y-%m-%d %H:%M:%S")(d.time)))
            .curve(d3.curveStepAfter);

        const areas = d3.group(minutes, d => d.area);
        
        const horizontalLine = svg.append('line')
            .attr('stroke', 'black')
            .attr('stroke-width', 1)
            .style('display', 'none');

        const xScaleTime = d3.scaleTime()
            .domain(timeRange)
            .range([0 + margin, height - margin]);

        areas.forEach((area, key) => {
            svg.append("path")
                .datum(area)
                .attr("fill", "none")
                .attr("stroke", "black")
                .attr("stroke-width", 1)
                .attr("d", line)
                .attr("fill", colorScale[key])
                .attr("fill-opacity", 0.6)
                .on('mouseover', function(event ,d, i) {
                    let hoverIndex = d;
                    svg.selectAll("path")
                    .style("opacity", function(d, i) {
                        if (d===hoverIndex) {
                            return 1
                        }
                        else {
                            return 0.2
                        }
                    }
                    )
                    .style("stroke-width", function(d, i) {
                        if (d===hoverIndex) {
                            return 2
                        }
                    }
                    )
                })
                .on('mouseout', function(d, i) {
                    tooltipRectangle.style("display", "none");
                    svg.selectAll("path")
                    .style("opacity", 0.6)
                    .style("stroke-width", 1);
                    horizontalLine.style('display', 'none');
                    })
                .on('mousemove', function(event, d, i) {
                    const mouseX = d3.pointer(event)[0];
                    const mouseY = d3.pointer(event)[1];

                    const timeValue = xScaleTime.invert(mouseY);


                    let low = 0;
                    let high = d.length - 1;
                    let index = -1;

                    while (low <= high) {
                    const mid = Math.floor((low + high) / 2);
                    const midTime = d3.timeParse("%Y-%m-%d %H:%M:%S")(d[mid].time);
                    if (midTime.getTime() === timeValue.getTime()) {
                        index = mid;
                        break;
                      } else if (midTime.getTime() < timeValue.getTime()) {
                        low = mid + 1;
                      } else {
                        high = mid - 1;
                      }
                    }
                    if (index===-1) {
                        index = high
                    }
                    const nearestData = d[index]
                    tooltipRectangle
                        // .attr("transform", `translate(${mouseX + 10}, ${mouseY - 25})`)
                        .attr("transform", `translate(${xScaleDiscrete(nearestData.area) + xScaleLinear(nearestData.count) - margin}, ${mouseY})`)
                        .style("display", "block")

                    tooltipText.text("");

                    tooltipText.text(`AREA: ${nearestData.area}`);
                    tooltipText.append('tspan')
                        .attr('x', 5)
                        .attr('dy', '1.2em')
                        .text(`TIME: ${nearestData.time.slice(10, 16)}`);

                    tooltipText.append('tspan')
                        .attr('x', 5)
                        .attr('dy', '1.2em')
                        .text(`COUNT: ${nearestData.count}`);
                    
                horizontalLine
                    .attr('x1', margin) 
                    .attr('y1', mouseY)
                    .attr('x2', d => xScaleDiscrete(nearestData.area) + xScaleLinear(nearestData.count) - margin)  
                    .attr('y2', mouseY)
                    .style('display', 'block');
                })
            })
            const tooltipRectangle = svg.append("g")
                                .attr("class", "tooltip")
                                .style("display", "none");

        tooltipRectangle.append("rect")
            .attr("width", 150)
            .attr("height", 75)
            .attr("fill", "white")
            .attr("stroke", "black")
            .attr("stroke-width", 3)
            .style("z-index", 3);

        const tooltipText = tooltipRectangle.append('text')
            .attr('x', 5)
            .attr('y', 20)
            .style('font-size', '16px')
            .style('fill', 'black');

        });
</script>

<div id="timeline"></div>

<style>
    #timeline {
        position: relative;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        z-index: 1;
        background-color: rgba(255, 255, 255, 0);
        /* border: 2px solid green; */
    }
</style>