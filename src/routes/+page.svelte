<script>
    import { onMount } from 'svelte';
    import { XMLParser } from 'fast-xml-parser';
    import textEncoding from 'text-encoding';

    let flightData = null;
    let airportData = null;
    let airlineData = null;

    let selectedAirport = 'OSL';

    function convertISOToTime(isoString) {
        const date = new Date(isoString);
        return date.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit', hour12: false });
    }

    function flightIsAfterCurrentTime(flight) {
        const scheduledTime = new Date(flight.schedule_time);
        const now = new Date();

        if (scheduledTime.getTime() >= now.getTime()) {
            return true;
        }

        if (flight.status && flight.status.code === 'E') {
            const newTime = new Date(flight.status.time);
            return newTime.getTime() >= now.getTime();
        }
        return false;
    }

    function airportCodeToName(airportIATA) {
        const airport = airportData.find(airport => airport.code === airportIATA);
        return airport ? airport.name : "airport not found";
    }

    function airlineCodeToName(airlineIATA) {
        const airline = airlineData.find(airline => airline.code === airlineIATA);
        return airline ? airline.name : "airline not found";
    }

    async function fetchFlights() {
        const flightDataURL = `https://flydata.avinor.no/XmlFeed.asp?TimeFrom=1TimeTo=7&airport=${selectedAirport}&direction=D&lastUpdate=2009-03-10T15:03:00Z`;
        const flightDataResult = await fetchData(flightDataURL);
        flightData = flightDataResult.airport.flights.flight.filter(flightIsAfterCurrentTime);
    }

    async function fetchData(url, e) {
        try {
            const encoding = 'windows-1252';

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

    onMount(async () => {
        const flightDataURL = `https://flydata.avinor.no/XmlFeed.asp?TimeFrom=1&TimeTo=7&airport=${selectedAirport}&direction=D&lastUpdate=2009-03-10T15:03:00Z`;
        const flightTestDataURL = 'flights-testData.xml'
        const flightDataResult = await fetchData(flightDataURL);
        flightData = flightDataResult.airport.flights.flight.filter(flightIsAfterCurrentTime);

        const airportDataURL = 'https://flydata.avinor.no/airportNames.asp';
        const airportTestDataURL = 'airportNames-testdata.xml'
        const airportDataResult = await fetchData(airportDataURL);
        airportData = airportDataResult.airportNames.airportName;

        const airlineDataURL = 'https://flydata.avinor.no/airlineNames.asp';
        const airlineTestDataURL = 'airlineNames-testData.xml';
        const airlineDataResult = await fetchData(airlineDataURL);
        airlineData = airlineDataResult.airlineNames.airlineName;
    });
</script>

<nav class="container-fluid">
    <ul>
      <li><strong>Avgangstavle</strong></li>
    </ul>
    <ul>
      <li><a href="https://avinor.no">Flydata fra avinor</a></li>
      <li><a href="#">Github</a></li>
    </ul>
  </nav>
<main class="container">
    <h1>Neste avganger fra</h1>
    <select name="valgt-flyplass" aria-label="velg flyplass" required bind:value={selectedAirport} on:change={fetchFlights}>
        <option selected disabled value="">
            velg flyplass
        </option>
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
                        <td>{airportCodeToName(flight.airport)}</td>
                        <td>{flight.flight_id} <br> {airlineCodeToName(flight.airline)}</td>
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