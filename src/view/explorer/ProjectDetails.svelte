<script lang="ts">
  import { last } from "lodash";
  import { normalizePath } from "obsidian";
  import { draftForPath, projectFolderPath } from "src/model/scene-navigation";
  import { pluginSettings, projects } from "src/model/stores";
  import {
    drafts,
    selectedDraft,
    selectedDraftVaultPath,
  } from "src/model/stores";
  import { onMount } from "svelte";
  import { FileSuggest } from "../settings/file-suggest";
  import { FolderSuggest } from "../settings/folder-suggest";
  import {
    selectedDraftWordCountStatus,
    goalProgress,
    activeFile,
  } from "../stores";
  import ActivityHeatmap from "./ActivityHeatmap.svelte";
  import { useApp } from "../utils";

  const app = useApp();

  function titleChanged(event: Event) {
    let newTitle = (event.target as any).value;
    drafts.update((_drafts) => {
      const currentDraftIndex = _drafts.findIndex(
        (d) => d.vaultPath === $selectedDraftVaultPath
      );
      if (currentDraftIndex >= 0) {
        const currentDraft = _drafts[currentDraftIndex];
        const currentTitle = currentDraft.title;
        let titleInFrontmatter = true;

        if (newTitle.length === 0) {
          newTitle = last(
            _drafts[currentDraftIndex].vaultPath.split("/")
          ).split(".md")[0];
          titleInFrontmatter = false;
        }

        return _drafts.map((d) => {
          if (d.title === currentTitle) {
            d.title = newTitle;
            d.titleInFrontmatter = titleInFrontmatter;
          }
          return d;
        });
      }
      return _drafts;
    });
  }

  let sceneFolderInput: HTMLInputElement;
  onMount(() => {
    if (sceneFolderInput && $selectedDraft.format === "scenes") {
      const projectPath = projectFolderPath($selectedDraft, app.vault);
      new FolderSuggest(app, sceneFolderInput, projectPath);
    }
  });

  async function sceneFolderChanged(event: Event) {
    const newFolder = (event.target as any).value;
    if (newFolder.length <= 0 || !$selectedDraft) {
      return;
    }
    const root = app.vault.getAbstractFileByPath($selectedDraft.vaultPath)
      .parent.path;
    const path = normalizePath(`${root}/${newFolder}`);
    const exists = await app.vault.adapter.exists(path);
    if (exists) {
      drafts.update((allDrafts) =>
        allDrafts.map((d) => {
          if (
            d.vaultPath === $selectedDraftVaultPath &&
            d.format === "scenes"
          ) {
            d.sceneFolder = newFolder;
          }
          return d;
        })
      );
    }
  }

  let sceneTemplateInput: HTMLInputElement;
  onMount(() => {
    if (sceneTemplateInput && $selectedDraft.format === "scenes") {
      new FileSuggest(app, sceneTemplateInput);
    }
  });
  async function sceneTemplateChanged(event: Event) {
    let newTemplate = (event.target as any).value;
    if (!$selectedDraft) {
      return;
    }
    let exists = true;
    if (newTemplate.length <= 0) {
      newTemplate = null;
    } else {
      exists = await app.vault.adapter.exists(newTemplate);
    }

    if (exists) {
      drafts.update((allDrafts) =>
        allDrafts.map((d) => {
          if (
            d.vaultPath === $selectedDraftVaultPath &&
            d.format === "scenes"
          ) {
            d.sceneTemplate = newTemplate;
          }
          return d;
        })
      );
    }
  }

  let projectCount: number;
  let draftCount: number | null;
  let sceneCount: number | null;
  $: {
    if ($selectedDraftWordCountStatus) {
      const { scene, draft, project } = $selectedDraftWordCountStatus;

      projectCount = project;
      draftCount = $projects[$selectedDraft.title].length > 1 ? draft : null;
      sceneCount = $selectedDraft.format === "scenes" ? scene : null;
    }
  }

  let showProgress = false;
  $: {
    if ($activeFile && $selectedDraft) {
      const draft = draftForPath($activeFile.path, $drafts);
      showProgress = draft && draft.vaultPath === $selectedDraft.vaultPath;
    }
  }

  let goalPercentage: number;
  let goalDescription: string;
  $: {
    goalPercentage = Math.ceil(Math.min($goalProgress, 1) * 100);
    goalDescription = `${Math.round(
      $goalProgress * $pluginSettings.sessionGoal
    )}/${$pluginSettings.sessionGoal}`;
  }

  function formatCount(count: number | null | undefined): string {
    if (count === undefined || count === null) {
      return "0";
    }
    return count.toLocaleString();
  }

</script>

<div class="longform-project">
  {#if $selectedDraft}
    <div class="longform-project-section">
      <div class="field">
        <label for="longform-project-title">Title</label>
        <input
          id="longform-project-title"
          type="text"
          value={$selectedDraft.title}
          on:change={titleChanged}
        />
      </div>
      {#if $selectedDraft.format === "scenes"}
        <div class="field">
          <label for="longform-project-scene-folder">Scene Folder</label>
          <input
            id="longform-project-scene-folder"
            type="text"
            value={$selectedDraft.sceneFolder}
            bind:this={sceneFolderInput}
            on:blur={sceneFolderChanged}
          />
          <p class="longform-project-warning">
            Changing scene folder does not move scenes. If you’re moving
            scenes to a new folder, move them in your vault first, then
            change this setting.
          </p>
        </div>
        <div class="field">
          <label for="longform-project-scene-template">Scene Template</label>
          <input
            id="longform-project-scene-template"
            type="text"
            value={$selectedDraft.sceneTemplate}
            bind:this={sceneTemplateInput}
            on:blur={sceneTemplateChanged}
          />
          <p class="longform-project-warning">
            This file will be used as a template when creating new scenes
            via the New scene… field. If you use a templating plugin
            (Templater or the core plugin) it will be used to process this
            template.
          </p>
        </div>
      {/if}
    </div>
  {/if}
  <div
    class="longform-project-section word-counts"
    style={`--progress-text-color:${
      goalPercentage >= 43 ? "var(--text-on-accent)" : "var(--text-accent)"
    }`}
  >
    {#if showProgress}
      <div
        class="progress"
        data-label={goalDescription}
        title={goalDescription}
      >
        <div class="value" style={`width:${goalPercentage}%;`} />
      </div>
    {/if}
    <div class="word-count-stats">
      {#if sceneCount !== null}
        <div
          class="word-count-stat"
          title="Word count in this scene of this project."
        >
          <span class="word-count-value">{formatCount(sceneCount)}</span>
          <span class="word-count-label">scene</span>
        </div>
      {/if}
      {#if draftCount !== null}
        <div
          class="word-count-stat"
          title="Word count in just this draft of this project."
        >
          <span class="word-count-value">{formatCount(draftCount)}</span>
          <span class="word-count-label">draft</span>
        </div>
      {/if}
      <div
        class="word-count-stat"
        title="Word count across all drafts of this project."
      >
        <span class="word-count-value">{formatCount(projectCount)}</span>
        <span class="word-count-label">project</span>
      </div>
    </div>
  </div>
  <div class="longform-project-section heatmap-section">
    <ActivityHeatmap />
  </div>
</div>

<style>
  .longform-project {
    display: flex;
    flex-direction: column;
    gap: var(--size-4-8);
    padding-top: var(--size-4-3);
    padding-bottom: var(--size-4-4);
  }

  .longform-project-section {
    display: flex;
    flex-direction: column;
    gap: var(--size-4-5);
  }

  .field {
    display: flex;
    flex-direction: column;
    gap: var(--size-4-1);
  }

  input {
    width: 100%;
  }

  label {
    display: block;
    font-size: var(--font-ui-smaller);
    color: var(--text-muted);
    line-height: var(--line-height-tight);
  }

  p.longform-project-warning {
    color: var(--text-faint);
    font-size: var(--font-smallest);
    margin: 0;
    line-height: normal;
  }

  .word-counts {
    gap: var(--size-4-3);
  }

  .word-count-stats {
    display: flex;
    flex-wrap: wrap;
    gap: var(--size-4-6);
  }

  .word-count-stat {
    display: flex;
    flex-direction: column;
    gap: 1px;
    min-width: 4em;
  }

  .word-count-value {
    font-size: var(--font-ui-medium);
    color: var(--text-normal);
    font-variant-numeric: tabular-nums;
    line-height: var(--line-height-tight);
  }

  .word-count-label {
    font-size: var(--font-smallest);
    color: var(--text-faint);
  }

  .progress {
    height: var(--size-4-6);
    width: 100%;
    background-color: var(--background-secondary-alt);
    border-radius: var(--radius-s);
    position: relative;
    overflow: hidden;
  }

  .progress:before {
    content: attr(data-label);
    font-size: var(--font-smallest);
    color: var(--progress-text-color);
    font-weight: bold;
    position: absolute;
    text-align: center;
    top: 0;
    left: 0;
    right: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    align-self: center;
    height: 100%;
  }

  .progress .value {
    height: 100%;
    background-color: var(--text-accent);
  }

  .heatmap-section {
    gap: 0;
  }
</style>
