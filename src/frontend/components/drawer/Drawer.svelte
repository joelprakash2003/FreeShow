<script lang="ts">
  import type { Bible } from "../../../types/Scripture"
  import type { DrawerTabIds } from "../../../types/Tabs"
  import {
    activeDrawerTab,
    activeEdit,
    activePage,
    activePopup,
    activeProject,
    activeShow,
    dictionary,
    drawer,
    drawerOpenedInEdit,
    drawerTabsData,
    focusMode,
    labelsDisabled,
    os,
    previousShow,
    projects,
    quickTextCache,
    selected,
  } from "../../stores"
  import {
    DEFAULT_DRAWER_HEIGHT,
    DEFAULT_WIDTH,
    MENU_BAR_HEIGHT,
  } from "../../utils/common"
  import { getAccess } from "../../utils/profile"
  import { drawerTabs } from "../../values/tabs"
  import Content from "../drawer/Content.svelte"
  import Navigation from "../drawer/Navigation.svelte"
  import { keysToID } from "../helpers/array"
  import { history } from "../helpers/history"
  import Icon from "../helpers/Icon.svelte"
  import { selectTextOnFocus } from "../helpers/inputActions"
  import T from "../helpers/T.svelte"
  import Button from "../inputs/Button.svelte"
  import Resizeable from "../system/Resizeable.svelte"
  import Info from "./info/Info.svelte"

  const minHeight = 40
  const topHeight = 40
  let maxHeight =
    window.innerHeight -
    topHeight -
    ($os.platform === "win32" ? MENU_BAR_HEIGHT - 0.3 : 0)
  $: height = $drawer.height

  let move = false
  let mouse: null | { x: number; y: number; offsetY: number } = null
  function mousedown(e: any) {
    if (e.target.closest(".search")) return

    maxHeight =
      window.innerHeight -
      topHeight -
      ($os.platform === "win32" ? MENU_BAR_HEIGHT - 0.3 : 0)
    mouse = {
      x: e.clientX,
      y: e.clientY,
      offsetY: window.innerHeight - height - e.clientY,
    }
  }

  function mousemove(e: any) {
    if (!mouse) return

    drawer.set({
      height: getHeight(window.innerHeight - e.clientY - mouse.offsetY),
      stored: null,
    })

    selected.set({ id: null, data: [] })

    const isClosed = $drawer.height <= minHeight
    if (isClosed) drawerOpenedInEdit.set(false)
    else if ($activePage === "edit") drawerOpenedInEdit.set(true)
  }

  function getHeight(height: number) {
    if (height < minHeight * 2) return minHeight
    if (height > maxHeight) return maxHeight
    move = true
    return height
  }

  function click(e: any) {
    if (
      (height > minHeight && e?.target?.closest("button")) ||
      move ||
      e?.target instanceof HTMLInputElement
    ) {
      move = false
      return
    }

    if (height > minHeight) {
      // close drawer
      if (
        e === null ||
        e?.target.classList.contains("top") ||
        e?.target?.closest("#" + $activeDrawerTab)
      )
        closeDrawer()
      drawerOpenedInEdit.set(false)
      return
    }

    // open drawer
    drawer.set({
      height: DEFAULT_DRAWER_HEIGHT,
      stored: null,
    })
    if ($activePage === "edit") drawerOpenedInEdit.set(true)

    // Set category to "all" for relevant tabs when opening drawer
    if (
      e === null &&
      ["shows", "templates", "media"].includes($activeDrawerTab)
    ) {
      drawerTabsData.update((a) => {
        a[$activeDrawerTab].activeSubTab = "all"
        return a
      })
    }
  }

  function closeDrawer() {
    drawer.set({ height: minHeight, stored: height })
    drawerOpenedInEdit.set(false)
  }

  function mouseup(e: any) {
    if (
      !e.target.closest("input") &&
      !e.target.closest(".contextMenu") &&
      !searchValue.length
    )
      // No need to reset searchActive since search is always visible

      mouse = null
    if (!e.target.closest(".top")) move = false
  }

  function openDrawerTab(tab: { id: string; name: string; icon: string }) {
    if ($activeDrawerTab === tab.id) return

    // allow click event first
    setTimeout(() => {
      activeDrawerTab.set(tab.id as DrawerTabIds)

      // remove focus for search function to work
      setTimeout(() => (document.activeElement as any)?.blur(), 10)
    }, 10)
  }

  let searchValue = ""
  $: searchValue = searchValue.endsWith(" ")
    ? removeWhitespace(searchValue) + " "
    : removeWhitespace(searchValue)
  $: if ($activeDrawerTab) searchValue = ""
  const removeWhitespace = (v: string) =>
    v
      .split(" ")
      .filter((n) => n)
      .join(" ")
  function search() {
    // Search is always visible now, no need to check drawer state
    // if ($activeDrawerTab === "shows") {
    // }
  }

  let bibles: Bible[] = []

  let firstMatch: null | any = null
  let searchElem: HTMLInputElement | undefined
  function keydown(e: KeyboardEvent) {
    if ((e.ctrlKey || e.metaKey) && e.key === "f") {
      if ($activePopup === "show") return
      // Search is always visible now, just focus it
      searchElem?.focus()
    } else if ((e.ctrlKey || e.metaKey) && e.key === "d") {
      if (!$selected?.id && !$activeEdit.items.length) click(null)
    } else if (e.key === "Enter") {
      if (
        document.activeElement !== searchElem ||
        !searchValue.length ||
        !firstMatch ||
        !$activeProject ||
        $focusMode
      )
        return
      if ($activeDrawerTab !== "shows") return

      let match =
        $activeShow?.data?.searchInput === true
          ? { id: $activeShow.id }
          : firstMatch

      // create from search
      if (match === "SEARCH_CREATE") {
        quickTextCache.set({
          name: searchValue[0].toUpperCase() + searchValue.slice(1),
          text: "",
          fromSearch: true,
        })
        activePopup.set("show")
        return
      }

      searchElem.select()
      let newIndex =
        ($activeShow?.index ?? $projects[$activeProject].shows.length - 1) + 1
      if ($activePage === "show")
        history({
          id: "UPDATE",
          newData: { key: "shows", index: newIndex, data: { id: match.id } },
          oldData: { id: $activeProject },
          location: { page: "show", id: "project_ref" },
        })
      activeShow.set({ ...match, index: newIndex })
      searchValue = ""
    } else if (e.key === "Escape") {
      // No need to handle searchActive since search is always visible
    }
  }

  $: if ($activeShow?.type === undefined || $activeShow?.type === "show")
    previousShow.set(JSON.stringify($activeShow))

  $: tabs = keysToID(drawerTabs)

  const hiddenInFocusMode = ["templates", "audio", "overlays", "calendar"]
</script>

<svelte:window
  on:mouseup={mouseup}
  on:mousemove={mousemove}
  on:keydown={keydown}
/>

<!-- <Resizeable id="drawer" side="bottom" minWidth={50}> -->
<section class="drawer" style="height: {height}px">
  <div
    class="top context #drawer_top"
    on:mousedown={mousedown}
    on:click={click}
    on:keydown={() => {
      // does not work and prevents search input keys
      // if (e.key === "Enter" || e.key === " ") {
      //     e.preventDefault()
      //     click(e)
      // }
    }}
  >
    <!-- role="button"
        tabindex="0" -->
    <input
      bind:this={searchElem}
      class="search edit"
      type="text"
      placeholder="{$dictionary.main?.search}..."
      bind:value={searchValue}
      on:input={search}
      use:selectTextOnFocus
    />

    <span class="tabs">
      {#each tabs as tab, i}
        {#if $drawerTabsData[tab.id]?.enabled !== false && getAccess(tab.id).global !== "none" && (!$focusMode || !hiddenInFocusMode.includes(tab.id))}
          <Button
            id={tab.id}
            on:click={() => openDrawerTab(tab)}
            on:dblclick={closeDrawer}
            active={$activeDrawerTab === tab.id}
            class="context #drawer_top"
            title="{$dictionary[tab.name.split('.')[0]]?.[
              tab.name.split('.')[1]
            ]} [Ctrl+{i + 1}]"
          >
            <Icon
              id={tab.icon}
              size={1.3}
              white={$activeDrawerTab === tab.id}
            />
            {#if !$labelsDisabled && !$focusMode}
              <span><T id={tab.name} /></span>
            {/if}
          </Button>
        {/if}
      {/each}
    </span>
  </div>

  <div class="content">
    <Resizeable id="leftPanelDrawer">
      <Navigation id={$activeDrawerTab} />
    </Resizeable>
    <Content
      id={$activeDrawerTab}
      bind:searchValue
      bind:firstMatch
      bind:bibles
    />
    <Resizeable id="rightPanelDrawer" let:width side="right">
      <div class="right" class:row={width > DEFAULT_WIDTH * 1.8}>
        <Info id={$activeDrawerTab} {bibles} />
      </div>
    </Resizeable>
  </div>
</section>

<style>
  section {
    display: flex;
    flex-direction: column;
    width: 100%;
    height: 100px;
    z-index: 20;

    background-color: var(--primary);

    /* box-shadow: 0 -2px 14px rgb(0 0 0 / 0.12); */
  }

  .top {
    position: relative;
    height: 40px;
    display: flex;
    justify-content: flex-start;
    padding-top: 4px;

    box-shadow: 0 -1.8px 8px rgb(0 0 0 / 0.2);
    z-index: 1;
  }
  .top::after {
    content: "";
    background-color: var(--primary-lighter);
    position: absolute;
    top: 0;
    width: 100%;
    height: 4px;
    cursor: ns-resize;
  }

  .top .tabs {
    display: flex;
    overflow-x: auto;
    overflow-y: hidden;
    margin-left: 0;
  }
  .top .tabs span {
    margin-inline-start: 0.5em;
  }

  .search {
    background-color: rgb(0 0 0 / 0.2);
    color: var(--text);
    font-size: inherit;
    font-family: inherit;
    width: var(--navigation-width);
    min-width: var(--navigation-width);
    /* width: 50%; */
    padding: 0 8px;
    border: none;
    border-inline-start: 4px solid var(--primary-darker);
    margin-right: 10px;
  }
  .search:active,
  .search:focus {
    outline: 2px solid var(--secondary);
    outline-offset: -2px;
  }
  .search::placeholder {
    color: inherit;
    opacity: 0.4;
  }

  .hidden {
    display: none;
  }

  .content {
    display: flex;
    height: calc(100% - 40px);
    justify-content: space-between;
  }

  .right {
    display: contents;
  }
  .right.row :global(.scroll.split) {
    flex-direction: row-reverse;
  }
  .right.row :global(.scroll.split .border) {
    border-inline-end: 2px solid var(--primary-lighter);
    overflow-y: auto;
  }
  /* scripture preview */
  .right.row :global(.scroll.split .zoomed) {
    flex: 1;
  }

  @media screen and (max-width: 750px) {
    .tabs span {
      display: none;
    }
  }
</style>
