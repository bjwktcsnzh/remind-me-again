<script lang="ts">
  import { cubicOut } from 'svelte/easing'
  import { crossfade } from 'svelte/transition'
  import { flip } from 'svelte/animate'
  import './app.css'
  import Item from './lib/Item.svelte'
  import New from './lib/New.svelte'
  import { checkShortcut, runCmd } from './lib/helpers'
  import type { Group } from './lib/types'
  import { onDestroy } from 'svelte'
  import { event, window as tauriWindow } from '@tauri-apps/api'

  let groupElements: Item[] = []
  let focusedGroup: number | null = null

  let groups: Group[] = []
  runCmd<Group[]>('get_groups').then((g) => {
    groups = g
    console.log(groups)
  })

  const [send, receive] = crossfade({
    fallback(node) {
      const style = getComputedStyle(node)
      const transform = style.transform === 'none' ? '' : style.transform

      return {
        duration: 400,
        easing: cubicOut,
        css: (t) => `
  				transform: ${transform} scale(${1 + (t - 1) * 0.2});
  				opacity: ${t}
  			`,
      }
    },
  })

  let creatorComponent: New
  const unlistenFuture = event.listen('tauri://menu', ({ payload }) => {
    console.log(payload, focusedGroup)
    if (payload === 'Edit Reminder' && focusedGroup !== null) {
      console.log('--w', groupElements[focusedGroup])
      groupElements[focusedGroup]?.edit()
    } else if (payload === 'New Reminder') {
      if (focusedGroup) {
        groupElements[focusedGroup].cancel()
      }
      creatorComponent.open()
    }
  })
  onDestroy(async () => {
    ;(await unlistenFuture)()
  })
</script>

<svelte:body
  on:keydown={async (e) => {
    if (checkShortcut(e, 'Escape')) {
      tauriWindow.appWindow.hide()
    }
  }}
/>

<div class="flex min-h-screen w-full flex-col overflow-y-scroll px-4 pb-2">
  <h1 class="mb-2 cursor-default select-none text-center text-2xl font-normal text-white">
    Reminders
  </h1>
  {#if groups}
    <div class="relative select-none outline-none">
      <New
        bind:this={creatorComponent}
        onCreate={(newGroups) => {
          groups = newGroups
        }}
      />
      {#each groups as group, i (group.id)}
        <div
          in:receive={{ key: group.id, duration: 400 }}
          out:send={{ key: group.id, duration: 400 }}
          animate:flip={{ duration: 400 }}
          on:focusin={() => {
            focusedGroup = i
          }}
          on:focusout={() => {
            focusedGroup = null
          }}
          on:keydown={async (e) => {
            if (checkShortcut(e, 'ArrowUp')) {
              groupElements[i - 1]?.focus()
              e.preventDefault()
            } else if (checkShortcut(e, 'ArrowDown')) {
              groupElements[i + 1]?.focus()
              e.preventDefault()
            }
          }}
        >
          <Item
            bind:this={groupElements[i]}
            bind:group
            onUpdate={async () => {
              groups = await runCmd('update_group', { group })
            }}
            onDelete={async () => {
              groups = await runCmd('delete_group', { index: i })
            }}
          />
        </div>
      {/each}
    </div>
  {/if}
</div>

<style lang="sass">
  :global(body)
    font-family: -apple-system, BlinkMacSystemFont, Segoe UI, Helvetica, Arial, sans-serif, Apple Color Emoji, Segoe UI Emoji
    font-size: 18px
    color: #ffffff
    background-color: rgb(8, 9, 13) // also used in main.rs
</style>
