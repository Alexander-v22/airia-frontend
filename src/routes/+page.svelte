<script>
    import { onMount } from 'svelte';
    import { texts } from '$lib/texts/index.js';

    // ─── localStorage helpers ────────────────────────────────────────────────
    function lsGet(key, fallback) {
        try {
            const raw = localStorage.getItem(key);
            return raw ? JSON.parse(raw) : fallback;
        } catch { return fallback; }
    }
    function lsSet(key, value) {
        try { localStorage.setItem(key, JSON.stringify(value)); } catch {}
    }
    function lsGetRaw(key) {
        try { return localStorage.getItem(key); } catch { return null; }
    }
    function lsSetRaw(key, value) {
        try { localStorage.setItem(key, value); } catch {}
    }

    // ─── Persistent state ────────────────────────────────────────────────────
    let trainingSamples = lsGet('airia_training_samples', []);
    let articles        = lsGet('airia_articles', []);

    function getTrainingStats() {
        const counts = { too_hard: 0, just_right: 0, too_easy: 0 };
        const labels = { 0: 'too_hard', 1: 'just_right', 2: 'too_easy' };
        for (const s of trainingSamples) {
            const name = labels[s.label];
            if (name) counts[name]++;
        }
        return {
            total_samples: trainingSamples.length,
            samples_by_class: counts,
            oldest_sample: trainingSamples[0]?.timestamp ?? null,
            newest_sample: trainingSamples[trainingSamples.length - 1]?.timestamp ?? null
        };
    }

    // ─── Reading state ───────────────────────────────────────────────────────
    let mode             = $state("focused");
    let currentParagraph = $state(0);
    let urlInput         = $state("");
    let isLoadingUrl     = $state(false);
    let urlError         = $state("");
    let customText       = $state(null);
    let reading          = $state(false);

    function getParagraphs() {
        return customText?.content.split('\n\n') ?? [];
    }

    // ─── Timing ──────────────────────────────────────────────────────────────
    let paragraphStartTime = $state(Date.now());
    let sessionStartTime   = $state(Date.now());
    let elapsedTime        = $state(0);
    let paragraphMetrics   = $state([]);
    let backPresses        = $state(0);
    let blurCount          = $state(0);

    function getCurrentWPM() {
        if (elapsedTime < 1) return 0;
        const wordCount = getParagraphs()[currentParagraph]?.split(/\s+/).length || 0;
        return Math.round((wordCount / elapsedTime) * 60);
    }

    function getSessionStats() {
        const completed = paragraphMetrics.filter(m => m);
        if (!completed.length) return { avgWpm: 0, variance: 0, totalTime: 0 };
        const wpms   = completed.map(m => m.wpm);
        const avgWpm = wpms.reduce((a, b) => a + b, 0) / wpms.length;
        const variance = Math.sqrt(wpms.map(w => Math.pow(w - avgWpm, 2)).reduce((a, b) => a + b, 0) / wpms.length);
        const totalTime = completed.reduce((a, m) => a + m.timeSeconds, 0);
        return { avgWpm: Math.round(avgWpm), variance: Math.round(variance), totalTime: Math.round(totalTime) };
    }

    function getNormalized() {
        const stats     = getSessionStats();
        const completed = paragraphMetrics.filter(m => m);
        let slowdownRatio = 1;
        if (completed.length > 0 && stats.avgWpm > 0) {
            const slowest = Math.min(...completed.map(m => m.wpm));
            slowdownRatio = slowest / stats.avgWpm;
        }
        return {
            avg_wpm:         Math.min(stats.avgWpm / 500, 1),
            wpm_variance:    Math.min(stats.variance / 200, 1),
            back_presses:    Math.min(backPresses / 10, 1),
            completion_rate: getParagraphs().length > 0 ? completed.length / getParagraphs().length : 0,
            slowdown_ratio:  slowdownRatio,
            blur_count:      Math.min(blurCount / 5, 1)
        };
    }

    // ─── SNN membrane state ──────────────────────────────────────────────────
    let sessionMem1    = $state(null);
    let sessionMem2    = $state(null);
    let sessionMem3    = $state(null);
    let membraneCharge = $state(0);

    // ─── Intervention state ──────────────────────────────────────────────────
    let interventionText      = $state(null);
    let interventionParagraph = $state(null);
    let isLoadingIntervention = $state(false);
    let usingIntervention     = $state(false);
    let interventionLog       = $state([]);

    // ─── Pending article ─────────────────────────────────────────────────────
    let pendingArticleMetadata = $state(null);

    // ─── Training UI ─────────────────────────────────────────────────────────
    let prediction      = $state("keep_same");
    let feedbackGiven   = $state(false);
    let lastFeedback    = $state(null);
    let isRetraining    = $state(false);
    let sessionComplete = $state(false);
    let trainingStats   = $state(getTrainingStats());
    let sampleCount     = $state(trainingSamples.length);

    // ─── Timer ───────────────────────────────────────────────────────────────
    let timerInterval;

    function startTimer() {
        paragraphStartTime = Date.now();
        elapsedTime = 0;
        reading = false;
        clearInterval(timerInterval);
    }

    function beginReading() {
        if (reading) return;
        reading = true;
        paragraphStartTime = Date.now();
        timerInterval = setInterval(() => {
            elapsedTime = (Date.now() - paragraphStartTime) / 1000;
        }, 100);
    }

    function pauseReading() { reading = false; clearInterval(timerInterval); }
    function stopTimer()    { clearInterval(timerInterval); }

    function recordParagraphMetrics() {
        const timeSpent = (Date.now() - paragraphStartTime) / 1000;
        if (timeSpent >= 1) {
            const wordCount = getParagraphs()[currentParagraph].split(/\s+/).length;
            paragraphMetrics[currentParagraph] = {
                wpm: Math.round((wordCount / timeSpent) * 60),
                timeSeconds: Math.round(timeSpent * 10) / 10,
                wordCount
            };
        }
    }

    // ─── SNN per-paragraph inference ─────────────────────────────────────────
    async function runParagraphSNN(paragraphIndex) {
        try {
            const weights  = lsGetRaw('airia_model_weights');
            const features = getNormalized();
            const payload  = {
                features: {
                    avg_wpm:         features.avg_wpm,
                    wpm_variance:    features.wpm_variance,
                    back_presses:    features.back_presses,
                    completion_rate: features.completion_rate,
                    slowdown_ratio:  features.slowdown_ratio,
                    blur_count:      features.blur_count
                },
                mem1: sessionMem1,
                mem2: sessionMem2,
                mem3: sessionMem3
            };
            if (weights) payload.weights = weights;

            const res  = await fetch('https://web-production-a3b6.up.railway.app/predict-paragraph', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });
            const data = await res.json();
            sessionMem1    = data.mem1;
            sessionMem2    = data.mem2;
            sessionMem3    = data.mem3;
            membraneCharge = data.membrane_charge;
            if (data.spiked && data.spike_class === 'too_hard') {
                await fetchIntervention(paragraphIndex);
            }
        } catch { console.log('SNN paragraph inference failed'); }
    }

    // ─── Intervention ─────────────────────────────────────────────────────────
    async function fetchIntervention(paragraphIndex) {
        isLoadingIntervention = true;
        interventionText      = null;
        interventionParagraph = paragraphIndex;
        try {
            const paragraphText   = getParagraphs()[paragraphIndex];
            const genreDifficulty = customText?.classification?.genre_difficulty ?? 0.5;
            const specificGenre   = customText?.classification?.specific_genre ?? 'general';
            const res  = await fetch('https://web-production-a3b6.up.railway.app/intervene', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ paragraph: paragraphText, genre_difficulty: genreDifficulty, specific_genre: specificGenre })
            });
            const data = await res.json();
            interventionText = data.rewritten;
            interventionLog  = [...interventionLog, { paragraphIndex, level: data.level, timestamp: new Date().toISOString() }];
        } catch { console.log('Intervention fetch failed'); }
        isLoadingIntervention = false;
    }

    function dismissIntervention() { interventionText = null; interventionParagraph = null; usingIntervention = false; }
    function useIntervention()     { usingIntervention = true; }

    // ─── Navigation ──────────────────────────────────────────────────────────
    async function nextParagraph() {
        recordParagraphMetrics();
        dismissIntervention();
        if (currentParagraph < getParagraphs().length - 1) {
            await runParagraphSNN(currentParagraph);
            currentParagraph++;
            startTimer();
        } else {
            stopTimer();
            sessionComplete = true;
        }
    }

    function prevParagraph() {
        if (currentParagraph > 0) {
            recordParagraphMetrics();
            dismissIntervention();
            backPresses++;
            currentParagraph--;
            startTimer();
        }
    }

    function toggleMode() {
        if (mode === "focused") { pauseReading(); mode = "fulltext"; }
        else { mode = "focused"; }
    }

    function jumpToParagraph(index) {
        if (mode === "fulltext") { currentParagraph = index; mode = "focused"; startTimer(); }
    }

    function handleVisibilityChange() { if (document.hidden) blurCount++; }

    // ─── End-of-session prediction ───────────────────────────────────────────
    async function sendToBackend() {
        try {
            const weights = lsGetRaw('airia_model_weights');
            const payload = { ...getNormalized() };
            if (weights) payload.weights = weights;
            const res  = await fetch('https://web-production-a3b6.up.railway.app/predict', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });
            const data = await res.json();
            prediction    = data.action;
            feedbackGiven = false;
            lastFeedback  = null;
        } catch { console.log('Backend not connected'); }
    }

    // ─── Feedback ────────────────────────────────────────────────────────────
    function submitFeedback(feedback) {
        const sample = {
            features:      Object.values(getNormalized()),
            label:         feedback,
            timestamp:     new Date().toISOString(),
            article_title: customText?.title ?? 'Unknown'
        };
        trainingSamples = [...trainingSamples, sample];
        lsSet('airia_training_samples', trainingSamples);

        if (pendingArticleMetadata) {
            articles = [...articles, pendingArticleMetadata];
            lsSet('airia_articles', articles);
            pendingArticleMetadata = null;
        }

        sampleCount   = trainingSamples.length;
        trainingStats = getTrainingStats();
        feedbackGiven = true;
        lastFeedback  = feedback;
        if (sampleCount >= 10 && sampleCount % 5 === 0) retrainModel();
    }

    // ─── Retrain ─────────────────────────────────────────────────────────────
    async function retrainModel() {
        if (trainingSamples.length < 10) return;
        isRetraining = true;
        try {
            const weights = lsGetRaw('airia_model_weights');
            const payload = { samples: trainingSamples };
            if (weights) payload.weights = weights;
            const res    = await fetch('https://web-production-a3b6.up.railway.app/retrain', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });
            const result = await res.json();
            if (result.weights) lsSetRaw('airia_model_weights', result.weights);
        } catch { console.log('Failed to retrain'); }
        isRetraining = false;
    }

    // ─── Reset ───────────────────────────────────────────────────────────────
    function resetTrainingData() {
        if (!confirm('This will delete all your training data and model weights. Continue?')) return;
        trainingSamples = [];
        lsSet('airia_training_samples', []);
        localStorage.removeItem('airia_model_weights');
        sampleCount   = 0;
        trainingStats = getTrainingStats();
    }

    function resetSession() {
        currentParagraph      = 0;
        paragraphMetrics      = [];
        backPresses           = 0;
        blurCount             = 0;
        sessionComplete       = false;
        feedbackGiven         = false;
        lastFeedback          = null;
        prediction            = "keep_same";
        sessionStartTime      = Date.now();
        sessionMem1           = null;
        sessionMem2           = null;
        sessionMem3           = null;
        membraneCharge        = 0;
        interventionText      = null;
        interventionParagraph = null;
        usingIntervention     = false;
        interventionLog       = [];
        startTimer();
    }

    function formatTime(seconds) {
        if (!seconds || isNaN(seconds)) return "0:00";
        return `${Math.floor(seconds / 60)}:${Math.floor(seconds % 60).toString().padStart(2, '0')}`;
    }

    // ─── URL ingestion ────────────────────────────────────────────────────────
    async function loadArticleFromUrl() {
        if (!urlInput.trim()) { urlError = "Please enter a URL"; return; }
        isLoadingUrl = true;
        urlError     = "";
        try {
            const res  = await fetch('https://web-production-a3b6.up.railway.app/ingest-url', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ url: urlInput.trim() })
            });
            const data = await res.json();
            if (data.status === "error") { urlError = data.content; isLoadingUrl = false; return; }
            customText = {
                title:          data.title,
                content:        data.content,
                level:          `${data.estimated_lexile}L (estimated)`,
                lexile:         data.estimated_lexile,
                classification: data.classification
            };
            pendingArticleMetadata = {
                id:               `article_${Date.now()}`,
                url:              urlInput.trim(),
                title:            data.title,
                estimated_lexile: data.estimated_lexile,
                word_count:       data.word_count,
                paragraph_count:  data.paragraph_count,
                classification:   data.classification,
                timestamp:        new Date().toISOString()
            };
            resetSession();
            urlInput = "";
        } catch { urlError = "Failed to load article. Is backend running?"; }
        isLoadingUrl = false;
    }

    function clearArticle() {
        customText             = null;
        pendingArticleMetadata = null;
        resetSession();
    }

    // ─── onMount ──────────────────────────────────────────────────────────────
    onMount(async () => {
        document.addEventListener('visibilitychange', handleVisibilityChange);
        startTimer();
        initCanvas();

        const existingWeights = lsGetRaw('airia_model_weights');
        if (!existingWeights) {
            try {
                const res  = await fetch('https://web-production-a3b6.up.railway.app/base-weights');
                const data = await res.json();
                if (data.weights) lsSetRaw('airia_model_weights', data.weights);
            } catch {}
        }

        return () => {
            document.removeEventListener('visibilitychange', handleVisibilityChange);
            stopTimer();
            if (animFrame) cancelAnimationFrame(animFrame);
        };
    });

    // ─── Neural network background canvas ────────────────────────────────────
    let canvas;
    let animFrame;

    function initCanvas() {
        if (!canvas) return;
        const ctx = canvas.getContext('2d');
        const W   = canvas.width  = window.innerWidth;
        const H   = canvas.height = window.innerHeight;

        // Nodes
        const NODE_COUNT = 42;
        const nodes = Array.from({ length: NODE_COUNT }, () => ({
            x:    Math.random() * W,
            y:    Math.random() * H,
            vx:   (Math.random() - 0.5) * 0.18,
            vy:   (Math.random() - 0.5) * 0.18,
            r:    1.5 + Math.random() * 2,
            pulse: Math.random() * Math.PI * 2,   // phase offset
            pulseSpeed: 0.012 + Math.random() * 0.018
        }));

        // Connections: only pairs within threshold distance
        const THRESHOLD = 180;

        function draw(t) {
            ctx.clearRect(0, 0, W, H);

            // Update positions
            for (const n of nodes) {
                n.x += n.vx;
                n.y += n.vy;
                if (n.x < 0 || n.x > W) n.vx *= -1;
                if (n.y < 0 || n.y > H) n.vy *= -1;
                n.pulse += n.pulseSpeed;
            }

            // Draw connections
            for (let i = 0; i < nodes.length; i++) {
                for (let j = i + 1; j < nodes.length; j++) {
                    const a  = nodes[i];
                    const b  = nodes[j];
                    const dx = a.x - b.x;
                    const dy = a.y - b.y;
                    const d  = Math.sqrt(dx * dx + dy * dy);
                    if (d < THRESHOLD) {
                        // flicker: combine both nodes' pulses
                        const flicker = (Math.sin(a.pulse) + Math.sin(b.pulse)) * 0.25 + 0.5;
                        const alpha   = (1 - d / THRESHOLD) * 0.09 * flicker;
                        ctx.strokeStyle = `rgba(0,0,0,${alpha})`;
                        ctx.lineWidth   = 0.7;
                        ctx.beginPath();
                        ctx.moveTo(a.x, a.y);
                        ctx.lineTo(b.x, b.y);
                        ctx.stroke();
                    }
                }
            }

            // Draw nodes
            for (const n of nodes) {
                const glow  = (Math.sin(n.pulse) * 0.5 + 0.5);
                const alpha = 0.12 + glow * 0.22;
                const r     = n.r + glow * 1.2;
                ctx.beginPath();
                ctx.arc(n.x, n.y, r, 0, Math.PI * 2);
                ctx.fillStyle = `rgba(0,0,0,${alpha})`;
                ctx.fill();
            }

            animFrame = requestAnimationFrame(draw);
        }

        animFrame = requestAnimationFrame(draw);
    }

    // ─── Derived ─────────────────────────────────────────────────────────────
    let membranePercent     = $derived(Math.round(membraneCharge * 100));
    let activeParagraphText = $derived(
        usingIntervention && interventionParagraph === currentParagraph
            ? interventionText
            : getParagraphs()[currentParagraph]
    );
</script>

<!-- ══════════════════════════════════════════════════════════════════
     TOP NAV — always visible
     ══════════════════════════════════════════════════════════════════ -->
<nav class="topnav">
    <span class="topnav-logo">AIRIA</span>
    <div class="topnav-links">
        <a href="/stats"    class="topnav-link">Your data</a>
        <a href="/infopage" class="topnav-link">How it works</a>
    </div>
</nav>

{#if !customText}
<!-- ══════════════════════════════════════════════════════════════════
     HERO — shown when no article is loaded
     ══════════════════════════════════════════════════════════════════ -->
<div class="hero">
    <canvas bind:this={canvas} class="hero-canvas"></canvas>

    <div class="hero-content">
        <h1 class="hero-logo">AIRIA</h1>
        <p class="hero-tagline">Adaptive reading, powered by a spiking neural network.</p>

        <!-- URL loader -->
        <div class="hero-url-row">
            <input
                type="text"
                placeholder="Paste an article URL to begin"
                bind:value={urlInput}
                class="hero-url-input"
                disabled={isLoadingUrl}
                onkeydown={(e) => e.key === 'Enter' && loadArticleFromUrl()}
            />
            <button class="hero-url-btn" onclick={loadArticleFromUrl} disabled={isLoadingUrl}>
                {isLoadingUrl ? '…' : '→'}
            </button>
        </div>
        {#if urlError}
            <p class="hero-url-error">{urlError}</p>
        {/if}

        <!-- Steps -->
        <div class="hero-steps">
            <div class="hero-step">
                <span class="hero-step-num">01</span>
                <span class="hero-step-title">Load an article</span>
                <span class="hero-step-desc">Paste any URL above. AIRIA extracts the text and classifies its genre and difficulty before you read a word.</span>
            </div>
            <div class="hero-step">
                <span class="hero-step-num">02</span>
                <span class="hero-step-title">Read paragraph by paragraph</span>
                <span class="hero-step-desc">Press Start on each paragraph. A spiking neural network watches your reading speed in real time and intervenes when it detects struggle.</span>
            </div>
            <div class="hero-step">
                <span class="hero-step-num">03</span>
                <span class="hero-step-title">Finish and rate it</span>
                <span class="hero-step-desc">After the last paragraph, tell AIRIA how the text felt. Your rating trains the model so future sessions adapt to you personally.</span>
            </div>
            <div class="hero-step">
                <span class="hero-step-num">04</span>
                <span class="hero-step-title">Watch it personalize</span>
                <span class="hero-step-desc">After 10 sessions, AIRIA retrains on your data. By article 7, interventions fire silently — only when you actually need them.</span>
            </div>
        </div>

        <a href="/infopage" class="hero-learn">Deep dive into how it works →</a>
    </div>
</div>

{:else}
<!-- ══════════════════════════════════════════════════════════════════
     READER — shown when article is loaded
     ══════════════════════════════════════════════════════════════════ -->
<div class="app">

    <!-- Left Sidebar -->
    <aside class="left-sidebar">

        <div class="url-input-section">
            <div class="sidebar-label">Load article</div>
            <div class="url-row">
                <input
                    type="text"
                    placeholder="Paste URL..."
                    bind:value={urlInput}
                    class="url-input"
                    disabled={isLoadingUrl}
                />
                <button class="url-btn" onclick={loadArticleFromUrl} disabled={isLoadingUrl}>
                    {isLoadingUrl ? '…' : '↓'}
                </button>
            </div>
            {#if urlError}<div class="url-error">{urlError}</div>{/if}
            <div class="article-loaded">
                <span>{customText.title.slice(0, 28)}{customText.title.length > 28 ? '…' : ''}</span>
                <button class="clear-btn" onclick={clearArticle}>×</button>
            </div>
        </div>

        <div class="sidebar-divider"></div>

        <div class="mode-row">
            <button class="mode-btn" class:active={mode === "focused"} onclick={() => mode !== "focused" && toggleMode()}>Paragraph</button>
            <button class="mode-btn" class:active={mode === "fulltext"} onclick={() => mode !== "fulltext" && toggleMode()}>Full text</button>
        </div>

        <div class="sidebar-divider"></div>

        <div class="sidebar-label">Current paragraph</div>
        <div class="stat-pair">
            <div class="stat-box">
                <span class="stat-val">{formatTime(elapsedTime)}</span>
                <span class="stat-key">time</span>
            </div>
            <div class="stat-box">
                <span class="stat-val">{getCurrentWPM()}</span>
                <span class="stat-key">WPM</span>
            </div>
        </div>

        <div class="sidebar-divider"></div>

        <div class="sidebar-label">Session</div>
        <div class="metric-list">
            <div class="metric-row"><span class="metric-key">Avg WPM</span><span class="metric-val">{getSessionStats().avgWpm || 0}</span></div>
            <div class="metric-row"><span class="metric-key">Std dev</span><span class="metric-val">{getSessionStats().variance || 0}</span></div>
            <div class="metric-row"><span class="metric-key">Back presses</span><span class="metric-val">{backPresses}</span></div>
            <div class="metric-row"><span class="metric-key">Tab away</span><span class="metric-val">{blurCount}</span></div>
        </div>

        <div class="sidebar-divider"></div>

        <div class="sidebar-label">SNN membrane</div>
        <div class="membrane-row">
            <div class="membrane-track">
                <div class="membrane-fill" style="width: {membranePercent}%"></div>
            </div>
            <span class="membrane-val">{membranePercent}%</span>
        </div>
        <div class="membrane-labels"><span>calm</span><span>spike</span></div>

        <div class="sidebar-divider"></div>

        <div class="sidebar-label">Progress</div>
        <div class="progress-track">
            {#each getParagraphs() as _, i}
                <div
                    class="progress-seg"
                    class:done={paragraphMetrics[i]}
                    class:curr={i === currentParagraph && !sessionComplete}
                    class:intervention={interventionLog.some(l => l.paragraphIndex === i)}
                    onclick={() => jumpToParagraph(i)}
                    title={paragraphMetrics[i] ? `${paragraphMetrics[i].wpm} WPM` : 'Not read'}
                ></div>
            {/each}
        </div>

    </aside>

    <!-- Main -->
    <main>
        <header class="article-header">
            <div class="article-meta">
                <span class="meta-tag dark">{customText.classification?.broad_genre ?? 'article'}</span>
                <span class="meta-tag">{customText.lexile}L</span>
                {#if customText.classification?.specific_genre}
                    <span class="meta-tag">{customText.classification.specific_genre}</span>
                {/if}
            </div>
            <h1 class="article-title">{customText.title}</h1>
        </header>

        {#if mode === "focused"}
            <div class="focused-reader">
                <div class="para-num">{String(currentParagraph + 1).padStart(2, '0')} / {String(getParagraphs().length).padStart(2, '0')}</div>

                <div class="para-card">
                    <p class="para-text">{activeParagraphText}</p>
                    <div class="para-footer">
                        <div class="wpm-display">
                            {#if paragraphMetrics[currentParagraph]}
                                <span class="wpm-number">{paragraphMetrics[currentParagraph].wpm}</span>
                                <span class="wpm-label">WPM</span>
                            {:else if reading}
                                <span class="wpm-number">{getCurrentWPM()}</span>
                                <span class="wpm-label">WPM live</span>
                            {/if}
                        </div>
                        {#if usingIntervention && interventionParagraph === currentParagraph}
                            <span class="using-badge">using simplified</span>
                        {/if}
                    </div>
                </div>

                {#if isLoadingIntervention}
                    <div class="intervention-loading">
                        <span class="int-loading-text">AIRIA is analyzing this paragraph…</span>
                    </div>
                {/if}

                {#if interventionText && interventionParagraph === currentParagraph && !usingIntervention}
                    <div class="intervention-panel">
                        <div class="int-header">
                            <span class="int-badge">AIRIA suggestion</span>
                            <button class="int-dismiss" onclick={dismissIntervention}>dismiss ×</button>
                        </div>
                        <p class="int-label">Simplified version of this paragraph</p>
                        <p class="int-text">{interventionText}</p>
                        <div class="int-actions">
                            <button class="int-btn-use" onclick={useIntervention}>Use this version</button>
                            <button class="int-btn-skip" onclick={dismissIntervention}>Keep original</button>
                        </div>
                    </div>
                {/if}

                <div class="nav-row">
                    <button class="nav-btn" onclick={prevParagraph} disabled={currentParagraph === 0}>← Back</button>
                    {#if !reading}
                        <button class="nav-btn primary" onclick={beginReading}>Start</button>
                    {:else}
                        <button class="nav-btn" onclick={pauseReading}>Pause</button>
                    {/if}
                    <button class="nav-btn primary" onclick={nextParagraph} disabled={!reading && elapsedTime === 0}>
                        {currentParagraph === getParagraphs().length - 1 ? 'Finish' : 'Next →'}
                    </button>
                </div>
            </div>

        {:else}
            <div class="fulltext-reader">
                {#each getParagraphs() as p, i}
                    <p
                        class="fulltext-para"
                        class:current={i === currentParagraph}
                        class:completed={paragraphMetrics[i]}
                        onclick={() => jumpToParagraph(i)}
                    >
                        {p}
                        {#if paragraphMetrics[i]}
                            <span class="ft-wpm">{paragraphMetrics[i].wpm} WPM</span>
                        {/if}
                    </p>
                {/each}
            </div>
        {/if}
    </main>

    <!-- Right Sidebar -->
    <aside class="right-sidebar">
        <div class="sidebar-label" style="margin-bottom: 16px;">Training</div>

        {#if sessionComplete}
            <div class="complete-badge">Session complete</div>

            <div class="final-stats">
                <div class="final-row"><span class="final-key">Avg WPM</span><span class="final-val">{getSessionStats().avgWpm || 0}</span></div>
                <div class="final-row"><span class="final-key">Total time</span><span class="final-val">{formatTime(getSessionStats().totalTime)}</span></div>
                <div class="final-row"><span class="final-key">Back presses</span><span class="final-val">{backPresses}</span></div>
                {#if interventionLog.length > 0}
                    <div class="final-row"><span class="final-key">Interventions</span><span class="final-val">{interventionLog.length}</span></div>
                {/if}
            </div>

            <button class="analyze-btn" onclick={sendToBackend}>Analyze with SNN</button>

            {#if prediction !== "keep_same"}
                <div class="prediction-box">
                    <span class="prediction-label">Prediction</span>
                    <span class="prediction-val">{prediction.replace('_', ' ')}</span>
                </div>
            {/if}

            <div class="sidebar-divider"></div>

            <div class="sidebar-label" style="margin-bottom: 10px;">How was this text?</div>
            <div class="feedback-col">
                <button class="fb-btn" class:selected={lastFeedback === 0} onclick={() => submitFeedback(0)} disabled={feedbackGiven}>Too hard</button>
                <button class="fb-btn" class:selected={lastFeedback === 1} onclick={() => submitFeedback(1)} disabled={feedbackGiven}>Just right</button>
                <button class="fb-btn" class:selected={lastFeedback === 2} onclick={() => submitFeedback(2)} disabled={feedbackGiven}>Too easy</button>
            </div>
            {#if feedbackGiven}<p class="feedback-confirm">Saved to your browser</p>{/if}

            <button class="secondary-btn" onclick={resetSession}>Read another</button>

        {:else}
            <div class="training-block">
                <div class="metric-list">
                    <div class="metric-row"><span class="metric-key">Total samples</span><span class="metric-val">{sampleCount}</span></div>
                    <div class="metric-row"><span class="metric-key">Too hard</span><span class="metric-val">{trainingStats.samples_by_class.too_hard}</span></div>
                    <div class="metric-row"><span class="metric-key">Just right</span><span class="metric-val">{trainingStats.samples_by_class.just_right}</span></div>
                    <div class="metric-row"><span class="metric-key">Too easy</span><span class="metric-val">{trainingStats.samples_by_class.too_easy}</span></div>
                </div>

                <div class="sidebar-divider"></div>

                {#if sampleCount >= 10}
                    <div class="ready-badge">Model can be personalized</div>
                {:else}
                    <div class="sample-bar-track">
                        <div class="sample-bar-fill" style="width: {(sampleCount / 10) * 100}%"></div>
                    </div>
                    <p class="samples-needed">{10 - sampleCount} more to personalize</p>
                {/if}

                <button class="retrain-btn" onclick={retrainModel} disabled={sampleCount < 10 || isRetraining}>
                    {isRetraining ? 'Retraining…' : 'Retrain model'}
                </button>
                <button class="reset-btn" onclick={resetTrainingData}>Reset all data</button>
            </div>
        {/if}
    </aside>

</div>
{/if}
