<template>
  <div class="app">
    <h1>⚖️ Weighted <span>Overtime</span> Calculator</h1>
    <p class="subtitle">FLSA regular-rate overtime: all earnings ÷ all hours = regular rate. Half of regular rate × OT hours = OT premium.</p>

    <!-- Payroll Period Toggle -->
    <div class="card">
      <div class="period-toggle">
        <button
          class="period-btn"
          :class="{ active: payrollPeriod === 'weekly' }"
          @click="setPayrollPeriod('weekly')"
        >Weekly</button>
        <button
          class="period-btn"
          :class="{ active: payrollPeriod === 'biweekly' }"
          @click="setPayrollPeriod('biweekly')"
        >Bi-Weekly</button>
      </div>
    </div>

    <!-- Minimum Wage / OT Floor -->
    <div class="card">
      <div class="field-row">
        <div class="field">
          <label>Minimum Wage Rate ($/hr)</label>
          <input type="number" min="0" step="0.01" v-model.number="minimumWage" placeholder="7.25">
        </div>
      </div>
      <p class="card-hint" style="margin-top:4px;">If set, OT premium uses the higher of the calculated regular rate or this minimum wage. Leave at 0 to use the regular rate only.</p>
    </div>

    <!-- Bi-Weekly Mode -->
    <template v-if="payrollPeriod === 'biweekly'">
      <div class="week-tabs">
        <button class="week-tab" :class="{ active: activeWeek === 0 }" @click="activeWeek = 0">Week 1</button>
        <button class="week-tab" :class="{ active: activeWeek === 1 }" @click="activeWeek = 1">Week 2</button>
      </div>

      <div class="week-section" v-for="(week, wi) in weeks" :key="wi" v-show="activeWeek === wi">
        <!-- Pay Rates -->
        <div class="card">
          <h2><span class="icon">💵</span> {{ wi === 0 ? 'Week 1' : 'Week 2' }} — Pay Rates</h2>
          <div class="entry-list">
            <div class="rate-group" v-for="(rate, i) in week.payRates" :key="i">
              <div class="rate-header">
                <div class="field" style="flex:1;">
                  <label>Role / Label</label>
                  <input type="text" v-model="rate.label" placeholder="e.g. Warehouse">
                </div>
                <button class="btn-danger-sm" @click="week.payRates.splice(i, 1)" title="Remove" v-if="week.payRates.length > 1">✕</button>
              </div>
              <div class="rate-fields">
                <div class="field">
                  <label>Base Rate ($/hr)</label>
                  <input type="number" min="0" step="0.01" v-model.number="rate.hourly" placeholder="22.00">
                </div>
                <div class="field">
                  <label>Straight Hours</label>
                  <input type="number" min="0" step="0.5" v-model.number="rate.hours" placeholder="30">
                </div>
                <div class="field">
                  <label>OT Hours</label>
                  <input type="number" min="0" step="0.5" v-model.number="rate.otHours" placeholder="0">
                </div>
                <div class="field" style="justify-content:flex-end;">
                  <label>Earnings</label>
                  <span class="pay-rate-subtotal">{{ formatCurrency(rate.hourly * (rate.hours + rate.otHours)) }}</span>
                </div>
              </div>
            </div>
          </div>
          <div class="btn-row">
            <button class="btn btn-secondary" @click="addPayRate(wi)">+ Add Pay Rate</button>
          </div>
        </div>

        <!-- Bonuses (Non-Discretionary) -->
        <div class="card">
          <h2><span class="icon">🎁</span> {{ wi === 0 ? 'Week 1' : 'Week 2' }} — Non-Discretionary Bonuses &amp; Differentials</h2>
          <p class="card-hint">These are included in the regular rate of pay for OT calculation.</p>
          <div class="entry-list">
            <div class="entry-row" v-for="(bonus, i) in week.bonuses" :key="i">
              <div class="field">
                <label>Label</label>
                <input type="text" v-model="bonus.label" placeholder="e.g. Shift Diff">
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
              <button class="btn-danger-sm" @click="week.bonuses.splice(i, 1)" title="Remove">✕</button>
            </div>
          </div>
          <div class="btn-row">
            <button class="btn btn-secondary" @click="addBonus(wi)">+ Add Bonus</button>
          </div>
        </div>

        <!-- Discretionary Bonuses -->
        <div class="card">
          <h2><span class="icon">🏆</span> {{ wi === 0 ? 'Week 1' : 'Week 2' }} — Discretionary Bonuses</h2>
          <p class="card-hint">Not included in the regular rate. Added to total pay after OT premium.</p>
          <div class="entry-list">
            <div class="entry-row" v-for="(b, i) in week.discretionaryBonuses" :key="i">
              <div class="field">
                <label>Label</label>
                <input type="text" v-model="b.label" placeholder="e.g. Holiday Bonus">
              </div>
              <div class="field">
                <label>Amount ($)</label>
                <input type="number" min="0" step="0.01" v-model.number="b.value" placeholder="0">
              </div>
              <button class="btn-danger-sm" @click="week.discretionaryBonuses.splice(i, 1)" title="Remove">✕</button>
            </div>
          </div>
          <div class="btn-row">
            <button class="btn btn-secondary" @click="addDiscBonus(wi)">+ Add Discretionary Bonus</button>
          </div>
        </div>

        <!-- Tips -->
        <div class="card">
          <h2><span class="icon">🪙</span> {{ wi === 0 ? 'Week 1' : 'Week 2' }} — Tips</h2>
          <p class="card-hint">Excluded from the regular rate. Added to total pay after OT premium.</p>
          <div class="field" style="margin-bottom:0;">
            <label>Weekly Tips ($)</label>
            <input type="number" min="0" step="0.01" v-model.number="week.tips" placeholder="0.00" style="max-width:200px;">
          </div>
        </div>
      </div>

      <!-- Bi-Weekly Combined Results -->
      <div class="card results" v-if="biweeklyTotal > 0">
        <h2><span class="icon">💰</span> Bi-Weekly Earnings Breakdown</h2>
        <div class="breakdown">
          <template v-for="(wk, wi) in weeks" :key="wi">
            <div class="breakdown-group">
              <div class="breakdown-group-label">{{ wi === 0 ? 'Week 1' : 'Week 2' }}</div>
              <template v-for="line in weekBreakdown(wk)" :key="line.label">
                <div class="breakdown-row">
                  <span class="label">{{ line.label }}</span>
                  <span class="value">{{ formatCurrency(line.value) }}</span>
                </div>
              </template>
              <div class="breakdown-row group-sub">
                <span class="label">{{ wi === 0 ? 'Week 1' : 'Week 2' }} Subtotal</span>
                <span class="value">{{ formatCurrency(weekTotal(wk)) }}</span>
              </div>
            </div>
          </template>
          <div class="breakdown-row total">
            <span class="label">Bi-Weekly Total Pay</span>
            <span class="value">{{ formatCurrency(biweeklyTotal) }}</span>
          </div>
        </div>
      </div>
    </template>

    <!-- Weekly Mode -->
    <template v-else>
      <!-- Pay Rates -->
      <div class="card">
        <h2><span class="icon">💵</span> Pay Rates</h2>
        <div class="entry-list">
          <div class="rate-group" v-for="(rate, i) in weeks[0].payRates" :key="i">
            <div class="rate-header">
              <div class="field" style="flex:1;">
                <label>Role / Label</label>
                <input type="text" v-model="rate.label" placeholder="e.g. Warehouse">
              </div>
              <button class="btn-danger-sm" @click="weeks[0].payRates.splice(i, 1)" title="Remove" v-if="weeks[0].payRates.length > 1">✕</button>
            </div>
            <div class="rate-fields">
              <div class="field">
                <label>Base Rate ($/hr)</label>
                <input type="number" min="0" step="0.01" v-model.number="rate.hourly" placeholder="22.00">
              </div>
              <div class="field">
                <label>Straight Hours</label>
                <input type="number" min="0" step="0.5" v-model.number="rate.hours" placeholder="30">
              </div>
              <div class="field">
                <label>OT Hours</label>
                <input type="number" min="0" step="0.5" v-model.number="rate.otHours" placeholder="0">
              </div>
              <div class="field" style="justify-content:flex-end;">
                <label>Earnings</label>
                <span class="pay-rate-subtotal">{{ formatCurrency(rate.hourly * (rate.hours + rate.otHours)) }}</span>
              </div>
            </div>
          </div>
        </div>
        <div class="btn-row">
          <button class="btn btn-secondary" @click="addPayRate(0)">+ Add Pay Rate</button>
        </div>
      </div>

      <!-- Bonuses (Non-Discretionary) -->
      <div class="card">
        <h2><span class="icon">🎁</span> Non-Discretionary Bonuses &amp; Differentials</h2>
        <p class="card-hint">These are included in the regular rate of pay for OT calculation.</p>
        <div class="entry-list">
          <div class="entry-row" v-for="(bonus, i) in weeks[0].bonuses" :key="i">
            <div class="field">
              <label>Label</label>
              <input type="text" v-model="bonus.label" placeholder="e.g. Shift Diff">
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
            <button class="btn-danger-sm" @click="weeks[0].bonuses.splice(i, 1)" title="Remove">✕</button>
          </div>
        </div>
        <div class="btn-row">
          <button class="btn btn-secondary" @click="addBonus(0)">+ Add Bonus</button>
        </div>
      </div>

      <!-- Discretionary Bonuses -->
      <div class="card">
        <h2><span class="icon">🏆</span> Discretionary Bonuses</h2>
        <p class="card-hint">Not included in the regular rate. Added to total pay after OT premium.</p>
        <div class="entry-list">
          <div class="entry-row" v-for="(b, i) in weeks[0].discretionaryBonuses" :key="i">
            <div class="field">
              <label>Label</label>
              <input type="text" v-model="b.label" placeholder="e.g. Holiday Bonus">
            </div>
            <div class="field">
              <label>Amount ($)</label>
              <input type="number" min="0" step="0.01" v-model.number="b.value" placeholder="0">
            </div>
            <button class="btn-danger-sm" @click="weeks[0].discretionaryBonuses.splice(i, 1)" title="Remove">✕</button>
          </div>
        </div>
        <div class="btn-row">
          <button class="btn btn-secondary" @click="addDiscBonus(0)">+ Add Discretionary Bonus</button>
        </div>
      </div>

      <!-- Tips -->
      <div class="card">
        <h2><span class="icon">🪙</span> Tips</h2>
        <p class="card-hint">Excluded from the regular rate. Added to total pay after OT premium.</p>
        <div class="field" style="margin-bottom:0;">
          <label>Weekly Tips ($)</label>
          <input type="number" min="0" step="0.01" v-model.number="weeks[0].tips" placeholder="0.00" style="max-width:200px;">
        </div>
      </div>

      <!-- Results -->
      <div class="card results" v-if="weeklyTotal > 0">
        <h2><span class="icon">💰</span> Weekly Earnings Breakdown</h2>
        <div class="breakdown">
          <template v-for="group in weeklyBreakdown" :key="group.label">
            <div class="breakdown-group">
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
          </template>
          <div class="breakdown-row total">
            <span class="label">Total Weekly Pay</span>
            <span class="value">{{ formatCurrency(weeklyTotal) }}</span>
          </div>
        </div>
      </div>
    </template>

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
          <span class="saved-period-badge">{{ item.period_type || 'weekly' }}</span>
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

function makeDefaultRate(label = 'Warehouse', hourly = 22, hours = 30, otHours = 4) {
  return { label, hourly, hours, otHours }
}

function makeWeek(overrides = {}) {
  return {
    payRates: [makeDefaultRate(), makeDefaultRate('Forklift Cert', 28, 10, 0)],
    bonuses: [],
    discretionaryBonuses: [],
    tips: 0,
    ...overrides,
  }
}

export default {
  data() {
    return {
      payrollPeriod: 'weekly',
      activeWeek: 0,
      minimumWage: 7.25,
      weeks: [makeWeek(), makeWeek()],
      saved: [],
      loading: false,
    }
  },

  computed: {
    weeklyBreakdown() {
      return this.buildBreakdown(this.weeks[0])
    },

    weeklyTotal() {
      return this.weeklyBreakdown.reduce((s, g) => s + (g.subtotal || 0), 0)
    },

    biweeklyTotal() {
      return this.weeks.reduce((s, w) => s + this.weekTotal(w), 0)
    },
  },

  methods: {
    setPayrollPeriod(type) {
      this.payrollPeriod = type
    },

    addPayRate(weekIndex) {
      this.weeks[weekIndex].payRates.push({ label: '', hourly: 0, hours: 0, otHours: 0 })
    },

    addBonus(weekIndex) {
      this.weeks[weekIndex].bonuses.push({ label: '', type: 'flat', value: 0 })
    },

    addDiscBonus(weekIndex) {
      this.weeks[weekIndex].discretionaryBonuses.push({ label: '', value: 0 })
    },

    /**
     * FLSA Regular Rate calculation for one week:
     * 1. Straight-time earnings: (straight hours + OT hours) × base rate for each role
     * 2. Add bonuses/differentials
     * 3. Regular rate = total earnings ÷ total hours
     *    - When minimum wage is set, use max(rate, min wage) for the regular rate denominator
     *    - Actual straight-time pay stays at the real rate
     * 4. OT premium = (regular rate ÷ 2) × OT hours
     */
    weekBreakdown(week) {
      const lines = []
      const hasMinWage = this.minimumWage > 0

      // Step 1: Straight-time earnings per rate (actual pay)
      for (const r of week.payRates) {
        const totalHoursForRate = r.hours + r.otHours
        const earnings = r.hourly * totalHoursForRate
        if (totalHoursForRate > 0) {
          lines.push({
            label: `${r.label || 'Rate'} (${totalHoursForRate}h × $${r.hourly.toFixed(2)})`,
            value: earnings,
          })
        }
      }

      // Step 2: Bonuses
      const bonusTotal = this.calcBonusTotal(week)
      if (bonusTotal > 0) {
        lines.push({ label: 'Bonuses & Differentials', value: bonusTotal })
      }

      // Step 3: Compute totals
      const totalEarnings = lines.reduce((s, l) => s + l.value, 0)
      const totalHours = week.payRates.reduce((s, r) => s + r.hours + r.otHours, 0)

      // Step 4: Regular rate & OT premium
      const otHours = week.payRates.reduce((s, r) => s + r.otHours, 0)
      if (totalHours > 0 && otHours > 0) {
        // For regular rate, use min wage where base rate is below it
        const effectiveEarnings = hasMinWage
          ? week.payRates.reduce((s, r) => s + Math.max(r.hourly, this.minimumWage) * (r.hours + r.otHours), 0) + bonusTotal
          : totalEarnings
        const regularRate = effectiveEarnings / totalHours
        const otPremium = (regularRate / 2) * otHours
        if (hasMinWage && effectiveEarnings > totalEarnings) {
          lines.push({
            label: `Regular Rate: $${regularRate.toFixed(2)}/hr (incl. min wage adjustment)`,
            value: 0,
            isInfo: true,
          })
        } else {
          lines.push({
            label: `Regular Rate: $${regularRate.toFixed(2)}/hr`,
            value: 0,
            isInfo: true,
          })
        }
        lines.push({
          label: `OT Premium (${otHours}h × $${(regularRate / 2).toFixed(2)}/hr half-time)`,
          value: otPremium,
        })
      }

      // Step 5: Discretionary bonuses (excluded from regular rate)
      const discTotal = (week.discretionaryBonuses || []).reduce((s, b) => s + b.value, 0)
      if (discTotal > 0) {
        lines.push({ label: 'Discretionary Bonuses (excl. from reg. rate)', value: discTotal })
      }

      // Step 6: Tips (excluded from regular rate)
      if (week.tips > 0) {
        lines.push({ label: 'Tips (excl. from reg. rate)', value: week.tips })
      }

      return lines
    },

    weekTotal(week) {
      return this.weekBreakdown(week).reduce((s, l) => s + l.value, 0)
    },

    buildBreakdown(week) {
      const groups = []
      const lines = this.weekBreakdown(week)

      // Group straight-time earnings by rate
      const rateLines = lines.filter(l => !l.isInfo && l.label !== 'Bonuses & Differentials' && !l.label.includes('OT Premium') && !l.label.includes('Regular Rate'))
      if (rateLines.length > 0) {
        groups.push({
          label: 'Straight-Time Earnings',
          lines: rateLines,
          subtotal: rateLines.reduce((s, l) => s + l.value, 0),
        })
      }

      // Bonuses group
      const bonusLine = lines.find(l => l.label === 'Bonuses & Differentials')
      if (bonusLine) {
        groups.push({ label: 'Bonuses & Differentials', lines: [bonusLine], subtotal: bonusLine.value })
      }

      // OT premium group
      const otLines = lines.filter(l => l.label.includes('OT Premium') || l.label.includes('Regular Rate'))
      if (otLines.length > 0) {
        const otPay = lines.find(l => l.label.includes('OT Premium'))
        groups.push({
          label: 'Overtime Premium',
          lines: otLines,
          subtotal: otPay ? otPay.value : 0,
        })
      }

      // Discretionary bonuses group
      const discLine = lines.find(l => l.label.includes('Discretionary Bonuses'))
      if (discLine) {
        groups.push({ label: 'Discretionary Bonuses', lines: [discLine], subtotal: discLine.value })
      }

      // Tips group
      const tipsLine = lines.find(l => l.label.includes('Tips'))
      if (tipsLine) {
        groups.push({ label: 'Tips', lines: [tipsLine], subtotal: tipsLine.value })
      }

      return groups
    },

    calcBonusTotal(week) {
      const totalHours = week.payRates.reduce((s, r) => s + r.hours + r.otHours, 0)
      return week.bonuses.reduce((total, b) => {
        if (b.type === 'flat') return total + b.value
        // Multiplier: applies to straight-time blended rate × total hours
        const regularHoursTotal = week.payRates.reduce((s, r) => s + r.hours, 0)
        const blended = regularHoursTotal > 0 ? week.payRates.reduce((s, r) => s + r.hourly * r.hours, 0) / regularHoursTotal : 0
        return total + (blended * totalHours * (b.value - 1))
      }, 0)
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
      const periodLabel = this.payrollPeriod === 'biweekly' ? 'Bi-Weekly' : 'Weekly'
      const label = `${periodLabel} — ${new Date().toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })}`
      const total = this.payrollPeriod === 'biweekly' ? this.biweeklyTotal : this.weeklyTotal
      const payload = {
        label,
        total,
        period_type: this.payrollPeriod,
        minimumWage: this.minimumWage,
        weeks: JSON.parse(JSON.stringify(this.weeks)),
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
      if (item.weeks) {
        this.weeks = JSON.parse(item.weeks)
        this.payrollPeriod = item.period_type || 'weekly'
        this.minimumWage = item.minimumWage ?? 7.25
      } else if (item.payRates) {
        this.weeks = [makeWeek({ payRates: JSON.parse(item.payRates), bonuses: item.bonuses ? JSON.parse(item.bonuses) : [] })]
        this.payrollPeriod = 'weekly'
      }
    },
  },

  mounted() {
    this.loadSaved()
  },
}
</script>
