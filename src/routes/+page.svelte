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

const apiUrl = "https://api-v3.mbta.com/services?filter[route]="

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

function createServiceDateMap(json: any) {
    let dateToServices = new Map();
    for (const jsonService of json.data) {
        const service = buildService(jsonService.attributes, jsonService.id)
        const startDate = new Date(service.startDate + "T00:00:00")
        const endDate = new Date(service.endDate + "T00:00:00")

        const relevantDetails = {"description": service.description, "id": service.id, "color": stringToColour(service.id)}


        while (startDate <= endDate) {
            const gtfsDay = toGTFSDayOfWeek(startDate.getDay());
            const isoDateString = toISODateString(startDate);

            if (!service.validDays.includes(gtfsDay) || service.removedDates.includes(isoDateString)) {
                startDate.setDate(startDate.getDate() + 1);
                continue;
            }


            if (dateToServices.has(isoDateString)) {
                const existingServices = dateToServices.get(startDate);
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


    return dateToServices;
}

let serviceToDates = $derived.by(async () => {
    let res = await fetch(apiUrl + routeId);
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
    let res = await fetch("https://api-v3.mbta.com/routes");
    if (res.ok) {
        let json = await res.json();

        return extractRoutes(json);
    } else {
        return Promise.reject();
    }
});



</script>

<h1>Service Calendar</h1>

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
{/await}



{#await serviceToDates}
    <p>Loading...</p>
{:then serviceToDates}
    <div class="day-container">
        {#each [...serviceToDates.keys()].sort() as day}
            <div class="service-day" style="color: {serviceToDates.get(day)[0].color}">
                <p>{day}</p>
                <p>{serviceToDates.get(day)[0].id}</p>
                <p>{serviceToDates.get(day)[0].description}</p>
            </div>
        {/each}
    </div>
{:catch}
    <p style="color: red">Encountered error from API:</p>
{/await}


<style>

.service-day {
    border-style: solid;
    text-align: center;

}

.day-container {
    display: grid;
    grid-template-columns: auto auto auto auto auto auto auto;
    font-size: 14px;
}


</style>
