<!--
// Copyright © 2022-2023 Hardcore Engineering Inc.
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
  import { Employee } from '@hcengineering/contact'
  import { AccountArrayEditor, AssigneeBox } from '@hcengineering/contact-resources'
  import core, { Account, Data, DocumentUpdate, Ref, generateId, getCurrentAccount } from '@hcengineering/core'
  import { Asset } from '@hcengineering/platform'
  import presentation, { Card, createQuery, getClient } from '@hcengineering/presentation'
  import task, { ProjectType } from '@hcengineering/task'
  import { StyledTextBox } from '@hcengineering/text-editor'
  import { IssueStatus, Project, TimeReportDayType } from '@hcengineering/tracker'
  import {
    Button,
    Component,
    EditBox,
    IconEdit,
    IconWithEmoji,
    Label,
    Toggle,
    eventToHTMLElement,
    getColorNumberByText,
    getPlatformColorDef,
    getPlatformColorForTextDef,
    showPopup,
    themeStore
  } from '@hcengineering/ui'
  import view from '@hcengineering/view'
  import { IconPicker } from '@hcengineering/view-resources'
  import { createEventDispatcher } from 'svelte'
  import tracker from '../../plugin'
  import StatusSelector from '../issues/StatusSelector.svelte'
  import ChangeIdentity from './ChangeIdentity.svelte'
  import { typeStore, taskTypeStore } from '@hcengineering/task-resources'

  export let project: Project | undefined = undefined

  export let namePlaceholder: string = ''
  export let descriptionPlaceholder: string = ''

  const client = getClient()
  const hierarchy = client.getHierarchy()
  const projectsQuery = createQuery()

  let name: string = project?.name ?? namePlaceholder
  let description: string = project?.description ?? descriptionPlaceholder
  let isPrivate: boolean = project?.private ?? false
  let icon: Asset | undefined = project?.icon ?? undefined
  let color = project?.color ?? getColorNumberByText(name)
  let isColorSelected = false
  let defaultAssignee: Ref<Employee> | null | undefined = project?.defaultAssignee ?? null
  let members: Ref<Account>[] =
    project?.members !== undefined ? hierarchy.clone(project.members) : [getCurrentAccount()._id]
  let projectsIdentifiers = new Set<string>()
  let isSaving = false
  let defaultStatus: Ref<IssueStatus> | undefined = project?.defaultIssueStatus

  let changeIdentityRef: HTMLElement

  const dispatch = createEventDispatcher()

  $: isNew = project == null

  async function handleSave (): Promise<void> {
    if (isNew) {
      await createProject()
    } else {
      await updateProject()
    }
  }

  let identifier: string = project?.identifier ?? 'TSK'

  function getProjectData (): Omit<Data<Project>, 'type'> {
    return {
      name,
      description,
      private: isPrivate,
      members,
      archived: false,
      identifier: identifier.toUpperCase(),
      sequence: 0,
      defaultAssignee: defaultAssignee ?? undefined,
      icon,
      color,
      defaultIssueStatus: defaultStatus ?? ('' as Ref<IssueStatus>),
      defaultTimeReportDay: project?.defaultTimeReportDay ?? TimeReportDayType.PreviousWorkDay
    }
  }

  async function updateProject (): Promise<void> {
    if (!project) {
      return
    }

    const { sequence, ...projectData } = getProjectData()
    const update: DocumentUpdate<Project> = {}
    if (projectData.name !== project?.name) {
      update.name = projectData.name
    }
    if (projectData.description !== project?.description) {
      update.description = projectData.description
    }
    if (projectData.private !== project?.private) {
      update.private = projectData.private
    }
    if (projectData.defaultAssignee !== project?.defaultAssignee) {
      update.defaultAssignee = projectData.defaultAssignee
    }
    if (projectData.defaultIssueStatus !== project?.defaultIssueStatus) {
      update.defaultIssueStatus = projectData.defaultIssueStatus
    }
    if (projectData.icon !== project?.icon) {
      update.icon = projectData.icon
    }
    if (projectData.color !== project?.color) {
      update.color = projectData.color
    }
    if (projectData.defaultTimeReportDay !== project?.defaultTimeReportDay) {
      update.defaultTimeReportDay = projectData.defaultTimeReportDay
    }
    if (projectData.members.length !== project?.members.length) {
      update.members = projectData.members
    } else {
      for (const member of projectData.members) {
        if (project.members.findIndex((p) => p === member) === -1) {
          update.members = projectData.members
          break
        }
      }
    }

    if (Object.keys(update).length > 0) {
      isSaving = true
      await client.update(project, update)
      isSaving = false
    }

    close()
  }

  let typeId: Ref<ProjectType> | undefined = project?.type

  $: typeType = typeId !== undefined ? $typeStore.get(typeId) : undefined

  $: if (defaultStatus === undefined && typeType !== undefined) {
    defaultStatus = typeType.statuses[0]?._id
  }

  async function createProject (): Promise<void> {
    const projectId = generateId<Project>()
    const projectData = getProjectData()
    if (typeId !== undefined && typeType !== undefined) {
      const ops = client
        .apply(projectId)
        .notMatch(tracker.class.Project, { identifier: projectData.identifier.toUpperCase() })

      isSaving = true
      await ops.createDoc(tracker.class.Project, core.space.Space, { ...projectData, type: typeId }, projectId)
      const succeeded = await ops.commit()

      if (succeeded) {
        // Add vacancy mixin
        await client.createMixin(projectId, tracker.class.Project, core.space.Space, typeType.targetClass, {})

        close(projectId)
      } else {
        isSaving = false
      }
    }
  }

  function chooseIcon (ev: MouseEvent): void {
    const icons = [tracker.icon.Home, tracker.icon.RedCircle]
    const update = (result: any) => {
      if (result !== undefined && result !== null) {
        icon = result.icon
        color = result.color
        isColorSelected = true
      }
    }
    showPopup(IconPicker, { icon, color, icons }, 'top', update, update)
  }

  function close (id?: Ref<Project>): void {
    dispatch('close', id)
  }

  $: projectsQuery.query(tracker.class.Project, { _id: { $nin: project ? [project._id] : [] } }, (res) => {
    projectsIdentifiers = new Set(res.map(({ identifier }) => identifier))
  })

  function handleTypeChange (evt: CustomEvent<Ref<ProjectType>>): void {
    typeId = evt.detail
    defaultStatus = undefined
  }

  $: identifier = identifier.toLocaleUpperCase().replaceAll('-', '_').replaceAll(' ', '_').substring(0, 5)
</script>

<Card
  label={isNew ? tracker.string.NewProject : tracker.string.EditProject}
  okLabel={isNew ? presentation.string.Create : presentation.string.Save}
  okAction={handleSave}
  canSave={name.length > 0 &&
    identifier.length > 0 &&
    !projectsIdentifiers.has(identifier.toUpperCase()) &&
    !(members.length === 0 && isPrivate)}
  accentHeader
  width={'medium'}
  gap={'gapV-6'}
  onCancel={close}
  on:changeContent
>
  <div class="antiGrid">
    <div class="antiGrid-row">
      <div class="antiGrid-row__header">
        <Label label={task.string.ProjectType} />
      </div>

      <Component
        is={task.component.ProjectTypeSelector}
        disabled={!isNew}
        props={{
          descriptors: [tracker.descriptors.ProjectType],
          type: typeId,
          focusIndex: 4,
          kind: 'regular',
          size: 'large'
        }}
        on:change={handleTypeChange}
      />
    </div>
    <div class="antiGrid-row">
      <div class="antiGrid-row__header">
        <Label label={tracker.string.ProjectTitle} />
      </div>
      <div class="padding">
        <EditBox
          bind:value={name}
          placeholder={tracker.string.ProjectTitlePlaceholder}
          kind={'large-style'}
          autoFocus
          on:input={() => {
            if (isNew) {
              identifier = name.toLocaleUpperCase().replaceAll('-', '_').replaceAll(' ', '_').substring(0, 5)
              color = isColorSelected ? color : getColorNumberByText(name)
            }
          }}
        />
      </div>
    </div>

    <div class="antiGrid-row">
      <div class="antiGrid-row__header withDesciption">
        <Label label={tracker.string.Identifier} />
        <span><Label label={tracker.string.UsedInIssueIDs} /></span>
      </div>
      <div bind:this={changeIdentityRef} class="padding flex-row-center relative">
        <EditBox
          bind:value={identifier}
          disabled={!isNew}
          placeholder={tracker.string.ProjectIdentifierPlaceholder}
          kind={'large-style'}
          uppercase
        />
        {#if !isSaving && projectsIdentifiers.has(identifier.toUpperCase())}
          <div class="absolute overflow-label duplicated-identifier">
            <Label label={tracker.string.IdentifierExists} />
          </div>
        {/if}
      </div>
    </div>

    <div class="antiGrid-row">
      <div class="antiGrid-row__header topAlign">
        <Label label={tracker.string.Description} />
      </div>
      <div class="padding clear-mins">
        <StyledTextBox
          alwaysEdit
          showButtons={false}
          bind:content={description}
          placeholder={tracker.string.IssueDescriptionPlaceholder}
        />
      </div>
    </div>
  </div>

  <div class="antiGrid">
    <div class="antiGrid-row">
      <div class="antiGrid-row__header">
        <Label label={tracker.string.ChooseIcon} />
      </div>
      <Button
        icon={icon === view.ids.IconWithEmoji ? IconWithEmoji : icon ?? tracker.icon.Home}
        iconProps={icon === view.ids.IconWithEmoji
          ? { icon: color }
          : {
              fill:
                color !== undefined
                  ? getPlatformColorDef(color, $themeStore.dark).icon
                  : getPlatformColorForTextDef(name, $themeStore.dark).icon
            }}
        size={'large'}
        on:click={chooseIcon}
      />
    </div>

    <div class="antiGrid-row">
      <div class="antiGrid-row__header withDesciption">
        <Label label={presentation.string.MakePrivate} />
        <span><Label label={presentation.string.MakePrivateDescription} /></span>
      </div>
      <Toggle bind:on={isPrivate} disabled={!isPrivate && members.length === 0} />
    </div>

    <div class="antiGrid-row">
      <div class="antiGrid-row__header">
        <Label label={tracker.string.Members} />
      </div>
      <AccountArrayEditor
        value={members}
        label={tracker.string.Members}
        onChange={(refs) => (members = refs)}
        kind={'regular'}
        size={'large'}
      />
    </div>

    <div class="antiGrid-row">
      <div class="antiGrid-row__header">
        <Label label={tracker.string.DefaultAssignee} />
      </div>
      <AssigneeBox
        label={tracker.string.Assignee}
        placeholder={tracker.string.Assignee}
        kind={'regular'}
        size={'large'}
        avatarSize={'card'}
        bind:value={defaultAssignee}
        titleDeselect={tracker.string.Unassigned}
        showNavigate={false}
        showTooltip={{ label: tracker.string.DefaultAssignee }}
      />
    </div>
    <div class="antiGrid-row">
      <div class="antiGrid-row__header">
        <Label label={tracker.string.DefaultIssueStatus} />
      </div>
      <StatusSelector
        taskType={Array.from($taskTypeStore.values()).filter(
          (it) => it.parent === typeId && it.ofClass === tracker.class.Issue
        )[0]?._id}
        bind:value={defaultStatus}
        type={typeId}
        kind={'regular'}
        size={'large'}
      />
    </div>
  </div>
</Card>

<style lang="scss">
  .duplicated-identifier {
    left: 0;
    bottom: -0.25rem;
    color: var(--theme-warning-color);
  }
</style>
