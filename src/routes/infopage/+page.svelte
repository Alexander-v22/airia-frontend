<script>
  // No data needed — this is a static explainer page
</script>

<div class="hiw-page">

  <!-- ── Top bar ── -->
  <div class="stats-topbar">
    <div class="pg-logo">AIRIA</div>
    <nav class="pg-nav">
      <a href="/"             class="pg-navitem">Reader</a>
      <a href="/stats"        class="pg-navitem">Your data</a>
      <a href="/infopage"     class="pg-navitem">How it works    </a>
    </nav>
    <a href="/" class="pg-back">← Reader</a>
  </div>

  <div class="hiw-body">

    <!-- ── Hero ── -->
    <div class="hiw-hero">
      <h1 class="hiw-hero-title">AIRIA learns how you read,<br>then helps where you need it.</h1>
      <p class="hiw-hero-sub">A spiking neural network watches your reading behavior paragraph by paragraph, building a personal model of when you struggle and when you don't. Interventions appear only when the data says you need them.</p>
    </div>

    <!-- ── Arc timeline ── -->
    <div class="hiw-section-label">The arc — your first 7 articles</div>
    <div class="hiw-timeline">

      <div class="hiw-tl-item">
        <div class="hiw-tl-left">
          <div class="hiw-tl-dot active">1</div>
          <div class="hiw-tl-line"></div>
        </div>
        <div class="hiw-tl-content">
          <div class="hiw-tl-phase">Article 1</div>
          <div class="hiw-tl-title">Full analysis</div>
          <div class="hiw-tl-desc">No interventions. AIRIA reads how you read — your WPM curve across every paragraph, where you slow down, how your speed changes as an article gets harder. Your data page begins populating with real observations from this session.</div>
        </div>
      </div>

      <div class="hiw-tl-item">
        <div class="hiw-tl-left">
          <div class="hiw-tl-dot active">2</div>
          <div class="hiw-tl-line"></div>
        </div>
        <div class="hiw-tl-content">
          <div class="hiw-tl-phase">Article 2</div>
          <div class="hiw-tl-title">Pattern comparison</div>
          <div class="hiw-tl-desc">AIRIA compares this session to your first. Genre-specific patterns start to emerge. If you consistently slow on technical content but cruise through narrative, that signal gets recorded. Your data page starts showing the differences.</div>
        </div>
      </div>

      <div class="hiw-tl-item">
        <div class="hiw-tl-left">
          <div class="hiw-tl-dot active">3</div>
          <div class="hiw-tl-line"></div>
        </div>
        <div class="hiw-tl-content">
          <div class="hiw-tl-phase">Article 3</div>
          <div class="hiw-tl-title">Prediction preview</div>
          <div class="hiw-tl-desc">At the end of the article, AIRIA shows where it would have intervened if interventions were active. You can see whether those match how the content actually felt. This sets accurate expectations before help starts appearing automatically.</div>
        </div>
      </div>

      <div class="hiw-tl-item">
        <div class="hiw-tl-left">
          <div class="hiw-tl-dot upcoming">4–5</div>
          <div class="hiw-tl-line"></div>
        </div>
        <div class="hiw-tl-content">
          <div class="hiw-tl-phase">Articles 4–5</div>
          <div class="hiw-tl-title">Soft interventions</div>
          <div class="hiw-tl-desc">Interventions appear with a brief explanation of why they fired — which paragraph triggered the SNN, what the difficulty score was. You're still in control. Accepting or dismissing each intervention is itself a training signal that sharpens the model.</div>
        </div>
      </div>

      <div class="hiw-tl-item">
        <div class="hiw-tl-left">
          <div class="hiw-tl-dot upcoming">6–7</div>
        </div>
        <div class="hiw-tl-content">
          <div class="hiw-tl-phase">Articles 6–7</div>
          <div class="hiw-tl-title">Silent personalization</div>
          <div class="hiw-tl-desc">Interventions fire quietly when you need them and disappear when you don't. The model is now calibrated to your specific reading rhythm — your natural WPM range, your tolerance for different genres, how much deceleration means you're genuinely struggling versus just slowing for a dense sentence.</div>
        </div>
      </div>

    </div>

    <div class="hiw-divider"></div>

    <!-- ── System flow ── -->
    <div class="hiw-section-label">The system — three layers</div>
    <div class="hiw-flow">
      <div class="hiw-flow-step">
        <div class="hiw-flow-num">01</div>
        <div class="hiw-flow-title">Claude classifies</div>
        <div class="hiw-flow-desc">Every article gets a broad genre, a specific genre, and a difficulty score from 0 to 1 before you read a single word. A 1500-word excerpt is analyzed to assess cognitive density for a general reader.</div>
      </div>
      <div class="hiw-flow-step">
        <div class="hiw-flow-num">02</div>
        <div class="hiw-flow-title">SNN watches</div>
        <div class="hiw-flow-desc">The spiking neural network processes each paragraph as a real timestep. Membrane potential accumulates with struggle and decays with ease — it remembers earlier difficulty within the same article, so later struggle fires faster.</div>
      </div>
      <div class="hiw-flow-step">
        <div class="hiw-flow-num">03</div>
        <div class="hiw-flow-title">Claude helps</div>
        <div class="hiw-flow-desc">When the membrane fires on a too-hard spike, Claude rewrites or annotates the paragraph. The level of intervention depends on the article's difficulty score. You choose whether to use the simplified version or keep the original.</div>
      </div>
    </div>

    <div class="hiw-divider"></div>

    <!-- ── Intervention examples ── -->
    <div class="hiw-section-label">What interventions look like</div>
    <div class="hiw-int-row">

      <div class="hiw-int-card">
        <span class="hiw-int-level">Level 2</span>
        <div class="hiw-int-title">Full rewrite</div>
        <div class="hiw-int-desc">The whole paragraph is rewritten in simpler language while preserving the core meaning. Triggered when the article's genre difficulty score is between 0.5 and 0.7 — technical but not highly specialized.</div>
        <div class="hiw-int-example">The F-35 program has officially labeled the heat damage issues in the B and C variants as its most serious deficiency classification — meaning they directly threaten the aircraft's core combat missions.</div>
      </div>

      <div class="hiw-int-card strong">
        <span class="hiw-int-level">Level 3</span>
        <div class="hiw-int-title">Surgical annotation</div>
        <div class="hiw-int-desc">Only the hard parts are touched. Domain terms get inline definitions, complex syntax gets untangled. The rest stays exactly as written. Triggered when genre difficulty is above 0.7 — dense technical or academic content.</div>
        <div class="hiw-int-example">The F-35 Joint Program Office has classified the issues as <strong>category 1 deficiencies</strong> <em style="color:#aaa;">[most serious classification — directly threatens mission capability]</em> for the B and C models.</div>
      </div>

    </div>

    <div class="hiw-divider"></div>

    <!-- ── Privacy ── -->
    <div class="hiw-privacy">
      <div class="hiw-privacy-icon">⊘</div>
      <div>
        <div class="hiw-privacy-title">Your data never leaves your browser</div>
        <div class="hiw-privacy-text">Training samples, model weights, and article history are stored only in your browser's localStorage. The backend is stateless compute — it receives data, processes it, returns a result, and immediately forgets everything. Nothing is logged or stored server-side. Clearing your browser data resets everything.</div>
        <div class="hiw-privacy-keys">
          <span class="hiw-key-tag">airia_training_samples</span>
          <span class="hiw-key-tag">airia_model_weights</span>
          <span class="hiw-key-tag">airia_articles</span>
        </div>
      </div>
    </div>

  </div>
</div>