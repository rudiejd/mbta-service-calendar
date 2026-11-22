<script lang="ts">

import { toISODateString } from "./utils";

let { dateToServices, maxDate } = $props();

// gets the date of monday (day index 1 in JS) immediately before given day
function calculateStartDate(date: Date) {
    const dayOfWeek = date.getDay();

    // JS day of week is 0-6 starting on Sunday
    let daysToSubtract = dayOfWeek - 1;

    // if it's monday, do nothing
    if (dayOfWeek == 1) {
        daysToSubtract = 0;
    // if it's sunday, start on the monday of the previous week
    } else if (dayOfWeek == 0) {
        daysToSubtract = 6;
    }

    let startDate = new Date(date);
    startDate.setDate(startDate.getDate() - daysToSubtract);

    return startDate;
}

function calculateOffsetDate(startDate: Date, days: number, weeks: number) {
    let offsetDate = new Date(startDate);
    offsetDate.setDate(startDate.getDate() + weeks * 7 + days)

    return offsetDate;
}

</script>

<table class="day-container">
    <thead>
        <tr>
            <th>Monday</th>
            <th>Tuesday</th>
            <th>Wednesday</th>
            <th>Thursday</th>
            <th>Friday</th>
            <th>Saturday</th>
            <th>Sunday</th>
        </tr>
    </thead>
    <tbody>
    {let startDate = calculateStartDate(new Date())}
    {#each { length: 8 }, weeks}
        <tr>
            {#each { length: 7 }, days}
                {let offsetDateKey = toISODateString(calculateOffsetDate(startDate, days, weeks))}

                {#if dateToServices.has(offsetDateKey)}
                    {let service = dateToServices.get(offsetDateKey)[0]}
                    <td class="service-day" style="background-color: {service.color}; --backgroundColor: {service.color}" > 
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
