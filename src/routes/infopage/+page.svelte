<script>
  // No data needed — this is a static explainer page
</script>

<nav class="topnav">
  <a href="/" class="topnav-logo">AIRIA</a>
  <div class="topnav-links">
    <a href="/stats"    class="topnav-link">Your data</a>
    <a href="/infopage" class="topnav-link active">How it works</a>
  </div>
</nav>

<div class="hiw-page">
  <div class="hiw-body">

    <div class="hiw-hero">
      <h1 class="hiw-hero-title">AIRIA learns how you read,<br>then helps where you need it.</h1>
      <p class="hiw-hero-sub">A spiking neural network watches your reading behavior paragraph by paragraph, building a personal model of when you struggle and when you don't. Interventions appear only when the data says you need them — and they escalate only as far as you actually need.</p>
    </div>

    <!-- ── The arc ──────────────────────────────────────────────────────── -->
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

    <!-- ── The system ───────────────────────────────────────────────────── -->
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
        <div class="hiw-flow-desc">When the membrane fires on a too-hard spike, Claude first gives you the background context the paragraph assumes you have. If that's not enough, a rewrite is one click away. The depth of that rewrite depends on the article's difficulty score.</div>
      </div>
    </div>

    <div class="hiw-divider"></div>

    <!-- ── Intervention levels ──────────────────────────────────────────── -->
    <div class="hiw-section-label">What interventions look like</div>
    <p class="hiw-section-sub">Every intervention follows the same two-step flow. The primer comes first — always. The rewrite is only shown if you ask for it.</p>

    <div class="hiw-int-row">

      <div class="hiw-int-card">
        <span class="hiw-int-level">Both levels — Step 1</span>
        <div class="hiw-int-title">Background primer</div>
        <div class="hiw-int-desc">
          Before anything else, AIRIA shows a 2 to 4 sentence card explaining the background knowledge the paragraph assumes you already have. Not a summary of the paragraph — an answer to the question "what would I need to already know to understand this?"
        </div>
        <div class="hiw-int-example">
          <strong>Background context</strong><br><br>
          Central banks use interest rates as their main tool for controlling inflation. When they raise rates, borrowing becomes more expensive, which slows spending and cools prices. A yield curve shows the relationship between short-term and long-term borrowing costs — when it inverts, markets are signaling they expect a downturn ahead.
        </div>
        <div class="hiw-int-cta-row">
          <span class="hiw-int-btn-demo dismiss">That helped, keep original</span>
          <span class="hiw-int-btn-demo use">I still need help →</span>
        </div>
      </div>

      <div class="hiw-int-card-col">

        <div class="hiw-int-card">
          <span class="hiw-int-level">Level 2 — Step 2</span>
          <div class="hiw-int-title">Moderate rewrite</div>
          <div class="hiw-int-desc">
            Triggered when genre difficulty is 0.5 to 0.7. The paragraph is rewritten with simpler vocabulary and shorter sentences. The original structure and order of ideas is preserved — it's the same paragraph, easier to read.
          </div>
          <div class="hiw-int-example">
            The Fed decided to keep interest rates the same, even though economic signals suggest a recession may be coming. They're accepting that risk because they want inflation to keep falling. Markets had expected a rate cut, so this decision surprised many investors.
          </div>
        </div>

        <div class="hiw-int-card strong">
          <span class="hiw-int-level">Level 3 — Step 2</span>
          <div class="hiw-int-title">Full reconstruction</div>
          <div class="hiw-int-desc">
            Triggered when genre difficulty is above 0.7 — dense technical or academic content. The paragraph is rebuilt from scratch at roughly an 8th grade reading level. Structure is not preserved. Clarity is the only goal. Level 3 unlocks after you accept 3 rewrites.
          </div>
          <div class="hiw-int-example">
            The Fed kept rates high on purpose. They know this raises the chance of a recession, but they think that's worth it to get inflation fully under control. Investors were caught off guard — most had expected a cut.
          </div>
        </div>

      </div>

    </div>

    <div class="hiw-divider"></div>

    <!-- ── Annotation mode ──────────────────────────────────────────────── -->
    <div class="hiw-section-label">Annotation mode</div>
    <div class="hiw-annotation-block">
      <div class="hiw-annotation-text">
        <p class="hiw-annotation-desc">
          Separate from the intervention system, annotation mode runs on your terms. Toggle it on in the sidebar before or during any session. After you complete each paragraph, AIRIA scans it for domain-specific terms a general reader might not know and highlights them inline with plain-language definitions you can expand on hover.
        </p>
        <p class="hiw-annotation-desc">
          Annotation is not triggered by the SNN — it fires on every paragraph when the mode is on, regardless of whether you struggled. It's designed for articles where you want richer comprehension, not just to get through.
        </p>
        <div class="hiw-annotation-data">
          <div class="hiw-data-point">
            <span class="hiw-data-label">What gets collected</span>
            <span class="hiw-data-val">Which terms were flagged, in which genres, across which articles — stored in your browser as <span class="hiw-key-tag">airia_annotation_log</span></span>
          </div>
          <div class="hiw-data-point">
            <span class="hiw-data-label">What it tells you</span>
            <span class="hiw-data-val">Over time, your stats page shows annotation density by genre — how many domain terms per paragraph each subject area produces. If that number drops across sessions, you're building vocabulary.</span>
          </div>
        </div>
      </div>
      <div class="hiw-annotation-example">
        <p class="hiw-annot-para">
          The Fed's decision to maintain the federal funds rate amid
          <span class="annotated-term-demo">yield curve inversion<span class="annot-tooltip-demo">When short-term bonds pay more interest than long-term ones — often signals a coming recession.</span></span>
          signals a deliberate tolerance for recession risk in exchange for sustained
          <span class="annotated-term-demo">disinflation<span class="annot-tooltip-demo">A slowdown in the rate of inflation — prices still rising, but more slowly.</span></span>.
        </p>
        <p class="hiw-annot-caption">Hover the highlighted terms to expand definitions.</p>
      </div>
    </div>

    <div class="hiw-divider"></div>

    <!-- ── Privacy ──────────────────────────────────────────────────────── -->
    <div class="hiw-privacy">
      <div class="hiw-privacy-icon">⊘</div>
      <div>
        <div class="hiw-privacy-title">Your data never leaves your browser</div>
        <div class="hiw-privacy-text">Training samples, model weights, article history, intervention logs, and annotation logs are stored only in your browser's localStorage. The backend is stateless compute — it receives data, processes it, returns a result, and immediately forgets everything. Nothing is logged or stored server-side. Clearing your browser data resets everything.</div>
        <div class="hiw-privacy-keys">
          <span class="hiw-key-tag">airia_training_samples</span>
          <span class="hiw-key-tag">airia_model_weights</span>
          <span class="hiw-key-tag">airia_articles</span>
          <span class="hiw-key-tag">airia_intervention_log</span>
          <span class="hiw-key-tag">airia_annotation_log</span>
          <span class="hiw-key-tag">airia_unlock_status</span>
        </div>
      </div>
    </div>

  </div>
</div>