<template>
  <div class="app">
    <h1>⚖️ Weighted <span>Overtime</span> Calculator</h1>
    <p class="subtitle">Calculate earnings with tiered overtime rates, bonuses &amp; shift differentials.</p>

    <!-- Base Settings -->
    <div class="card">
      <h2><span class="icon">🔧</span> Base Settings</h2>
      <div class="field-row">
        <div class="field">
          <label for="rate">Hourly Rate ($)</label>
          <input id="rate" type="number" min="0" step="0.01" v-model.number="baseRate" placeholder="25.00">
        </div>
        <div class="field">
          <label for="regularHours">Regular Hours / Week</label>
          <input id="regularHours" type="number" min="0" step="0.5" v-model.number="regularHours" placeholder="40">
        </div>
      </div>
    </div>

    <!-- Overtime Tiers -->
    <div class="card">
      <h2><span class="icon">⏱️</span> Overtime Tiers</h2>
      <div class="entry-list">
        <div class="entry-row" v-for="(tier, i) in overtimeTiers" :key="i">
          <div class="field">
            <label>Hours</label>
            <input type="number" min="0" step="0.5" v-model.number="tier.hours" placeholder="0">
          </div>
          <div class="field">
            <label>Multiplier</label>
            <input type="number" min="1" step="0.1" v-model.number="tier.multiplier" placeholder="1.5">
          </div>
          <div class="field">
            <label>Label</label>
            <input type="text" v-model="tier.label" placeholder="e.g. Time & Half">
          </div>
          <button class="btn-danger-sm" @click="overtimeTiers.splice(i, 1)" title="Remove">✕</button>
        </div>
      </div>
      <div class="btn-row">
        <button class="btn btn-secondary" @click="addOvertimeTier">+ Add Overtime Tier</button>
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
    <div class="card results" v-if="baseRate > 0 && regularHours > 0">
      <h2><span class="icon">💰</span> Weekly Earnings Breakdown</h2>
      <div class="breakdown">
        <div class="breakdown-row" v-for="line in breakdown" :key="line.label">
          <span class="label">{{ line.label }}</span>
          <span class="value">{{ formatCurrency(line.value) }}</span>
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
      baseRate: 25,
      regularHours: 40,
      overtimeTiers: [
        { hours: 4, multiplier: 1.5, label: 'Time & Half' },
        { hours: 4, multiplier: 2.0, label: 'Double Time' },
      ],
      bonuses: [],
      saved: [],
      loading: false,
    }
  },

  computed: {
    regularPay() {
      return this.baseRate * this.regularHours
    },

    overtimeBreakdown() {
      return this.overtimeTiers.map(tier => ({
        label: tier.label || `${tier.multiplier}× OT`,
        hours: tier.hours,
        pay: this.baseRate * tier.hours * tier.multiplier,
      }))
    },

    bonusBreakdown() {
      return this.bonuses.map(b => {
        if (b.type === 'flat') {
          return { label: b.label || 'Bonus', pay: b.value }
        }
        const otHours = this.overtimeTiers.reduce((s, t) => s + t.hours, 0)
        const totalHours = this.regularHours + otHours
        return { label: b.label || 'Differential', pay: this.baseRate * totalHours * (b.value - 1) }
      }).filter(b => b.pay > 0)
    },

    breakdown() {
      const lines = [{ label: `Regular (${this.regularHours}h × $${this.baseRate.toFixed(2)})`, value: this.regularPay }]
      for (const ot of this.overtimeBreakdown) {
        lines.push({ label: `${ot.label} (${ot.hours}h × ${this.overtimeTiers[this.overtimeBreakdown.indexOf(ot)].multiplier}×)`, value: ot.pay })
      }
      for (const b of this.bonusBreakdown) {
        lines.push({ label: b.label, value: b.pay })
      }
      return lines
    },

    totalPay() {
      return this.breakdown.reduce((s, l) => s + l.value, 0)
    },
  },

  methods: {
    addOvertimeTier() {
      this.overtimeTiers.push({ hours: 0, multiplier: 1.5, label: '' })
    },

    addBonus() {
      this.bonuses.push({ label: '', type: 'flat', value: 0 })
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
        baseRate: this.baseRate,
        regularHours: this.regularHours,
        overtimeTiers: JSON.parse(JSON.stringify(this.overtimeTiers)),
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
      this.baseRate = item.baseRate || 25
      this.regularHours = item.regularHours || 40
      this.overtimeTiers = item.overtimeTiers ? JSON.parse(item.overtimeTiers) : []
      this.bonuses = item.bonuses ? JSON.parse(item.bonuses) : []
    },
  },

  mounted() {
    this.loadSaved()
  },
}
</script>
