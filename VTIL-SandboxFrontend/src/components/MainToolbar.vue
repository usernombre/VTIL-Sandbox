<template>
    <header class="toolbar">
        <div class="toolbar-brand">
            <svg class="brand-icon" viewBox="0 0 20 20" fill="none" xmlns="http://www.w3.org/2000/svg">
                <rect x="2" y="2" width="7" height="7" rx="1.5" fill="var(--accent)" opacity="0.9"/>
                <rect x="11" y="2" width="7" height="7" rx="1.5" fill="var(--accent)" opacity="0.5"/>
                <rect x="2" y="11" width="7" height="7" rx="1.5" fill="var(--accent)" opacity="0.5"/>
                <rect x="11" y="11" width="7" height="7" rx="1.5" fill="var(--accent)" opacity="0.2"/>
            </svg>
            <span class="brand-name">VTIL<span class="brand-dim">Sandbox</span></span>
        </div>

        <div class="toolbar-divider" />

        <div class="toolbar-actions">
            <label class="btn btn-accent" title="Upload a .vtil file">
                <svg class="btn-icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M2 11.5v1.75C2 13.66 2.34 14 2.75 14h10.5c.41 0 .75-.34.75-.75V11.5"/>
                    <path d="M8 2v8M5 5l3-3 3 3"/>
                </svg>
                Upload
                <input type="file" accept=".vtil" @change="onFileChanged" />
            </label>

            <button class="btn btn-ghost" @click="$emit('download-file')" title="Download modified .vtil">
                <svg class="btn-icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M2 11.5v1.75C2 13.66 2.34 14 2.75 14h10.5c.41 0 .75-.34.75-.75V11.5"/>
                    <path d="M8 2v8M5 9l3 3 3-3"/>
                </svg>
                Download
            </button>

            <button class="btn btn-icon-only" @click="$emit('refresh')" title="Refresh state">
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M13.5 2.5A7 7 0 1 0 14 8"/>
                    <path d="M10.5 2.5H13.5V5.5"/>
                </svg>
            </button>

            <button class="btn btn-icon-only" @click="$emit('toggle-theme')" :title="darkMode ? 'Switch to light theme' : 'Switch to dark theme'">
                <svg v-if="darkMode" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <circle cx="8" cy="8" r="3"/>
                    <path d="M8 1v2M8 13v2M1 8h2M13 8h2M3.22 3.22l1.42 1.42M11.36 11.36l1.42 1.42M3.22 12.78l1.42-1.42M11.36 4.64l1.42-1.42"/>
                </svg>
                <svg v-else viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M13.5 10.5A6 6 0 0 1 5.5 2.5a6 6 0 1 0 8 8z"/>
                </svg>
            </button>
        </div>

        <template v-if="routine.ok">
            <div class="toolbar-divider" />
            <div class="toolbar-meta">
                <span class="meta-item">
                    <span class="meta-label">file</span>
                    <code class="meta-value">{{ routine.file_name || 'n/a' }}</code>
                </span>
                <span class="meta-sep">·</span>
                <span class="meta-item">
                    <span class="meta-label">entry</span>
                    <code class="meta-value accent">{{ routine.entry_point || 'n/a' }}</code>
                </span>
                <span class="meta-sep">·</span>
                <span class="meta-item">
                    <span class="meta-label">blocks</span>
                    <span class="meta-badge">{{ blockCount }}</span>
                </span>
            </div>
        </template>

        <div class="toolbar-spacer" />

        <span v-if="routine.last_error" class="toolbar-error" :title="routine.last_error">
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" style="width:13px;height:13px;flex-shrink:0">
                <circle cx="8" cy="8" r="6.5"/>
                <path d="M8 5v3.5M8 11v.5"/>
            </svg>
            {{ routine.last_error }}
        </span>

        <div class="status-pill" :class="{ connected: backendReady }">
            <span class="status-dot"></span>
            <span class="status-label">{{ backendReady ? 'Connected' : 'Offline' }}</span>
        </div>
    </header>
</template>

<script>
export default {
    props: {
        routine: {
            type: Object,
            required: true
        },
        backendReady: {
            type: Boolean,
            default: false
        },
        darkMode: {
            type: Boolean,
            default: true
        }
    },
    computed: {
        blockCount() {
            return (this.routine.blocks || []).length
        }
    },
    methods: {
        onFileChanged(evt) {
            const file = evt.target.files && evt.target.files[0]
            if (file) this.$emit('upload-file', file)
            evt.target.value = ''
        }
    }
}
</script>

<style scoped>
.toolbar {
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 0 12px;
    height: 44px;
    flex-shrink: 0;
    background: var(--panel);
    border-bottom: 1px solid var(--panel-border);
    z-index: 100;
    overflow: hidden;
}

.toolbar-brand {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-shrink: 0;
}
.brand-icon {
    width: 20px;
    height: 20px;
}
.brand-name {
    font-size: 14px;
    font-weight: 600;
    letter-spacing: -0.01em;
    color: var(--text);
}
.brand-dim {
    font-weight: 400;
    color: var(--muted);
    margin-left: 1px;
}

.toolbar-divider {
    width: 1px;
    height: 20px;
    background: var(--panel-border);
    margin: 0 6px;
    flex-shrink: 0;
}

.toolbar-actions {
    display: flex;
    align-items: center;
    gap: 4px;
}
.btn {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 0 10px;
    height: 28px;
    border-radius: var(--radius);
    border: 1px solid var(--panel-border);
    font-family: inherit;
    font-size: 12.5px;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.12s, border-color 0.12s, color 0.12s;
    white-space: nowrap;
    line-height: 1;
}
.btn-icon {
    width: 13px;
    height: 13px;
    flex-shrink: 0;
}
.btn-ghost {
    background: transparent;
    color: var(--text);
}
.btn-ghost:hover {
    background: var(--panel-2);
    border-color: var(--panel-border);
}
.btn-accent {
    background: var(--accent-soft);
    border-color: transparent;
    color: var(--accent-text);
}
.btn-accent:hover {
    background: var(--accent-hover);
}
.btn-accent input {
    display: none;
}
.btn-icon-only {
    padding: 0;
    width: 28px;
    height: 28px;
    justify-content: center;
    background: transparent;
    color: var(--muted);
    flex-shrink: 0;
}
.btn-icon-only:hover {
    background: var(--panel-2);
    color: var(--text);
    border-color: var(--panel-border);
}
.btn-icon-only svg {
    width: 14px;
    height: 14px;
}

.toolbar-meta {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 12px;
    flex-shrink: 0;
}
.meta-item {
    display: flex;
    align-items: center;
    gap: 4px;
}
.meta-label {
    color: var(--muted);
    user-select: none;
}
.meta-value {
    font-family: var(--mono);
    font-size: 11.5px;
    color: var(--text);
}
.meta-value.accent {
    color: var(--accent-text);
}
.meta-badge {
    background: var(--panel-2);
    border: 1px solid var(--panel-border);
    border-radius: 999px;
    padding: 0 6px;
    font-size: 11px;
    font-weight: 600;
    color: var(--text);
    min-width: 20px;
    text-align: center;
}
.meta-sep {
    color: var(--panel-border);
    font-size: 14px;
    user-select: none;
}

.toolbar-spacer {
    flex: 1;
    min-width: 8px;
}

.toolbar-error {
    display: flex;
    align-items: center;
    gap: 4px;
    color: var(--danger);
    font-size: 12px;
    max-width: 260px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    flex-shrink: 0;
}

.status-pill {
    display: flex;
    align-items: center;
    gap: 5px;
    padding: 0 8px;
    height: 22px;
    border-radius: 999px;
    background: var(--danger-bg);
    border: 1px solid transparent;
    flex-shrink: 0;
    margin-left: 6px;
}
.status-pill.connected {
    background: var(--success-bg);
}
.status-dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: var(--danger);
    flex-shrink: 0;
}
.status-pill.connected .status-dot {
    background: var(--success);
    box-shadow: 0 0 0 2px var(--success-bg);
}
.status-label {
    font-size: 11.5px;
    font-weight: 500;
    color: var(--danger);
}
.status-pill.connected .status-label {
    color: var(--success);
}

@media (max-width: 900px) {
    .toolbar-meta { display: none; }
    .btn-ghost { display: none; }
}
@media (max-width: 600px) {
    .brand-name { display: none; }
    .toolbar-divider { display: none; }
}
</style>
