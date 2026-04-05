<template>
<div id="main" :class="{ 'theme-dark': darkMode }">
    <main-toolbar
        :routine="routine"
        :backend-ready="backendReady"
        :dark-mode="darkMode"
        @upload-file="handleUpload"
        @download-file="downloadCurrentRoutine"
        @refresh="refreshRoutine"
        @toggle-theme="toggleTheme" />
    <splittable-pane
        id="main-pane"
        :routine="routine"
        :editor-schema="editorSchema"
        @edit-immediate="handleEditImmediate"
        @edit-instruction="handleEditInstruction" />
</div>
</template>
<script>
import MainToolbar from './components/MainToolbar'
import SplittablePane from './components/SplittablePane'

const API_BASE = 'http://127.0.0.1:8090'

export default {
    data() {
        return {
            backendReady: false,
            darkMode: true,
            routine: {
                ok: false,
                file_name: '',
                entry_point: null,
                blocks: [],
                cfg_edges: [],
                last_error: ''
            },
            editorSchema: {
                mnemonics: [],
                registers: [],
                instructions: []
            },
            pollTimer: null
        }
    },
    components: {
        'main-toolbar': MainToolbar,
        SplittablePane
    },
    methods: {
        buildApiUrl(path, query = {}) {
            const url = new URL(`${API_BASE}${path}`)
            Object.keys(query).forEach((key) => {
                if (query[key] !== undefined && query[key] !== null) {
                    url.searchParams.set(key, String(query[key]))
                }
            })
            return url.toString()
        },
        normalizeRoutine(payload) {
            const blocks = (payload.blocks || []).slice().sort((a, b) => {
                const aa = a.address || ''
                const bb = b.address || ''
                return aa.localeCompare(bb)
            })

            return {
                ok: !!payload.ok,
                file_name: payload.file_name || '',
                entry_point: payload.entry_point || null,
                blocks,
                cfg_edges: payload.cfg_edges || [],
                last_error: payload.last_error || ''
            }
        },
        async checkHealth() {
            try {
                const response = await fetch(`${API_BASE}/health`)
                this.backendReady = response.ok
            } catch {
                this.backendReady = false
            }
        },
        async refreshRoutine() {
            await this.checkHealth()
            if (!this.backendReady) return

            try {
                const response = await fetch(`${API_BASE}/api/state`)
                if (!response.ok) return
                const payload = await response.json()
                this.routine = this.normalizeRoutine(payload)
            } catch {
                this.backendReady = false
            }
        },
        async refreshEditorSchema() {
            await this.checkHealth()
            if (!this.backendReady) return

            try {
                const response = await fetch(this.buildApiUrl('/api/schema'))
                if (!response.ok) return
                const payload = await response.json()
                this.editorSchema = {
                    mnemonics: payload.mnemonics || [],
                    registers: payload.registers || [],
                    instructions: payload.instructions || []
                }
            } catch {
                this.backendReady = false
            }
        },
        async handleUpload(file) {
            if (!file) return

            await this.checkHealth()
            if (!this.backendReady) {
                window.alert('Backend is not reachable on http://127.0.0.1:8090')
                return
            }

            const buffer = await file.arrayBuffer()
            const response = await fetch(`${API_BASE}/api/upload?name=${encodeURIComponent(file.name)}`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/octet-stream'
                },
                body: buffer
            })

            if (!response.ok) {
                const text = await response.text()
                window.alert(text || 'Upload failed.')
            }
            await this.refreshRoutine()
            await this.refreshEditorSchema()
        },
        async handleEditImmediate(payload) {
            if (!payload) return

            await this.checkHealth()
            if (!this.backendReady) {
                window.alert('Backend is not reachable on http://127.0.0.1:8090')
                return
            }

            const endpoint = this.buildApiUrl('/api/edit/immediate', {
                block: payload.block,
                instruction: payload.instruction,
                operand: payload.operand,
                value: payload.value
            })

            try {
                const response = await fetch(endpoint, { method: 'POST' })
                if (!response.ok) {
                    const text = await response.text()
                    window.alert(text || 'Immediate edit failed.')
                }
            } catch {
                this.backendReady = false
                window.alert('Immediate edit failed.')
            }

            await this.refreshRoutine()
            await this.refreshEditorSchema()
        },
        async handleEditInstruction(payload) {
            if (!payload || !payload.text) return

            await this.checkHealth()
            if (!this.backendReady) {
                window.alert('Backend is not reachable on http://127.0.0.1:8090')
                return
            }

            const endpoint = this.buildApiUrl('/api/edit/instruction', {
                block: payload.block,
                instruction: payload.instruction
            })

            try {
                const response = await fetch(endpoint, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'text/plain; charset=utf-8'
                    },
                    body: payload.text
                })

                if (!response.ok) {
                    const text = await response.text()
                    window.alert(text || 'Instruction edit failed.')
                }
            } catch {
                this.backendReady = false
                window.alert('Instruction edit failed.')
            }

            await this.refreshRoutine()
            await this.refreshEditorSchema()
        },
        async downloadCurrentRoutine() {
            await this.checkHealth()
            if (!this.backendReady) {
                window.alert('Backend is not reachable on http://127.0.0.1:8090')
                return
            }

            try {
                const response = await fetch(this.buildApiUrl('/api/download'))
                if (!response.ok) {
                    const text = await response.text()
                    window.alert(text || 'Download failed.')
                    return
                }

                const blob = await response.blob()
                const objectUrl = URL.createObjectURL(blob)
                const disposition = response.headers.get('Content-Disposition') || ''
                const match = disposition.match(/filename="?([^";]+)"?/i)
                const fallback = this.routine.file_name || 'edited.vtil'
                const filename = (match && match[1]) ? match[1] : fallback

                const anchor = document.createElement('a')
                anchor.href = objectUrl
                anchor.download = filename
                document.body.appendChild(anchor)
                anchor.click()
                anchor.remove()
                URL.revokeObjectURL(objectUrl)
            } catch {
                this.backendReady = false
                window.alert('Download failed.')
            }
        },
        toggleTheme() {
            this.darkMode = !this.darkMode
        }
    },
    mounted() {
        this.refreshRoutine()
        this.refreshEditorSchema()
        this.pollTimer = setInterval(() => {
            this.refreshRoutine()
        }, 1200)
    },
    beforeDestroy() {
        clearInterval(this.pollTimer)
    }
}

</script>
<style>
*, *::before, *::after { box-sizing: border-box; }

html, body {
    height: 100%;
    margin: 0;
    font-family: 'Inter', -apple-system, 'Segoe UI', sans-serif;
    background: #0d1117;
    -webkit-font-smoothing: antialiased;
}
#app {
    height: 100%;
}


#main {
    --bg:            #f6f8fa;
    --panel:         #ffffff;
    --panel-2:       #f6f8fa;
    --panel-3:       #eaeef2;
    --panel-border:  #d0d7de;
    --text:          #1f2328;
    --muted:         #656d76;
    --muted-2:       #9198a1;
    --accent:        #0969da;
    --accent-soft:   rgba(9, 105, 218, 0.08);
    --accent-hover:  rgba(9, 105, 218, 0.12);
    --accent-text:   #0969da;
    --success:       #1a7f37;
    --success-bg:    rgba(26, 127, 55, 0.08);
    --danger:        #cf222e;
    --danger-bg:     rgba(207, 34, 46, 0.08);
    --warning:       #9a6700;
    --edge:          #8c959f;
    --edge-out:      #1a7f37;
    --edge-in:       #9a6700;
    --cfg-dot:       rgba(0,0,0,0.06);
    --shadow-sm:     0 1px 3px rgba(27,31,36,0.12), 0 1px 2px rgba(27,31,36,0.08);
    --shadow-md:     0 4px 12px rgba(27,31,36,0.15), 0 2px 4px rgba(27,31,36,0.1);
    --shadow-xl:     0 16px 48px rgba(27,31,36,0.25);
    --radius:        6px;
    --radius-lg:     10px;
    --mono:          'JetBrains Mono', 'Fira Code', Consolas, monospace;

    height: 100%;
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    color: var(--text);
    background: var(--bg);
    font-size: 13px;
    line-height: 1.5;
}

#main.theme-dark {
    --bg:            #0d1117;
    --panel:         #161b22;
    --panel-2:       #1c2128;
    --panel-3:       #21262d;
    --panel-border:  #30363d;
    --text:          #e6edf3;
    --muted:         #8b949e;
    --muted-2:       #6e7781;
    --accent:        #58a6ff;
    --accent-soft:   rgba(88, 166, 255, 0.1);
    --accent-hover:  rgba(88, 166, 255, 0.18);
    --accent-text:   #58a6ff;
    --success:       #3fb950;
    --success-bg:    rgba(63, 185, 80, 0.1);
    --danger:        #f85149;
    --danger-bg:     rgba(248, 81, 73, 0.1);
    --warning:       #d29922;
    --edge:          #484f58;
    --edge-out:      #3fb950;
    --edge-in:       #d29922;
    --cfg-dot:       rgba(255,255,255,0.04);
    --shadow-sm:     0 1px 3px rgba(0,0,0,0.4), 0 1px 2px rgba(0,0,0,0.3);
    --shadow-md:     0 4px 12px rgba(0,0,0,0.5), 0 2px 4px rgba(0,0,0,0.3);
    --shadow-xl:     0 16px 48px rgba(0,0,0,0.7);
}

#main-pane {
    position: relative;
    flex: 1;
    min-height: 0;
}
</style>
