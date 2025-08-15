<script lang="ts">
  import { activePopup, popupData } from "../../../stores"
  import Icon from "../../helpers/Icon.svelte"
  import T from "../../helpers/T.svelte"
  import Button from "../../inputs/Button.svelte"
  import CombinedInput from "../../inputs/CombinedInput.svelte"
  import TextInput from "../../inputs/TextInput.svelte"

  let liveSetId = ""

  function keydown(e: KeyboardEvent) {
    if (e.key === "Enter") join()
  }

  function close() {
    popupData.set({})
    activePopup.set(null)
  }

  function join() {
    if (!liveSetId.trim()) return

    popupData.set({ ...$popupData, id: "join_set", value: liveSetId.trim() })
    activePopup.set(null)
  }
</script>

<svelte:window on:keydown={keydown} />

<p>Enter the Live Set ID to join:</p>

<TextInput
  bind:value={liveSetId}
  placeholder="Live Set ID"
  autofocus
  style="width: 100%; margin: 20px 0;"
/>

<CombinedInput style="margin-top: 20px;">
  <Button style="flex: 1;" on:click={close} center dark>
    <Icon id="close" size={1.1} right />
    <T id="join.cancel" />
  </Button>
  <Button on:click={join} center dark disabled={!liveSetId.trim()}>
    <Icon id="check" size={1.1} right />
    <T id="join.join" />
  </Button>
</CombinedInput>

<style>
  p {
    white-space: initial;
    margin-bottom: 10px;
    text-align: center;
  }
</style>
