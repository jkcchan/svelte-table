<script>
	import { createEventDispatcher } from 'svelte';

	const dispatch = createEventDispatcher();

	export let classes = [];
	export let columns;
	export let rows;
	export let sortBy = "";
	export let sortOrder = 1;
	export let iconAsc = '▲';
	export let iconDesc = '▼';

	let sortFunction = () => "";
	let showFilterHeader = columns.some(c => c.filterOptions !== undefined);
	let filterValues = {};
	let filterSettings = {};
	let columnByKey = {};
	columns.forEach(col => {
	  columnByKey[col.key] = col;
	});

	$: c_rows = rows
	  .filter(r =>
	    Object.keys(filterSettings).every(f => {
	      let ret =  (
					filterSettings[f] === undefined ||
					// default to value() if filterValue() not provided in col
					filterSettings[f] === (typeof columnByKey[f].filterValue === 'function' ? 
						columnByKey[f].filterValue(r) : columnByKey[f].value(r))
				);
				return ret;
	    })
	  )
	  .map(r => (Object.assign({}, r, {$sortOn: sortFunction(r)} ) ) )
	  .sort((a, b) => {
	    if (a.$sortOn > b.$sortOn) return sortOrder;
	    else if (a.$sortOn < b.$sortOn) return -sortOrder;
	    return 0;
	  });

	const calculateFilterValues = () => {
	  filterValues = {};
	  columns.forEach(c => {
	    if (typeof c.filterOptions === "function") {
				filterValues[c.key] = c.filterOptions(rows);
	    } else if (Array.isArray(c.filterOptions)) {
				// if array of strings is provided, use it for name and value
				filterValues[c.key] = c.filterOptions.map(val => ({name:val, value:val}));
	    }
	  });
	};


	$: {
		let col = columnByKey[sortBy];
	  if (col !== undefined && col.sortable === true && typeof col.value === "function") {
	    sortFunction = r => col.value(r);
	  }
	};

	const updateSortOrder = (colKey) => {
		if (colKey === sortBy) {
			sortOrder = sortOrder === 1 ? -1 : 1
		} else {
			sortOrder = 1;
		}
	}
	
	const handleClickCol = (col) => {
		updateSortOrder(col.key)
		sortBy = col.key;
		dispatch('clickCol', {key:col.key} );
	}
	
	const handleClickRow = (row) => {
		dispatch('clickRow', {row} );
	}

	const handleClickCell = (row, key) => {
		dispatch('clickCell', {row, key} );
	}

	if (showFilterHeader) {
	  calculateFilterValues();
	}
</script>

<style>
	table {
	  width: 100%;
	}
	.isSortable {
	  cursor: pointer;
	}

	tr th select {
		width: 100%;
	}
</style>

<table class="{classes.join(' ')}">
	<thead>
		{#if showFilterHeader}
			<tr>
			{#each columns as col}
				<th>
						{#if filterValues[col.key] !== undefined}
							<div class="select">
							<select bind:value={filterSettings[col.key]}>
								<option value={undefined}></option>
							{#each filterValues[col.key] as option}
									<option value={option.value}>{option.name}</option>
							{/each}
							</select>
							</div>
						{/if}
					</th>
			{/each}
			</tr>
		{/if}
		<slot name="header" sortOrder={sortOrder} sortBy={sortBy}>
			<tr>
			  {#each columns as col}
			  	<th
						on:click={() => handleClickCol(col)}
						class="{[(col.sortable ? 'isSortable' : '' ),(col.headerClass !== undefined && col.headerClass)].join(' ')}"
					>
						{col.title}
						{#if sortBy === col.key}
							{ sortOrder === 1 ? iconAsc : iconDesc}
						{/if}
					</th>
			  {/each}
			</tr>
		</slot>
	</thead>
	<tbody>
		{#each c_rows as row, n}
			<slot name="row" row={row} n={n} >
				<tr on:click={()=>{handleClickRow(row)}} >
				{#each columns as col}
					<td
							on:click={()=>{handleClickCell(row, col.key)}}
							class="{col.class !== undefined && col.class}"
						>{@html col.renderValue ? col.renderValue(row) : col.value(row)}</td>
				{/each}
				</tr>
			</slot>
		{/each}
	</tbody>
</table>
