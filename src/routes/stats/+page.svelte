<script>
  import { onMount } from 'svelte';

  let stats    = null;
  let articles = [];
  let loading  = true;

  function lsGet(key, fallback) {
    try {
      const raw = localStorage.getItem(key);
      return raw ? JSON.parse(raw) : fallback;
    } catch { return fallback; }
  }

  onMount(() => {
    const samples = lsGet('airia_training_samples', []);
    articles      = lsGet('airia_articles', []);

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

  // reactive helpers
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

  $: allSamples  = lsGet('airia_training_samples', []);
  $: insights    = getInsights(allSamples);
  $: uniqueGenres = [...new Set(articles.map(a => a.classification?.broad_genre).filter(Boolean))];
</script>

<div class="stats-page">

  <!-- ── Top bar ── -->
  <div class="stats-topbar">
    <div class="pg-logo">AIRIA</div>
    <nav class="pg-nav">
      <a href="/"              class="pg-navitem">Reader</a>
      <span                    class="pg-navitem active">Your data</span>
      <a href="/infopage"  class="pg-navitem">How it works</a>
    </nav>
    <a href="/" class="pg-back">← Reader</a>
  </div>

  {#if loading}
    <div class="stats-loading">Loading…</div>
  {:else}

    <div class="stats-body">

      <!-- ── Header row ── -->
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

      <!-- ── Stats strip ── -->
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
      </div>

      <!-- ── Mid row ── -->
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
            </div>
          {:else}
            <div class="card-empty">Read at least 3 articles for AIRIA to start building your reading profile.</div>
          {/if}
        </div>

      </div>

      <!-- ── Articles table ── -->
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
                  <th>Date</th>
                </tr>
              </thead>
              <tbody>
                {#each articles as article}
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