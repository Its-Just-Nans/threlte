<!--
@component `<Controller />` represents a THREE.XRTargetRaySpace, a THREE.XRGripSpace, and a controller model.
-->
<script
  lang="ts"
  context="module"
>
  import type { XRTargetRaySpace } from 'three'
  import { XRControllerModelFactory } from 'three/examples/jsm/webxr/XRControllerModelFactory'
  import { onDestroy } from 'svelte'
  import { T, createRawEventDispatcher, useThrelte } from '@threlte/core'
  import { gaze, left as leftStore, right as rightStore } from '../hooks/useController'
  import { isHandTracking, pointerState, teleportState } from '../internal/stores'
  import { useHandTrackingState } from '../internal/useHandTrackingState'
  import type { XRController, XRControllerEvent } from '../types'
  import PointerCursor from './internal/PointerCursor.svelte'
  import ShortRay from './internal/ShortRay.svelte'
  import ScenePortal from './internal/ScenePortal.svelte'
  import TeleportCursor from './internal/TeleportCursor.svelte'
  import TeleportRay from './internal/TeleportRay.svelte'

  const factory = new XRControllerModelFactory()

  const stores = {
    left: leftStore,
    right: rightStore,
    none: gaze
  } as const

  const events = [
    'select',
    'selectstart',
    'selectend',
    'squeeze',
    'squeezeend',
    'squeezestart'
  ] as const

  const eventMap = new WeakMap<XRTargetRaySpace, Omit<XRController, 'inputSource'>>()
</script>

<script lang="ts">
  type $$Props =
    | {
        /** Whether the controller should be matched with the left hand. */
        left: true
        right?: undefined
        hand?: undefined
      }
    | {
        /** Whether the controller should be matched with the right hand. */
        right: true
        left?: undefined
        hand?: undefined
      }
    | {
        /** Whether the controller should be matched with the left or right hand. */
        hand: 'left' | 'right'
        left?: undefined
        right?: undefined
      }

  export let left: $$Props['left'] = undefined
  export let right: $$Props['right'] = undefined
  export let hand: $$Props['hand'] = undefined

  type $$Events = {
    connected: XRControllerEvent<'connected'>
    disconnected: XRControllerEvent<'disconnected'>
    select: XRControllerEvent<'select'>
    selectstart: XRControllerEvent<'selectstart'>
    selectend: XRControllerEvent<'selectend'>
    squeeze: XRControllerEvent<'squeeze'>
    squeezeend: XRControllerEvent<'squeezeend'>
    squeezestart: XRControllerEvent<'squeezestart'>
  }

  $: handedness = (left ? 'left' : right ? 'right' : hand) as 'left' | 'right'

  const dispatch = createRawEventDispatcher<$$Events>()
  const { xr } = useThrelte().renderer
  const handTrackingNow = useHandTrackingState()

  const handleEvent = (event: XRControllerEvent) => {
    if (!handTrackingNow()) {
      dispatch(event.type, event)
    }
  }

  const handleConnected = (event: XRControllerEvent<'connected'>) => {
    const targetData = eventMap.get(event.target)

    if (event.data.handedness !== handedness || !targetData) return

    stores[handedness].set({ ...targetData, inputSource: event.data })

    if (!handTrackingNow()) {
      dispatch('connected', event)
    }

    events.forEach((name) => event.target.addEventListener(name, handleEvent))
  }

  const handleDisconnected = (event: XRControllerEvent<'disconnected'>) => {
    if (event.data.handedness !== handedness) return

    stores[handedness].set(undefined)

    if (!$isHandTracking) {
      dispatch('disconnected', event)
    }

    events.forEach((name) => event.target.removeEventListener(name, handleEvent))
  }

  for (const index of [0, 1]) {
    const targetRay = xr.getController(index)
    const grip = xr.getControllerGrip(index)

    // "createControllerModel" currently only will attach a model if the "connected" event is fired,
    // so it must be called immediately before a controller connects.
    const model = factory.createControllerModel(grip)

    eventMap.set(targetRay, { targetRay, model, grip })

    /**
     * @todo(mp) event.data is missing from @three/types. Need to make a PR there.
     */
    targetRay.addEventListener('connected', handleConnected as any)
    targetRay.addEventListener('disconnected', handleDisconnected as any)
  }

  $: store = stores[handedness]
  $: grip = $store?.grip
  $: targetRay = $store?.targetRay
  $: model = $store?.model
  $: hasPointerControls = $pointerState[handedness].enabled
  $: hasTeleportControls = $teleportState[handedness].enabled

  onDestroy(() => {
    for (const index of [0, 1]) {
      const controller = xr.getController(index)
      controller.removeEventListener('connected', handleConnected as any)
      controller.removeEventListener('disconnected', handleDisconnected as any)
    }

    const controller = $store?.targetRay
    events.forEach((name) => controller?.removeEventListener(name, handleEvent as any))

    store.set(undefined)
  })
</script>

{#if !$isHandTracking}
  {#if grip}
    <T
      is={grip}
      name="XR controller grip {handedness}"
    >
      <slot>
        <T is={model} />
      </slot>

      <slot name="grip" />
    </T>
  {/if}

  {#if targetRay}
    <T
      is={targetRay}
      name="XR controller {handedness}"
    >
      <slot name="target-ray" />

      {#if hasPointerControls || hasTeleportControls}
        {#if $$slots['pointer-ray']}
          <ShortRay {handedness}>
            <slot name="pointer-ray" />
          </ShortRay>
        {:else}
          <ShortRay {handedness} />
        {/if}
      {/if}
    </T>
  {/if}
{/if}

<ScenePortal>
  {#if hasPointerControls}
    {#if $$slots['pointer-cursor']}
      <PointerCursor {handedness}>
        <slot name="pointer-cursor" />
      </PointerCursor>
    {:else}
      <PointerCursor {handedness} />
    {/if}
  {/if}

  {#if hasTeleportControls && targetRay !== undefined}
    {#if $$slots['teleport-ray']}
      <TeleportRay
        {targetRay}
        {handedness}
      >
        <slot name="teleport-ray" />
      </TeleportRay>
    {:else}
      <TeleportRay
        {targetRay}
        {handedness}
      />
    {/if}

    {#if $$slots['teleport-ray']}
      <TeleportCursor {handedness}>
        <slot name="teleport-cursor" />
      </TeleportCursor>
    {:else}
      <TeleportCursor {handedness} />
    {/if}
  {/if}
</ScenePortal>
