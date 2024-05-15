<style>
    body {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        margin: 0;
        height: 100vh;
        background-color: #f9f9f9;
    }

    main {
        text-align: center;
        margin-bottom: 20px;
    }

    h1 {
        color: #2b2d42;
        padding: 20px 0;
        margin: 0;
    }

    .controls-container {
        position: absolute;
        left: 50%;
        transform: translateX(-50%);
        display: flex;
        align-items: center;
        justify-content: center;
    }

    #controls {
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: center;
        gap: 10px; /* Add some spacing between the elements */
        margin-bottom: 20px;
    }

    #map-container {
        display: flex;
        align-items: center;
        justify-content: center;
        flex-grow: 1;
    }

    svg {
        border: 1px solid #ccc;
        background-color: #2C4E80;
    }

    #date-slider {
        width: 300px;  /* Adjust the width of the slider */
    }

    #date-display {
        position: absolute;
        left: 50%;
        transform: translateY(100%);
        font-weight: bold;
        color: #8d99ae;
    }

    .point, .ray {
        transition: all 0.60s ease;
    }
</style>

<script>
    import * as d3 from 'd3';
    import { onMount } from 'svelte';

    let data;
    let geojsonData;
    let selectedYear = 2020;
    let selectedMonth = 0;  // 0 corresponds to January
    const width = 1200;  // Increased width
    const height = 800;  // Increased height

    const months = [
        'January', 'February', 'March', 'April', 'May', 'June',
        'July', 'August', 'September', 'October', 'November', 'December'
    ];

    let selectedDate = `${selectedYear}-${String(selectedMonth + 1).padStart(2, '0')}-01`;

    // Initialize color scale here, we'll set its domain after data is loaded
    let colorScale = d3.scaleSequential(d3.interpolateOrRd);

    // Use Albers USA projection to include Alaska and Hawaii
    let projection = d3.geoAlbersUsa()
        .scale(1400)  // Adjust the scale for better fit
        .translate([width / 2, height / 2]);

    const pathGenerator = d3.geoPath().projection(projection);

    onMount(async () => {
        try {
            data = await d3.csv('public/proj3_us_df.csv', d => {
                return {
                    ...d,
                    sunshine_total_min: +d.sunshine_total_min,
                    longitude: +d.longitude,
                    latitude: +d.latitude,
                    date: new Date(d.date).toISOString().split('T')[0]  // Normalize date format
                };  // convert string to number and normalize date format
            });
            console.log('Weather data loaded:', data.slice(0, 5));  // Debugging

            geojsonData = await d3.json('public/UnitedStates.json');
            console.log('GeoJSON data loaded:', geojsonData);  // Debugging

            // Set domain based on the sunshine_total_min range of the data
            const tempExtents = d3.extent(data, d => d.sunshine_total_min);
            colorScale.domain(tempExtents);

            drawMap();
        } catch (error) {
            console.error('Error loading data:', error);
        }
    });

    function drawMap() {
        const svg = d3.select('#map')
            .attr('width', width)
            .attr('height', height);

        svg.selectAll('path')
            .data(geojsonData.features)
            .enter()
            .append('path')
            .attr('d', pathGenerator)
            .attr('fill', 'lightgray');  // Default color before data is processed

        updateMap(selectedDate);
    }

    function updateMap(date) {
        const svg = d3.select('#map');
        const filteredData = data.filter(d => d.date === date);

        console.log('Filtered data for date:', date, filteredData);  // Debugging

        // Ensure we have valid filtered data
        if (filteredData.length === 0) {
            console.warn('No data available for the selected date:', date);
            return;
        }

        // Update existing points and rays
        const points = svg.selectAll('.point')
            .data(filteredData, d => d.station_id);

        const pointsEnter = points.enter()
            .append('circle')
            .attr('class', 'point')
            .attr('cx', d => {
                const coords = projection([d.longitude, d.latitude]);
                return coords ? coords[0] : -9999;
            })
            .attr('cy', d => {
                const coords = projection([d.longitude, d.latitude]);
                return coords ? coords[1] : -9999;
            })
            .attr('r', 0)  // Initial radius for smooth transition
            .attr('fill', 'rgba(255, 255, 153, 0.7)')
            .on('mouseover', function(event, d) {
                d3.select(this)
                    .attr('r', 3 + (d.sunshine_total_min / 50))
                    .attr('fill', 'orange');

                svg.append('text')
                    .attr('id', 'tooltip')
                    .attr('x', projection([d.longitude, d.latitude])[0] + 10)
                    .attr('y', projection([d.longitude, d.latitude])[1] - 10)
                    .attr('fill', 'black')
                    .text(`Average Sunshine: ${d3.format('.2f')(d.sunshine_total_min)} mins`);
            })
            .on('mouseout', function(event, d) {
                d3.select(this)
                    .attr('r', 2 + (d.sunshine_total_min / 50))
                    .attr('fill', 'rgba(255, 255, 153, 0.7)');  // Revert to lighter yellow

                svg.select('#tooltip').remove();
            });

        pointsEnter.merge(points)
            .transition()
            .duration(800)
            .attr('r', d => 2 + (d.sunshine_total_min / 50))
            .attr('fill', 'rgba(255, 255, 153, 0.7)');

        points.exit().transition().duration(500)
            .attr('r', 0)  // Smooth exit
            .remove();

        // Update sun rays
        svg.selectAll('.ray').remove();  // Remove existing rays before updating

        filteredData.forEach(d => {
            const coords = projection([d.longitude, d.latitude]);
            if (!coords) return;  // Skip if no coordinates found
            const rayLength = 4 + (d.sunshine_total_min / 25);
            const rayColor = 'rgba(255, 255, 153, 0.7)';

            for (let i = 0; i < 8; i++) {
                const angle = (Math.PI * 2 / 8) * i;
                svg.append('line')
                    .attr('class', 'ray')
                    .attr('x1', coords[0])
                    .attr('y1', coords[1])
                    .attr('x2', coords[0])
                    .attr('y2', coords[1])
                    .attr('stroke', rayColor)
                    .attr('stroke-width', 1.5)
                    .transition().duration(500)
                    .attr('x2', coords[0] + rayLength * Math.cos(angle))
                    .attr('y2', coords[1] + rayLength * Math.sin(angle));
            }
        });
    }

    function handleSliderChange(event) {
        selectedMonth = event.target.value;
        selectedDate = `${selectedYear}-${String(parseInt(selectedMonth) + 1).padStart(2, '0')}-01`;
        document.getElementById('date-display').innerText = months[selectedMonth] + ' ' + selectedYear;
        updateMap(selectedDate);
    }
</script>

<main>
    <h1 color = 'darkgray'>Minutes of Sunshine in the United States 2010-2020</h1>
    <div id="controls">
        <select bind:value={selectedYear} on:change={() => { selectedDate = `${selectedYear}-${String(parseInt(selectedMonth) + 1).padStart(2, '0')}-01`; updateMap(selectedDate); }}>
            {#each Array.from({length: 11}, (_, i) => 2010 + i) as year}
                <option value={year}>{year}</option>
            {/each}
        </select>

        <input id="date-slider" type="range" min="0" max="11" value={selectedMonth} on:input={handleSliderChange}>
        <span id="date-display">{months[selectedMonth]} {selectedYear}</span>
    </div>
</main>

<div id="map-container">
    <svg id="map"></svg>
</div>
