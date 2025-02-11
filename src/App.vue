<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref, useTemplateRef, watch } from "vue"
import { VList } from "virtua/vue"
import { convertToPinyin } from "tiny-pinyin"

const data = reactive({
    keyword: '',
    curr: {},
    list: computed(() => {
        if (data.keyword) {
            return data.raw.filter(({ t, p, py }) =>
                t.includes(data.keyword)
                || p.startsWith(data.keyword.toLowerCase())
                || py.startsWith(data.keyword.toLowerCase())
            )
        }
        return data.raw
    }),
    raw: [],
    full: false,
    theme: 'dark',
    loading: true,
    // Mobile-only: which view takes the full screen ('list' | 'stage')
    mobileView: 'list'
})

// Reactive breakpoint state — mirrors the CSS @media (max-width: 820px), (orientation: portrait)
// Used to disable desktop-only affordances (FULLSCREEN button, double-click fullscreen).
const isNarrow = ref(false)
let mql = null
const updateNarrow = e => {
    isNarrow.value = e.matches
}

const themeIcon = computed(() => data.theme === 'dark' ? 'sun' : 'moon')

const toggleTheme = () => {
    data.theme = data.theme === 'dark' ? 'light' : 'dark'
    document.documentElement.dataset.theme = data.theme
}

const init = async _ => {
    const resp = await fetch('./data/dict.json')
    const list = await resp.json()
    data.raw = list.map(({ i, t, c }) => {
        let pys = convertToPinyin(t, ' ', true).split(' ')
        return { i, t, c, p: pys.map(p => p[0]).join(''), py: pys.join('') }
    })
    data.loading = false
}

const tablature = useTemplateRef('right')
const vListRef = useTemplateRef('vList')

watch(() => data.curr, curr => {
    tablature.value?.scrollTo({
        top: 0,
        behavior: 'smooth'
    })
    // Mirror the selection in the URL hash so the link is shareable.
    syncHash(curr?.i || null)
    // On mobile, picking a song switches the full-screen view to stage
    if (curr && curr.i) {
        data.mobileView = 'stage'
    }
})

const goList = () => { data.mobileView = 'list' }
const goStage = () => { data.mobileView = 'stage' }

// Sync the URL hash with the current track. We use replaceState so
// browsing between songs never bloats browser history.
const syncHash = id => {
    const target = id ? `#/${id}` : location.pathname + location.search
    if (location.hash === (id ? `#/${id}` : '')) return
    const url = id ? `${location.pathname}${location.search}#/${id}` : location.pathname + location.search
    history.replaceState(null, '', url)
}

// Go back to the dashboard (hero) — clears selection and URL hash.
const goHome = () => {
    data.curr = {}
    data.mobileView = 'list'
    history.replaceState(null, '', location.pathname + location.search)
}

// Restore a track selection from a hash like "#/1234".
// Called after data.raw is loaded; no-op if the hash is missing or invalid.
const restoreFromHash = () => {
    const m = location.hash.match(/^#\/(\d+)$/)
    if (!m) return
    const id = Number(m[1])
    const found = data.raw.find(x => x.i === id)
    if (found) {
        data.curr = found
        // Direct link → on mobile we want the reader, not the list
        data.mobileView = 'stage'
        // Bring the corresponding row into view in the sidebar list
        scrollListToId(id)
    }
}

// Scroll the sidebar virtual list so that the row matching `id`
// is visible. Runs after the next render + one animation frame so
// virtua has had a chance to measure item sizes.
const scrollListToId = async id => {
    await nextTick()
    // Give virtua a frame to settle item sizes
    requestAnimationFrame(() => {
        const idx = data.list.findIndex(x => x.i === id)
        if (idx === -1 || !vListRef.value) return
        vListRef.value.scrollToIndex(idx, { align: 'center' })
    })
}

const totalSongs = computed(() => data.raw.length)
const matchedCount = computed(() => data.list.length)
const progressWidth = computed(() => {
    if (!data.curr.c) return '0%'
    return '100%'
})

onMounted(async () => {
    document.documentElement.dataset.theme = data.theme
    mql = window.matchMedia('(max-width: 820px), (orientation: portrait)')
    isNarrow.value = mql.matches
    mql.addEventListener('change', updateNarrow)
    await init()
    // If the page was opened with #/<id>, open that track directly
    restoreFromHash()
})

onBeforeUnmount(() => {
    mql?.removeEventListener('change', updateNarrow)
})
</script>

<template>
    <div class="main" :class="{ full: data.full, light: data.theme === 'light', 'mobile-stage': data.mobileView === 'stage' }">
        <!-- LEFT: Sidebar / Tracklist -->
        <aside class="left ani">
            <header class="brand">
                <button
                    class="brand__home"
                    type="button"
                    :aria-label="data.curr.i ? '返回主页' : 'Tab.archive 主页'"
                    @click="goHome"
                >
                    <div class="brand__mark" aria-hidden="true">
                        <svg viewBox="0 0 24 24" width="20" height="20">
                            <path d="M9 18V5l12-2v13" fill="none" stroke="currentColor" stroke-width="1.4" stroke-linecap="round"/>
                            <circle cx="6" cy="18" r="3" fill="currentColor"/>
                            <circle cx="18" cy="16" r="3" fill="currentColor"/>
                        </svg>
                    </div>
                    <div class="brand__text">
                        <div class="brand__title">TAB<span class="brand__dot">.</span><span class="brand__sub-brand">archive</span></div>
                        <div class="brand__sub">吉他谱 · Guitar Tablature</div>
                    </div>
                </button>
                <button class="theme-btn" :aria-label="`Switch to ${themeIcon === 'sun' ? 'light' : 'dark'} mode`" @click="toggleTheme">
                    <svg v-if="data.theme === 'dark'" viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                        <circle cx="12" cy="12" r="4"/>
                        <path d="M12 2v2M12 20v2M4.93 4.93l1.41 1.41M17.66 17.66l1.41 1.41M2 12h2M20 12h2M4.93 19.07l1.41-1.41M17.66 6.34l1.41-1.41"/>
                    </svg>
                    <svg v-else viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"/>
                    </svg>
                </button>
            </header>

            <div class="search-wrap">
                <svg class="search-icon" viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round">
                    <circle cx="11" cy="11" r="7"/>
                    <path d="m21 21-4.3-4.3"/>
                </svg>
                <input
                    class="search"
                    v-model="data.keyword"
                    placeholder="搜索 曲目 / 拼音"
                    spellcheck="false"
                    autocomplete="off"
                />
                <span v-if="data.keyword" class="search-clear" @click="data.keyword = ''">×</span>
            </div>

            <div class="meta-row">
                <span class="meta-label">{{ data.keyword ? '搜索结果' : '曲库' }}</span>
                <span class="meta-count">
                    <template v-if="data.loading">···</template>
                    <template v-else-if="data.keyword">{{ matchedCount }} / {{ totalSongs }}</template>
                    <template v-else>{{ totalSongs }}</template>
                </span>
            </div>

            <div class="list-wrap" :class="{ 'list-wrap--empty': !data.loading && data.list.length === 0 }">
                <v-list
                    v-if="data.list.length"
                    ref="vList"
                    :data="data.list"
                    :style="{ height: '100%', width: '100%' }"
                    #default="{ item, index }"
                >
                    <div
                        :key="item.i"
                        class="song ani"
                        :class="{ song_curr: item == data.curr }"
                        @click="data.curr = item"
                    >
                        <span class="song__index">{{ String(index + 1).padStart(2, '0') }}</span>
                        <span class="song__title">{{ item.t }}</span>
                    </div>
                </v-list>

                <!-- Empty: no result -->
                <div v-else-if="!data.loading && data.raw.length" class="state state--empty">
                    <p class="state__title">未找到相关曲谱</p>
                    <p class="state__sub">换个关键词试试,或清空搜索查看全部</p>
                </div>

                <!-- Loading state -->
                <div v-else class="state state--loading">
                    <p class="state__title">载入中</p>
                    <p class="state__sub">正在读取曲库…</p>
                </div>
            </div>

            <footer class="side-foot">
                <span class="foot-label">REC · {{ new Date().getFullYear() }}</span>
            </footer>

            <!-- Mobile-only floating "Now Playing" entry -->
            <button
                v-if="data.curr.i"
                class="now-playing"
                @click="goStage"
                aria-label="查看当前选中曲目的曲谱"
            >
                <div class="now-playing__meta">
                    <div class="now-playing__label">NOW PLAYING</div>
                    <div class="now-playing__title">{{ data.curr.t }}</div>
                </div>
                <div class="now-playing__cta">
                    <span>查看</span>
                    <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M9 18l6-6-6-6"/>
                    </svg>
                </div>
            </button>
        </aside>

        <!-- RIGHT: Detail / Stage -->
        <section class="right" ref="right">
            <button
                v-if="data.curr.i"
                class="expand-btn"
                :class="{ 'expand-btn--active': data.full }"
                @click="data.full = !data.full"
                :aria-label="data.full ? 'Restore list' : 'Fullscreen'"
                v-show="!isNarrow"
            >
                <svg v-if="!data.full" viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M3 8V3h5M21 8V3h-5M3 16v5h5M21 16v5h-5"/>
                </svg>
                <svg v-else viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M9 3v5H3M15 3v5h6M9 21v-5H3M15 21v-5h6"/>
                </svg>
                <span>{{ data.full ? 'RESTORE' : 'FULLSCREEN' }}</span>
            </button>

            <div v-if="data.curr.i" class="stage" v-on="{ dblclick: isNarrow ? null : (_ => data.full = !data.full) }">
                <button class="back-btn" @click="goList" aria-label="返回曲库">
                    <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M19 12H5M12 19l-7-7 7-7"/>
                    </svg>
                    <span>曲库</span>
                </button>
                <header class="stage__header">
                    <div class="stage__track">
                        <span class="stage__id">TRACK · {{ String(data.curr.i).padStart(4, '0') }}</span>
                    </div>
                    <h1 class="stage__title">
                        <span class="stage__title-zh">{{ data.curr.t }}</span>
                        <span class="stage__title-pinyin">{{ data.curr.py }}</span>
                    </h1>
                </header>

                <div class="gallery" :class="{ 'gallery--full': data.full }" :key="data.curr.i">
                    <figure
                        v-for="i in data.curr.c"
                        :key="i"
                        class="sheet"
                        :style="{ '--delay': `${(i - 1) * 70}ms` }"
                    >
                        <div class="sheet__number">{{ String(i).padStart(2, '0') }} / {{ String(data.curr.c).padStart(2, '0') }}</div>
                        <img
                            :src="`./data/pics/${data.curr.i}/${i - 1}.webp`"
                            :alt="`${data.curr.t} 吉他谱 第 ${i} 页`"
                            loading="lazy"
                            decoding="async"
                        />
                    </figure>
                </div>
            </div>

            <div v-else class="hero" v-on="{ dblclick: isNarrow ? null : (_ => data.full = !data.full) }">
                <div class="hero__inner">
                    <div class="hero__eyebrow">吉他六弦谱 · 私人曲库</div>
                    <h1 class="hero__title">
                        <span class="hero__title-zh">Tab</span><span class="hero__title-dot">.</span><span class="hero__title-it">archive</span>
                    </h1>
                    <p class="hero__lede">
                        收录 <em>{{ totalSongs || '—' }}</em> 首曲目的吉他六弦谱,
                        曲谱素材均来自互联网整理分享,仅供学习交流使用。
                    </p>
                </div>
            </div>
        </section>
    </div>
</template>

<style>
:root {
    /* Pure system font stack — no external fonts */
    --font-serif: "Songti SC", "Noto Serif SC", "Source Han Serif SC", "STSong", "SimSun", Georgia, "Times New Roman", serif;
    --font-sans: -apple-system, BlinkMacSystemFont, "PingFang SC", "Hiragino Sans GB", "Microsoft YaHei", "Segoe UI", Roboto, system-ui, sans-serif;
    --font-mono: ui-monospace, "SF Mono", Menlo, Consolas, "Roboto Mono", monospace;

    /* Dark theme — softened: never pure black, never pure white */
    --bg: #1c1c1f;
    --bg-elev: #232327;
    --bg-soft: #2a2a2e;
    --line: rgba(255, 255, 255, 0.05);
    --line-strong: rgba(255, 255, 255, 0.1);
    --ink: #d8d8d8;
    --ink-soft: #9a9a9e;
    --ink-dim: #6a6a70;
    --accent: #d8d8d8;
    --accent-ink: #1c1c1f;
    /* In dark mode the sheet image is `multiply`-blended against this
       cushion colour. Pick a light grey so the original white paper
       compresses to a near-white tone while the dark linework stays
       crisp. */
    --paper: #b8b8bd;
    --paper-cushion: #b8b8bd;
    --paper-shadow: 0 1px 1px rgba(0, 0, 0, 0.3), 0 14px 30px -18px rgba(0, 0, 0, 0.45);
}

:root[data-theme="light"] {
    --bg: #f6f6f7;
    --bg-elev: #ffffff;
    --bg-soft: #ececef;
    --line: rgba(0, 0, 0, 0.06);
    --line-strong: rgba(0, 0, 0, 0.1);
    --ink: #1a1a1d;
    --ink-soft: #5e5e63;
    --ink-dim: #97979c;
    --accent: #1a1a1d;
    --accent-ink: #ffffff;
    --paper: #ffffff;
    --paper-cushion: #ececef;
    --paper-shadow: 0 1px 2px rgba(0, 0, 0, 0.05), 0 16px 36px -22px rgba(0, 0, 0, 0.18);
}

* {
    box-sizing: border-box;
}

html,
body,
#app {
    width: 100%;
    height: 100%;
    overflow: hidden;
    padding: 0;
    margin: 0;
    background: var(--bg);
    color: var(--ink);
    font-family: var(--font-sans);
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-rendering: optimizeLegibility;
}

body {
    transition: background 0.3s ease, color 0.3s ease;
}

.ani {
    transition: background 0.25s ease, color 0.25s ease, border-color 0.25s ease;
}

.main {
    display: flex;
    width: 100%;
    height: 100%;
    position: relative;
}

/* ============ LEFT — TRACKLIST ============ */
.left {
    height: 100%;
    width: 280px;
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    background: var(--bg-elev);
    border-right: 1px solid var(--line);
    position: relative;
    overflow: hidden;
}

.brand {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 20px 20px 16px;
    border-bottom: 1px solid var(--line);
}

.brand__home {
    flex: 1;
    min-width: 0;
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 0;
    margin: 0;
    border: 0;
    background: transparent;
    color: inherit;
    font: inherit;
    text-align: left;
    cursor: pointer;
    border-radius: 6px;
    -webkit-tap-highlight-color: transparent;
}

.brand__home:hover {
    opacity: 0.7;
}

.brand__home:focus-visible {
    outline: 2px solid var(--ink);
    outline-offset: 3px;
}

.brand__mark {
    width: 32px;
    height: 32px;
    border-radius: 7px;
    display: grid;
    place-items: center;
    background: var(--ink-soft);
    color: var(--bg);
    flex-shrink: 0;
    transition: opacity 0.2s ease;
}

.brand__home:hover .brand__mark {
    background: var(--ink);
}

.brand__text {
    flex: 1;
    min-width: 0;
    line-height: 1.2;
}

.brand__title {
    font-family: var(--font-serif);
    font-weight: 500;
    font-size: 16px;
    letter-spacing: 0.02em;
    color: var(--ink);
    display: flex;
    align-items: baseline;
    gap: 1px;
}

.brand__dot {
    color: var(--ink-dim);
    margin: 0 1px;
}

.brand__sub-brand {
    font-family: var(--font-serif);
    font-style: italic;
    font-weight: 400;
    color: var(--ink);
}

.brand__sub {
    margin-top: 3px;
    font-family: var(--font-sans);
    font-size: 10.5px;
    letter-spacing: 0.02em;
    color: var(--ink-dim);
}

.theme-btn {
    width: 30px;
    height: 30px;
    border: 1px solid var(--line);
    background: transparent;
    border-radius: 6px;
    color: var(--ink-soft);
    cursor: pointer;
    display: grid;
    place-items: center;
    transition: all 0.2s ease;
}

.theme-btn:hover {
    color: var(--ink);
    border-color: var(--line-strong);
}

.search-wrap {
    position: relative;
    padding: 14px 16px 10px;
}

.search-icon {
    position: absolute;
    left: 26px;
    top: 50%;
    transform: translateY(-50%);
    color: var(--ink-dim);
    pointer-events: none;
    transition: color 0.2s ease;
}

.search-wrap:focus-within .search-icon {
    color: var(--ink);
}

.search {
    width: 100%;
    height: 36px;
    padding: 0 30px 0 32px;
    border: 1px solid var(--line);
    background: transparent;
    border-radius: 6px;
    color: var(--ink);
    font-size: 13px;
    font-family: var(--font-sans);
    outline: none;
    transition: all 0.2s ease;
}

.search::placeholder {
    color: var(--ink-dim);
    font-size: 12.5px;
}

.search:focus {
    border-color: var(--line-strong);
    background: var(--bg-soft);
}

.search-clear {
    position: absolute;
    right: 22px;
    top: 50%;
    transform: translateY(-50%);
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: var(--line-strong);
    color: var(--ink-soft);
    display: grid;
    place-items: center;
    font-size: 14px;
    line-height: 1;
    cursor: pointer;
    transition: all 0.2s ease;
}

.search-clear:hover {
    background: var(--ink);
    color: var(--bg);
}

.meta-row {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    padding: 4px 20px 10px;
    border-bottom: 1px solid var(--line);
}

.meta-label {
    font-family: var(--font-sans);
    font-size: 11px;
    color: var(--ink-dim);
    letter-spacing: 0.04em;
}

.meta-count {
    font-family: var(--font-mono);
    font-size: 11px;
    color: var(--ink-soft);
    letter-spacing: 0.04em;
}

.list-wrap {
    flex: 1;
    min-height: 0;
    position: relative;
}

.list-wrap--empty {
    display: flex;
    align-items: center;
    justify-content: center;
}

/* Track row */
.song {
    height: 48px;
    width: 100%;
    padding: 0 20px;
    display: flex;
    align-items: center;
    gap: 12px;
    cursor: pointer;
    overflow: hidden;
    white-space: nowrap;
    border-bottom: 1px solid var(--line);
    position: relative;
}

.song:hover {
    background: var(--bg-soft);
}

.song_curr {
    background: var(--bg-soft);
}

.song_curr .song__title {
    color: var(--ink);
    font-weight: 600;
}

.song_curr .song__index {
    color: var(--ink);
}

.song__index {
    font-family: var(--font-mono);
    font-size: 11px;
    letter-spacing: 0.02em;
    color: var(--ink-dim);
    flex-shrink: 0;
    width: 22px;
    transition: color 0.2s ease;
}

.song__title {
    font-size: 13.5px;
    color: var(--ink-soft);
    overflow: hidden;
    text-overflow: ellipsis;
    font-weight: 400;
    flex: 1;
    min-width: 0;
    transition: color 0.2s ease;
}

.song:hover .song__title {
    color: var(--ink);
}

/* States */
.state {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    justify-content: center;
    padding: 24px;
    text-align: left;
    color: var(--ink-soft);
    gap: 6px;
}

.state__title {
    font-family: var(--font-sans);
    font-size: 13px;
    color: var(--ink);
    margin: 0;
    font-weight: 500;
}

.state__sub {
    font-size: 12px;
    color: var(--ink-dim);
    margin: 0;
    line-height: 1.6;
}

.side-foot {
    padding: 12px 20px 14px;
    border-top: 1px solid var(--line);
    display: flex;
    align-items: center;
    gap: 12px;
}

.foot-label {
    font-family: var(--font-mono);
    font-size: 10px;
    letter-spacing: 0.06em;
    color: var(--ink-dim);
}

/* ============ RIGHT — STAGE ============ */
.right {
    height: 100%;
    flex: 1;
    overflow-y: auto;
    position: relative;
    user-select: none;
    background: var(--bg);
}

.expand-btn {
    position: sticky;
    top: 18px;
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 5px 10px 5px 8px;
    background: var(--bg-elev);
    border: 1px solid var(--line);
    border-radius: 6px;
    color: var(--ink-soft);
    font-family: var(--font-sans);
    font-size: 11px;
    letter-spacing: 0.06em;
    cursor: pointer;
    z-index: 5;
    transition: all 0.2s ease;
    margin: 18px 18px 0 auto;
    width: max-content;
}

.expand-btn:hover {
    color: var(--ink);
    border-color: var(--line-strong);
}

.expand-btn--active {
    color: var(--ink);
    border-color: var(--line-strong);
    background: var(--bg-soft);
}

/* HERO (empty state) */
.hero {
    min-height: 100%;
    display: flex;
    align-items: center;
    justify-content: flex-start;
    padding: 64px;
    position: relative;
    z-index: 1;
}

.hero__inner {
    max-width: 640px;
    animation: hero-rise 0.6s cubic-bezier(0.22, 0.61, 0.36, 1) both;
}

@keyframes hero-rise {
    from { opacity: 0; transform: translateY(12px); }
    to { opacity: 1; transform: translateY(0); }
}

.hero__eyebrow {
    font-family: var(--font-sans);
    font-size: 12px;
    letter-spacing: 0.04em;
    color: var(--ink-dim);
    margin-bottom: 28px;
}

.hero__title {
    font-family: var(--font-serif);
    font-weight: 400;
    font-size: clamp(56px, 9vw, 96px);
    line-height: 1;
    letter-spacing: -0.02em;
    margin: 0 0 28px;
    color: var(--ink);
    display: flex;
    align-items: baseline;
}

.hero__title-zh {
    font-weight: 500;
}

.hero__title-dot {
    color: var(--ink-dim);
    margin: 0 4px;
}

.hero__title-it {
    font-style: italic;
    font-weight: 400;
}

.hero__lede {
    font-size: 15px;
    line-height: 1.8;
    color: var(--ink-soft);
    margin: 0;
    max-width: 460px;
}

.hero__lede em {
    font-style: normal;
    color: var(--ink);
    font-weight: 500;
    font-family: var(--font-mono);
    font-size: 0.92em;
    padding: 0 2px;
}

/* STAGE — selected song */
.stage {
    max-width: 920px;
    margin: 0 auto;
    padding: 10px 28px 80px;
    position: relative;
    z-index: 1;
    animation: stage-in 0.4s cubic-bezier(0.22, 0.61, 0.36, 1);
}

@keyframes stage-in {
    from { opacity: 0; transform: translateY(8px); }
    to { opacity: 1; transform: translateY(0); }
}

.stage__header {
    padding: 0 0 18px;
    border-bottom: 1px solid var(--line);
    margin-bottom: 26px;
}

.stage__track {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-bottom: 8px;
}

.stage__id {
    font-family: var(--font-mono);
    font-size: 11px;
    letter-spacing: 0.04em;
    color: var(--ink-dim);
}

.stage__title {
    margin: 0;
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.stage__title-zh {
    font-family: var(--font-sans);
    font-weight: 600;
    font-size: clamp(28px, 4.2vw, 40px);
    line-height: 1.2;
    letter-spacing: 0.005em;
    color: var(--ink);
    /* Slightly tighten Chinese letter-spacing for display use */
    font-feature-settings: "palt";
}

.stage__title-pinyin {
    font-family: var(--font-mono);
    font-size: 12.5px;
    font-weight: 400;
    letter-spacing: 0.04em;
    color: var(--ink-dim);
}

/* Gallery */
.gallery {
    display: flex;
    flex-direction: column;
    gap: 18px;
}

@keyframes sheet-in {
    from {
        opacity: 0;
        transform: scale(0.96);
    }
    to {
        opacity: 1;
        transform: scale(1);
    }
}

.sheet {
    position: relative;
    background: var(--paper);
    border-radius: 2px;
    box-shadow: var(--paper-shadow);
    overflow: hidden;
    /* Hairline border softens the transition between the light-grey
       cushion and the dark page background. */
    outline: 1px solid rgba(255, 255, 255, 0.18);
    outline-offset: -1px;
    /* `isolation: isolate` creates a new stacking context so
       the image's `mix-blend-mode` only blends against the
       sheet's own backdrop, not the whole page. */
    isolation: isolate;
    animation: sheet-in 0.5s cubic-bezier(0.4, 0, 0.2, 1) both;
    animation-delay: var(--delay, 0ms);
}

.sheet img {
    display: block;
    width: 100%;
    height: auto;
    -webkit-user-drag: none;
    transition: filter 0.3s ease, opacity 0.3s ease, mix-blend-mode 0.3s ease;
}

/* Dark theme: blend the sheet image into the cushion so its bright
   white paper compresses to a readable mid-grey while the dark
   linework stays crisp. We avoid `invert()` because coloured photo
   backgrounds (performers etc.) become garish. */
:root .sheet img {
    mix-blend-mode: multiply;
    opacity: 1;
}

:root[data-theme="light"] .sheet img {
    mix-blend-mode: normal;
    opacity: 1;
}

.sheet__number {
    position: absolute;
    top: 10px;
    right: 10px;
    z-index: 2;
    font-family: var(--font-mono);
    font-size: 10px;
    letter-spacing: 0.06em;
    color: rgba(20, 20, 22, 0.4);
    background: rgba(255, 255, 255, 0.7);
    backdrop-filter: blur(6px);
    padding: 2px 6px;
    border-radius: 2px;
}

/* Fullscreen — hide list, expand stage */
.main.full .left {
    width: 0;
    border-right: 0;
    overflow: hidden;
    opacity: 0;
}

.main.full .stage {
    max-width: 1100px;
    padding: 10px 48px 80px;
}

.main.full .gallery {
    gap: 28px;
}

/* ============ SCROLLBAR ============ */
* {
    scrollbar-width: thin;
    scrollbar-color: var(--line-strong) transparent;
}

::-webkit-scrollbar {
    width: 8px;
    height: 8px;
}

::-webkit-scrollbar-track {
    background: transparent;
}

::-webkit-scrollbar-thumb {
    background: var(--line-strong);
    border-radius: 999px;
    border: 2px solid transparent;
    background-clip: padding-box;
    transition: background 0.2s ease;
}

::-webkit-scrollbar-thumb:hover {
    background: var(--ink-dim);
    background-clip: padding-box;
}

::-webkit-scrollbar-corner {
    background: transparent;
}

/* ============ RESPONSIVE — narrow screens ============ */
.back-btn,
.now-playing {
    display: none;
}

@media (max-width: 820px), (orientation: portrait) {
    .main {
        flex-direction: column;
    }

    .left {
        width: 100%;
        height: 100%;
        border-right: 0;
        border-bottom: 0;
        position: absolute;
        inset: 0;
        z-index: 2;
        transform: translateX(0);
        transition: transform 0.3s cubic-bezier(0.22, 0.61, 0.36, 1),
                    opacity 0.25s ease;
    }

    .main.mobile-stage .left {
        transform: translateX(-100%);
        opacity: 0;
        pointer-events: none;
    }

    .brand {
        padding: 14px 16px 12px;
    }
    .brand__mark {
        width: 30px;
        height: 30px;
    }
    .brand__title {
        font-size: 15px;
    }

    .search-wrap {
        padding: 12px 14px 8px;
    }
    .search-wrap .search-icon { left: 24px; }
    .search-clear { right: 22px; }

    .meta-row {
        padding: 4px 16px 10px;
    }

    .song {
        height: 50px;
        padding: 0 16px;
    }
    .song__title {
        font-size: 14px;
    }

    .right {
        position: absolute;
        inset: 0;
        z-index: 1;
        opacity: 0;
        pointer-events: none;
        transition: opacity 0.25s ease;
    }
    .main.mobile-stage .right {
        opacity: 1;
        pointer-events: auto;
    }

    .stage {
        padding: 10px 18px 80px;
    }
    .stage__title-zh {
        font-size: clamp(28px, 7vw, 38px);
    }

    .hero {
        padding: 32px 24px;
    }
    .hero__title {
        font-size: clamp(48px, 12vw, 64px);
    }

    .back-btn {
        display: inline-flex;
        align-items: center;
        gap: 6px;
        margin-bottom: 18px;
        padding: 6px 12px;
        background: transparent;
        border: 1px solid var(--line);
        border-radius: 6px;
        color: var(--ink-soft);
        font-family: var(--font-sans);
        font-size: 12px;
        cursor: pointer;
        transition: all 0.2s ease;
    }
    .back-btn:hover,
    .back-btn:active {
        color: var(--ink);
        border-color: var(--line-strong);
    }

    .now-playing {
        display: flex;
        align-items: center;
        gap: 12px;
        position: fixed;
        left: 12px;
        right: 12px;
        bottom: calc(12px + env(safe-area-inset-bottom));
        padding: 10px 12px;
        background: var(--bg-elev);
        border: 1px solid var(--line-strong);
        border-radius: 8px;
        color: var(--ink);
        text-align: left;
        cursor: pointer;
        z-index: 50;
        animation: np-rise 0.3s cubic-bezier(0.22, 0.61, 0.36, 1);
    }
    .now-playing:active {
        transform: scale(0.98);
    }
    @keyframes np-rise {
        from { opacity: 0; transform: translateY(8px); }
        to { opacity: 1; transform: translateY(0); }
    }

    .now-playing__meta {
        flex: 1;
        min-width: 0;
        line-height: 1.2;
    }
    .now-playing__label {
        font-family: var(--font-mono);
        font-size: 10px;
        letter-spacing: 0.08em;
        color: var(--ink-dim);
        margin-bottom: 3px;
    }
    .now-playing__title {
        font-family: var(--font-sans);
        font-size: 15px;
        color: var(--ink);
        font-weight: 600;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .now-playing__cta {
        display: flex;
        align-items: center;
        gap: 4px;
        color: var(--ink-soft);
        font-family: var(--font-sans);
        font-size: 12px;
        flex-shrink: 0;
    }

    .main.full .left {
        display: none;
    }
}

@media (max-width: 380px) {
    .hero__title {
        font-size: 44px;
    }
    .stage__title-zh {
        font-size: 28px;
    }
    .brand__sub {
        display: none;
    }
}
</style>