<script setup>
/**
 * Asignación 4 - Métodos de Raíces
 * Vue 3 + JS (sin librerías externas)
 * Métodos: Bisección, Regla Falsa, Newton-Raphson (derivada numérica), Secante
 * Criterio de paro: error aproximado relativo (%) <= tolerancia o |f(x)| <= fTol
 * Salida: raíz, f(raíz), # de iteraciones — en notación fija con 6 decimales
 */

import { ref, reactive, computed } from 'vue'

// ----------- Utilidades numéricas -----------

/** Compila una expresión f(x) escrita por el usuario a una función JS segura con Math en scope. */
function compileFn(expr) {
  // Permite usar: sin, cos, tan, log, exp, sqrt, abs, pow, etc. y constantes como PI, E
  // Se permite ^? No. Indica al usuario que use ** para potencias.
  // throw si la expresión no compila.
  const fn = Function('x', `with (Math) { return (${expr}); }`)
  // Prueba rápida para dar error temprano
  // eslint-disable-next-line no-unused-vars
  const _ = fn(0)
  return (x) => fn(x)
}

/** Derivada numérica central de f en x. */
function dfdx(f, x) {
  const h = 1e-6 * (1 + Math.abs(x))
  return (f(x + h) - f(x - h)) / (2 * h)
}

/** Formatea a notación fija de 6 decimales. */
function fx6(v) {
  if (!Number.isFinite(v)) return 'NaN'
  return v.toFixed(6)
}

/** Error aproximado relativo porcentual entre x_new y x_old. */
function eaPercent(xNew, xOld) {
  const denom = Math.abs(xNew) > 0 ? Math.abs(xNew) : 1
  return Math.abs((xNew - xOld) / denom) * 100
}

// ----------- Estado (inputs) -----------

const expr = ref('x**3 - x - 2') // editable
const a = ref(1)
const b = ref(2)
const x0 = ref(1.5)   // para Newton
const x1 = ref(1.0)   // para Secante (x0 y x1)
const tol = ref(0.001)  // tolerancia de error aproximado (%) — p.ej. 0.001 = 0.001%
const fTol = ref(1e-12) // tolerancia absoluta para |f(x)|
const maxIter = ref(100)

const errorMsg = ref('')
const iterRows = ref([]) // tabla de iteraciones de la última corrida (cualquiera)
const lastMethod = ref('')

// Resultados por método
const out = reactive({
  biseccion: { root: null, froot: null, iters: 0 },
  falsapos:  { root: null, froot: null, iters: 0 },
  newton:    { root: null, froot: null, iters: 0 },
  secante:   { root: null, froot: null, iters: 0 },
})

// ----------- Run wrappers -----------

function resetTable() {
  iterRows.value = []
  errorMsg.value = ''
}

function pushRow(row) {
  iterRows.value.push(row)
}

function validateCommon(f) {
  // Valida que la función compile
  try {
    compileFn(expr.value)
  } catch (e) {
    throw new Error('La expresión de f(x) es inválida. Usa sintaxis JS y funciones de Math. Ej: x**3 - x - 2, sin(x), exp(-x), etc.')
  }
  // Valida maxIter y tol
  if (!(maxIter.value > 0 && Number.isFinite(+maxIter.value))) {
    throw new Error('maxIter debe ser un entero positivo.')
  }
  if (!(tol.value > 0 && Number.isFinite(+tol.value))) {
    throw new Error('La tolerancia (%) debe ser un número positivo.')
  }
  if (!(fTol.value > 0 && Number.isFinite(+fTol.value))) {
    throw new Error('La tolerancia de |f(x)| debe ser positiva.')
  }
}

// ----------- Métodos -----------

function runBisection() {
  resetTable()
  lastMethod.value = 'Bisección'
  try {
    validateCommon()
    const f = compileFn(expr.value)
    let xl = +a.value
    let xu = +b.value
    if (!(Number.isFinite(xl) && Number.isFinite(xu))) throw new Error('a y b deben ser números.')
    if (xl >= xu) throw new Error('Se requiere a < b.')
    const fl = f(xl), fu = f(xu)
    if (fl === 0) { out.biseccion = { root: xl, froot: 0, iters: 0 }; return }
    if (fu === 0) { out.biseccion = { root: xu, froot: 0, iters: 0 }; return }
    if (fl * fu > 0) throw new Error('f(a) y f(b) tienen el mismo signo. No hay garantía de raíz en [a, b].')

    let xr = (xl + xu) / 2
    let xrOld = xr
    let ea = 100
    let it = 0

    pushRow({ it, xl, xu, xr, fxr: f(xr), ea: null })

    while (it < +maxIter.value) {
      it++
      const fr = f(xr)
      if (Math.abs(fr) <= +fTol.value) break

      // Decide subintervalo
      if (fl * fr < 0) {
        xu = xr
      } else {
        xl = xr
      }

      const xrNew = (xl + xu) / 2
      ea = eaPercent(xrNew, xr)
      xrOld = xr
      xr = xrNew

      pushRow({ it, xl, xu, xr, fxr: f(xr), ea })

      if (ea <= +tol.value) break
    }

    out.biseccion = { root: xr, froot: f(xr), iters: it }
  } catch (e) {
    errorMsg.value = e.message || String(e)
  }
}

function runFalsePosition() {
  resetTable()
  lastMethod.value = 'Regla Falsa'
  try {
    validateCommon()
    const f = compileFn(expr.value)
    let xl = +a.value
    let xu = +b.value
    if (!(Number.isFinite(xl) && Number.isFinite(xu))) throw new Error('a y b deben ser números.')
    if (xl >= xu) throw new Error('Se requiere a < b.')
    let fl = f(xl), fu = f(xu)
    if (fl === 0) { out.falsapos = { root: xl, froot: 0, iters: 0 }; return }
    if (fu === 0) { out.falsapos = { root: xu, froot: 0, iters: 0 }; return }
    if (fl * fu > 0) throw new Error('f(a) y f(b) tienen el mismo signo. No hay garantía de raíz en [a, b].')

    let xr = xu - fu * (xl - xu) / (fl - fu)
    let ea = 100
    let it = 0

    pushRow({ it, xl, xu, xr, fxr: f(xr), ea: null })

    while (it < +maxIter.value) {
      it++
      const fr = f(xr)
      if (Math.abs(fr) <= +fTol.value) break

      if (fl * fr < 0) {
        xu = xr
        fu = fr
      } else {
        xl = xr
        fl = fr
      }

      const xrNew = xu - fu * (xl - xu) / (fl - fu)
      ea = eaPercent(xrNew, xr)
      xr = xrNew

      pushRow({ it, xl, xu, xr, fxr: f(xr), ea })

      if (ea <= +tol.value) break
    }

    out.falsapos = { root: xr, froot: f(xr), iters: it }
  } catch (e) {
    errorMsg.value = e.message || String(e)
  }
}

function runNewton() {
  resetTable()
  lastMethod.value = 'Newton–Raphson'
  try {
    validateCommon()
    const f = compileFn(expr.value)
    let x = +x0.value
    if (!Number.isFinite(x)) throw new Error('x0 debe ser un número.')

    let it = 0
    let ea = 100
    let fx = f(x)

    pushRow({ it, x, fx, dfx: dfdx(f, x), ea: null })

    while (it < +maxIter.value) {
      it++
      const d = dfdx(f, x)
      if (!Number.isFinite(d) || Math.abs(d) < 1e-15) throw new Error('Derivada ~0: Newton no puede continuar (posible punto crítico).')
      const xNew = x - fx / d
      ea = eaPercent(xNew, x)
      x = xNew
      fx = f(x)
      pushRow({ it, x, fx, dfx: d, ea })
      if (Math.abs(fx) <= +fTol.value || ea <= +tol.value) break
    }

    out.newton = { root: x, froot: fx, iters: it }
  } catch (e) {
    errorMsg.value = e.message || String(e)
  }
}

function runSecant() {
  resetTable()
  lastMethod.value = 'Secante'
  try {
    validateCommon()
    const f = compileFn(expr.value)
    let x_prev = +x0.value
    let x_curr = +x1.value
    if (!(Number.isFinite(x_prev) && Number.isFinite(x_curr))) throw new Error('x0 y x1 deben ser números.')
    if (x_prev === x_curr) throw new Error('x0 y x1 no deben ser iguales.')

    let it = 0
    let ea = 100
    let fprev = f(x_prev)
    let fcurr = f(x_curr)

    pushRow({ it, x0: x_prev, x1: x_curr, fx0: fprev, fx1: fcurr, ea: null })

    while (it < +maxIter.value) {
      it++
      const denom = (fcurr - fprev)
      if (Math.abs(denom) < 1e-30) throw new Error('División por ~0 en Secante: f(x1) ≈ f(x0).')
      const x_next = x_curr - fcurr * (x_curr - x_prev) / denom
      ea = eaPercent(x_next, x_curr)

      x_prev = x_curr
      fprev = fcurr
      x_curr = x_next
      fcurr = f(x_curr)

      pushRow({ it, x0: x_prev, x1: x_curr, fx0: fprev, fx1: fcurr, ea })

      if (Math.abs(fcurr) <= +fTol.value || ea <= +tol.value) break
    }

    out.secante = { root: x_curr, froot: fcurr, iters: it }
  } catch (e) {
    errorMsg.value = e.message || String(e)
  }
}

// ----------- Derivados para UI -----------

const fPreview = computed(() => {
  try {
    const f = compileFn(expr.value)
    return `f(1) = ${f(1).toFixed(6)}`
  } catch {
    return 'f(1) = —'
  }
})

</script>

<template>
  <div class="container">
    <h1>Implementación de Métodos de Raíces</h1>
    <p class="sub">
      Bisección · Regla Falsa · Newton–Raphson · Secante — salida en 6 decimales. Usa <code>**</code> para potencias (p.ej. <code>x**3 - x - 2</code>).
    </p>

    <div class="grid">
      <!-- Panel de entrada -->
      <div class="card">
        <label>Función f(x) <span class="small">— puedes usar funciones de <code>Math</code> como <code>sin</code>, <code>cos</code>, <code>exp</code>, <code>sqrt</code>, <code>PI</code>.</span></label>
        <input v-model="expr" placeholder="Ej: x**3 - x - 2" />

        <div class="row" style="margin-top:10px">
          <div>
            <label>Intervalo [a, b] (Bisección y Regla Falsa)</label>
            <div class="row">
              <input v-model.number="a" type="number" placeholder="a" />
              <input v-model.number="b" type="number" placeholder="b" />
            </div>
          </div>
          <div>
            <label>Inicial(es) (Newton: x0 · Secante: x0, x1)</label>
            <div class="row">
              <input v-model.number="x0" type="number" placeholder="x0" />
              <input v-model.number="x1" type="number" placeholder="x1 (secante)" />
            </div>
          </div>
        </div>

        <div class="row" style="margin-top:10px">
          <div>
            <label>Tolerancia de error aproximado (%)</label>
            <input v-model.number="tol" type="number" step="any" />
            <div class="small">Criterio: ea ≤ tol o |f(x)| ≤ |fTol|</div>
          </div>
          <div>
            <label>|f(x)| tolerancia absoluta (fTol)</label>
            <input v-model.number="fTol" type="number" step="any" />
          </div>
        </div>

        <div class="row" style="margin-top:10px">
          <div>
            <label>Máximo de iteraciones</label>
            <input v-model.number="maxIter" type="number" />
          </div>
          <div>
            <label>Previsualización rápida</label>
            <input :value="fPreview" disabled />
          </div>
        </div>

        <div class="btns btns-4" style="margin-top:12px">
          <button @click="runBisection">Bisección</button>
          <button @click="runFalsePosition">Regla Falsa</button>
          <button @click="runNewton">Newton–Raphson</button>
          <button @click="runSecant">Secante</button>
        </div>

        <div v-if="errorMsg" class="alert">{{ errorMsg }}</div>
      </div>

      <!-- Salidas / KPIs -->
      <div class="out">
        <div class="card">
          <h2 style="margin:0 0 10px">Resultados</h2>
          <div class="row">
            <div class="kpi">
              <h3>Bisección — raíz</h3>
              <div class="val">{{ fx6(out.biseccion.root ?? NaN) }}</div>
              <div class="small">f(raíz) = {{ fx6(out.biseccion.froot ?? NaN) }} · it = {{ out.biseccion.iters }}</div>
            </div>
            <div class="kpi">
              <h3>Regla Falsa — raíz</h3>
              <div class="val">{{ fx6(out.falsapos.root ?? NaN) }}</div>
              <div class="small">f(raíz) = {{ fx6(out.falsapos.froot ?? NaN) }} · it = {{ out.falsapos.iters }}</div>
            </div>
          </div>
          <div class="row" style="margin-top:10px">
            <div class="kpi">
              <h3>Newton–Raphson — raíz</h3>
              <div class="val">{{ fx6(out.newton.root ?? NaN) }}</div>
              <div class="small">f(raíz) = {{ fx6(out.newton.froot ?? NaN) }} · it = {{ out.newton.iters }}</div>
            </div>
            <div class="kpi">
              <h3>Secante — raíz</h3>
              <div class="val">{{ fx6(out.secante.root ?? NaN) }}</div>
              <div class="small">f(raíz) = {{ fx6(out.secante.froot ?? NaN) }} · it = {{ out.secante.iters }}</div>
            </div>
          </div>
          <footer>Formato fijo de 6 decimales.</footer>
        </div>

        <div class="card">
          <h2 style="margin:0 0 10px">Tabla de iteraciones <span class="small">({{ lastMethod || '—' }})</span></h2>
          <div class="small" style="margin-bottom:6px">Las columnas cambian según el método ejecutado.</div>
          <table class="table" v-if="iterRows.length">
            <thead>
              <tr v-if="lastMethod === 'Bisección' || lastMethod === 'Regla Falsa'">
                <th>it</th><th>xl</th><th>xu</th><th>xr</th><th>f(xr)</th><th>ea (%)</th>
              </tr>
              <tr v-else-if="lastMethod === 'Newton–Raphson'">
                <th>it</th><th>x</th><th>f(x)</th><th>f'(x)</th><th>ea (%)</th>
              </tr>
              <tr v-else-if="lastMethod === 'Secante'">
                <th>it</th><th>x₀</th><th>x₁</th><th>f(x₀)</th><th>f(x₁)</th><th>ea (%)</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="r in iterRows" :key="r.it">
                <template v-if="lastMethod === 'Bisección' || lastMethod === 'Regla Falsa'">
                  <td>{{ r.it }}</td>
                  <td>{{ fx6(r.xl) }}</td>
                  <td>{{ fx6(r.xu) }}</td>
                  <td>{{ fx6(r.xr) }}</td>
                  <td>{{ fx6(r.fxr) }}</td>
                  <td>{{ r.ea == null ? '—' : fx6(r.ea) }}</td>
                </template>
                <template v-else-if="lastMethod === 'Newton–Raphson'">
                  <td>{{ r.it }}</td>
                  <td>{{ fx6(r.x) }}</td>
                  <td>{{ fx6(r.fx) }}</td>
                  <td>{{ fx6(r.dfx) }}</td>
                  <td>{{ r.ea == null ? '—' : fx6(r.ea) }}</td>
                </template>
                <template v-else-if="lastMethod === 'Secante'">
                  <td>{{ r.it }}</td>
                  <td>{{ fx6(r.x0) }}</td>
                  <td>{{ fx6(r.x1) }}</td>
                  <td>{{ fx6(r.fx0) }}</td>
                  <td>{{ fx6(r.fx1) }}</td>
                  <td>{{ r.ea == null ? '—' : fx6(r.ea) }}</td>
                </template>
              </tr>
            </tbody>
          </table>
          <div v-else class="small">Ejecuta un método para ver sus iteraciones.</div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
h2 { font-size: 18px; }
</style>
