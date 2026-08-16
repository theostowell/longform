<script lang="ts">
  import { sessions, selectedProject } from "src/model/stores";
  import type { WordCountSession } from "src/model/types";

  // Number of weeks to render (roughly 3 months).
  const WEEKS = 13;
  const DAYS_PER_WEEK = 7;

  const MONTH_NAMES = [
    "Jan",
    "Feb",
    "Mar",
    "Apr",
    "May",
    "Jun",
    "Jul",
    "Aug",
    "Sep",
    "Oct",
    "Nov",
    "Dec",
  ];

  // GitHub-style: only label a few weekdays down the left gutter.
  const WEEKDAY_LABELS = ["", "Mon", "", "Wed", "", "Fri", ""];

  type DayCell = {
    date: Date;
    key: string;
    words: number;
    level: number;
    future: boolean;
  };
  type Week = DayCell[];

  function pad(value: number): string {
    return value < 10 ? `0${value}` : `${value}`;
  }

  function dateKey(date: Date): string {
    return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(
      date.getDate()
    )}`;
  }

  function startOfDay(date: Date): Date {
    const copy = new Date(date);
    copy.setHours(0, 0, 0, 0);
    return copy;
  }

  function formatDate(date: Date): string {
    return `${MONTH_NAMES[date.getMonth()]} ${date.getDate()}, ${date.getFullYear()}`;
  }

  // Draft vault paths that make up the currently selected project.
  $: projectPaths = $selectedProject
    ? $selectedProject.map((d) => d.vaultPath)
    : [];

  // Map of "YYYY-MM-DD" -> words written in this project that day.
  $: wordsByDay = aggregateWordsByDay($sessions, projectPaths);

  function aggregateWordsByDay(
    allSessions: WordCountSession[],
    paths: string[]
  ): Map<string, number> {
    const map = new Map<string, number>();
    for (const session of allSessions) {
      if (!session || !session.start) {
        continue;
      }
      const start =
        session.start instanceof Date ? session.start : new Date(session.start);

      let words = 0;
      if (paths.length === 0) {
        words = session.total;
      } else {
        for (const path of paths) {
          const draftStats = session.drafts && session.drafts[path];
          words += (draftStats && draftStats.total) || 0;
        }
      }

      if (words > 0) {
        const key = dateKey(start);
        map.set(key, (map.get(key) || 0) + words);
      }
    }
    return map;
  }

  // 0 (none) to 4 (most), scaled against the heaviest day in the window.
  function intensity(words: number, max: number): number {
    if (words <= 0 || max <= 0) {
      return 0;
    }
    const ratio = words / max;
    if (ratio < 0.25) return 1;
    if (ratio < 0.5) return 2;
    if (ratio < 0.75) return 3;
    return 4;
  }

  let weeks: Week[] = [];
  let monthLabels: (string | null)[] = [];
  let totalWords = 0;
  let activeDays = 0;

  $: {
    const today = startOfDay(new Date());

    // Sunday of the current week.
    const currentWeekStart = new Date(today);
    currentWeekStart.setDate(currentWeekStart.getDate() - currentWeekStart.getDay());

    // Sunday of the first week in the window.
    const gridStart = new Date(currentWeekStart);
    gridStart.setDate(gridStart.getDate() - (WEEKS - 1) * DAYS_PER_WEEK);

    // Stats across the whole window (used for scaling + footer).
    let max = 0;
    let total = 0;
    let days = 0;
    wordsByDay.forEach((value) => {
      if (value > max) max = value;
      total += value;
      days += 1;
    });
    totalWords = total;
    activeDays = days;

    const builtWeeks: Week[] = [];
    const labels: (string | null)[] = [];
    let lastMonth = -1;
    const cursor = new Date(gridStart);

    for (let w = 0; w < WEEKS; w++) {
      const week: DayCell[] = [];
      for (let d = 0; d < DAYS_PER_WEEK; d++) {
        const date = new Date(cursor);
        const future = date.getTime() > today.getTime();
        const key = dateKey(date);
        const words = future ? 0 : wordsByDay.get(key) || 0;
        week.push({
          date,
          key,
          words,
          level: intensity(words, max),
          future,
        });
        cursor.setDate(cursor.getDate() + 1);
      }

      const month = week[0].date.getMonth();
      if (month !== lastMonth) {
        labels.push(MONTH_NAMES[month]);
        lastMonth = month;
      } else {
        labels.push(null);
      }

      builtWeeks.push(week);
    }

    weeks = builtWeeks;
    monthLabels = labels;
  }

  function cellTitle(cell: DayCell): string {
    if (cell.future) {
      return "";
    }
    const noun = cell.words === 1 ? "word" : "words";
    return `${cell.words.toLocaleString()} ${noun} · ${formatDate(cell.date)}`;
  }
</script>

<div class="longform-heatmap">
  <div class="longform-heatmap-months">
    {#each monthLabels as label}
      <span class="longform-heatmap-month">{label || ""}</span>
    {/each}
  </div>
  <div class="longform-heatmap-body">
    <div class="longform-heatmap-weekdays">
      {#each WEEKDAY_LABELS as label}
        <span class="longform-heatmap-weekday">{label}</span>
      {/each}
    </div>
    <div class="longform-heatmap-grid">
      {#each weeks as week}
        <div class="longform-heatmap-week">
          {#each week as cell}
            <div
              class="longform-heatmap-cell level-{cell.level}"
              class:future={cell.future}
              title={cellTitle(cell)}
            ></div>
          {/each}
        </div>
      {/each}
    </div>
  </div>
  <div class="longform-heatmap-footer">
    <span>
      {totalWords.toLocaleString()} words · {activeDays} active
      {activeDays === 1 ? "day" : "days"}
    </span>
    <span class="longform-heatmap-legend">
      Less
      <span class="longform-heatmap-cell level-0"></span>
      <span class="longform-heatmap-cell level-1"></span>
      <span class="longform-heatmap-cell level-2"></span>
      <span class="longform-heatmap-cell level-3"></span>
      <span class="longform-heatmap-cell level-4"></span>
      More
    </span>
  </div>
</div>

<style>
  .longform-heatmap {
    --heat-cell-size: 10px;
    --heat-gap: 3px;
    --heat-gutter: 26px;
    margin-top: var(--size-4-3);
  }

  .longform-heatmap-months {
    display: flex;
    gap: var(--heat-gap);
    margin-left: var(--heat-gutter);
    margin-bottom: var(--size-2-1);
    height: 12px;
  }

  .longform-heatmap-month {
    width: var(--heat-cell-size);
    font-size: 9px;
    line-height: 12px;
    color: var(--text-muted);
    white-space: nowrap;
    overflow: visible;
  }

  .longform-heatmap-body {
    display: flex;
    gap: var(--heat-gap);
  }

  .longform-heatmap-weekdays {
    display: flex;
    flex-direction: column;
    gap: var(--heat-gap);
    width: var(--heat-gutter);
    margin-right: var(--heat-gap);
    flex-shrink: 0;
  }

  .longform-heatmap-weekday {
    height: var(--heat-cell-size);
    line-height: var(--heat-cell-size);
    font-size: 9px;
    color: var(--text-faint);
    text-align: right;
    padding-right: var(--size-2-1);
  }

  .longform-heatmap-grid {
    display: flex;
    gap: var(--heat-gap);
  }

  .longform-heatmap-week {
    display: flex;
    flex-direction: column;
    gap: var(--heat-gap);
  }

  .longform-heatmap-cell {
    width: var(--heat-cell-size);
    height: var(--heat-cell-size);
    border-radius: 2px;
    background-color: var(--background-secondary-alt);
    flex-shrink: 0;
  }

  .longform-heatmap-cell.future {
    background-color: transparent;
  }

  .longform-heatmap-cell.level-1 {
    background-color: var(--text-accent);
    opacity: 0.25;
  }

  .longform-heatmap-cell.level-2 {
    background-color: var(--text-accent);
    opacity: 0.45;
  }

  .longform-heatmap-cell.level-3 {
    background-color: var(--text-accent);
    opacity: 0.7;
  }

  .longform-heatmap-cell.level-4 {
    background-color: var(--text-accent);
    opacity: 1;
  }

  .longform-heatmap-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: var(--size-4-2);
    margin-top: var(--size-4-2);
    font-size: var(--font-smallest);
    color: var(--text-muted);
    flex-wrap: wrap;
  }

  .longform-heatmap-legend {
    display: flex;
    align-items: center;
    gap: 2px;
  }

  .longform-heatmap-legend .longform-heatmap-cell {
    width: calc(var(--heat-cell-size) - 2px);
    height: calc(var(--heat-cell-size) - 2px);
  }
</style>
