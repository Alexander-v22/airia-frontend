<script>
    let { onDone } = $props();

    const WORD = 'AIRIA';
    const SUBTITLE = 'Adaptive reading, powered by a spiking neural network.';
    const LETTER_STAGGER = 120;
    const CHAR_STAGGER = 25;
    const HOLD = 300;
    const EXIT_DURATION = 900;

    let lettersVisible = $state(0);
    let subtitleChars = $state(0);
    let exiting = $state(false);

    let allLettersVisible = $derived(lettersVisible >= WORD.length);
    let cursorBlinking = $derived(lettersVisible > 0 && !allLettersVisible);

    $effect(() => {
        const timers = [];

        for (let i = 0; i < WORD.length; i++) {
            timers.push(setTimeout(() => {
                lettersVisible = i + 1;
            }, i * LETTER_STAGGER));
        }

        // one stagger gap after the last letter before subtitle begins
        const subtitleStart = WORD.length * LETTER_STAGGER;

        for (let j = 0; j < SUBTITLE.length; j++) {
            timers.push(setTimeout(() => {
                subtitleChars = j + 1;
            }, subtitleStart + j * CHAR_STAGGER));
        }

        const exitStart = subtitleStart + (SUBTITLE.length - 1) * CHAR_STAGGER + HOLD;

        timers.push(setTimeout(() => {
            exiting = true;
        }, exitStart));

        timers.push(setTimeout(() => {
            onDone?.();
        }, exitStart + EXIT_DURATION));

        return () => timers.forEach(clearTimeout);
    });
</script>

<svelte:head>
    <link
        href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:wght@300;400;500&family=Inter:wght@300;400;500&display=swap"
        rel="stylesheet"
    />
</svelte:head>

<div class="outer">
    <div
        class="inner"
        style="transform: {exiting ? 'translateZ(600px)' : 'translateZ(0)'}; opacity: {exiting ? 0 : 1};"
    >
        <div class="logo-line">
            {#each WORD.split('') as letter, i}
                {#if i < lettersVisible}
                    <span class="letter">{letter}</span>
                {/if}
            {/each}
            {#if lettersVisible > 0}
                <span class="cursor" class:blinking={cursorBlinking}>|</span>
            {/if}
        </div>
        <p class="subtitle">{SUBTITLE.slice(0, subtitleChars)}</p>
    </div>
</div>

<style>
    .outer {
        perspective: 800px;
        perspective-origin: center center;
        position: fixed;
        inset: 0;
        z-index: 9999;
    }

    .inner {
        will-change: transform, opacity;
        transform-origin: center center;
        backface-visibility: hidden;
        -webkit-backface-visibility: hidden;
        background: black;
        width: 100%;
        height: 100%;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        transition: transform 900ms cubic-bezier(0.2, 0, 0.8, 1), opacity 900ms cubic-bezier(0.2, 0, 1, 1);
    }

    .logo-line {
        display: flex;
        align-items: baseline;
        letter-spacing: 0.3em;
        font-family: 'Cormorant Garamond', serif;
        font-size: 5rem;
        font-weight: 300;
        color: white;
        line-height: 1;
    }

    .letter {
        display: inline-block;
    }

    .cursor {
        display: inline-block;
        letter-spacing: 0;
        font-family: 'Cormorant Garamond', serif;
        font-size: 5rem;
        font-weight: 300;
        color: white;
    }

    .cursor.blinking {
        animation: blink 0.7s step-start infinite;
    }

    @keyframes blink {
        0%, 100% { opacity: 1; }
        50% { opacity: 0; }
    }

    .subtitle {
        font-family: 'Inter', sans-serif;
        font-size: 0.85rem;
        color: #888;
        font-weight: 300;
        margin-top: 1.5rem;
        min-height: 1.5em;
        white-space: nowrap;
        margin-block: 0;
    }
</style>
