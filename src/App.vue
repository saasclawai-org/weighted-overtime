<template>
  <div class="app">
    <h1>⚖️ Weighted <span>Overtime</span> Calculator</h1>
    <p class="subtitle">Multiple pay rates with per-rate overtime, bonuses &amp; shift differentials.</p>

    <!-- Pay Rates -->
    <div class="card">
      <h2><span class="icon">💵</span> Pay Rates &amp; Overtime</h2>
      <div class="entry-list">
        <div class="rate-group" v-for="(rate, i) in payRates" :key="i">
          <div class="rate-header">
            <div class="field" style="flex:1;">
              <label>Role / Label</label>
              <input type="text" v-model="rate.label" placeholder="e.g. Warehouse">
            </div>
            <button class="btn-danger-sm" @click="payRates.splice(i, 1)" title="Remove" v-if="payRates.length > 1">✕</button>
          </div>
          <div class="rate-fields">
            <div class="field">
              <label>Base Rate ($/hr)</label>
              <input type="number" min="0" step="0.01" v-model.number="rate.hourly" placeholder="22.00">
            </div>
            <div class="field">
              <label>Regular Hours</label>
              <input type="number" min="0" step="0.5" v-model.number="rate.hours" placeholder="30">
            </div>
            <div class="field">
              <label>OT Hours</label>
              <input type="number" min="0" step="0.5" v-model.number="rate.otHours" placeholder="0">
            </div>
            <div class="field">
              <label>OT Multiplier</label>
              <input type="number" min="1" step="0.1" v-model.number="rate.otMultiplier" placeholder="1.5">
            </div>
            <div class="field">
              <label>OT Rate ($/hr)</label>
              <input type="number" min="0" step="0.01" v-model.number="rate.otRate" placeholder="0 (auto)">
            </div>
            <div class="field" style="justify-content:flex-end;">
              <label>Subtotal</label>
              <span class="pay-rate-subtotal">{{ formatCurrency(rateTotal(rate)) }}</span>
            </div>
          </div>
        </div>
      </div>
      <div class="btn-row">
        <button class="btn btn-secondary" @click="addPayRate">+ Add Pay Rate</button>
      </div>
    </div>

    <!-- Bonuses -->
    <div class="card">
      <h2><span class="icon">🎁</span> Bonuses &amp; Differentials</h2>
      <div class="entry-list">
        <div class="entry-row" v-for="(bonus, i) in bonuses" :key="i">
          <div class="field">
            <label>Label</label>
            <input type="text" v-model="bonus.label" placeholder="e.g. Night Shift">
          </div>
          <div class="field">
            <label>Type</label>
            <select v-model="bonus.type">
              <option value="flat">Flat ($)</option>
              <option value="multiplier">Rate ×</option>
            </select>
          </div>
          <div class="field">
            <label>Amount / Multiplier</label>
            <input type="number" min="0" step="0.01" v-model.number="bonus.value" placeholder="0">
          </div>
          <button class="btn-danger-sm" @click="bonuses.splice(i, 1)" title="Remove">✕</button>
        </div>
      </div>
      <div class="btn-row">
        <button class="btn btn-secondary" @click="addBonus">+ Add Bonus</button>
      </div>
    </div>

    <!-- Results -->
    <div class="card results" v-if="totalPay > 0">
      <h2><span class="icon">💰</span> Weekly Earnings Breakdown</h2>
      <div class="breakdown">
        <div class="breakdown-group" v-for="(group, gi) in breakdown" :key="gi">
          <div class="breakdown-group-label">{{ group.label }}</div>
          <div class="breakdown-row" v-for="line in group.lines" :key="line.label">
            <span class="label">{{ line.label }}</span>
            <span class="value">{{ formatCurrency(line.value) }}</span>
          </div>
          <div class="breakdown-row group-sub" v-if="group.subtotal != null">
            <span class="label">{{ group.label }} Subtotal</span>
            <span class="value">{{ formatCurrency(group.subtotal) }}</span>
          </div>
        </div>
        <div class="breakdown-row total">
          <span class="label">Total Weekly Pay</span>
          <span class="value">{{ formatCurrency(totalPay) }}</span>
        </div>
      </div>
    </div>

    <!-- Save / Load -->
    <div class="card">
      <h2><span class="icon">💾</span> Saved Calculations</h2>
      <div class="btn-row" style="margin-top:0; margin-bottom:12px;">
        <button class="btn btn-primary" @click="saveCalculation">Save Current</button>
        <button class="btn btn-secondary" @click="loadSaved" :disabled="loading">↻ Refresh</button>
      </div>
      <div v-if="saved.length === 0 && !loading" class="empty-state">No saved calculations yet. Click "Save Current" to store one.</div>
      <div v-for="item in saved" :key="item.id" class="saved-item">
        <div class="saved-meta">
          <span class="saved-name">{{ item.label || 'Untitled' }}</span>
          <span class="saved-date">{{ formatDate(item.created_at) }}</span>
        </div>
        <div style="display:flex;align-items:center;gap:12px;">
          <span class="saved-total">{{ formatCurrency(item.total) }}</span>
          <button class="btn btn-secondary" style="padding:6px 12px;font-size:.8rem;" @click="loadItem(item)">Load</button>
          <button class="btn-danger-sm" @click="deleteItem(item.id)">Delete</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
const API_KEY = import.meta.env.VITE_FORM_API_KEY || 'VHoqh9fp8PPLeBP78_v49Vhauq0_c3XaenqydCA-PzXGv72As2NSPg'
const API_BASE = '/api/forms/weighted-overtime-calculator'

export default {
  data() {
    return {
      payRates: [
        { label: 'Warehouse', hourly: 22, hours: 30, otHours: 4, otMultiplier: 1.5, otRate: 0 },
        { label: 'Forklift Cert', hourly: 28, hours: 10, otHours: 0, otMultiplier: 1.5, otRate: 0 },
      ],
      bonuses: [],
      saved: [],
      loading: false,
    }
  },

  computed: {
    breakdown() {
      const groups = []
      for (const r of this.payRates) {
        const lines = []
        const regularPay = r.hourly * r.hours
        if (r.hours > 0) {
          lines.push({ label: `Regular (${r.hours}h × $${r.hourly.toFixed(2)})`, value: regularPay })
        }
        const effectiveOtRate = r.otRate > 0 ? r.otRate : r.hourly
        const otPay = effectiveOtRate * r.otHours * r.otMultiplier
        if (r.otHours > 0) {
          const rateLabel = r.otRate > 0 ? `$${effectiveOtRate.toFixed(2)}` : `$${r.hourly.toFixed(2)}`
          lines.push({
            label: `OT (${r.otHours}h × ${rateLabel} × ${r.otMultiplier}×)`,
            value: otPay,
          })
        }
        if (lines.length > 0) {
          groups.push({
            label: r.label || 'Rate',
            lines,
            subtotal: regularPay + otPay,
          })
        }
      }
      const bonusLines = this.bonusBreakdown
      if (bonusLines.length > 0) {
        groups.push({
          label: 'Bonuses & Differentials',
          lines: bonusLines.map(b => ({ label: b.label, value: b.pay })),
          subtotal: bonusLines.reduce((s, b) => s + b.pay, 0),
        })
      }
      return groups
    },

    bonusBreakdown() {
      const totalHours = this.payRates.reduce((s, r) => s + r.hours + r.otHours, 0)
      return this.bonuses.map(b => {
        if (b.type === 'flat') {
          return { label: b.label || 'Bonus', pay: b.value }
        }
        const blended = totalHours > 0 ? this.payRates.reduce((s, r) => s + r.hourly * r.hours, 0) / this.payRates.reduce((s, r) => s + r.hours, 0) : 0
        return { label: b.label || 'Differential', pay: blended * totalHours * (b.value - 1) }
      }).filter(b => b.pay > 0)
    },

    totalPay() {
      return this.breakdown.reduce((s, g) => s + (g.subtotal || 0), 0)
    },
  },

  methods: {
    addPayRate() {
      this.payRates.push({ label: '', hourly: 0, hours: 0, otHours: 0, otMultiplier: 1.5, otRate: 0 })
    },

    addBonus() {
      this.bonuses.push({ label: '', type: 'flat', value: 0 })
    },

    rateTotal(rate) {
      const regularPay = rate.hourly * rate.hours
      const effectiveOtRate = rate.otRate > 0 ? rate.otRate : rate.hourly
      const otPay = effectiveOtRate * rate.otHours * rate.otMultiplier
      return regularPay + otPay
    },

    formatCurrency(val) {
      return '$' + val.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
    },

    formatDate(iso) {
      if (!iso) return ''
      const d = new Date(iso)
      return d.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric', hour: 'numeric', minute: '2-digit' })
    },

    async saveCalculation() {
      const label = `Week of ${new Date().toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })}`
      const payload = {
        label,
        total: this.totalPay,
        payRates: JSON.parse(JSON.stringify(this.payRates)),
        bonuses: JSON.parse(JSON.stringify(this.bonuses)),
      }
      try {
        await fetch(API_BASE + '/', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json', 'X-Form-Key': API_KEY },
          body: JSON.stringify(payload),
        })
        await this.loadSaved()
      } catch (e) {
        console.error('Save failed', e)
      }
    },

    async loadSaved() {
      this.loading = true
      try {
        const res = await fetch(API_BASE + '/data/', { headers: { 'X-Form-Key': API_KEY } })
        const json = await res.json()
        this.saved = (json.ok ? json.data : []).sort((a, b) => new Date(b.created_at) - new Date(a.created_at))
      } catch (e) {
        console.error('Load failed', e)
      } finally {
        this.loading = false
      }
    },

    async deleteItem(id) {
      try {
        await fetch(`${API_BASE}/${id}/`, { method: 'DELETE', headers: { 'X-Form-Key': API_KEY } })
        this.saved = this.saved.filter(s => s.id !== id)
      } catch (e) {
        console.error('Delete failed', e)
      }
    },

    loadItem(item) {
      this.payRates = item.payRates ? JSON.parse(item.payRates) : [{ label: 'Regular', hourly: 25, hours: 40, otHours: 0, otMultiplier: 1.5, otRate: 0 }]
      this.bonuses = item.bonuses ? JSON.parse(item.bonuses) : []
    },
  },

  mounted() {
    this.loadSaved()
  },
}
</script>
