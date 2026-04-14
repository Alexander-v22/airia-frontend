<script>
  import { onMount } from 'svelte';

  const PROFILE_DEFAULT = {
    calibration_complete: false,
    calibration_sessions: [],
    baseline_wpm:         null,
    wpm_std:              null,
    charge_threshold:     0.68,
    slowdown_threshold:   0.75,
    last_updated:         null
  };

  let profile = PROFILE_DEFAULT;
  let loading = true;

  function lsGet(key, fallback) {
    try {
      const raw = localStorage.getItem(key);
      return raw ? JSON.parse(raw) : fallback;
    } catch { return fallback; }
  }

  function lsSet(key, value) {
    try { localStorage.setItem(key, JSON.stringify(value)); } catch {}
  }

  onMount(() => {
    profile = lsGet('airia_user_profile', PROFILE_DEFAULT);
    loading = false;
  });

  function resetCalibration() {
    if (!confirm('This will delete your calibration data and reset thresholds to defaults. Continue?')) return;
    profile = { ...PROFILE_DEFAULT };
    lsSet('airia_user_profile', profile);
  }

  const FEEDBACK_LABELS = ['Too hard', 'Just right', 'Too easy'];
  const FEEDBACK_CLASSES = ['label-hard', 'label-good', 'label-easy'];

  function sensitivityLabel(chargeThreshold) {
    if (chargeThreshold <= 0.58) return 'High — fires earlier for frequent struggles';
    if (chargeThreshold >= 0.76) return 'Low — fires only for clear struggle signals';
    return 'Standard — balanced intervention timing';
  }

  function consistencyLabel(wpmStd) {
    if (wpmStd === null) return '—';
    if (wpmStd < 25)  return 'Very consistent';
    if (wpmStd < 60)  return 'Moderately variable';
    return 'Highly variable';
  }

  function formatDate(ts) {
    if (!ts) return '—';
    return new Date(ts).toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' });
  }

  $: articlesNeeded = 3 - (profile.calibration_sessions?.length ?? 0);
  $: sessionCount   = profile.calibration_sessions?.length ?? 0;
</script>

<nav class="topnav">
  <a href="/" class="topnav-logo">AIRIA</a>
  <div class="topnav-links">
    <a href="/stats"    class="topnav-link">Your data</a>
    <a href="/profile"  class="topnav-link active">Profile</a>
    <a href="/infopage" class="topnav-link">How it works</a>
  </div>
</nav>

<div class="profile-page">
  {#if loading}
    <div class="profile-loading">Loading…</div>
  {:else}
  <div class="profile-body">

    <!-- ── Header ───────────────────────────────────────────────────────── -->
    <div class="profile-header">
      <div>
        <h1 class="profile-title">Reading profile</h1>
        <p class="profile-sub">Your personal reading thresholds, built from a 3-article calibration baseline.</p>
      </div>
      {#if profile.calibration_complete}
        <div class="profile-status-badge complete">Calibrated</div>
      {:else if sessionCount > 0}
        <div class="profile-status-badge partial">{sessionCount} / 3 articles</div>
      {:else}
        <div class="profile-status-badge empty">Not started</div>
      {/if}
    </div>

    <!-- ── Calibration arc ───────────────────────────────────────────────── -->
    <div class="profile-section-label">Calibration arc</div>
    <p class="profile-section-sub">AIRIA observes your first 3 articles before deriving personalized thresholds. Until then, default values are used.</p>

    <div class="calib-arc">
      {#each [0, 1, 2] as i}
        {@const session = profile.calibration_sessions[i]}
        {@const isNext  = !session && i === sessionCount}
        <div class="calib-step">
          <div class="calib-step-left">
            <div class="calib-dot"
              class:done={!!session}
              class:next={isNext}
            >
              {#if session}✓{:else}{i + 1}{/if}
            </div>
            {#if i < 2}
              <div class="calib-line" class:done={!!session}></div>
            {/if}
          </div>
          <div class="calib-step-content">
            {#if session}
              <div class="calib-article-title">{session.article_title}</div>
              <div class="calib-meta">
                <span class="calib-meta-item">{session.avg_wpm} WPM</span>
                <span class="calib-meta-sep">·</span>
                <span class="calib-meta-item">{session.genre}</span>
                <span class="calib-meta-sep">·</span>
                <span class="calib-meta-item {FEEDBACK_CLASSES[session.feedback_label]}">
                  {FEEDBACK_LABELS[session.feedback_label] ?? '—'}
                </span>
                {#if session.intervention_count > 0}
                  <span class="calib-meta-sep">·</span>
                  <span class="calib-meta-item calib-interventions">
                    {session.intervention_count} intervention{session.intervention_count !== 1 ? 's' : ''}
                  </span>
                {/if}
              </div>
              <div class="calib-date">{formatDate(session.timestamp)}</div>
            {:else if isNext}
              <div class="calib-next-label">Read another article to continue →</div>
              <a href="/" class="calib-read-link">Start reading</a>
            {:else}
              <div class="calib-pending-label">Pending</div>
            {/if}
          </div>
        </div>
      {/each}
    </div>

    <!-- ── Thresholds (post-calibration) ────────────────────────────────── -->
    {#if profile.calibration_complete}
      <div class="profile-divider"></div>
      <div class="profile-section-label">Your reading thresholds</div>
      <p class="profile-section-sub">These values replace the hardcoded defaults. AIRIA will now fire interventions tuned to your reading patterns.</p>

      <div class="threshold-grid">
        <div class="threshold-card">
          <div class="threshold-label">Baseline speed</div>
          <div class="threshold-val">{profile.baseline_wpm}<span class="threshold-unit"> WPM</span></div>
          <div class="threshold-desc">Average reading speed across your 3 calibration articles</div>
        </div>
        <div class="threshold-card">
          <div class="threshold-label">Speed consistency</div>
          <div class="threshold-val">{profile.wpm_std}<span class="threshold-unit"> σ</span></div>
          <div class="threshold-desc">{consistencyLabel(profile.wpm_std)} — std deviation of WPM across sessions</div>
        </div>
        <div class="threshold-card">
          <div class="threshold-label">Charge threshold</div>
          <div class="threshold-val">{Math.round(profile.charge_threshold * 100)}<span class="threshold-unit">%</span></div>
          <div class="threshold-desc">{sensitivityLabel(profile.charge_threshold)}</div>
          <div class="threshold-bar-wrap">
            <div class="threshold-bar">
              <div class="threshold-fill" style="width:{profile.charge_threshold * 100}%"></div>
            </div>
            <span class="threshold-bar-label">default 68%</span>
          </div>
        </div>
        <div class="threshold-card">
          <div class="threshold-label">Slowdown sensitivity</div>
          <div class="threshold-val">{Math.round(profile.slowdown_threshold * 100)}<span class="threshold-unit">%</span></div>
          <div class="threshold-desc">Triggers when a paragraph is this much slower than your running average</div>
          <div class="threshold-bar-wrap">
            <div class="threshold-bar">
              <div class="threshold-fill" style="width:{profile.slowdown_threshold * 100}%"></div>
            </div>
            <span class="threshold-bar-label">default 75%</span>
          </div>
        </div>
      </div>

      <div class="profile-calibrated-note">
        Calibrated on {formatDate(profile.last_updated)}. Thresholds update automatically after each new article if you reset calibration.
      </div>

    {:else}
      <div class="calib-incomplete">
        <div class="calib-incomplete-count">
          {articlesNeeded} article{articlesNeeded !== 1 ? 's' : ''} remaining
        </div>
        <p class="calib-incomplete-desc">
          Until calibration completes, AIRIA uses the default charge threshold (68%) and slowdown threshold (75%). After 3 articles, thresholds are tuned to your reading pace and struggle frequency.
        </p>
        <a href="/" class="calib-cta-btn">Read an article →</a>
      </div>
    {/if}

    <!-- ── Reset ─────────────────────────────────────────────────────────── -->
    {#if sessionCount > 0}
      <div class="profile-divider"></div>
      <div class="profile-reset-row">
        <div>
          <div class="profile-reset-label">Reset calibration</div>
          <div class="profile-reset-sub">Clears your calibration sessions and reverts thresholds to defaults. Training samples and model weights are not affected.</div>
        </div>
        <button class="calib-reset-btn" onclick={resetCalibration}>Reset</button>
      </div>
    {/if}

  </div>
  {/if}
</div>
