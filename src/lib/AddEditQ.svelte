<script lang="ts">
  // Dependencies
  import { goto } from "$app/navigation";
  import { browser } from "$app/environment";
  import type { ItemType, QueueType } from "$lib/types";
  import { slide } from "svelte/transition";
  import {
    CrossIcon,
    EditIcon,
    PlusIcon,
    SaveIcon,
    ThreeBarsIcon,
  } from "./icons";
  import { labels } from "./stores";

  // Props
  export let editMode = false;
  export let queueId = "";
  export let queueName = "";
  export let queueItems: ItemType[] = [];
  export let queueShowCounter = false;

  // State
  let name = queueName;
  let items = queueItems;
  let showCounter = queueShowCounter;
  let newItemInput: HTMLInputElement;
  let newItemValue = "";

  // Utils
  const syncWithLocalStorage = () => {
    const id = queueId || crypto.randomUUID();
    const newQueue = { id, name, items, showCounter };
    const currentState = window.localStorage.getItem("fifo");

    if (currentState) {
      let newState = JSON.parse(currentState);

      if (editMode)
        newState = newState.map((queue: QueueType) =>
          queue.id === id ? newQueue : queue
        );
      else newState = [...newState, newQueue];

      window.localStorage.setItem("fifo", JSON.stringify(newState));
    } else window.localStorage.setItem("fifo", JSON.stringify([newQueue]));
  };

  // Handlers
  const handleItemAdd = () => {
    if (!newItemValue) return;

    if (newItemValue.length > 5) {
      browser && window.alert($labels.ITEM_LESS_THAN_5_ERROR);
      return;
    }

    const newItem = {
      id: crypto.randomUUID(),
      value: newItemValue,
      count: 0,
    };

    items = [...items, newItem];
    newItemValue = "";
    newItemInput.focus();
  };

  const handleItemDelete = (id: string) => {
    items = items.filter((item) => item.id !== id);
  };

  const handleQueueSave = () => {
    if (!name) {
      browser && window.alert($labels.ENTER_QUEUE_NAME_ERROR);
      return;
    }

    if (name.length > 10) {
      browser && window.alert($labels.QUEUE_LESS_THAN_10_ERROR);
      return;
    }

    if (!items.length) {
      browser && window.alert($labels.ITEM_ATLEAST_ONE_ERROR);
      return;
    }

    browser && syncWithLocalStorage();
    goto("/");
  };

  const handleDrag = (event: DragEvent, startIndex: number) => {
    if (!event.dataTransfer) return;
    event.dataTransfer.setData("text/plain", `${startIndex}`);
    event.dataTransfer.effectAllowed = "move";
  };

  const handleDrop = (event: DragEvent, targetIndex: number) => {
    event.preventDefault();
    if (!event.dataTransfer) return;

    const startIndex = parseInt(event.dataTransfer.getData("text/plain"));
    if (startIndex === targetIndex) return;

    const newItems = items;
    if (startIndex < targetIndex) {
      newItems.splice(targetIndex + 1, 0, newItems[startIndex]);
      newItems.splice(startIndex, 1);
    } else {
      newItems.splice(targetIndex, 0, newItems[startIndex]);
      newItems.splice(startIndex + 1, 1);
    }

    items = newItems;
  };

  const handleItemEdit = (id: string, value: string) => {
    const newValue = window && window.prompt($labels.EDIT_ITEM_NAME, value);
    if (!newValue || newValue.length > 5) return;

    items = items.map((item) => {
      if (item.id === id) return { ...item, value: newValue };
      return item;
    });
  };
</script>

<input
  class="outlined focus:outline-none text-left py-1"
  type="text"
  placeholder={$labels.ENTER_QUEUE_NAME}
  bind:value={name}
  on:keypress={({ key }) => key === "Enter" && newItemInput.focus()}
/>

{#if items.length}
  <ul class="flex flex-col gap-2 ml-8" transition:slide|local>
    {#each items as { id, value }, index (id)}
      <li
        class="outlined flex items-center justify-between"
        draggable="true"
        on:dragstart={(event) => handleDrag(event, index)}
        on:dragover={(event) => event.preventDefault()}
        on:drop={(event) => handleDrop(event, index)}
        transition:slide|local
      >
        <span class="font-bold flex items-center gap-2">
          <span class="cursor-move"> {@html ThreeBarsIcon}</span>
          {value}
          <button
            class="cursor-pointer"
            on:click={() => handleItemEdit(id, value)}
          >
            {@html EditIcon}
          </button>
        </span>
        <button
          class="flex items-center gap-2"
          on:click={() => handleItemDelete(id)}
        >
          {$labels.REMOVE}
          {@html CrossIcon}
        </button>
      </li>
    {/each}
  </ul>
{/if}

<div class="flex gap-2">
  <input
    class="outlined w-full focus:outline-none text-left py-1"
    type="text"
    placeholder={$labels.ENTER_ITEM_NAME}
    bind:this={newItemInput}
    bind:value={newItemValue}
    on:keypress={({ key }) => key === "Enter" && handleItemAdd()}
  />
  <button
    class="filled w-fit py-1 flex items-center gap-2"
    on:click={handleItemAdd}
  >
    {$labels.ADD}
    {@html PlusIcon}
  </button>
</div>

<label class="text-xl flex items-center gap-2 cursor-pointer">
  <input
    class="appearance-none outlined checked:filled p-2 checked:p-2"
    type="checkbox"
    bind:checked={showCounter}
  />
  {$labels.ADD_COUNTER}
</label>

<button
  class="filled w-full flex items-center justify-center gap-2"
  on:click={handleQueueSave}
>
  {$labels.SAVE}
  {@html SaveIcon}
</button>
