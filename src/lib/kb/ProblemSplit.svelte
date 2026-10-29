<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket. Unlike CauseTable, reports the desk read as a one-day
  // move (desk-notes.json) are their own problem, not a footnote under the tenant's cause.
  type Problem = 'parser' | 'hint' | 'one-day' | 'unplaced';
  let { ticket, links = {} }: { ticket: number; links?: Partial<Record<Problem, string>> } = $props();

  const LABEL: Record<Problem, string> = {
    parser: 'shared parser reads day-first dates month first',
    hint: 'booking-v3 form hint says MM/DD/YYYY to day-first tenants',
    'one-day': 'booking moves by one day, month and day intact',
    unplaced: 'no tenant on file: not placed',
  };

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const problemOf = (r: Report): Problem =>
    notes.oneDayMove.includes(r.id) ? 'one-day' : (causeOf(r, tenantOf(r)) ?? 'unplaced');
  const rows = $derived.by(() => {
    const by = new Map<Problem, Report[]>();
    for (const r of merged) by.set(problemOf(r), [...(by.get(problemOf(r)) ?? []), r]);
    return [...by.entries()]
      .map(([problem, rs]) => {
        const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
        return {
          problem,
          count: rs.length,
          tenants: new Set(rs.map((r) => r.tenant)).size,
          intakeAgrees: rs.filter((r) => r.meta.intakeTag === problem).length,
          firstSeen: days[0],
          lastSeen: days[days.length - 1],
        };
      })
      .sort((a, b) => b.count - a.count);
  });
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.length}</strong> {rows.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>intake tag agrees</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.problem)}
      <tr>
        <td><code>{r.problem}</code><br /><span class="what">{LABEL[r.problem]}</span></td>
        <td>{r.count}</td>
        <td>{r.tenants}</td>
        <td>{r.intakeAgrees} of {r.count}</td>
        <td>{r.firstSeen} → {r.lastSeen}</td>
        <td>{#if links[r.problem]}<a href={links[r.problem]}>{links[r.problem]?.split('/').slice(-3).join('/').replace('/issues/', '#')}</a>{:else}none yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .what { color: var(--muted); font-size: 0.85em; }
</style>
