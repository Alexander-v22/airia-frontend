<script>
  import { onMount } from 'svelte';

  let stats           = null;
  let articles        = [];
  let annotationLog   = [];
  let interventionLog = [];
  let unlockStatus    = { level3_unlocked: false, accepted_interventions: 0 };
  let loading         = true;

  function lsGet(key, fallback) {
    try {
      const raw = localStorage.getItem(key);
      return raw ? JSON.parse(raw) : fallback;
    } catch { return fallback; }
  }

  onMount(() => {
    const samples   = lsGet('airia_training_samples', []);
    articles        = lsGet('airia_articles', []);
    annotationLog   = lsGet('airia_annotation_log', []);
    interventionLog = lsGet('airia_intervention_log', []);
    unlockStatus    = lsGet('airia_unlock_status', { level3_unlocked: false, accepted_interventions: 0 });

    const counts = { too_hard: 0, just_right: 0, too_easy: 0 };
    const labels = { 0: 'too_hard', 1: 'just_right', 2: 'too_easy' };
    for (const s of samples) {
      const name = labels[s.label];
      if (name) counts[name]++;
    }

    stats = {
      total_samples:    samples.length,
      samples_by_class: counts,
      oldest_sample:    samples[0]?.timestamp ?? null,
      newest_sample:    samples[samples.length - 1]?.timestamp ?? null
    };

    loading = false;
  });

  function formatDate(ts) {
    if (!ts) return '—';
    return new Date(ts).toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' });
  }

  function difficultyLabel(val) {
    if (val <= 0.2) return 'Easy';
    if (val <= 0.5) return 'Moderate';
    if (val <= 0.8) return 'Technical';
    return 'Dense';
  }

  function difficultyPct(val) {
    return Math.round((val ?? 0.5) * 100);
  }

  function getInsights(samples) {
    if (!samples || !samples.length) return { avgWpm: 0, daysDiff: 0 };
    const wpms   = samples.map(s => s.features[0] * 500).filter(w => w > 0);
    const avgWpm = wpms.length
      ? Math.round(wpms.reduce((a, b) => a + b, 0) / wpms.length)
      : 0;
    const firstTs  = samples[0]?.timestamp;
    const lastTs   = samples[samples.length - 1]?.timestamp;
    const daysDiff = firstTs && lastTs
      ? Math.max(1, Math.round((new Date(lastTs) - new Date(firstTs)) / (1000 * 60 * 60 * 24)))
      : 0;
    return { avgWpm, daysDiff };
  }

  // ─── Annotation stats ────────────────────────────────────────────────────
  // Term frequency across all annotation events
  function getTopTerms(log, n = 8) {
    const freq = {};
    for (const event of log) {
      for (const term of (event.terms ?? [])) {
        freq[term] = (freq[term] ?? 0) + 1;
      }
    }
    return Object.entries(freq)
      .sort((a, b) => b[1] - a[1])
      .slice(0, n)
      .map(([term, count]) => ({ term, count }));
  }

  // Annotation density by genre: avg terms per paragraph per genre
  function getAnnotationByGenre(log) {
    const byGenre = {};
    for (const event of log) {
      const g = event.genre ?? 'unknown';
      if (!byGenre[g]) byGenre[g] = { total: 0, events: 0 };
      byGenre[g].total  += event.terms?.length ?? 0;
      byGenre[g].events += 1;
    }
    return Object.entries(byGenre)
      .map(([genre, d]) => ({ genre, avg: parseFloat((d.total / d.events).toFixed(1)), events: d.events }))
      .sort((a, b) => b.avg - a.avg)
      .slice(0, 6);
  }

  // ─── Intervention stats ───────────────────────────────────────────────────
  function getInterventionStats(log) {
    if (!log.length) return null;
    const total      = log.length;
    const primerOnly = log.filter(e => e.primer_only).length;
    const rewrites   = log.filter(e => e.rewrite_used).length;
    const level2     = log.filter(e => e.level === 2).length;
    const level3     = log.filter(e => e.level === 3).length;
    return { total, primerOnly, rewrites, level2, level3 };
  }

  // Intervention rate by genre: how often interventions fire per article genre
  function getInterventionByGenre(intLog, articleList) {
    // Build a map of article_id -> genre from the articles list
    const idToGenre = {};
    for (const a of articleList) {
      idToGenre[a.id] = a.classification?.specific_genre ?? 'unknown';
    }
    const byGenre = {};
    for (const event of intLog) {
      const g = idToGenre[event.article_id] ?? event.genre ?? 'unknown';
      if (!byGenre[g]) byGenre[g] = { interventions: 0, rewrites: 0 };
      byGenre[g].interventions += 1;
      if (event.rewrite_used) byGenre[g].rewrites += 1;
    }
    return Object.entries(byGenre)
      .map(([genre, d]) => ({ genre, ...d, rewriteRate: Math.round((d.rewrites / d.interventions) * 100) }))
      .sort((a, b) => b.interventions - a.interventions)
      .slice(0, 6);
  }

  $: arcTotal     = 7;
  $: articleCount = articles.length;
  $: arcLabel     = articleCount >= arcTotal ? 'Fully personalized' : `Article ${articleCount} of ${arcTotal}`;
  $: arcSub       = articleCount >= arcTotal
      ? 'Model is fully calibrated to your reading'
      : articleCount >= 5 ? 'Interventions active — personalizing'
      : articleCount >= 3 ? 'Soft interventions beginning'
      : 'Calibrating your reading profile';

  $: total   = (stats?.total_samples || 1);
  $: hardPct = Math.round(((stats?.samples_by_class.too_hard   ?? 0) / total) * 100);
  $: goodPct = Math.round(((stats?.samples_by_class.just_right ?? 0) / total) * 100);
  $: easyPct = Math.round(((stats?.samples_by_class.too_easy   ?? 0) / total) * 100);

  $: allSamples        = lsGet('airia_training_samples', []);
  $: insights          = getInsights(allSamples);
  $: uniqueGenres      = [...new Set(articles.map(a => a.classification?.broad_genre).filter(Boolean))];
  $: topTerms          = getTopTerms(annotationLog);
  $: annotationGenres  = getAnnotationByGenre(annotationLog);
  $: interventionStats = getInterventionStats(interventionLog);
  $: interventionGenres = getInterventionByGenre(interventionLog, articles);
</script>

<nav class="topnav">
  <a href="/" class="topnav-logo">AIRIA</a>
  <div class="topnav-links">
    <a href="/stats"    class="topnav-link active">Your data</a>
    <a href="/infopage" class="topnav-link">How it works</a>
  </div>
</nav>

<div class="stats-page">

  {#if loading}
    <div class="stats-loading">Loading…</div>
  {:else}

    <div class="stats-body">

      <div class="stats-header-row">
        <div>
          <h1 class="stats-page-title">Your data</h1>
          <p class="stats-subtitle">Stored in your browser only — never sent anywhere</p>
        </div>
        <div class="arc-block">
          <div class="arc-label-top">Personalization progress</div>
          <div class="arc-row">
            <div class="arc-steps">
              {#each Array(arcTotal) as _, i}
                <div
                  class="arc-step"
                  class:done={i < articleCount}
                  class:curr={i === articleCount && articleCount < arcTotal}
                >{i + 1}</div>
              {/each}
            </div>
            <span class="arc-text">{arcLabel}</span>
          </div>
          <div class="arc-sub">{arcSub}</div>
        </div>
      </div>

      <!-- ── Top stats strip ──────────────────────────────────────────────── -->
      <div class="stats-strip">
        <div class="strip-stat">
          <div class="strip-val">{stats.total_samples}</div>
          <div class="strip-label">Total samples</div>
          <div class="strip-sub">
            {stats.oldest_sample
              ? `${formatDate(stats.oldest_sample)} — ${formatDate(stats.newest_sample)}`
              : 'No data yet'}
          </div>
        </div>
        <div class="strip-stat">
          <div class="strip-val">{insights.avgWpm}</div>
          <div class="strip-label">Avg WPM</div>
          <div class="strip-sub">across all sessions</div>
        </div>
        <div class="strip-stat">
          <div class="strip-val">{articleCount}</div>
          <div class="strip-label">Articles read</div>
          <div class="strip-sub">{uniqueGenres.length} genre{uniqueGenres.length !== 1 ? 's' : ''} covered</div>
        </div>
        <div class="strip-stat">
          <div class="strip-val">{insights.daysDiff}d</div>
          <div class="strip-label">Active</div>
          <div class="strip-sub">days since first session</div>
        </div>
        <div class="strip-stat">
          <div class="strip-val">{interventionStats?.total ?? 0}</div>
          <div class="strip-label">Interventions</div>
          <div class="strip-sub">{interventionStats?.rewrites ?? 0} rewrites used</div>
        </div>
        <div class="strip-stat">
          <div class="strip-val">{annotationLog.length}</div>
          <div class="strip-label">Annotated</div>
          <div class="strip-sub">paragraphs with terms</div>
        </div>
      </div>

      <!-- ── Training balance + AIRIA observed ───────────────────────────── -->
      <div class="stats-mid-row">

        <div class="stats-card">
          <div class="card-title">Training balance</div>
          {#if stats.total_samples > 0}
            <div class="balance-bar">
              <div class="bb-hard" style="width: {hardPct}%"></div>
              <div class="bb-good" style="width: {goodPct}%"></div>
              <div class="bb-easy" style="width: {easyPct}%"></div>
            </div>
            <div class="balance-rows">
              <div class="balance-row">
                <span class="bal-key"><span class="bal-dot" style="background:#000"></span>Too hard</span>
                <span class="bal-val">{stats.samples_by_class.too_hard}</span>
              </div>
              <div class="balance-row">
                <span class="bal-key"><span class="bal-dot" style="background:#888"></span>Just right</span>
                <span class="bal-val">{stats.samples_by_class.just_right}</span>
              </div>
              <div class="balance-row">
                <span class="bal-key"><span class="bal-dot bal-dot-easy"></span>Too easy</span>
                <span class="bal-val">{stats.samples_by_class.too_easy}</span>
              </div>
            </div>
            {#if stats.total_samples < 10}
              <div class="balance-note">{10 - stats.total_samples} more samples to enable retraining</div>
            {:else}
              <div class="balance-ready">Model can be retrained</div>
            {/if}
          {:else}
            <div class="card-empty">No samples yet. Read an article and give feedback to start training.</div>
          {/if}
        </div>

        <div class="stats-card">
          <div class="card-title">AIRIA observed</div>
          {#if stats.total_samples >= 3}
            <div class="obs-list">
              {#if insights.avgWpm > 0}
                <div class="obs-item">Your average reading speed is <strong>{insights.avgWpm} WPM</strong> across all sessions.</div>
              {/if}
              {#if stats.samples_by_class.too_hard > stats.samples_by_class.too_easy}
                <div class="obs-item">You find most content <strong>challenging</strong> — the model will lean toward simplification.</div>
              {:else if stats.samples_by_class.too_easy > stats.samples_by_class.too_hard}
                <div class="obs-item">You find most content <strong>comfortable</strong> — the model will target harder material.</div>
              {:else}
                <div class="obs-item">Your feedback is <strong>well balanced</strong> — the model has a strong signal to work with.</div>
              {/if}
              {#if articleCount >= 2}
                <div class="obs-item">You have covered <strong>{uniqueGenres.length} genre{uniqueGenres.length !== 1 ? 's' : ''}</strong> — reading more genres improves personalization accuracy.</div>
              {/if}
              {#if interventionStats && interventionStats.total > 0}
                <div class="obs-item">
                  The primer alone resolved
                  <strong>{interventionStats.primerOnly} of {interventionStats.total}</strong>
                  intervention{interventionStats.total !== 1 ? 's' : ''} — you needed a rewrite
                  {interventionStats.rewrites} time{interventionStats.rewrites !== 1 ? 's' : ''}.
                </div>
              {/if}
              {#if topTerms.length > 0}
                <div class="obs-item">
                  Your most annotated term is <strong>{topTerms[0].term}</strong>,
                  flagged {topTerms[0].count} time{topTerms[0].count !== 1 ? 's' : ''} across your sessions.
                </div>
              {/if}
            </div>
          {:else}
            <div class="card-empty">Read at least 3 articles for AIRIA to start building your reading profile.</div>
          {/if}
        </div>

      </div>

      <!-- ── Intervention breakdown ─────────────────────────────────────── -->
      {#if interventionStats && interventionStats.total > 0}
        <div class="stats-mid-row">

          <div class="stats-card">
            <div class="card-title">Intervention breakdown</div>
            <div class="balance-rows">
              <div class="balance-row">
                <span class="bal-key">Total fired</span>
                <span class="bal-val">{interventionStats.total}</span>
              </div>
              <div class="balance-row">
                <span class="bal-key">Primer resolved</span>
                <span class="bal-val">{interventionStats.primerOnly}</span>
              </div>
              <div class="balance-row">
                <span class="bal-key">Needed rewrite</span>
                <span class="bal-val">{interventionStats.total - interventionStats.primerOnly}</span>
              </div>
              <div class="balance-row">
                <span class="bal-key">Rewrite accepted</span>
                <span class="bal-val">{interventionStats.rewrites}</span>
              </div>
              <div class="balance-row">
                <span class="bal-key">Level 2 fired</span>
                <span class="bal-val">{interventionStats.level2}</span>
              </div>
              <div class="balance-row">
                <span class="bal-key">Level 3 fired</span>
                <span class="bal-val">{interventionStats.level3}</span>
              </div>
            </div>

            <!-- Level 3 unlock progress -->
            <div class="sidebar-divider" style="margin: 12px 0"></div>
            {#if unlockStatus.level3_unlocked}
              <div class="balance-ready">Level 3 unlocked</div>
            {:else}
              <div class="balance-note">
                Level 3 unlock: {unlockStatus.accepted_interventions} / 3 rewrites accepted
              </div>
              <div class="balance-bar" style="margin-top: 6px">
                <div class="bb-good" style="width: {Math.min((unlockStatus.accepted_interventions / 3) * 100, 100)}%"></div>
              </div>
            {/if}
          </div>

          <div class="stats-card">
            <div class="card-title">Interventions by genre</div>
            {#if interventionGenres.length > 0}
              <div class="genre-list">
                {#each interventionGenres as g}
                  <div class="genre-row">
                    <span class="genre-name">{g.genre}</span>
                    <span class="genre-count">{g.interventions} fired</span>
                    <span class="genre-rewrite">{g.rewriteRate}% rewritten</span>
                  </div>
                {/each}
              </div>
            {:else}
              <div class="card-empty">No intervention data yet.</div>
            {/if}
          </div>

        </div>
      {/if}

      <!-- ── Annotation breakdown ───────────────────────────────────────── -->
      {#if annotationLog.length > 0}
        <div class="stats-mid-row">

          <div class="stats-card">
            <div class="card-title">Top annotated terms</div>
            <div class="obs-list">
              {#each topTerms as t}
                <div class="obs-item term-row">
                  <span class="term-name">{t.term}</span>
                  <span class="term-count">{t.count}×</span>
                </div>
              {/each}
              {#if topTerms.length === 0}
                <div class="card-empty">No annotations yet. Turn on Annotate terms in the sidebar while reading.</div>
              {/if}
            </div>
          </div>

          <div class="stats-card">
            <div class="card-title">Annotation density by genre</div>
            <p class="card-sub">Avg domain terms flagged per paragraph</p>
            {#if annotationGenres.length > 0}
              <div class="genre-list">
                {#each annotationGenres as g}
                  <div class="genre-row">
                    <span class="genre-name">{g.genre}</span>
                    <span class="genre-count">{g.avg} terms/para</span>
                    <div class="diff-bar" style="width: 80px; display: inline-block; vertical-align: middle; margin-left: 8px">
                      <div class="diff-fill" style="width: {Math.min(g.avg / 6 * 100, 100)}%"></div>
                    </div>
                  </div>
                {/each}
              </div>
            {:else}
              <div class="card-empty">No annotation data yet.</div>
            {/if}
          </div>

        </div>
      {/if}

      <!-- ── Articles table ─────────────────────────────────────────────── -->
      <div class="table-section">
        <div class="table-header-row">
          <span class="table-title">Ingested articles</span>
          <span class="table-count">{articles.length} article{articles.length !== 1 ? 's' : ''}</span>
        </div>

        {#if articles.length === 0}
          <div class="table-empty">No articles ingested yet. Paste a URL in the reader to get started.</div>
        {:else}
          <div class="table-wrap">
            <table class="stats-table">
              <thead>
                <tr>
                  <th>Title</th>
                  <th>Genre</th>
                  <th>Lexile</th>
                  <th>Difficulty</th>
                  <th>Words</th>
                  <th>Interventions</th>
                  <th>Date</th>
                </tr>
              </thead>
              <tbody>
                {#each articles as article}
                  {@const articleInterventions = interventionLog.filter(e => e.article_id === article.id)}
                  <tr>
                    <td class="td-title">
                      <a href={article.url} target="_blank" rel="noreferrer" class="article-link">
                        {article.title}
                      </a>
                    </td>
                    <td><span class="genre-tag">{article.classification?.specific_genre ?? '—'}</span></td>
                    <td class="td-mono">{article.estimated_lexile}</td>
                    <td>
                      <div class="diff-wrap">
                        <div class="diff-bar">
                          <div class="diff-fill" style="width: {difficultyPct(article.classification?.genre_difficulty)}%"></div>
                        </div>
                        <span class="diff-label">{difficultyLabel(article.classification?.genre_difficulty ?? 0.5)}</span>
                      </div>
                    </td>
                    <td class="td-mono">{article.word_count.toLocaleString()}</td>
                    <td class="td-mono">
                      {#if articleInterventions.length > 0}
                        {articleInterventions.length}
                        <span class="int-sub">({articleInterventions.filter(e => e.rewrite_used).length} rewritten)</span>
                      {:else}
                        —
                      {/if}
                    </td>
                    <td class="td-date">{formatDate(article.timestamp)}</td>
                  </tr>
                {/each}
              </tbody>
            </table>
          </div>
        {/if}
      </div>

    </div>
  {/if}
</div>