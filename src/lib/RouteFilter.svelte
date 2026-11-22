<script lang="ts">
// TODO: clean up type checking errors in here
let { routes, setRouteId } = $props();

let routeFilter = $state(null);

let filteredRoutes = $derived.by(() => {
    if (routeFilter) {
        return routes.filter(routeId => routeId.toLowerCase().startsWith(routeFilter.toLowerCase()));
    } else {
        return routes;
    }
})

// if there's only one route after filtering, select that one
$effect(() => {
    if (filteredRoutes.length == 1) {
        setRouteId(filteredRoutes[0]);
        document.getElementById("route-selector").value = filteredRoutes[0]
    } else {
        document.getElementById("route-selector").click();
    }
})

</script>

<div style="display: flex; flex-direction: column;">
    <label>Filter Routes</label>
    <input bind:value={routeFilter} />
</div>


<div style="display: flex; flex-direction: column;">
    <label>Select Route ID</label>
    <select id="route-selector" onchange={e => setRouteId(e.target?.value)} value="Red">
        {#each filteredRoutes as routeId}
            <option value="{routeId}">{routeId}</option>
        {/each}
    </select>
</div>

