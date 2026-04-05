<template>
    <div :class="['layout', { dragging }]" ref="container">
        <div class="pane pane-left" :style="{ width: sizes[0] + '%' }">
            <div class="pane-header">
                <span class="pane-title">Blocks</span>
                <span class="pane-badge" v-if="blockRows.length">{{ blockRows.length }}</span>
            </div>

            <div class="pane-body">
                <div v-if="blockRows.length === 0" class="empty-state">
                    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.3" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
                        <polyline points="14 2 14 8 20 8"/>
                        <line x1="9" y1="15" x2="15" y2="15"/>
                    </svg>
                    <span>Upload a .vtil file to view blocks</span>
                </div>

                <div
                    v-for="blk in blockRows"
                    :key="blk.address"
                    class="block-row"
                    :class="{ selected: selectedKey === blk.address, entry: blk.address === entryAddress }"
                    @click="selectBlock(blk.address)">
                    <div class="block-row-indicator">
                        <svg v-if="blk.address === entryAddress" class="entry-icon" viewBox="0 0 10 10" fill="currentColor">
                            <polygon points="2,2 8,5 2,8"/>
                        </svg>
                    </div>
                    <div class="block-row-content">
                        <code class="block-addr">{{ blk.address }}</code>
                        <span class="block-count">{{ blk.instruction_count || 0 }}</span>
                    </div>
                </div>
            </div>
        </div>

        <div class="resizer" @mousedown.stop.prevent="mouseDown"></div>
        <div class="pane pane-right" :style="{ width: sizes[1] + '%' }">
            <div v-if="!selectedBlock" class="empty-state full-center">
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.3" stroke-linecap="round" stroke-linejoin="round">
                    <rect x="3" y="3" width="7" height="7" rx="1"/>
                    <rect x="14" y="3" width="7" height="7" rx="1"/>
                    <rect x="14" y="14" width="7" height="7" rx="1"/>
                    <rect x="3" y="14" width="7" height="7" rx="1"/>
                </svg>
                <span>Select a block from the left panel</span>
            </div>

            <template v-if="selectedBlock">
                <div class="block-info-strip">
                    <div class="block-info-item">
                        <span class="info-label">Address</span>
                        <code class="info-code accent">{{ selectedBlock.address }}</code>
                    </div>
                    <div class="block-info-sep" />
                    <div class="block-info-item">
                        <span class="info-label">Instructions</span>
                        <span class="info-badge">{{ selectedBlock.instruction_count || 0 }}</span>
                    </div>
                    <div class="block-info-sep" />
                    <div class="block-info-item">
                        <span class="info-label">Predecessors</span>
                        <div class="info-addr-list">
                            <code v-for="p in (selectedBlock.predecessors || [])" :key="p" class="info-addr" @click="selectBlock(p)">{{ p }}</code>
                            <span v-if="!(selectedBlock.predecessors || []).length" class="info-none">—</span>
                        </div>
                    </div>
                    <div class="block-info-sep" />
                    <div class="block-info-item">
                        <span class="info-label">Successors</span>
                        <div class="info-addr-list">
                            <code v-for="s in (selectedBlock.successors || [])" :key="s" class="info-addr" @click="selectBlock(s)">{{ s }}</code>
                            <span v-if="!(selectedBlock.successors || []).length" class="info-none">—</span>
                        </div>
                    </div>
                </div>
                <div class="section" v-if="cfgNodes.length">
                    <div class="section-header">
                        <span class="section-title">
                            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
                                <circle cx="4" cy="8" r="2.5"/>
                                <circle cx="12" cy="3.5" r="2.5"/>
                                <circle cx="12" cy="12.5" r="2.5"/>
                                <path d="M6.5 8h1.5l1.5-4.5M6.5 8h1.5l1.5 4.5"/>
                            </svg>
                            Control Flow Graph
                        </span>
                        <div class="cfg-controls">
                            <button class="ctrl-btn" @click="zoomOut" title="Zoom out">
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"><line x1="3" y1="7" x2="11" y2="7"/></svg>
                            </button>
                            <span class="zoom-label">{{ Math.round(cfgZoom * 100) }}%</span>
                            <button class="ctrl-btn" @click="zoomIn" title="Zoom in">
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"><line x1="7" y1="3" x2="7" y2="11"/><line x1="3" y1="7" x2="11" y2="7"/></svg>
                            </button>
                            <div class="ctrl-divider" />
                            <button class="ctrl-btn ctrl-text" @click="resetZoom" title="Reset zoom">1:1</button>
                            <button class="ctrl-btn ctrl-text" @click="fitZoom" title="Fit to view">Fit</button>
                        </div>
                    </div>

                    <div class="cfg-canvas"
                        ref="cfgWrap"
                        @wheel.prevent="onCfgWheel"
                        @mousedown="onCfgPanStart"
                        @mousemove="onCfgPanMove"
                        @mouseup="onCfgPanEnd"
                        @mouseleave="onCfgPanEnd"
                        :class="{ panning: cfgIsPanning }">
                        <svg class="cfg-grid" :width="cfgCanvasDisplayW" :height="cfgCanvasDisplayH" xmlns="http://www.w3.org/2000/svg">
                            <defs>
                                <pattern id="cfg-dot-pat" x="0" y="0" width="24" height="24" patternUnits="userSpaceOnUse">
                                    <circle cx="1" cy="1" r="1" fill="var(--cfg-dot)"/>
                                </pattern>
                            </defs>
                            <rect width="100%" height="100%" fill="url(#cfg-dot-pat)"/>
                        </svg>
                        <div class="cfg-viewport" :style="{ transform: `translate(${cfgPanX}px, ${cfgPanY}px) scale(${cfgZoom})` }">
                            <svg
                                :viewBox="`0 0 ${cfgWidth} ${cfgHeight}`"
                                :width="cfgWidth"
                                :height="cfgHeight"
                                class="cfg-svg"
                                xmlns="http://www.w3.org/2000/svg">
                                <defs>
                                    <marker id="arrow-default" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto">
                                        <path d="M0,0.5 L6,3.5 L0,6.5 z" fill="var(--edge)"/>
                                    </marker>
                                    <marker id="arrow-out" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto">
                                        <path d="M0,0.5 L6,3.5 L0,6.5 z" fill="var(--edge-out)"/>
                                    </marker>
                                    <marker id="arrow-in" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto">
                                        <path d="M0,0.5 L6,3.5 L0,6.5 z" fill="var(--edge-in)"/>
                                    </marker>
                                    <filter id="node-glow" x="-30%" y="-30%" width="160%" height="160%">
                                        <feDropShadow dx="0" dy="0" stdDeviation="3" flood-color="var(--accent)" flood-opacity="0.4"/>
                                    </filter>
                                </defs>

                                <path
                                    v-for="(edge, idx) in cfgRenderEdges"
                                    :key="`e-${idx}`"
                                    :d="edgePath(edge)"
                                    class="cfg-edge"
                                    :class="{ 'cfg-edge-out': edge.isOut, 'cfg-edge-in': edge.isIn }"
                                    :marker-end="edge.isOut ? 'url(#arrow-out)' : edge.isIn ? 'url(#arrow-in)' : 'url(#arrow-default)'" />

                                <g
                                    v-for="node in cfgNodes"
                                    :key="node.address"
                                    :transform="`translate(${node.x}, ${node.y})`"
                                    class="cfg-node"
                                    :class="{ active: selectedKey === node.address, entry: node.address === entryAddress }"
                                    @click.stop="selectBlock(node.address)">
                                    <rect class="cfg-node-shadow" :width="nodeW" :height="nodeH" rx="6" ry="6" />
                                    <rect class="cfg-node-body" :width="nodeW" :height="nodeH" rx="6" ry="6" />
                                    <rect class="cfg-node-accent" x="0" y="0" width="3" :height="nodeH" rx="1.5" ry="1.5"/>
                                    <text class="cfg-node-addr" x="14" y="24">{{ node.address }}</text>
                                    <text class="cfg-node-meta" x="14" y="42">{{ node.instruction_count || 0 }} insn</text>
                                </g>
                            </svg>
                        </div>
                        <div class="cfg-hint">Scroll to zoom · Drag to pan · Click node to select</div>
                    </div>
                </div>
                <div v-else-if="selectedBlock" class="section">
                    <div class="section-header">
                        <span class="section-title">Control Flow Graph</span>
                    </div>
                    <div class="muted-msg">No CFG edges found for this routine.</div>
                </div>

                <div class="section section-instructions">
                    <div class="section-header">
                        <span class="section-title">
                            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
                                <line x1="3" y1="4" x2="13" y2="4"/>
                                <line x1="3" y1="8" x2="10" y2="8"/>
                                <line x1="3" y1="12" x2="12" y2="12"/>
                            </svg>
                            Instructions
                        </span>
                        <span class="insn-match-count" v-if="instructionQuery">
                            {{ filteredInstructions.length }} match{{ filteredInstructions.length !== 1 ? 'es' : '' }}
                        </span>
                    </div>

                    <div class="insn-toolbar">
                        <div class="insn-search-wrap">
                            <svg class="insn-search-icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                                <circle cx="6.5" cy="6.5" r="4.5"/>
                                <path d="M10 10l3.5 3.5"/>
                            </svg>
                            <input
                                v-model.trim="instructionQuery"
                                class="insn-search"
                                placeholder="Search VIP, mnemonic, instruction…"
                                @keydown.enter.prevent="jumpToNextInstruction" />
                        </div>
                        <select v-model="searchPriority" class="field-select" title="Search priority">
                            <option value="vip">VIP first</option>
                            <option value="mnemonic">Mnemonic</option>
                            <option value="text">Text first</option>
                        </select>
                        <button class="ctrl-btn" @click="jumpToPrevInstruction" title="Previous match">
                            <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><polyline points="9,3 5,7 9,11"/></svg>
                        </button>
                        <button class="ctrl-btn" @click="jumpToNextInstruction" title="Next match">
                            <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><polyline points="5,3 9,7 5,11"/></svg>
                        </button>
                        <div class="ctrl-divider" />
                        <button class="ctrl-btn ctrl-text" @click="promptEditImmediate" title="Edit selected instruction's immediate operand">Imm</button>
                        <button class="ctrl-btn ctrl-text" @click="openInstructionEditor" title="Edit selected instruction">Edit</button>
                        <button class="ctrl-btn ctrl-text" :class="{ active: showPseudoPanel }" @click="showPseudoPanel = !showPseudoPanel" title="Toggle pseudo-ASM view">ASM</button>
                    </div>

                    <div class="asm-panel" v-if="showPseudoPanel && activeInstruction">
                        <div class="asm-panel-label">pseudo-asm</div>
                        <code class="asm-code">{{ toPseudoAsm(activeInstruction) }}</code>
                    </div>

                    <div class="insn-table-wrap" v-if="shownInstructions.length">
                        <table class="insn-table">
                            <thead>
                                <tr>
                                    <th class="col-expand"></th>
                                    <th class="col-idx">#</th>
                                    <th class="col-vip">VIP</th>
                                    <th class="col-mnem">Mnemonic</th>
                                    <th class="col-ops">Ops</th>
                                    <th class="col-insn">Instruction</th>
                                </tr>
                            </thead>
                            <tbody>
                                <template v-for="insn in shownInstructions">
                                    <tr
                                        :key="`row-${insn.index}`"
                                        :data-insn-index="insn.index"
                                        :class="{ 'insn-active': activeInstructionIndex === insn.index }"
                                        @click="activeInstructionIndex = insn.index">
                                        <td class="col-expand">
                                            <button class="expand-btn" @click.stop="toggleExpanded(insn.index)">
                                                <svg viewBox="0 0 10 10" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                                                    <template v-if="isExpanded(insn.index)">
                                                        <line x1="2" y1="5" x2="8" y2="5"/>
                                                    </template>
                                                    <template v-else>
                                                        <line x1="5" y1="2" x2="5" y2="8"/>
                                                        <line x1="2" y1="5" x2="8" y2="5"/>
                                                    </template>
                                                </svg>
                                            </button>
                                        </td>
                                        <td class="col-idx muted">{{ insn.index }}</td>
                                        <td class="col-vip"><code class="mono muted">{{ insn.vip }}</code></td>
                                        <td class="col-mnem"><code class="mono accent">{{ (insn.display_mnemonic || insn.mnemonic || '').toUpperCase() }}</code></td>
                                        <td class="col-ops muted">{{ insn.operand_count }}</td>
                                        <td class="col-insn"><code class="mono insn-text">{{ insn.display_text || insn.text }}</code></td>
                                    </tr>
                                    <tr v-if="isExpanded(insn.index)" :key="`detail-${insn.index}`" class="insn-detail-row">
                                        <td colspan="6">
                                            <div class="insn-detail-grid">
                                                <div class="insn-detail-col">
                                                    <div class="detail-label">Descriptor</div>
                                                    <div class="chip-group">
                                                        <span class="chip">name: {{ insn.desc ? insn.desc.name : '—' }}</span>
                                                        <span class="chip" :class="{ 'chip-warn': insn.desc && insn.desc.is_volatile }">volatile: {{ insn.desc && insn.desc.is_volatile ? 'yes' : 'no' }}</span>
                                                        <span class="chip" :class="{ 'chip-warn': insn.desc && insn.desc.memory_write }">mem_write: {{ insn.desc && insn.desc.memory_write ? 'yes' : 'no' }}</span>
                                                        <span class="chip">access_sz: {{ insn.desc ? insn.desc.access_size_index : '—' }}</span>
                                                        <span class="chip">mem_op: {{ insn.desc ? insn.desc.memory_operand_index : '—' }}</span>
                                                    </div>
                                                    <div class="chip-group">
                                                        <span class="chip">op_types: {{ toTextList(insn.desc ? insn.desc.operand_types : []) }}</span>
                                                    </div>
                                                    <div class="chip-group">
                                                        <span class="chip">br_vip: {{ toTextList(insn.desc ? insn.desc.branch_operands_vip : []) }}</span>
                                                        <span class="chip">br_rip: {{ toTextList(insn.desc ? insn.desc.branch_operands_rip : []) }}</span>
                                                    </div>
                                                </div>
                                                <div class="insn-detail-col">
                                                    <div class="detail-label">Operands</div>
                                                    <table class="operand-table">
                                                        <thead>
                                                            <tr>
                                                                <th>#</th>
                                                                <th>Type</th>
                                                                <th>Kind</th>
                                                                <th>Bits</th>
                                                                <th>Value</th>
                                                            </tr>
                                                        </thead>
                                                        <tbody>
                                                            <tr v-for="(op, opIndex) in (insn.operands || [])" :key="`op-${insn.index}-${opIndex}`">
                                                                <td class="muted">{{ opIndex }}</td>
                                                                <td><code class="mono muted">{{ op.type }}</code></td>
                                                                <td class="muted">{{ op.kind }}</td>
                                                                <td class="muted">{{ op.bit_count }}</td>
                                                                <td><code class="mono">{{ op.text }}</code></td>
                                                            </tr>
                                                        </tbody>
                                                    </table>
                                                </div>
                                            </div>
                                        </td>
                                    </tr>
                                </template>
                            </tbody>
                        </table>
                    </div>
                    <div v-else class="muted-msg">No instructions match the current filter.</div>
                </div>
            </template>
        </div>

        <div v-if="instructionEditor.open" class="modal-overlay" @click.self="closeInstructionEditor">
            <div class="modal-card">
                <div class="modal-header">
                    <h3 class="modal-title">Edit Instruction</h3>
                    <p class="modal-subtitle">Schema-guided editing with autocomplete</p>
                    <button class="modal-close" @click="closeInstructionEditor">
                        <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><line x1="2" y1="2" x2="12" y2="12"/><line x1="12" y1="2" x2="2" y2="12"/></svg>
                    </button>
                </div>

                <div class="modal-body">
                    <div class="field-group">
                        <label class="field-label">Mnemonic</label>
                        <div class="field-wrap">
                            <input
                                v-model.trim="instructionEditor.mnemonic"
                                ref="mnemonicInput"
                                class="field-input"
                                @focus="onMnemonicFocus"
                                @input="onMnemonicInput"
                                @keydown="onFieldKeyDown($event, 'mnemonic')"
                                placeholder="e.g. mov" />
                            <div v-if="showSuggestionsFor('mnemonic')" class="suggestion-box">
                                <button
                                    v-for="(entry, idx) in activeSuggestions"
                                    :key="`m-s-${entry}-${idx}`"
                                    class="suggestion-item"
                                    :class="{ active: idx === suggestionState.highlighted }"
                                    @mousedown.prevent="commitSuggestion('mnemonic', entry)">
                                    {{ entry }}
                                </button>
                            </div>
                        </div>
                        <span v-if="mnemonicValidationError" class="field-error">{{ mnemonicValidationError }}</span>
                    </div>

                    <div class="chip-group" v-if="activeInstructionSchema">
                        <span class="chip chip-accent">op_types: {{ (activeInstructionSchema.operand_types || []).join(', ') || '—' }}</span>
                        <span class="chip" :class="{ 'chip-warn': activeInstructionSchema.is_volatile }">volatile: {{ activeInstructionSchema.is_volatile ? 'yes' : 'no' }}</span>
                        <span class="chip" :class="{ 'chip-warn': activeInstructionSchema.memory_write }">mem_write: {{ activeInstructionSchema.memory_write ? 'yes' : 'no' }}</span>
                        <span class="chip chip-muted" v-if="activeInstructionSchema.example_line">eg: {{ activeInstructionSchema.example_line }}</span>
                    </div>

                    <div class="field-group" v-if="expectedOperandTypes.length > 0">
                        <label class="field-label">Operands</label>
                        <div v-for="(opType, opIndex) in expectedOperandTypes" :key="`ed-op-${opIndex}`" class="operand-row">
                            <span class="op-type-badge">{{ opType }}</span>
                            <div class="field-wrap">
                                <input
                                    v-model.trim="instructionEditor.operands[opIndex]"
                                    class="field-input"
                                    :ref="`operandInput-${opIndex}`"
                                    @focus="onOperandFocus(opIndex, opType)"
                                    @input="onOperandInput(opIndex, opType)"
                                    @keydown="onFieldKeyDown($event, `operand:${opIndex}`)"
                                    :placeholder="operandPlaceholder(opType)" />
                                <div v-if="showSuggestionsFor(`operand:${opIndex}`)" class="suggestion-box">
                                    <button
                                        v-for="(entry, idx) in activeSuggestions"
                                        :key="`o-s-${opIndex}-${entry}-${idx}`"
                                        class="suggestion-item"
                                        :class="{ active: idx === suggestionState.highlighted }"
                                        @mousedown.prevent="commitSuggestion(`operand:${opIndex}`, entry)">
                                        {{ entry }}
                                    </button>
                                </div>
                            </div>
                            <div class="operand-hint">{{ operandHint(opType, opIndex) }}</div>
                            <span v-if="operandValidationErrors[opIndex]" class="field-error">{{ operandValidationErrors[opIndex] }}</span>
                        </div>
                    </div>
                    <div v-else-if="instructionEditor.mnemonic && !mnemonicValidationError" class="muted-msg">No operands for this mnemonic.</div>

                    <div class="field-group">
                        <label class="field-label">Preview</label>
                        <code class="preview-code">{{ instructionEditorPreview || '—' }}</code>
                        <span v-if="instructionValidationError" class="field-error">{{ instructionValidationError }}</span>
                    </div>
                </div>

                <div class="modal-footer">
                    <button class="btn btn-ghost" @click="closeInstructionEditor">Cancel</button>
                    <button class="btn btn-primary" :disabled="!instructionEditorCanSubmit" @click="submitInstructionEditor">Apply changes</button>
                </div>
            </div>
        </div>

        <div v-if="immediateEditor.open" class="modal-overlay" @click.self="closeImmediateEditor">
            <div class="modal-card modal-card-sm">
                <div class="modal-header">
                    <h3 class="modal-title">Edit Immediate</h3>
                    <p class="modal-subtitle">Update a single immediate operand value</p>
                    <button class="modal-close" @click="closeImmediateEditor">
                        <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><line x1="2" y1="2" x2="12" y2="12"/><line x1="12" y1="2" x2="2" y2="12"/></svg>
                    </button>
                </div>

                <div class="modal-body">
                    <div class="chip-group" v-if="selectedBlock && activeInstruction">
                        <span class="chip chip-accent">block: {{ selectedBlock.address }}</span>
                        <span class="chip">insn: {{ immediateEditor.instruction }}</span>
                    </div>

                    <div class="field-group">
                        <label class="field-label">Immediate operand</label>
                        <select v-if="immediateOperandOptions.length > 1" v-model.number="immediateEditor.operand" class="field-select full-width" @change="syncImmediateEditorValue">
                            <option v-for="op in immediateOperandOptions" :key="`imm-${op.index}`" :value="op.index">
                                Operand {{ op.index }} · {{ op.text || op.kind }}
                            </option>
                        </select>
                        <span v-else class="field-static muted">
                            Operand {{ currentImmediateOperand ? currentImmediateOperand.index : '—' }} · {{ currentImmediateOperand ? (currentImmediateOperand.text || currentImmediateOperand.kind) : 'n/a' }}
                        </span>
                    </div>

                    <div class="chip-group" v-if="currentImmediateOperand">
                        <span class="chip">type: {{ currentImmediateOperand.type || 'immediate' }}</span>
                        <span class="chip">kind: {{ currentImmediateOperand.kind || 'immediate' }}</span>
                        <span class="chip">bits: {{ currentImmediateOperand.bit_count || '—' }}</span>
                        <span class="chip chip-accent">current: {{ currentImmediateOperand.text || '—' }}</span>
                    </div>

                    <div class="field-group">
                        <label class="field-label">New value</label>
                        <input
                            v-model.trim="immediateEditor.value"
                            ref="immediateValueInput"
                            class="field-input"
                            placeholder="decimal, hex (0x…), or negative"
                            @keydown.enter.prevent="submitImmediateEditor" />
                        <span class="field-error" v-if="immediateEditorValidationError">{{ immediateEditorValidationError }}</span>
                    </div>

                    <div class="field-group">
                        <label class="field-label">Preview</label>
                        <code class="preview-code">{{ immediateEditorPreview || '—' }}</code>
                    </div>
                </div>

                <div class="modal-footer">
                    <button class="btn btn-ghost" @click="closeImmediateEditor">Cancel</button>
                    <button class="btn btn-primary" :disabled="!immediateEditorCanSubmit" @click="submitImmediateEditor">Apply</button>
                </div>
            </div>
        </div>

    </div>
</template>

<script>
import Vue from 'vue'

export default {
    props: {
        routine: {
            type: Object,
            required: true
        },
        editorSchema: {
            type: Object,
            default: () => ({
                mnemonics: [],
                registers: [],
                instructions: []
            })
        }
    },
    data() {
        return {
            sizes: [32, 68],
            resizeIdx: 0,

            selectedKey: null,

            instructionQuery: '',
            searchPriority: 'vip',
            showPseudoPanel: true,
            activeInstructionIndex: null,
            expandedInstructions: {},
            currentMatchPos: -1,

            nodeW: 195,
            nodeH: 54,
            xGap: 64,
            yGap: 28,
            cfgPadding: 20,

            cfgZoom: 1,
            cfgPanX: 0,
            cfgPanY: 0,
            cfgIsPanning: false,
            cfgPanStartX: 0,
            cfgPanStartY: 0,
            cfgCanvasDisplayW: 0,
            cfgCanvasDisplayH: 0,

            instructionEditor: {
                open: false,
                mnemonic: '',
                operands: []
            },
            immediateEditor: {
                open: false,
                block: '',
                instruction: null,
                operand: null,
                value: ''
            },
            suggestionState: {
                field: '',
                visible: false,
                highlighted: 0,
                items: []
            }
        }
    },

    methods: {
        mouseDown(evt) {
            evt.preventDefault()
            evt.stopPropagation()
            this.resizeIdx = 1
        },
        mouseUp() {
            if (this.dragging) this.resizeIdx = 0
        },
        mouseMove(evt) {
            if (!this.dragging) return
            if (evt.buttons === 0) { this.resizeIdx = 0; return }
            evt.preventDefault()
            const maxWidth = this.$refs.container.clientWidth
            let pct = (evt.clientX / maxWidth) * 100
            if (pct < 15) pct = 15
            if (pct > 75) pct = 75
            Vue.set(this.sizes, 0, pct)
            Vue.set(this.sizes, 1, 100 - pct)
        },

        selectBlock(key) {
            this.selectedKey = key
            this.activeInstructionIndex = null
            this.currentMatchPos = -1
            this.$nextTick(() => this.fitZoom())
        },

        nodeAnchor(node, side) {
            if (side === 'left') return { x: node.x, y: node.y + this.nodeH / 2 }
            return { x: node.x + this.nodeW, y: node.y + this.nodeH / 2 }
        },
        edgePath(edge) {
            const dx = Math.max(24, Math.abs(edge.x2 - edge.x1) * 0.42)
            const c1x = edge.x1 + dx
            const c2x = edge.x2 - dx
            return `M ${edge.x1} ${edge.y1} C ${c1x} ${edge.y1}, ${c2x} ${edge.y2}, ${edge.x2} ${edge.y2}`
        },

        onCfgPanStart(e) {
            if (e.button !== 0) return
            if (e.target && e.target.closest && e.target.closest('.cfg-node')) return
            this.cfgIsPanning = true
            this.cfgPanStartX = e.clientX - this.cfgPanX
            this.cfgPanStartY = e.clientY - this.cfgPanY
            e.preventDefault()
        },
        onCfgPanMove(e) {
            if (!this.cfgIsPanning) return
            this.cfgPanX = e.clientX - this.cfgPanStartX
            this.cfgPanY = e.clientY - this.cfgPanStartY
        },
        onCfgPanEnd() {
            this.cfgIsPanning = false
        },

        onCfgWheel(e) {
            const wrap = this.$refs.cfgWrap
            if (!wrap) return
            const rect = wrap.getBoundingClientRect()
            const mx = e.clientX - rect.left
            const my = e.clientY - rect.top
            const factor = e.deltaY < 0 ? 1.12 : 0.89
            const newZoom = Math.max(0.12, Math.min(4, this.cfgZoom * factor))
            this.cfgPanX = mx + (this.cfgPanX - mx) * (newZoom / this.cfgZoom)
            this.cfgPanY = my + (this.cfgPanY - my) * (newZoom / this.cfgZoom)
            this.cfgZoom = +newZoom.toFixed(3)
        },
        zoomIn() {
            const wrap = this.$refs.cfgWrap
            const cx = wrap ? wrap.clientWidth / 2 : 0
            const cy = wrap ? wrap.clientHeight / 2 : 0
            const newZoom = Math.min(4, +(this.cfgZoom * 1.2).toFixed(3))
            this.cfgPanX = cx + (this.cfgPanX - cx) * (newZoom / this.cfgZoom)
            this.cfgPanY = cy + (this.cfgPanY - cy) * (newZoom / this.cfgZoom)
            this.cfgZoom = newZoom
        },
        zoomOut() {
            const wrap = this.$refs.cfgWrap
            const cx = wrap ? wrap.clientWidth / 2 : 0
            const cy = wrap ? wrap.clientHeight / 2 : 0
            const newZoom = Math.max(0.12, +(this.cfgZoom / 1.2).toFixed(3))
            this.cfgPanX = cx + (this.cfgPanX - cx) * (newZoom / this.cfgZoom)
            this.cfgPanY = cy + (this.cfgPanY - cy) * (newZoom / this.cfgZoom)
            this.cfgZoom = newZoom
        },
        resetZoom() {
            this.cfgZoom = 1
            this.cfgPanX = 0
            this.cfgPanY = 0
        },
        fitZoom() {
            this.$nextTick(() => {
                const wrap = this.$refs.cfgWrap
                if (!wrap || !this.cfgWidth || !this.cfgHeight) return
                this.cfgCanvasDisplayW = wrap.clientWidth
                this.cfgCanvasDisplayH = wrap.clientHeight
                const pad = 28
                const fitW = (wrap.clientWidth - pad * 2) / this.cfgWidth
                const fitH = (wrap.clientHeight - pad * 2) / this.cfgHeight
                const fit = Math.min(fitW, fitH, 3)
                this.cfgZoom = Math.max(0.12, +fit.toFixed(3))
                this.cfgPanX = (wrap.clientWidth - this.cfgWidth * this.cfgZoom) / 2
                this.cfgPanY = (wrap.clientHeight - this.cfgHeight * this.cfgZoom) / 2
            })
        },

        toPseudoAsm(insn) {
            const raw = (insn.display_text || insn.text || '').trim()
            const mnemonic = (insn.display_mnemonic || insn.mnemonic || '').toUpperCase()
            const escapedMnemonic = (insn.mnemonic || '').replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
            let operands = raw
            if (escapedMnemonic) {
                operands = operands.replace(new RegExp(`^${escapedMnemonic}\\s*`, 'i'), '')
            }
            operands = operands
                .replace(/\bsp\b/gi, 'rsp')
                .replace(/\bvr(\d+)\b/gi, 'v$1')
                .replace(/\s+/g, ' ')
                .trim()
            return operands ? `${mnemonic} ${operands}` : mnemonic
        },
        instructionMatches(insn) {
            const q = this.instructionQuery.toLowerCase()
            if (!q) return true
            return (
                (insn.vip || '').toLowerCase().includes(q) ||
                (insn.display_mnemonic || insn.mnemonic || '').toLowerCase().includes(q) ||
                (insn.display_text || insn.text || '').toLowerCase().includes(q)
            )
        },
        fieldMatchLevel(value, query) {
            if (!query) return 0
            const normalized = (value || '').toLowerCase()
            if (!normalized) return 99
            if (normalized === query) return 0
            if (normalized.startsWith(query)) return 1
            if (normalized.includes(query)) return 2
            return 99
        },
        scoreInstruction(insn) {
            const q = this.instructionQuery.toLowerCase()
            const levels = {
                vip: this.fieldMatchLevel(insn.vip, q),
                mnemonic: this.fieldMatchLevel(insn.display_mnemonic || insn.mnemonic, q),
                text: this.fieldMatchLevel(insn.display_text || insn.text, q)
            }
            const orders = { vip: ['vip', 'mnemonic', 'text'], mnemonic: ['mnemonic', 'vip', 'text'], text: ['text', 'mnemonic', 'vip'] }
            const order = orders[this.searchPriority] || orders.vip
            return [levels[order[0]], levels[order[1]], levels[order[2]], insn.index]
        },
        jumpToInstructionAt(pos) {
            if (!this.filteredInstructions.length) return
            const idx = (pos + this.filteredInstructions.length) % this.filteredInstructions.length
            this.currentMatchPos = idx
            const target = this.filteredInstructions[idx]
            this.activeInstructionIndex = target.index
            this.$nextTick(() => {
                const row = this.$el.querySelector(`tr[data-insn-index='${target.index}']`)
                if (row) row.scrollIntoView({ block: 'nearest' })
            })
        },
        jumpToNextInstruction() { this.jumpToInstructionAt(this.currentMatchPos + 1) },
        jumpToPrevInstruction() { this.jumpToInstructionAt(this.currentMatchPos - 1) },
        toggleExpanded(index) {
            const next = { ...this.expandedInstructions }
            next[index] = !next[index]
            this.expandedInstructions = next
        },
        isExpanded(index) { return !!this.expandedInstructions[index] },
        toTextList(values) {
            if (!values || !values.length) return '—'
            return values.join(', ')
        },

        promptEditImmediate() { this.openImmediateEditor() },
        openImmediateEditor() {
            if (!this.selectedBlock || !this.activeInstruction) {
                window.alert('Select a block and instruction first.')
                return
            }
            const immediateOperands = this.immediateOperandOptions
            if (!immediateOperands.length) {
                window.alert('Selected instruction has no immediate operand.')
                return
            }
            const selectedOperand = immediateOperands[0]
            this.immediateEditor = {
                open: true,
                block: this.selectedBlock.address,
                instruction: this.activeInstruction.index,
                operand: selectedOperand.index,
                value: this.initialImmediateValue(selectedOperand)
            }
            this.$nextTick(() => {
                const target = this.$refs.immediateValueInput
                if (target && target.focus) target.focus()
            })
        },
        initialImmediateValue(op) {
            if (!op) return '0'
            if (op.immediate && op.immediate.i64 !== undefined && op.immediate.i64 !== null) return String(op.immediate.i64)
            if (op.immediate && op.immediate.u64 !== undefined && op.immediate.u64 !== null) return String(op.immediate.u64)
            if (op.text !== undefined && op.text !== null && String(op.text).trim()) return String(op.text).trim()
            return '0'
        },
        syncImmediateEditorValue() {
            const current = this.currentImmediateOperand
            if (current) this.immediateEditor.value = this.initialImmediateValue(current)
        },
        closeImmediateEditor() { this.immediateEditor.open = false },
        submitImmediateEditor() {
            if (!this.selectedBlock || !this.activeInstruction) return
            if (!this.immediateEditorCanSubmit) {
                window.alert(this.immediateEditorValidationError || 'Fix validation issues first.')
                return
            }
            this.$emit('edit-immediate', {
                block: this.selectedBlock.address,
                instruction: this.activeInstruction.index,
                operand: this.immediateEditor.operand,
                value: String(this.immediateEditor.value || '').trim()
            })
            this.closeImmediateEditor()
        },

        openInstructionEditor() {
            if (!this.selectedBlock || !this.activeInstruction) {
                window.alert('Select a block and instruction first.')
                return
            }
            const source = this.activeInstruction
            const tokenized = this.tokenizeInstructionText(source.display_text || source.text || '')
            const mnemonic = (tokenized.mnemonic || source.mnemonic || source.display_mnemonic || '').toLowerCase()
            let operands = (tokenized.operands || []).map((x) => String(x || '').trim())
            if (!operands.length) operands = (source.display_operands || []).map((x) => String(x || '').trim())
            if (!operands.length) operands = (source.operands || []).map((x) => String((x && x.text) || '').trim())
            this.instructionEditor = { open: true, mnemonic, operands }
            this.suggestionState.visible = false
            this.syncEditorOperandCount()
        },
        closeInstructionEditor() {
            this.instructionEditor.open = false
            this.suggestionState.visible = false
        },
        syncEditorOperandCount() {
            const need = this.expectedOperandTypes.length
            const next = (this.instructionEditor.operands || []).slice(0, need)
            while (next.length < need) next.push('')
            this.instructionEditor.operands = next
        },
        tokenizeInstructionText(text) {
            const raw = String(text || '').trim()
            if (!raw) return { mnemonic: '', operands: [] }
            const parts = raw.split(/\s+/)
            const mnemonic = (parts.shift() || '').toLowerCase()
            const tail = raw.slice(raw.indexOf(mnemonic) + mnemonic.length).trim()
            if (!tail) return { mnemonic, operands: [] }
            const ops = tail.split(',').map((x) => x.trim()).filter(Boolean)
            return { mnemonic, operands: ops }
        },
        operandPlaceholder(opType) {
            if (opType === 'read_imm') return '0x20:64'
            if (opType === 'read_reg' || opType === 'write' || opType === 'readwrite') return 'register name'
            return 'register or immediate'
        },
        operandSuggestions(opType) {
            if (opType === 'read_reg' || opType === 'write' || opType === 'readwrite') return this.editorRegisterList
            if (opType === 'read_imm') return ['0x10:64', '0x20:64', '1:1', '0:1']
            return [...this.editorRegisterList.slice(0, 24), '0x10:64', '0x20:64', '1:1', '0:1']
        },
        operandExampleList(opType) {
            if (opType === 'read_reg' || opType === 'write' || opType === 'readwrite') return this.editorRegisterList.slice(0, 3)
            if (opType === 'read_imm') return ['0x10:64', '0x20:64', '1:1']
            return [...this.editorRegisterList.slice(0, 2), '0x10:64', '0x20:64']
        },
        operandHint(opType, index) {
            const examples = this.operandExampleList(opType, index)
            return `Expected: ${opType}. Accepted: ${examples.join(', ') || 'n/a'}.`
        },
        normalizeForCompare(value) { return String(value || '').trim().toLowerCase() },
        isKnownRegister(token) {
            const n = this.normalizeForCompare(token)
            if (!n) return false
            return this.editorRegisterList.some((x) => this.normalizeForCompare(x) === n)
        },
        isImmediateToken(token, requireBits) {
            const t = String(token || '').trim().toLowerCase()
            if (!t) return false
            const rNoBits = /^-?(0x[0-9a-f]+|\d+)$/i
            const rWithBits = /^-?(0x[0-9a-f]+|\d+):(\d+)$/i
            if (requireBits) {
                const m = t.match(rWithBits)
                if (!m) return false
                const bits = Number.parseInt(m[2], 10)
                return Number.isInteger(bits) && bits > 0 && bits <= 64
            }
            if (rNoBits.test(t)) return true
            const m = t.match(rWithBits)
            if (!m) return false
            const bits = Number.parseInt(m[2], 10)
            return Number.isInteger(bits) && bits > 0 && bits <= 64
        },
        validateOperand(opType, token) {
            const t = String(token || '').trim()
            if (!t) return 'Operand is required.'
            if (opType === 'read_reg' || opType === 'write' || opType === 'readwrite') {
                return this.isKnownRegister(t) ? '' : 'Must match a known VTIL register name.'
            }
            if (opType === 'read_imm') return this.isImmediateToken(t, true) ? '' : 'Immediate must be value:bits, e.g. 0x20:64.'
            if (opType === 'read_any') {
                if (this.isKnownRegister(t) || this.isImmediateToken(t, false)) return ''
                return 'Must be a known register or an immediate.'
            }
            return 'Unsupported operand type.'
        },
        refreshSuggestions(field, query, pool) {
            const q = this.normalizeForCompare(query)
            const unique = Array.from(new Set((pool || []).filter(Boolean)))
            const starts = unique.filter((x) => this.normalizeForCompare(x).startsWith(q))
            const contains = unique.filter((x) => !starts.includes(x) && this.normalizeForCompare(x).includes(q))
            const items = [...starts, ...contains].slice(0, 12)
            this.suggestionState = { field, visible: items.length > 0, highlighted: 0, items }
        },
        showSuggestionsFor(field) {
            return this.suggestionState.visible && this.suggestionState.field === field && this.activeSuggestions.length > 0
        },
        onMnemonicFocus() { this.refreshSuggestions('mnemonic', this.instructionEditor.mnemonic, this.editorMnemonicList) },
        onMnemonicInput() { this.refreshSuggestions('mnemonic', this.instructionEditor.mnemonic, this.editorMnemonicList) },
        onOperandFocus(index, opType) { this.refreshSuggestions(`operand:${index}`, this.instructionEditor.operands[index], this.operandSuggestions(opType)) },
        onOperandInput(index, opType) { this.refreshSuggestions(`operand:${index}`, this.instructionEditor.operands[index], this.operandSuggestions(opType)) },
        commitSuggestion(field, value) {
            if (field === 'mnemonic') {
                this.instructionEditor.mnemonic = value
            } else if (field.startsWith('operand:')) {
                const idx = Number.parseInt(field.split(':')[1], 10)
                if (Number.isInteger(idx)) this.$set(this.instructionEditor.operands, idx, value)
            }
            this.suggestionState.visible = false
        },
        focusNextEditorField(field) {
            if (field === 'mnemonic') {
                if (this.expectedOperandTypes.length > 0) {
                    const target = this.$refs['operandInput-0']
                    if (target && target.focus) target.focus()
                }
                return
            }
            if (field.startsWith('operand:')) {
                const idx = Number.parseInt(field.split(':')[1], 10)
                if (!Number.isInteger(idx)) return
                const next = idx + 1
                if (next < this.expectedOperandTypes.length) {
                    const target = this.$refs[`operandInput-${next}`]
                    if (target && target.focus) target.focus()
                }
            }
        },
        onFieldKeyDown(event, field) {
            if (!this.showSuggestionsFor(field)) {
                if (event.key === 'Enter' && field !== 'mnemonic' && this.instructionEditorCanSubmit) {
                    event.preventDefault()
                    this.submitInstructionEditor()
                }
                return
            }
            if (event.key === 'ArrowDown') {
                event.preventDefault()
                this.suggestionState.highlighted = (this.suggestionState.highlighted + 1) % this.activeSuggestions.length
                return
            }
            if (event.key === 'ArrowUp') {
                event.preventDefault()
                this.suggestionState.highlighted = (this.suggestionState.highlighted + this.activeSuggestions.length - 1) % this.activeSuggestions.length
                return
            }
            if (event.key === 'Enter' || event.key === 'Tab') {
                event.preventDefault()
                const pick = this.activeSuggestions[this.suggestionState.highlighted]
                if (pick) this.commitSuggestion(field, pick)
                this.focusNextEditorField(field)
                return
            }
            if (event.key === 'Escape') this.suggestionState.visible = false
        },
        submitInstructionEditor() {
            if (!this.selectedBlock || !this.activeInstruction) return
            if (!this.instructionEditorCanSubmit) {
                window.alert(this.instructionValidationError || 'Fix validation issues first.')
                return
            }
            const mnemonic = (this.instructionEditor.mnemonic || '').trim().toLowerCase()
            const operands = (this.instructionEditor.operands || []).map((x) => String(x || '').trim()).filter((x, idx) => {
                if (idx < this.expectedOperandTypes.length) return true
                return x.length > 0
            })
            if (operands.some((x) => !x)) { window.alert('All operand fields must be filled.'); return }
            const text = operands.length ? `${mnemonic} ${operands.join(', ')}` : mnemonic
            this.$emit('edit-instruction', {
                block: this.selectedBlock.address,
                instruction: this.activeInstruction.index,
                text
            })
            this.closeInstructionEditor()
        }
    },

    computed: {
        dragging() { return this.resizeIdx !== 0 },
        blockRows() { return this.routine.blocks || [] },
        entryAddress() {
            return this.routine.entry_point || (this.blockRows[0] ? this.blockRows[0].address : null)
        },
        selectedBlock() {
            return this.blockRows.find((x) => x.address === this.selectedKey) || this.blockRows[0] || null
        },
        filteredInstructions() {
            const list = this.selectedBlock ? (this.selectedBlock.instructions || []) : []
            const matches = list.filter((insn) => this.instructionMatches(insn))
            if (!this.instructionQuery) return matches
            const scored = matches.map((insn) => ({ insn, score: this.scoreInstruction(insn) }))
            scored.sort((a, b) => {
                for (let i = 0; i < a.score.length; i += 1) {
                    if (a.score[i] !== b.score[i]) return a.score[i] - b.score[i]
                }
                return 0
            })
            return scored.map((x) => x.insn)
        },
        shownInstructions() { return this.filteredInstructions },
        activeInstruction() {
            if (!this.selectedBlock) return null
            const all = this.selectedBlock.instructions || []
            if (this.activeInstructionIndex === null || this.activeInstructionIndex === undefined) {
                return this.shownInstructions[0] || all[0] || null
            }
            return all.find((x) => x.index === this.activeInstructionIndex) || null
        },
        levelMap() {
            const levels = {}
            if (!this.blockRows.length) return levels
            const adjacency = {}
            for (const blk of this.blockRows) adjacency[blk.address] = blk.successors || []
            const start = this.entryAddress
            const queue = []
            if (start) { levels[start] = 0; queue.push(start) }
            while (queue.length) {
                const cur = queue.shift()
                const curLevel = levels[cur]
                for (const nxt of (adjacency[cur] || [])) {
                    if (levels[nxt] !== undefined) continue
                    levels[nxt] = curLevel + 1
                    queue.push(nxt)
                }
            }
            let tail = Math.max(0, ...Object.values(levels))
            for (const blk of this.blockRows) {
                if (levels[blk.address] === undefined) { tail += 1; levels[blk.address] = tail }
            }
            return levels
        },
        nodesByLevel() {
            const grouped = {}
            for (const blk of this.blockRows) {
                const lvl = this.levelMap[blk.address] || 0
                if (!grouped[lvl]) grouped[lvl] = []
                grouped[lvl].push(blk)
            }
            const ordered = []
            for (const key of Object.keys(grouped).map(Number).sort((a, b) => a - b)) {
                ordered.push(grouped[key])
            }
            return ordered
        },
        cfgNodes() {
            const nodes = []
            this.nodesByLevel.forEach((levelNodes, level) => {
                levelNodes.forEach((blk, rowIndex) => {
                    nodes.push({
                        ...blk,
                        x: this.cfgPadding + level * (this.nodeW + this.xGap),
                        y: this.cfgPadding + rowIndex * (this.nodeH + this.yGap)
                    })
                })
            })
            return nodes
        },
        cfgNodeMap() {
            const map = {}
            for (const n of this.cfgNodes) map[n.address] = n
            return map
        },
        cfgRenderEdges() {
            const out = []
            for (const e of (this.routine.cfg_edges || [])) {
                const src = this.cfgNodeMap[e.from]
                const dst = this.cfgNodeMap[e.to]
                if (!src || !dst) continue
                const from = this.nodeAnchor(src, 'right')
                const to = this.nodeAnchor(dst, 'left')
                out.push({
                    x1: from.x, y1: from.y, x2: to.x, y2: to.y,
                    isOut: this.selectedKey ? this.selectedKey === e.from : false,
                    isIn: this.selectedKey ? this.selectedKey === e.to : false
                })
            }
            return out
        },
        cfgWidth() {
            const levelCount = Math.max(this.nodesByLevel.length, 1)
            return this.cfgPadding * 2 + levelCount * this.nodeW + (levelCount - 1) * this.xGap
        },
        cfgHeight() {
            const maxRows = Math.max(1, ...this.nodesByLevel.map((g) => g.length || 1))
            return this.cfgPadding * 2 + maxRows * this.nodeH + (maxRows - 1) * this.yGap
        },
        editorMnemonicList() { return (this.editorSchema && this.editorSchema.mnemonics) ? this.editorSchema.mnemonics : [] },
        editorRegisterList() { return (this.editorSchema && this.editorSchema.registers) ? this.editorSchema.registers : [] },
        editorInstructionRows() { return (this.editorSchema && this.editorSchema.instructions) ? this.editorSchema.instructions : [] },
        editorInstructionMap() {
            const map = {}
            for (const row of this.editorInstructionRows) {
                if (row && row.name) map[String(row.name).toLowerCase()] = row
            }
            return map
        },
        immediateOperandOptions() {
            if (!this.activeInstruction || !Array.isArray(this.activeInstruction.operands)) return []
            return this.activeInstruction.operands
                .map((op, index) => ({ ...op, index }))
                .filter((op) => op.kind === 'immediate')
        },
        currentImmediateOperand() {
            if (!this.immediateOperandOptions.length) return null
            const targetIndex = this.immediateEditor.operand
            return this.immediateOperandOptions.find((op) => op.index === targetIndex) || this.immediateOperandOptions[0] || null
        },
        activeInstructionSchema() {
            const key = (this.instructionEditor.mnemonic || '').toLowerCase()
            return this.editorInstructionMap[key] || null
        },
        expectedOperandTypes() { return this.activeInstructionSchema ? (this.activeInstructionSchema.operand_types || []) : [] },
        mnemonicValidationError() {
            const value = (this.instructionEditor.mnemonic || '').trim().toLowerCase()
            if (!value) return 'Mnemonic is required.'
            if (!this.editorInstructionMap[value]) return 'Unknown VTIL mnemonic.'
            return ''
        },
        operandValidationErrors() {
            return this.expectedOperandTypes.map((opType, idx) => {
                const token = (this.instructionEditor.operands || [])[idx] || ''
                return this.validateOperand(opType, token)
            })
        },
        instructionValidationError() {
            if (this.mnemonicValidationError) return this.mnemonicValidationError
            return this.operandValidationErrors.find((x) => !!x) || ''
        },
        instructionEditorCanSubmit() {
            if (!this.instructionEditor.open) return false
            if (this.mnemonicValidationError) return false
            return !this.operandValidationErrors.some((x) => !!x)
        },
        activeSuggestions() { return this.suggestionState.items || [] },
        instructionEditorPreview() {
            const mnemonic = (this.instructionEditor.mnemonic || '').trim()
            if (!mnemonic) return ''
            const ops = (this.instructionEditor.operands || []).map((x) => String(x || '').trim()).filter(Boolean)
            return ops.length ? `${mnemonic} ${ops.join(', ')}` : mnemonic
        },
        immediateEditorValidationError() {
            const value = String(this.immediateEditor.value || '').trim()
            if (!this.immediateEditor.open) return ''
            if (!this.currentImmediateOperand) return 'Select an immediate operand first.'
            if (!value) return 'Immediate value is required.'
            return /^-?(0x[0-9a-f]+|\d+)$/i.test(value) ? '' : 'Use decimal, hex (0x…), or a negative value.'
        },
        immediateEditorCanSubmit() { return this.immediateEditor.open && !this.immediateEditorValidationError },
        immediateEditorPreview() {
            if (!this.immediateEditor.open || !this.currentImmediateOperand) return ''
            const value = String(this.immediateEditor.value || '').trim()
            if (!value) return ''
            return `operand ${this.currentImmediateOperand.index}: ${value}`
        }
    },

    watch: {
        blockRows(newVal) {
            const hasSelected = newVal.some((x) => x.address === this.selectedKey)
            if (!hasSelected) {
                if (newVal.length) this.selectBlock(newVal[0].address)
                else this.selectedKey = null
            } else {
                this.$nextTick(() => this.fitZoom())
            }
        },
        instructionQuery() { this.currentMatchPos = -1; this.activeInstructionIndex = null },
        searchPriority() { this.currentMatchPos = -1; this.activeInstructionIndex = null },
        'instructionEditor.mnemonic'() { this.syncEditorOperandCount() },
        expectedOperandTypes() { this.syncEditorOperandCount() }
    },

    mounted() {
        window.addEventListener('mousemove', this.mouseMove)
        window.addEventListener('mouseup', this.mouseUp)
        this.$nextTick(() => this.fitZoom())
    },
    beforeDestroy() {
        window.removeEventListener('mousemove', this.mouseMove)
        window.removeEventListener('mouseup', this.mouseUp)
    }
}
</script>

<style scoped>
.layout {
    display: flex;
    height: 100%;
    width: 100%;
    overflow: hidden;
    background: var(--bg);
}
.layout.dragging { user-select: none; }

.pane {
    display: flex;
    flex-direction: column;
    height: 100%;
    overflow: hidden;
    flex-shrink: 0;
}
.pane-left {
    border-right: 1px solid var(--panel-border);
    background: var(--panel);
    min-width: 180px;
}
.pane-right {
    flex: 1;
    min-width: 0;
    overflow-y: auto;
    background: var(--bg);
}

.resizer {
    width: 4px;
    height: 100%;
    background: var(--panel-border);
    cursor: col-resize;
    flex-shrink: 0;
    transition: background 0.15s;
    position: relative;
    z-index: 2;
}
.resizer:hover, .layout.dragging .resizer {
    background: var(--accent);
}

.pane-header {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 0 12px;
    height: 38px;
    flex-shrink: 0;
    border-bottom: 1px solid var(--panel-border);
    background: var(--panel);
}
.pane-title {
    font-size: 12px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: var(--muted);
}
.pane-badge {
    background: var(--panel-2);
    border: 1px solid var(--panel-border);
    border-radius: 999px;
    padding: 0 6px;
    font-size: 11px;
    font-weight: 600;
    color: var(--muted);
    min-width: 18px;
    text-align: center;
    line-height: 16px;
    height: 16px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
}

.pane-body {
    flex: 1;
    overflow-y: auto;
    padding: 4px 0;
}
.block-row {
    display: flex;
    align-items: center;
    gap: 0;
    padding: 0 8px;
    height: 34px;
    cursor: pointer;
    border-radius: 0;
    transition: background 0.1s;
    user-select: none;
}
.block-row:hover { background: var(--panel-2); }
.block-row.selected {
    background: var(--accent-soft);
}
.block-row.selected .block-addr { color: var(--accent-text); }
.block-row-indicator {
    width: 16px;
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: center;
}
.entry-icon {
    width: 8px;
    height: 8px;
    color: var(--accent-text);
    flex-shrink: 0;
}
.block-row-content {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: space-between;
    min-width: 0;
}
.block-addr {
    font-family: var(--mono);
    font-size: 12px;
    color: var(--text);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}
.block-count {
    font-size: 11px;
    color: var(--muted);
    background: var(--panel-2);
    border: 1px solid var(--panel-border);
    border-radius: 999px;
    padding: 0 5px;
    line-height: 16px;
    min-width: 22px;
    text-align: center;
    flex-shrink: 0;
}
.block-row.selected .block-count {
    background: var(--accent-soft);
    border-color: var(--accent);
    color: var(--accent-text);
}

.empty-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    padding: 32px 16px;
    color: var(--muted);
    font-size: 12.5px;
    text-align: center;
}
.empty-state svg {
    width: 28px;
    height: 28px;
    opacity: 0.4;
}
.full-center {
    height: 100%;
    justify-content: center;
}

.block-info-strip {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 0;
    padding: 0 16px;
    height: 38px;
    background: var(--panel);
    border-bottom: 1px solid var(--panel-border);
    flex-shrink: 0;
    overflow: hidden;
}
.block-info-item {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-shrink: 0;
}
.block-info-sep {
    width: 1px;
    height: 16px;
    background: var(--panel-border);
    margin: 0 12px;
    flex-shrink: 0;
}
.info-label {
    font-size: 11px;
    font-weight: 500;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: var(--muted);
}
.info-code {
    font-family: var(--mono);
    font-size: 11.5px;
    color: var(--text);
}
.info-code.accent { color: var(--accent-text); }
.info-badge {
    background: var(--panel-2);
    border: 1px solid var(--panel-border);
    border-radius: 999px;
    padding: 0 6px;
    font-size: 11px;
    font-weight: 600;
    color: var(--text);
}
.info-addr-list {
    display: flex;
    gap: 4px;
    flex-wrap: wrap;
}
.info-addr {
    font-family: var(--mono);
    font-size: 11px;
    color: var(--accent-text);
    cursor: pointer;
    border: 1px solid transparent;
    border-radius: 4px;
    padding: 0 4px;
    transition: background 0.1s;
}
.info-addr:hover {
    background: var(--accent-soft);
    border-color: var(--accent);
}
.info-none { color: var(--muted); font-size: 12px; }

.section {
    border-bottom: 1px solid var(--panel-border);
}
.section-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 16px;
    height: 36px;
    background: var(--panel);
    border-bottom: 1px solid var(--panel-border);
    gap: 8px;
    flex-shrink: 0;
}
.section-title {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 11.5px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: var(--muted);
}
.section-title svg {
    width: 13px;
    height: 13px;
    flex-shrink: 0;
}
.section-instructions { border-bottom: none; }
.insn-match-count {
    font-size: 11px;
    color: var(--muted);
}
.muted-msg {
    padding: 12px 16px;
    font-size: 12.5px;
    color: var(--muted);
}

.cfg-controls {
    display: flex;
    align-items: center;
    gap: 2px;
}
.ctrl-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 26px;
    height: 26px;
    border: 1px solid var(--panel-border);
    border-radius: var(--radius);
    background: var(--panel-2);
    color: var(--muted);
    cursor: pointer;
    transition: background 0.1s, color 0.1s;
    font-family: var(--mono);
}
.ctrl-btn svg { width: 13px; height: 13px; }
.ctrl-btn:hover { background: var(--panel-3); color: var(--text); }
.ctrl-btn.active { background: var(--accent-soft); color: var(--accent-text); border-color: var(--accent); }
.ctrl-btn.ctrl-text { width: auto; padding: 0 8px; font-size: 11.5px; font-weight: 500; font-family: inherit; }
.ctrl-divider { width: 1px; height: 16px; background: var(--panel-border); margin: 0 2px; }
.zoom-label {
    min-width: 38px;
    text-align: center;
    font-family: var(--mono);
    font-size: 11.5px;
    color: var(--muted);
    pointer-events: none;
    padding: 0 2px;
}

.cfg-canvas {
    position: relative;
    height: 300px;
    overflow: hidden;
    cursor: grab;
    background: var(--panel-2);
    user-select: none;
}
.cfg-canvas.panning { cursor: grabbing; }
.cfg-grid {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
}
.cfg-viewport {
    transform-origin: 0 0;
    position: absolute;
    top: 0;
    left: 0;
    pointer-events: none;
}
.cfg-svg {
    display: block;
    overflow: visible;
}
.cfg-hint {
    position: absolute;
    bottom: 6px;
    right: 8px;
    font-size: 10.5px;
    color: var(--muted-2);
    pointer-events: none;
    user-select: none;
}

.cfg-edge {
    stroke: var(--edge);
    stroke-width: 1.5;
    fill: none;
    opacity: 0.45;
}
.cfg-edge-out { stroke: var(--edge-out); opacity: 1; stroke-width: 2; }
.cfg-edge-in { stroke: var(--edge-in); opacity: 1; stroke-width: 2; }

.cfg-node { cursor: pointer; pointer-events: all; }
.cfg-node-shadow {
    fill: rgba(0,0,0,0.25);
    transform: translate(2px, 2px);
}
.cfg-node-body {
    fill: var(--panel);
    stroke: var(--panel-border);
    stroke-width: 1;
    transition: fill 0.1s, stroke 0.1s;
}
.cfg-node-accent {
    fill: var(--panel-border);
    transition: fill 0.1s;
}
.cfg-node.entry .cfg-node-accent { fill: var(--success); }
.cfg-node.active .cfg-node-body { fill: var(--accent-soft); stroke: var(--accent); stroke-width: 1.5; }
.cfg-node.active .cfg-node-accent { fill: var(--accent); }
.cfg-node-addr {
    font-size: 12.5px;
    font-family: 'JetBrains Mono', Consolas, monospace;
    fill: var(--text);
    font-weight: 500;
}
.cfg-node-meta {
    font-size: 11px;
    font-family: 'JetBrains Mono', Consolas, monospace;
    fill: var(--muted);
}
.cfg-node.active .cfg-node-addr { fill: var(--accent-text); }

.insn-toolbar {
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 8px 12px;
    background: var(--panel);
    border-bottom: 1px solid var(--panel-border);
    flex-wrap: wrap;
}
.insn-search-wrap {
    position: relative;
    flex: 1;
    min-width: 140px;
}
.insn-search-icon {
    position: absolute;
    left: 8px;
    top: 50%;
    transform: translateY(-50%);
    width: 12px;
    height: 12px;
    color: var(--muted);
    pointer-events: none;
}
.insn-search {
    width: 100%;
    height: 28px;
    padding: 0 8px 0 28px;
    border: 1px solid var(--panel-border);
    border-radius: var(--radius);
    background: var(--panel-2);
    color: var(--text);
    font-family: inherit;
    font-size: 12.5px;
    outline: none;
    transition: border-color 0.12s;
}
.insn-search:focus { border-color: var(--accent); }
.insn-search::placeholder { color: var(--muted); }

.field-select {
    height: 28px;
    padding: 0 8px;
    border: 1px solid var(--panel-border);
    border-radius: var(--radius);
    background: var(--panel-2);
    color: var(--text);
    font-family: inherit;
    font-size: 12px;
    cursor: pointer;
    outline: none;
}

.asm-panel {
    padding: 8px 14px;
    background: var(--panel-2);
    border-bottom: 1px solid var(--panel-border);
}
.asm-panel-label {
    font-size: 10px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: var(--muted-2);
    margin-bottom: 4px;
}
.asm-code {
    font-family: var(--mono);
    font-size: 12.5px;
    color: var(--text);
    white-space: pre-wrap;
    word-break: break-all;
}

.insn-table-wrap {
    overflow: auto;
    background: var(--panel);
}
.insn-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 12px;
}
.insn-table thead th {
    position: sticky;
    top: 0;
    z-index: 1;
    background: var(--panel-2);
    border-bottom: 1px solid var(--panel-border);
    text-align: left;
    padding: 0 8px;
    height: 28px;
    font-size: 11px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: var(--muted);
    white-space: nowrap;
}
.insn-table td {
    border-bottom: 1px solid var(--panel-border);
    padding: 0 8px;
    height: 30px;
    vertical-align: middle;
    color: var(--text);
}
.insn-table tr:last-child td { border-bottom: none; }
.insn-table tbody tr:hover { background: var(--panel-2); }
.insn-active { background: var(--accent-soft) !important; }

.col-expand { width: 28px; padding: 0 4px 0 8px; }
.col-idx { width: 36px; }
.col-vip { width: 100px; }
.col-mnem { width: 100px; }
.col-ops { width: 36px; text-align: center; }
.col-insn { min-width: 0; }

.mono { font-family: var(--mono); font-size: 11.5px; }
.muted { color: var(--muted); }
.accent { color: var(--accent-text); }
.insn-text { white-space: pre; }

.expand-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 18px;
    height: 18px;
    border: 1px solid var(--panel-border);
    border-radius: 4px;
    background: var(--panel-2);
    color: var(--muted);
    cursor: pointer;
    transition: background 0.1s;
    padding: 0;
}
.expand-btn:hover { background: var(--panel-3); color: var(--text); }
.expand-btn svg { width: 9px; height: 9px; }

.insn-detail-row td {
    background: var(--panel-2);
    padding: 10px 12px;
}
.insn-detail-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 12px;
}
.insn-detail-col {}
.detail-label {
    font-size: 10.5px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: var(--muted);
    margin-bottom: 6px;
}
.chip-group {
    display: flex;
    flex-wrap: wrap;
    gap: 4px;
    margin-bottom: 4px;
}
.chip {
    border: 1px solid var(--panel-border);
    border-radius: 999px;
    padding: 1px 7px;
    font-size: 10.5px;
    color: var(--text);
    background: var(--panel);
    white-space: nowrap;
}
.chip-warn { border-color: var(--warning); color: var(--warning); background: transparent; }
.chip-accent { border-color: var(--accent); color: var(--accent-text); background: var(--accent-soft); }
.chip-muted { color: var(--muted); }

.operand-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 11px;
}
.operand-table th,
.operand-table td {
    border-bottom: 1px solid var(--panel-border);
    text-align: left;
    padding: 3px 6px;
}
.operand-table thead th {
    position: static;
    background: transparent;
    font-size: 10.5px;
}
.operand-table tr:last-child td { border-bottom: none; }

.modal-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.6);
    backdrop-filter: blur(3px);
    z-index: 300;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 16px;
}
.modal-card {
    width: min(780px, 100%);
    max-height: 90vh;
    display: flex;
    flex-direction: column;
    border: 1px solid var(--panel-border);
    border-radius: var(--radius-lg);
    background: var(--panel);
    box-shadow: var(--shadow-xl);
    overflow: hidden;
}
.modal-card-sm { width: min(540px, 100%); }

.modal-header {
    position: relative;
    padding: 16px 20px 14px;
    border-bottom: 1px solid var(--panel-border);
    flex-shrink: 0;
}
.modal-title {
    margin: 0 0 2px;
    font-size: 15px;
    font-weight: 600;
    color: var(--text);
}
.modal-subtitle {
    margin: 0;
    font-size: 12.5px;
    color: var(--muted);
}
.modal-close {
    position: absolute;
    top: 14px;
    right: 14px;
    width: 26px;
    height: 26px;
    border: 1px solid var(--panel-border);
    border-radius: var(--radius);
    background: transparent;
    color: var(--muted);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background 0.1s, color 0.1s;
    padding: 0;
}
.modal-close:hover { background: var(--danger-bg); color: var(--danger); border-color: var(--danger); }
.modal-close svg { width: 12px; height: 12px; }

.modal-body {
    flex: 1;
    overflow-y: auto;
    padding: 16px 20px;
    display: flex;
    flex-direction: column;
    gap: 12px;
}
.modal-footer {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    gap: 8px;
    padding: 12px 20px;
    border-top: 1px solid var(--panel-border);
    background: var(--panel-2);
    flex-shrink: 0;
}

.field-group {
    display: flex;
    flex-direction: column;
    gap: 4px;
}
.field-label {
    font-size: 11.5px;
    font-weight: 600;
    color: var(--muted);
    text-transform: uppercase;
    letter-spacing: 0.05em;
}
.field-wrap {
    position: relative;
}
.field-input {
    width: 100%;
    height: 32px;
    padding: 0 10px;
    border: 1px solid var(--panel-border);
    border-radius: var(--radius);
    background: var(--panel-2);
    color: var(--text);
    font-family: var(--mono);
    font-size: 12.5px;
    outline: none;
    transition: border-color 0.12s;
}
.field-input:focus { border-color: var(--accent); }
.field-input::placeholder { color: var(--muted); font-family: inherit; }
.field-select.full-width { width: 100%; }
.field-static { font-size: 12.5px; color: var(--text); padding: 4px 0; }
.field-error {
    font-size: 11.5px;
    color: var(--danger);
    display: flex;
    align-items: center;
    gap: 4px;
}
.preview-code {
    display: block;
    font-family: var(--mono);
    font-size: 12.5px;
    color: var(--text);
    background: var(--panel-2);
    border: 1px solid var(--panel-border);
    border-radius: var(--radius);
    padding: 8px 12px;
    white-space: pre-wrap;
    word-break: break-all;
    min-height: 36px;
}

.operand-row {
    display: grid;
    grid-template-columns: 130px 1fr;
    gap: 8px;
    align-items: start;
    margin-bottom: 4px;
}
.op-type-badge {
    font-family: var(--mono);
    font-size: 11px;
    color: var(--accent-text);
    background: var(--accent-soft);
    border: 1px solid var(--accent);
    border-radius: 999px;
    padding: 3px 8px;
    white-space: nowrap;
    align-self: center;
    margin-top: 4px;
}
.operand-hint {
    grid-column: 2;
    font-size: 11px;
    color: var(--muted);
    margin-top: 2px;
}

.suggestion-box {
    position: absolute;
    top: 100%;
    left: 0;
    right: 0;
    z-index: 50;
    border: 1px solid var(--panel-border);
    border-radius: var(--radius);
    background: var(--panel);
    box-shadow: var(--shadow-md);
    overflow: hidden;
    margin-top: 2px;
    max-height: 200px;
    overflow-y: auto;
}
.suggestion-item {
    display: block;
    width: 100%;
    border: 0;
    text-align: left;
    background: transparent;
    color: var(--text);
    cursor: pointer;
    padding: 5px 10px;
    font-family: var(--mono);
    font-size: 12px;
    transition: background 0.08s;
}
.suggestion-item:hover,
.suggestion-item.active {
    background: var(--accent-soft);
    color: var(--accent-text);
}

.btn {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 0 12px;
    height: 30px;
    border-radius: var(--radius);
    border: 1px solid var(--panel-border);
    font-family: inherit;
    font-size: 12.5px;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.12s, border-color 0.12s;
    white-space: nowrap;
}
.btn-ghost { background: transparent; color: var(--text); }
.btn-ghost:hover { background: var(--panel-2); }
.btn-primary {
    background: var(--accent-soft);
    border-color: var(--accent);
    color: var(--accent-text);
}
.btn-primary:hover { background: var(--accent-hover); }
.btn-primary:disabled {
    opacity: 0.4;
    cursor: not-allowed;
}

@media (max-width: 700px) {
    .insn-detail-grid { grid-template-columns: 1fr; }
    .operand-row { grid-template-columns: 1fr; }
    .block-info-strip { height: auto; padding: 6px 12px; gap: 6px; }
    .block-info-sep { display: none; }
}
</style>
