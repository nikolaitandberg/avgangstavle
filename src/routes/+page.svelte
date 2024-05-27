<script>
    import { onMount } from 'svelte';
    import { XMLParser } from 'fast-xml-parser';
    import textEncoding from 'text-encoding';

    // Data variables
    let flightData = null;
    let airportData = null;
    let airlineData = null;

    // Selected airport
    let selectedAirport = 'OSL';

    // Helper functions
    function convertISOToTime(isoString) {
        const date = new Date(isoString);
        return date.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit', hour12: false });
    }

    function flightIsAfterCurrentTime(flight) {
        const scheduledTime = new Date(flight.schedule_time);
        const now = new Date();
        const newTime = flight.status && flight.status.code === 'E' ? new Date(flight.status.time) : null;

        return scheduledTime.getTime() >= now.getTime() || (newTime && newTime.getTime() >= now.getTime());
    }

    function codeToName(data, code) {
        const item = data.find(item => item.code === code);
        return item ? item.name : "not found";
    }

    // Fetch data functions
    async function fetchData(url, encoding = 'windows-1252') {
        try {
            const response = await fetch(url);
            const buffer = await response.arrayBuffer();
            const decoder = new textEncoding.TextDecoder(encoding);
            const data = decoder.decode(buffer);

            const parser = new XMLParser({
                ignoreAttributes: false,
                attributeNamePrefix: ""
            });
            const result = parser.parse(data);

            return result;
        } catch (error) {
            console.error('Error fetching and parsing data:', error);
        }
    }

    async function fetchFlights() {
        const flightDataURL = `https://flydata.avinor.no/XmlFeed.asp?TimeFrom=1TimeTo=7&airport=${selectedAirport}&direction=D&lastUpdate=2009-03-10T15:03:00Z`;
        const flightDataResult = await fetchData(flightDataURL);
        flightData = flightDataResult.airport.flights.flight.filter(flightIsAfterCurrentTime);
    }

    // Fetch data on mount
    onMount(async () => {
        fetchFlights();

        const airportDataURL = 'https://flydata.avinor.no/airportNames.asp';
        const airportDataResult = await fetchData(airportDataURL);
        airportData = airportDataResult.airportNames.airportName;

        const airlineDataURL = 'https://flydata.avinor.no/airlineNames.asp';
        const airlineDataResult = await fetchData(airlineDataURL);
        airlineData = airlineDataResult.airlineNames.airlineName;
    });
</script>

<!-- HTML Markup -->
<nav class="container-fluid">
    <ul>
        <li>
            <svg xmlns="http://www.w3.org/2000/svg" width="1em" height="1em" viewBox="0 0 24 24" {...$$props}>
            <path fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M16 10h4a2 2 0 0 1 0 4h-4l-4 7H9l2-7H7l-2 2H2l2-4l-2-4h3l2 2h4L9 3h3z" />
        </svg>
        </li>
        <li><strong>Avgangstavle</strong></li>
    </ul>
    <ul>
        <li><a href="https://avinor.no">Flydata fra avinor</a></li>
        <li><a href="https://github.com/nikolaitandberg/avgangstavle">Github</a></li>
    </ul>
</nav>
<main class="container">
    <h1>Neste avganger fra</h1>
    <select name="valgt-flyplass" aria-label="velg flyplass" required bind:value={selectedAirport} on:change={fetchFlights}>
        <option selected disabled value="">velg flyplass</option>
        <option>OSL</option>
        <option>BGO</option>
        <option>SVG</option>
        <option>TRD</option>
    </select>
    <table>
        {#if flightData && airportData && airlineData}
        <thead>
          <tr>
            <th scope="col">Avgang</th>
            <th scope="col">Til</th>
            <th scope="col">Flight</th>
            <th scope="col">Gate</th>
          </tr>
        </thead>
        <tbody>
                {#each flightData as flight}
                    <tr>
                        <th scope="row">
                            {convertISOToTime(flight.schedule_time)}
                            {#if flight.status && flight.status.code == 'E'}
                                <br> 
                                <span style="color: orange">ny tid {convertISOToTime(flight.status.time)}</span>
                            {:else if flight.status && flight.status.code == 'C'}
                                <br>
                                <span style="background-color: red; color: white;">INNSTILLT</span>
                            {:else}
                                <br>
                                <span style="visibility: hidden;">placeholder</span>
                            {/if}
                        </th>
                        <td>{codeToName(airportData, flight.airport)}</td>
                        <td>{flight.flight_id} <br> {codeToName(airlineData, flight.airline)}</td>
                        <td>
                            {#if flight.status && flight.status.code != 'C' || !flight.status && flight.gate}
                                Gate {flight.gate}
                            {/if}
                        </td>
                    </tr>
                {/each}
            
        </tbody>
        {:else}
                <p>loading...</p>
            {/if}
    </table>
</main>