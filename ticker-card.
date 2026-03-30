/**
 * sensor-ticker-card
 * A scrolling sensor ticker for Home Assistant Lovelace.
 *
 * Drop into /config/www/sensor-ticker-card.js
 * Register in resources:
 *   url: /local/sensor-ticker-card.js
 *   type: module
 */

const STYLES = `
  :host {
    --stc-bg:      var(--card-background-color, #1c1c1c);
    --stc-border:  var(--divider-color, #333);
    --stc-text:    var(--primary-text-color, #e0e0e0);
    --stc-dim:     var(--secondary-text-color, #888);
    --stc-accent:  var(--primary-color, #58a6ff);
    --stc-ok:      var(--success-color, #3fb950);
    --stc-warn:    var(--warning-color, #f0883e);
    --stc-danger:  var(--error-color, #f85149);
    font-family: var(--paper-font-body1_-_font-family, sans-serif);
  }

  ha-card {
    overflow: hidden;
    padding: 0;
    background: var(--stc-bg);
  }

  .header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 8px 14px;
    border-bottom: 1px solid var(--stc-border);
  }

  .title {
    font-size: 12px;
    letter-spacing: .1em;
    text-transform: uppercase;
    color: var(--stc-dim);
    font-weight: 500;
  }

  .live {
    display: flex;
    align-items: center;
    gap: 5px;
    font-size: 10px;
    color: var(--stc-ok);
  }

  .live-dot {
    width: 6px; height: 6px;
    border-radius: 50%;
    background: var(--stc-ok);
    animation: pulse 2s ease-in-out infinite;
  }

  @keyframes pulse {
    0%,100% { opacity:1; transform:scale(1); }
    50%      { opacity:.3; transform:scale(.6); }
  }

  .viewport {
    overflow: hidden;
    position: relative;
  }

  .viewport::before,
  .viewport::after {
    content: '';
    position: absolute;
    left: 0; right: 0;
    height: 32px;
    z-index: 2;
    pointer-events: none;
  }
  .viewport::before { top:0;    background: linear-gradient(var(--stc-bg), transparent); }
  .viewport::after  { bottom:0; background: linear-gradient(transparent, var(--stc-bg)); }

  .track {
    display: flex;
    flex-direction: column;
    will-change: transform;
  }

  .track.anim {
    transition: transform .5s cubic-bezier(.4,0,.2,1);
  }

  .row {
    display: flex;
    align-items: center;
    padding: 0 14px;
    border-bottom: 1px solid color-mix(in srgb, var(--stc-border) 60%, transparent);
  }

  .name {
    flex: 1 1 auto;
    min-width: 0;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    color: var(--stc-text);
  }

  .sep {
    flex: 0 0 auto;
    padding: 0 10px;
    color: var(--stc-border);
    font-size: .8em;
    letter-spacing: .15em;
  }

  .value {
    flex: 0 0 auto;
    text-align: right;
    font-variant-numeric: tabular-nums;
    white-space: nowrap;
  }

  .unit {
    font-size: .8em;
    margin-left: 2px;
    color: var(--stc-dim);
  }

  .c-normal  { color: var(--stc-accent); }
  .c-ok      { color: var(--stc-ok); }
  .c-warn    { color: var(--stc-warn); }
  .c-danger  { color: var(--stc-danger); }
  .c-unavail { color: var(--stc-dim); font-style: italic; }
`;

class SensorTickerCard extends HTMLElement {

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    this._config   = {};
    this._hass     = null;
    this._step     = 0;
    this._timer    = null;
    this._rows     = [];
    this._built    = false;
  }

  // Called by Lovelace with the card's YAML config
  setConfig(config) {
    if (!config.sensors || !Array.isArray(config.sensors)) {
      throw new Error('sensor-ticker-card: "sensors" list required');
    }
    this._config = {
      title:          config.title          ?? 'Sensor Feed',
      height:         config.height         ?? 250,
      row_height:     config.row_height      ?? 36,
      font_size:      config.font_size       ?? 14,
      scroll_speed:   config.scroll_speed    ?? 2500,   // ms per row
      separator:      config.separator       ?? '···',
      show_header:    config.show_header     !== false,
      sensors:        config.sensors,
    };
    this._built = false;
    this._render();
  }

  set hass(hass) {
    this._hass = hass;
    if (!this._built) {
      this._render();
    } else {
      // Live update: rebuild rows in-place without resetting scroll
      this._refreshValues();
    }
  }

  _render() {
    if (!this._hass || !this._config.sensors) return;

    const cfg = this._config;
    const shadow = this.shadowRoot;

    shadow.innerHTML = `
      <style>${STYLES}</style>
      <ha-card>
        ${cfg.show_header ? `
        <div class="header">
          <span class="title">${this._esc(cfg.title)}</span>
          <span class="live"><span class="live-dot"></span>LIVE</span>
        </div>` : ''}
        <div class="viewport" style="height:${cfg.height}px">
          <div class="track" id="track"></div>
        </div>
      </ha-card>
    `;

    this._buildTrack();
    this._startScroll();
    this._built = true;
  }

  _resolveRows() {
    const cfg   = this._config;
    const hass  = this._hass;
    const rows  = [];

    for (const s of cfg.sensors) {
      const stateObj = hass.states[s.entity];

      // show_when: only display row when the entity's own state matches
      if (s.show_when) {
        if (!stateObj) continue;
        if (!this._evalCondition(stateObj.state, s.show_when.op, s.show_when.value)) continue;
      }
      if (!stateObj && !s.show_unavailable) continue;

      const raw      = stateObj ? stateObj.state : 'unavailable';
      const attrs    = stateObj ? stateObj.attributes : {};
      const name     = s.name   ?? attrs.friendly_name ?? s.entity;
      const unit     = s.unit   ?? attrs.unit_of_measurement ?? '';
      const numVal   = parseFloat(raw);

      // Evaluate conditions for color
      let colorClass = 'c-normal';
      if (!stateObj) {
        colorClass = 'c-unavail';
      } else if (s.conditions) {
        for (const cond of s.conditions) {
          if (this._evalCondition(raw, cond.op, cond.value)) {
            colorClass = 'c-' + (cond.color ?? 'warn');
          }
        }
      }

      rows.push({ name, value: raw, unit, colorClass });
    }

    return rows;
  }

  _evalCondition(stateVal, op, condVal) {
    const n  = parseFloat(stateVal);
    const cv = parseFloat(condVal);
    switch (op) {
      case '>':  return !isNaN(n) && n  >  cv;
      case '>=': return !isNaN(n) && n  >= cv;
      case '<':  return !isNaN(n) && n  <  cv;
      case '<=': return !isNaN(n) && n  <= cv;
      case '==': return String(stateVal) === String(condVal);
      case '!=': return String(stateVal) !== String(condVal);
      default:   return false;
    }
  }

  _buildTrack() {
    const cfg   = this._config;
    const track = this.shadowRoot.getElementById('track');
    if (!track) return;

    this._rows = this._resolveRows();
    if (!this._rows.length) return;

    track.innerHTML = '';
    this._step = 0;

    // Render 3x for seamless looping
    const total = this._rows.length;
    const count = Math.max(total * 3, 24);

    for (let i = 0; i < count; i++) {
      const r   = this._rows[i % total];
      const row = document.createElement('div');
      row.className = 'row';
      row.style.height    = cfg.row_height + 'px';
      row.style.minHeight = cfg.row_height + 'px';
      row.style.fontSize  = cfg.font_size  + 'px';
      row.dataset.index   = i % total;
      row.innerHTML = `
        <span class="name">${this._esc(r.name)}</span>
        <span class="sep">${this._esc(cfg.separator)}</span>
        <span class="value ${r.colorClass}">${this._esc(r.value)}<span class="unit">${this._esc(r.unit)}</span></span>
      `;
      track.appendChild(row);
    }
  }

  _refreshValues() {
    const newRows = this._resolveRows();

    // If the set of visible rows has changed, rebuild the track entirely
    const sameSet =
      newRows.length === this._rows.length &&
      newRows.every((r, i) => r.name === this._rows[i].name);

    if (!sameSet) {
      this._buildTrack();
      return;
    }

    // Same rows — just update values and colours in place
    this._rows = newRows;
    if (!newRows.length) return;

    const track   = this.shadowRoot.getElementById('track');
    if (!track) return;
    const domRows = track.querySelectorAll('.row');
    const total   = newRows.length;

    domRows.forEach(el => {
      const idx    = parseInt(el.dataset.index);
      const r      = newRows[idx % total];
      if (!r) return;
      const nameEl = el.querySelector('.name');
      const valEl  = el.querySelector('.value');
      const unitEl = el.querySelector('.unit');
      if (nameEl) nameEl.textContent = r.name;
      if (valEl)  valEl.className = 'value ' + r.colorClass;
      if (unitEl) { unitEl.textContent = r.unit; valEl.childNodes[0].textContent = r.value; }
    });
  }

  _startScroll() {
    if (this._timer) clearInterval(this._timer);
    this._step = 0;
    this._timer = setInterval(() => this._tick(), this._config.scroll_speed);
  }

  _tick() {
    const track = this.shadowRoot.getElementById('track');
    if (!track || !this._rows.length) return;

    const rh = this._config.row_height;
    this._step++;

    track.classList.add('anim');
    track.style.transform = `translateY(-${this._step * rh}px)`;

    if (this._step >= this._rows.length) {
      setTimeout(() => {
        track.classList.remove('anim');
        this._step = 0;
        track.style.transform = 'translateY(0)';
      }, 520);
    }
  }

  disconnectedCallback() {
    if (this._timer) clearInterval(this._timer);
  }

  // Lovelace card size hint
  getCardSize() {
    return Math.ceil(this._config.height / 50);
  }

  _esc(s) {
    return String(s ?? '')
      .replace(/&/g,'&amp;')
      .replace(/</g,'&lt;')
      .replace(/>/g,'&gt;');
  }

  static getStubConfig() {
    return {
      title: 'Sensor Feed',
      height: 250,
      row_height: 36,
      scroll_speed: 2500,
      sensors: [
        { entity: 'sensor.example_temperature', name: 'Temperature', conditions: [{ op: '>', value: '30', color: 'warn' }] },
        { entity: 'sensor.example_humidity' },
      ]
    };
  }
}

customElements.define('sensor-ticker-card', SensorTickerCard);

window.customCards = window.customCards || [];
window.customCards.push({
  type:        'sensor-ticker-card',
  name:        'Sensor Ticker',
  description: 'Scrolling sensor value ticker with conditional formatting',
  preview:     false,
});
