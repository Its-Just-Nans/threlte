<script lang="ts">
  import { useRender } from '@threlte/core'
  import { interactivity, transitions } from '@threlte/extras'
  import { Box } from '@threlte/flex'
  import { tick } from 'svelte'
  import Button from './Button.svelte'
  import Label from './Label.svelte'
  import Matcap from './Matcap.svelte'
  import Window from './Window.svelte'

  export let windowWidth: number
  export let windowHeight: number
  export let rows = 5
  export let columns = 5
  export let size: any

  let page = 1
  $: offset = (page - 1) * rows * columns

  interactivity()
  transitions()

  useRender(async ({ camera, scene, renderer }) => {
    await tick()
    renderer.render(scene, camera.current)
  })
</script>

<Window
  title="Matcaps"
  width={windowWidth}
  height={windowHeight}
>
  <Box class="h-full w-full flex-col items-stretch gap-10 p-10">
    {#each new Array(rows) as _, rowIndex}
      <Box class="h-auto w-full flex-1 items-center justify-evenly gap-10">
        {#each new Array(columns) as _, columnIndex}
          {@const index = rowIndex * columns + columnIndex}
          <Box
            class="h-full w-full flex-1"
            let:width
            let:height
          >
            <Matcap
              {width}
              {height}
              matcapIndex={offset + index}
              gridIndex={index}
              format={size}
            />
          </Box>
        {/each}
      </Box>
    {/each}

    <Box
      order={999}
      class="h-40 w-auto items-center justify-center gap-10"
    >
      <Button
        class="h-full w-auto flex-1"
        z={15}
        text="← PREVIOUS PAGE"
        order={0}
        on:click={() => {
          page = Math.max(1, page - 1)
        }}
      />

      <Box
        class="h-full w-auto flex-1"
        order={1}
      >
        <Label
          z={10.1}
          fontSize="xl"
          text={`PAGE: ${page}`}
        />
      </Box>

      <Button
        class="h-full w-auto flex-1"
        z={15}
        text="NEXT PAGE →"
        order={2}
        on:click={() => {
          page = Math.min(10, page + 1)
        }}
      />
    </Box>
  </Box>
</Window>
