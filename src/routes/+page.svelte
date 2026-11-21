<script lang="ts">


interface Service {
    startDate: string;
    endDate: string;
    validDays: number[];
    removedDates: string[];
    addedDates: string[];
    description: string;
    id: string;
}

let routeId = $state('Green-E');

let apiUrl = $state("https://api-v3.mbta.com")

const apiQuery = "/services?filter[route]="

function toGTFSDayOfWeek(day: number) {
    if (day == 0) {
        return 7;
    }

    return day;
}

function toISODateString(date: Date) {
    return date.toISOString().split('T')[0];
}


function stringToColour(str: string): string {
  let hash = 0;
  str.split('').forEach(char => {
    hash = char.charCodeAt(0) + ((hash << 5) - hash)
  })
  let color = '#'
  for (let i = 0; i < 3; i++) {
    const value = (hash >> (i * 8)) & 0xff
    color += value.toString(16).padStart(2, '0')
  }
  return color
}


function buildService(attributes: any, id: string): Service {
    return {
        startDate: attributes.start_date,
        endDate: attributes.end_date,
        validDays: attributes.valid_days,
        removedDates: attributes.removed_dates,
        addedDates: attributes.added_dates,
        description: attributes.description,
        id: id
    }
}

// map of ISO datestring => services on that day
// also returns maximum ISO date of all service
function createServiceDateMap(json: any) {
    let dateToServices = new Map();
    let maxDate = new Date().toISOString();
    for (const jsonService of json.data) {
        const service = buildService(jsonService.attributes, jsonService.id)
        const startDate = new Date(service.startDate + "T00:00:00")
        const endDate = new Date(service.endDate + "T00:00:00")

        if (endDate.toISOString() > maxDate) {
            maxDate = endDate.toISOString();
        }

        const relevantDetails = {"description": service.description, "id": service.id, "color": stringToColour(service.id)}

        while (startDate <= endDate) {
            const gtfsDay = toGTFSDayOfWeek(startDate.getDay());
            const isoDateString = toISODateString(startDate);

            if (!service.validDays.includes(gtfsDay) || service.removedDates.includes(isoDateString)) {
                startDate.setDate(startDate.getDate() + 1);
                continue;
            }


            if (dateToServices.has(isoDateString)) {
                // the default handles if the service has been removed
                const existingServices = dateToServices.get(startDate) || [];
                existingServices.push(relevantDetails);
                dateToServices.set(isoDateString, existingServices);
            } else {
                dateToServices.set(isoDateString, new Array(relevantDetails));
            }

            startDate.setDate(startDate.getDate() + 1);
        }

        for (const addedDate of service.addedDates) {
            if (dateToServices.has(addedDate)) {
                const existingServices = dateToServices.get(addedDate);
                existingServices.push(relevantDetails);
                dateToServices.set(addedDate, existingServices);
            } else {
                dateToServices.set(addedDate, new Array(relevantDetails));
            }
        }

    }


    return {dateToServices: dateToServices, maxDate: maxDate};
}

let serviceData = $derived.by(async () => {
    let res = await fetch(apiUrl + apiQuery + routeId);
    if (res.ok) {
        let json = await res.json();

        return createServiceDateMap(json);
    } else {
        return Promise.reject();
    }
});

function extractRoutes(json: any): string[] {
    let routes = [];
    for (const jsonRoute of json.data) {
        routes.push(jsonRoute.id)
    }

    return routes;
}

let routes = $derived.by(async () => {
    let res = await fetch(`${apiUrl}/routes`);
    if (res.ok) {
        let json = await res.json();

        return extractRoutes(json);
    } else {
        return Promise.reject();
    }
});



</script>

<h1>MBTA Service Calendar</h1>

<p>A tool for showing which schedules the MBTA is running in its <a href="https://gtfs.org">GTFS feed</a> </p>
<p>Commonly used by in-house teams to understand which schedules have been published.</p>

{#await routes}
{:then routes}
<div style="display: flex; flex-direction: column;">
    <label>Route ID</label>
    <select bind:value={routeId}>
        {#each routes as routeId}
            <option value="{routeId}">{routeId}</option>
        {/each}
    </select>
</div>

<div style="display: flex; flex-direction: column;">
    <label>API</label>
    <input type="text" bind:value={apiUrl} />
</div>

<br />
{/await}



{#await serviceData}
    <p>Loading...</p>
{:then serviceData}
    {let dateToServices = serviceData.dateToServices}
    {let maxDate = serviceData.maxDate}

    <table class="day-container">
        <thead>
            <tr>
                <th>Sunday</th>
                <th>Monday</th>
                <th>Tuesday</th>
                <th>Wednesday</th>
                <th>Thursday</th>
                <th>Friday</th>
                <th>Saturday</th>
            </tr>
        </thead>
        <tbody>
        {let startDay = new Date()}
        <!-- find a way to do this not in the template -->
        { let _ = startDay.setDate(startDay.getDate() - startDay.getDay()) }
        {#each { length: 8 }, week}
            <tr>
                {#each { length: 7 }, day}
                    {let daysOffset = week * 7 + day}
                    {let offsetDate = new Date(startDay)}
                    <!-- find a way to do this not in the template -->
                    {let _ = offsetDate.setDate(offsetDate.getDate() + daysOffset)}
                    {let offsetDateKey = toISODateString(offsetDate)}

                    {#if dateToServices.has(offsetDateKey)}
                        {let service = dateToServices.get(offsetDateKey)[0]}
                        <td class="service-day" style="background-color: {service.color}" > 
                           <p> {offsetDateKey}</p>
                           <p> {service.id} </p>
                           <p> {service.description} </p>
                        </td>
                    {:else if offsetDateKey < maxDate}
                        <td class="service-day">
                            {offsetDateKey}
                            <p>No service</p>
                            <p>(no service for this route on this day)</p>
                        </td>
                    {:else}
                        <td class="service-day">
                            {offsetDateKey}
                            <p>Unknown service</p>
                            <p>(schedule not yet available)</p>
                        </td>
                    {/if}
                {/each}
            </tr>
        {/each}
        </tbody>
    </table>
{:catch}
    <p style="color: red">Encountered error from API:</p>
{/await}

<style>

.service-day {
    border-style: solid;
    text-align: center;
    mix-blend-mode: difference;
    color: contrast-color(var(--backgroundColor));
}

table, th, td {
  border: 1px solid black;
  border-collapse: collapse;
}


</style>
