<!--
// Copyright © 2023 Hardcore Engineering Inc.
//
// Licensed under the Eclipse Public License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License. You may
// obtain a copy of the License at https://www.eclipse.org/legal/epl-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
//
// See the License for the specific language governing permissions and
// limitations under the License.
-->
<script lang="ts">
  import { Asset, Metadata } from '@hcengineering/platform'
  import { Button, EmojiPopup, TabsControl, getPlatformColor, getPlatformColorDef, themeStore } from '@hcengineering/ui'
  import { createEventDispatcher } from 'svelte'
  import view from '../plugin'
  import ColorsPopup from './ColorsPopup.svelte'
  import PopupDialog from './PopupDialog.svelte'

  export let icon: Metadata<string> | undefined = undefined
  export let icons: Asset[]
  export let iconWithEmoji: Asset = view.ids.IconWithEmoji
  export let color: number = 0
  export let showColor: boolean = true

  const dispatch = createEventDispatcher()

  let _color = color
  let _icon = icon ?? icons[0]
</script>

<PopupDialog label={view.string.ChooseIcon} on:changeContent>
  <div class="float-left-box w-85">
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <TabsControl size={'small'} model={[{ label: view.string.IconCategory }, { label: view.string.EmojiCategory }]}>
      <svelte:fragment slot="content" let:selected>
        {#if selected === 0}
          <div class="flex-col mt-2">
            {#if icons.length > 0}
              <div class="flex">
                {#each icons as obj}
                  <div class="float-left">
                    <Button
                      icon={obj}
                      iconProps={showColor ? { fill: getPlatformColor(_color ?? 0, $themeStore.dark) } : undefined}
                      size="medium"
                      kind={obj === _icon ? 'primary' : 'ghost'}
                      on:click={() => {
                        _icon = obj
                        dispatch(!showColor ? 'close' : 'update', { icon: _icon, color: _color })
                      }}
                    />
                  </div>
                {/each}
              </div>
            {/if}
            {#if showColor}
              <ColorsPopup
                selected={getPlatformColorDef(_color, $themeStore.dark).name}
                embedded
                on:close={(evt) => {
                  _color = evt.detail
                  dispatch(icons.length === 0 ? 'close' : 'update', {
                    icon: icons.length === 0 ? null : _icon,
                    color: _color
                  })
                }}
              />
            {/if}
          </div>
        {:else}
          <EmojiPopup
            embedded
            selected={String.fromCodePoint(color ?? 0)}
            on:close={(evt) => {
              dispatch('close', { icon: iconWithEmoji, color: evt.detail.codePointAt(0) })
            }}
          />
        {/if}
      </svelte:fragment>
    </TabsControl>
  </div>
</PopupDialog>

<style lang="scss">
  .color {
    position: relative;
    width: 1.75rem;
    height: 1.75rem;
    background-color: var(--accent-bg-color);
    border: 1px solid transparent;
    border-radius: 0.25rem;
    cursor: pointer;

    .dot {
      position: absolute;
      content: '';
      border-radius: 50%;
      inset: 30%;
    }
  }
</style>
