<script lang="ts">
  // Dependencies
  import type { PageData } from "./$types";
  import type { ItemType, QueueType } from "$lib/types";
  import { browser } from "$app/environment";
  import Divider from "$lib/Divider.svelte";
  import { BarsArrowDownIcon, EditIcon, RoundArrowDownIcon } from "$lib/icons";
  import { quintOut } from "svelte/easing";
  import { crossfade } from "svelte/transition";
  import { flip } from "svelte/animate";
  import { goto } from "$app/navigation";
  import { labels } from "$lib/stores";

  // Props
  export let data: PageData;

  // State
  let { id, name, items = [], showCounter } = data as QueueType;
  items = items.map((item) => ({ ...item, count: 0 }));
  $: activeItems = items.filter((_, index) => [0, 1].includes(index));
  $: inactiveItems = items.filter((_, index) => ![0, 1].includes(index));
  let states: ItemType[][] = [];

  // Utils
  const syncWithLocalStorage = () => {
    const newQ = { id, name, items, showCounter };
    const currentState = window.localStorage.getItem("fifo");

    if (currentState) {
      let newState = JSON.parse(currentState);
      newState = newState.map((q: QueueType) => (q.id === id ? newQ : q));
      window.localStorage.setItem("fifo", JSON.stringify(newState));
    }
  };

  const [send, receive] = crossfade({
    duration: 1000,
    fallback(node) {
      const style = getComputedStyle(node);
      const transform = style.transform === "none" ? "" : style.transform;

      return {
        duration: 1000,
        easing: quintOut,
        css: (t) => `
					transform: ${transform} scale(${t});
					opacity: ${t}
				`,
      };
    },
  });

  // Handlers
  const handleSendLast = (index: number) => {
    states = [...states, items];
    const itemToBeMoved = items[index];
    const newItems = items.filter((_: ItemType, i: number) => i !== index);
    items = [...newItems, itemToBeMoved];
    items[0].count = items[1].count = 0;
    browser && syncWithLocalStorage();
  };

  const handleCounterClick = (id: string) => {
    items = items.map((item) =>
      item.id === id ? { ...item, count: item.count + 1 } : item
    );
  };

  const handleUndo = () => {
    const lastState = states.pop();

    if (lastState) {
      items = lastState;
      browser && syncWithLocalStorage();
    }
  };
</script>

<Divider />

<div class="flex justify-between items-center">
  <div class="flex items-center gap-2">
    <h2>{name ?? "Loading..."}</h2>
    <button on:click={() => goto(`/edit/${id}`)}>
      {@html EditIcon}
    </button>
  </div>
  {#if states.length}
    <button class="flex items-center gap-2" on:click={handleUndo}>
      {$labels.UNDO}
      {@html RoundArrowDownIcon}
    </button>
  {/if}
</div>

<ul class="flex flex-col gap-2">
  {#each activeItems as { id, value, count }, index (id)}
    <li
      class="outlined flex items-center justify-between"
      in:receive|local={{ key: id }}
      out:send|local={{ key: id }}
      animate:flip
    >
      <span class="font-bold">{value}</span>

      <span class="flex gap-4">
        {#if showCounter}
          <button
            class="filled py-1 w-7 h-7 flex justify-center items-center text-sm"
            on:click={() => handleCounterClick(id)}
          >
            {count}
          </button>
        {/if}

        {#if items.length > 2}
          <button
            class="filled py-1 text-sm flex items-center gap-2"
            on:click={() => handleSendLast(index)}
          >
            {$labels.SEND_TO_LAST}
            {@html BarsArrowDownIcon}
          </button>
        {/if}
      </span>
    </li>
  {/each}
</ul>

{#if items.length > 2}
  <div class="my-4 w-1/2 m-auto"><Divider /></div>
{/if}

<ul class="flex flex-col gap-2">
  {#each inactiveItems as { id, value, count }, index (id)}
    <li
      class="outlined flex items-center justify-between"
      in:receive|local={{ key: id }}
      out:send|local={{ key: id }}
      animate:flip
    >
      <span class="font-bold">{value}</span>

      <span class="flex items-center">
        {#if index === 0}
          <span class="text-sm italic mr-2">{$labels.UP_NEXT}</span>
        {/if}

        {#if count}
          <span class="outlined py-1 text-sm border-2">
            {$labels.LAST_COUNT} | {count}
          </span>
        {/if}
      </span>
    </li>
  {/each}
</ul>
