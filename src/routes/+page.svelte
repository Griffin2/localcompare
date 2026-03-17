<script>
  import TrustBar from "$lib/TrustBar.svelte";
  import ToolHeader from "$lib/ToolHeader.svelte";
  import ToolFooter from "$lib/ToolFooter.svelte";

  let original = $state("");
  let modified = $state("");
  let diffResult = $state([]);
  let stats = $state(null);
  let copied = $state(false);
  let ignoreWhitespace = $state(false);
  let contextOnly = $state(false);
  let contextLines = 3;

  function lcs(a, b) {
    const m = a.length,
      n = b.length;
    const dp = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));
    for (let i = 1; i <= m; i++) {
      for (let j = 1; j <= n; j++) {
        dp[i][j] =
          a[i - 1] === b[j - 1]
            ? dp[i - 1][j - 1] + 1
            : Math.max(dp[i - 1][j], dp[i][j - 1]);
      }
    }
    const result = [];
    let i = m,
      j = n;
    while (i > 0 && j > 0) {
      if (a[i - 1] === b[j - 1]) {
        result.unshift({ type: "equal", value: a[i - 1] });
        i--;
        j--;
      } else if (dp[i - 1][j] >= dp[i][j - 1]) {
        result.unshift({ type: "removed", value: a[i - 1] });
        i--;
      } else {
        result.unshift({ type: "added", value: b[j - 1] });
        j--;
      }
    }
    while (i > 0) {
      result.unshift({ type: "removed", value: a[--i] });
    }
    while (j > 0) {
      result.unshift({ type: "added", value: b[--j] });
    }
    return result;
  }

  function computeDiff() {
    if (original.trim() === "" && modified.trim() === "") {
      diffResult = [];
      stats = null;
      return;
    }

    let origLines = original.split("\n");
    let modLines = modified.split("\n");

    if (ignoreWhitespace) {
      const norm = (s) => s.replace(/\s+/g, " ").trim();
      const origNorm = origLines.map(norm);
      const modNorm = modLines.map(norm);
      const raw = lcs(origNorm, modNorm);
      let oi = 0,
        mi = 0;
      diffResult = raw.map((d) => {
        if (d.type === "equal")
          return { ...d, value: origLines[oi++], modIndex: mi++ };
        if (d.type === "removed") return { ...d, value: origLines[oi++] };
        return { ...d, value: modLines[mi++] };
      });
    } else {
      diffResult = lcs(origLines, modLines);
    }

    let added = 0,
      removed = 0,
      unchanged = 0;
    for (const d of diffResult) {
      if (d.type === "added") added++;
      else if (d.type === "removed") removed++;
      else unchanged++;
    }
    stats = { added, removed, unchanged };
  }

  function filteredDiff() {
    if (!contextOnly || diffResult.length === 0) return diffResult;
    const show = new Set();
    for (let i = 0; i < diffResult.length; i++) {
      if (diffResult[i].type !== "equal") {
        for (
          let j = Math.max(0, i - contextLines);
          j <= Math.min(diffResult.length - 1, i + contextLines);
          j++
        ) {
          show.add(j);
        }
      }
    }
    const result = [];
    let lastShown = -1;
    for (let i = 0; i < diffResult.length; i++) {
      if (show.has(i)) {
        if (lastShown !== -1 && i - lastShown > 1) {
          const skipped = i - lastShown - 1;
          result.push({
            type: "separator",
            value: `··· ${skipped} unchanged line${skipped > 1 ? "s" : ""} ···`,
          });
        }
        result.push(diffResult[i]);
        lastShown = i;
      }
    }
    return result;
  }

  function copyDiff() {
    if (diffResult.length === 0) return;
    const text = diffResult
      .map((d) => {
        if (d.type === "added") return `+ ${d.value}`;
        if (d.type === "removed") return `- ${d.value}`;
        return `  ${d.value}`;
      })
      .join("\n");
    navigator.clipboard
      .writeText(text)
      .then(() => {
        copied = true;
        setTimeout(() => (copied = false), 1500);
      })
      .catch(() => {});
  }

  function clear() {
    original = "";
    modified = "";
    diffResult = [];
    stats = null;
    copied = false;
  }

  function swap() {
    const tmp = original;
    original = modified;
    modified = tmp;
    computeDiff();
  }

  function loadSample() {
    original = `function greet(name) {
  console.log("Hello, " + name);
  return true;
}

const users = ["Alice", "Bob"];
users.forEach(greet);`;

    modified = `function greet(name, greeting = "Hello") {
  console.log(greeting + ", " + name);
  return true;
}

const users = ["Alice", "Bob", "Charlie"];
users.forEach((u) => greet(u));`;

    computeDiff();
  }

  let displayDiff = $derived(filteredDiff());
</script>

<TrustBar sourceUrl="https://github.com/Griffin2/localcompare" />

<div class="page-shell">
  <ToolHeader
    title="Text Diff"
    description="Compare two texts side by side — nothing leaves your browser."
  />

  <div class="tool-area">
    <div class="panel">
      <label class="tool-label" for="original">original</label>
      <textarea
        id="original"
        class="tool-textarea"
        bind:value={original}
        placeholder="Paste original text..."
        spellcheck="false"
      ></textarea>
    </div>

    <div class="panel">
      <label class="tool-label" for="modified">modified</label>
      <textarea
        id="modified"
        class="tool-textarea"
        bind:value={modified}
        placeholder="Paste modified text..."
        spellcheck="false"
      ></textarea>
    </div>
  </div>

  <div class="controls">
    <button class="btn btn-primary" onclick={computeDiff}>Compare</button>
    <button class="btn btn-swap" onclick={swap} title="Swap inputs"
      >⇄ Swap</button
    >
    <button class="btn" onclick={copyDiff}>
      {copied ? "✓ Copied" : "Copy diff"}
    </button>
    <button class="btn" onclick={clear}>Clear</button>
    <button class="btn btn-sample" onclick={loadSample}>Sample</button>

    <div class="options">
      <label class="option-label">
        <input
          type="checkbox"
          bind:checked={ignoreWhitespace}
          onchange={computeDiff}
        />
        <span>Ignore whitespace</span>
      </label>
      <label class="option-label">
        <input type="checkbox" bind:checked={contextOnly} />
        <span>Context only</span>
      </label>
    </div>

    <div class="spacer"></div>

    {#if stats}
      <div class="stat-pills">
        {#if stats.added > 0}
          <span class="pill pill-added">+{stats.added}</span>
        {/if}
        {#if stats.removed > 0}
          <span class="pill pill-removed">−{stats.removed}</span>
        {/if}
        {#if stats.added === 0 && stats.removed === 0}
          <span class="pill pill-ok">
            <span class="pill-dot"></span>
            identical
          </span>
        {/if}
      </div>
    {/if}
  </div>

  {#if displayDiff.length > 0}
    <div class="diff-output">
      {#each displayDiff as line, i}
        {#if line.type === "separator"}
          <div class="diff-line diff-sep">{line.value}</div>
        {:else}
          <div
            class="diff-line"
            class:diff-added={line.type === "added"}
            class:diff-removed={line.type === "removed"}
          >
            <span class="diff-marker">
              {#if line.type === "added"}+{:else if line.type === "removed"}−{:else}&nbsp;{/if}
            </span>
            <span class="diff-text">{line.value || "\u00A0"}</span>
          </div>
        {/if}
      {/each}
    </div>
  {/if}

  <ToolFooter />
</div>

<style>
  .panel {
    display: flex;
    flex-direction: column;
    gap: 5px;
  }

  .btn-sample {
    font-size: 12px;
    padding: 8px 12px;
    color: var(--text-muted);
  }

  .btn-swap {
    font-size: 12px;
    padding: 8px 12px;
    color: var(--accent);
    border-color: var(--accent-border);
  }

  .btn-swap:hover {
    background: var(--accent-bg);
  }

  .options {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-left: 8px;
  }

  .option-label {
    display: flex;
    align-items: center;
    gap: 5px;
    font-family: var(--font-mono);
    font-size: 12px;
    color: var(--text-muted);
    cursor: pointer;
  }

  .option-label input[type="checkbox"] {
    accent-color: var(--accent);
    cursor: pointer;
  }

  .spacer {
    margin-left: auto;
  }

  .stat-pills {
    display: flex;
    gap: 6px;
  }

  .pill-added {
    font-family: var(--font-mono);
    font-size: 12px;
    font-weight: 600;
    padding: 4px 10px;
    border-radius: 999px;
    background: #ecfdf5;
    color: #16a34a;
  }

  .pill-removed {
    font-family: var(--font-mono);
    font-size: 12px;
    font-weight: 600;
    padding: 4px 10px;
    border-radius: 999px;
    background: #fef2f2;
    color: #dc2626;
  }

  /* ---- diff output ---- */
  .diff-output {
    margin-top: 16px;
    border: 1px solid var(--border-light);
    border-radius: var(--radius-md);
    background: var(--bg-surface);
    overflow: hidden;
    font-family: var(--font-mono);
    font-size: 13px;
    line-height: 1.6;
    max-height: 420px;
    overflow-y: auto;
  }

  .diff-line {
    display: flex;
    padding: 0 12px;
    border-bottom: 1px solid transparent;
  }

  .diff-marker {
    flex-shrink: 0;
    width: 20px;
    text-align: center;
    color: var(--text-muted);
    user-select: none;
  }

  .diff-text {
    flex: 1;
    white-space: pre-wrap;
    word-break: break-all;
    padding-left: 4px;
  }

  .diff-added {
    background: #ecfdf5;
  }
  .diff-added .diff-marker {
    color: #16a34a;
    font-weight: 700;
  }
  .diff-added .diff-text {
    color: #15803d;
  }

  .diff-removed {
    background: #fef2f2;
  }
  .diff-removed .diff-marker {
    color: #dc2626;
    font-weight: 700;
  }
  .diff-removed .diff-text {
    color: #b91c1c;
  }

  .diff-sep {
    justify-content: center;
    color: var(--text-muted);
    font-size: 11px;
    padding: 4px 12px;
    background: var(--bg-input);
    border-bottom: 1px solid var(--border-light);
    border-top: 1px solid var(--border-light);
  }

  @media (max-width: 640px) {
    .options {
      display: none;
    }

    .controls {
      flex-wrap: wrap;
    }
  }
</style>
