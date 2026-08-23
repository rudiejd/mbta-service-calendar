<script lang="ts">

const apiUrl = "https://api-v3.mbta.com/services?filter[route]=1"

function toGTFSDayOfWeek(day: number) {
    if (day == 0) {
        return 7;
    }

    return day;
}

function toISODateString(date: Date) {
    return date.toISOString().split('T')[0];
}

function createServiceDateMap(json: any) {
    let dateToServices = new Map();
    for (const service of json.data) {
        const startDate = new Date(service.attributes.start_date + "T00:00:00");
        const endDate = new Date(service.attributes.end_date + "T00:00:00");
        const validDays = service.attributes.valid_days;
        const removedDates = service.attributes.removed_dates;
        const addedDates = service.attributes.added_dates;

        const relevantDetails = {"description": service.attributes.description, "id": service.id}


        while (startDate <= endDate) {
            const gtfsDay = toGTFSDayOfWeek(startDate.getDay());
            const isoDateString = toISODateString(startDate);

            if (!validDays.includes(gtfsDay) || removedDates.includes(isoDateString)) {
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

        for (const addedDate of addedDates) {
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
    let res = await fetch(apiUrl);
    if (res.ok) {
        let json = await res.json();

        return createServiceDateMap(json);
    } else {
        return Promise.reject();
    }
});


</script>

<h1>Service Calendar</h1>



{#await serviceToDates}
    <p>Loading...</p>
{:then serviceToDates}
    {#each [...serviceToDates.keys()].sort() as day}
        <p>Day: {day}</p>
        <p>Services: {JSON.stringify(serviceToDates.get(day))} </p>
    {/each}
{:catch}
    <p style="color: red">Encountered error from API:</p>
{/await}

