<script setup>
import { ref, reactive } from 'vue'

// -------- Funciones disponibles --------
const functions = {
  f: (x) => 4 * x ** 3 - 6 * x ** 2 + 7 * x - 2.3,
  g: (x) => x ** 2 * Math.sqrt(Math.abs(Math.cos(x))) - 5,
}
const selectedFn = ref('f')

// -------- Parámetros --------
const a = ref(1)
const b = ref(2)
const tol = ref(0.001)
const maxIter = ref(100)

// -------- Resultados --------
const out = reactive({
  biseccion: { root: null, froot: null, iters: 0 },
  falsapos:  { root: null, froot: null, iters: 0 },
})

// -------- Utilidades --------
function fx6(v) {
  return Number.isFinite(v) ? v.toFixed(6) : 'NaN'
}
function eaPercent(xNew, xOld) {
  const denom = Math.abs(xNew) > 0 ? Math.abs(xNew) : 1
  return Math.abs((xNew - xOld) / denom) * 100
}

// -------- Métodos --------
function runBisection() {
  const f = functions[selectedFn.value]
  let xl = +a.value
  let xu = +b.value
  let fl = f(xl), fu = f(xu)
  if (fl * fu > 0) {
    alert('f(a) y f(b) no tienen signos opuestos.')
    return
  }

  let xr = (xl + xu) / 2
  let it = 0, ea = 100

  while (it < +maxIter.value && ea > +tol.value) {
    it++
    const fr = f(xr)
    if (fl * fr < 0) {
      xu = xr
      fu = fr
    } else {
      xl = xr
      fl = fr
    }
    const xrNew = (xl + xu) / 2
    ea = eaPercent(xrNew, xr)
    xr = xrNew
    if (Math.abs(f(xr)) < 1e-10) break
  }

  out.biseccion = { root: xr, froot: f(xr), iters: it }
}

function runFalsePosition() {
  const f = functions[selectedFn.value]
  let xl = +a.value
  let xu = +b.value
  let fl = f(xl), fu = f(xu)
  if (fl * fu > 0) {
    alert('f(a) y f(b) no tienen signos opuestos.')
    return
  }

  let xr = xu - fu * (xl - xu) / (fl - fu)
  let it = 0, ea = 100

  while (it < +maxIter.value && ea > +tol.value) {
    it++
    const fr = f(xr)
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
    if (Math.abs(f(xr)) < 1e-10) break
  }

  out.falsapos = { root: xr, froot: f(xr), iters: it }
}
</script>

<template>
  <div style="max-width:600px;margin:auto;padding:20px;">
    <h1>Métodos de Raíces</h1>

    <!-- Selección de función -->
    <label>Selecciona función:</label>
    <select v-model="selectedFn">
      <option value="f">f(x) = 4x³ - 6x² + 7x - 2.3</option>
      <option value="g">g(x) = x²·√(|cos(x)|) - 5</option>
    </select>

    <!-- Intervalo -->
    <div style="margin-top:10px;">
      <label>Intervalo [a, b]:</label>
      <input v-model.number="a" type="number" step="any" placeholder="a" />
      <input v-model.number="b" type="number" step="any" placeholder="b" />
    </div>

    <!-- Tolerancia y máx iter -->
    <div style="margin-top:10px;">
      <label>Tolerancia (%):</label>
      <input v-model.number="tol" type="number" step="any" />
      <label>Máx iteraciones:</label>
      <input v-model.number="maxIter" type="number" />
    </div>

    <!-- Botones -->
    <div style="margin-top:15px;">
      <button @click="runBisection">Bisección</button>
      <button @click="runFalsePosition">Regla Falsa</button>
    </div>

    <!-- Resultados -->
    <div style="margin-top:20px;">
      <h2>Resultados</h2>
      <p><b>Bisección:</b> Raíz = {{ fx6(out.biseccion.root) }}, 
         f(raíz) = {{ fx6(out.biseccion.froot) }}, 
         Iteraciones = {{ out.biseccion.iters }}</p>
      <p><b>Regla Falsa:</b> Raíz = {{ fx6(out.falsapos.root) }}, 
         f(raíz) = {{ fx6(out.falsapos.froot) }}, 
         Iteraciones = {{ out.falsapos.iters }}</p>
    </div>
  </div>
</template>
