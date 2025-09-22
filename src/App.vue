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
const tol = ref(0.001)    // en %
const maxIter = ref(100)

// -------- Resultados --------
const out = reactive({
  biseccion: { root: null, froot: null, iters: 0 },
  falsapos:  { root: null, froot: null, iters: 0 },
})

// -------- Tablas de tabulación (historial de iteraciones) --------
const table = reactive({
  biseccion: [],
  falsapos:  [],
})

// -------- Utilidades --------
function fx6(v) {
  return Number.isFinite(v) ? Number(v).toFixed(6) : 'NaN'
}
function eaPercent(xNew, xOld) {
  const denom = Math.abs(xNew) > 0 ? Math.abs(xNew) : 1
  return Math.abs((xNew - xOld) / denom) * 100
}
function resetTables() {
  table.biseccion = []
  table.falsapos  = []
}

// -------- Métodos --------
function runBisection() {
  const f = functions[selectedFn.value]
  let xl = +a.value
  let xu = +b.value
  let fl = f(xl), fu = f(xu)

  // limpia tabla previa de bisección
  table.biseccion = []

  if (fl * fu > 0) {
    alert('f(a) y f(b) no tienen signos opuestos.')
    out.biseccion = { root: null, froot: null, iters: 0 }
    return
  }

  let xr = (xl + xu) / 2
  let it = 0, ea = 100

  // guarda primera fila (iteración 0 con xr inicial)
  table.biseccion.push({
    i: it, xl, xu, xr,
    fl, fu, fr: f(xr),
    ea: null
  })

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

    // registra fila
    table.biseccion.push({
      i: it,
      xl, xu, xr: xrNew,
      fl, fu, fr: f(xrNew),
      ea
    })

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

  // limpia tabla previa de regla falsa
  table.falsapos = []

  if (fl * fu > 0) {
    alert('f(a) y f(b) no tienen signos opuestos.')
    out.falsapos = { root: null, froot: null, iters: 0 }
    return
  }

  let xr = xu - fu * (xl - xu) / (fl - fu)
  let it = 0, ea = 100

  // registra iteración 0
  table.falsapos.push({
    i: it, xl, xu, xr,
    fl, fu, fr: f(xr),
    ea: null
  })

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

    table.falsapos.push({
      i: it,
      xl, xu, xr: xrNew,
      fl, fu, fr: f(xrNew),
      ea
    })

    xr = xrNew
    if (Math.abs(f(xr)) < 1e-10) break
  }

  out.falsapos = { root: xr, froot: f(xr), iters: it }
}
</script>

<template>
  <div style="max-width:1000px;margin:auto;padding:20px;">
    <h1>Métodos de Raíces</h1>

    <!-- Selección de función -->
    <label>Selecciona función:</label>
    <select v-model="selectedFn" @change="resetTables">
      <option value="f">f(x) = 4x³ - 6x² + 7x - 2.3</option>
      <option value="g">g(x) = x²·√(|cos(x)|) - 5</option>
    </select>

    <!-- Intervalo -->
    <div style="margin-top:10px;">
      <label>Intervalo [a, b]:</label>
      <input v-model.number="a" type="number" step="any" placeholder="a" @change="resetTables" />
      <input v-model.number="b" type="number" step="any" placeholder="b" @change="resetTables" />
    </div>

    <!-- Tolerancia y máx iter -->
    <div style="margin-top:10px;">
      <label>Tolerancia (%):</label>
      <input v-model.number="tol" type="number" step="any" @change="resetTables" />
      <label>Máx iteraciones:</label>
      <input v-model.number="maxIter" type="number" @change="resetTables" />
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

    <!-- Tablas de tabulación -->
    <div style="margin-top:20px;">
      <h2>Tabulación de Bisección</h2>
      <div v-if="table.biseccion.length">
        <div style="overflow:auto;">
          <table border="1" cellpadding="6" style="border-collapse:collapse; min-width:900px;">
            <thead>
              <tr>
                <th>i</th>
                <th>xl</th>
                <th>xu</th>
                <th>xr</th>
                <th>f(xl)</th>
                <th>f(xu)</th>
                <th>f(xr)</th>
                <th>ea (%)</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in table.biseccion" :key="'bis'+row.i">
                <td>{{ row.i }}</td>
                <td>{{ fx6(row.xl) }}</td>
                <td>{{ fx6(row.xu) }}</td>
                <td>{{ fx6(row.xr) }}</td>
                <td>{{ fx6(row.fl) }}</td>
                <td>{{ fx6(row.fu) }}</td>
                <td>{{ fx6(row.fr) }}</td>
                <td>{{ row.ea == null ? '-' : fx6(row.ea) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
      <p v-else>Corre Bisección para ver la tabla.</p>
    </div>

    <div style="margin-top:25px;">
      <h2>Tabulación de Regla Falsa</h2>
      <div v-if="table.falsapos.length">
        <div style="overflow:auto;">
          <table border="1" cellpadding="6" style="border-collapse:collapse; min-width:900px;">
            <thead>
              <tr>
                <th>i</th>
                <th>xl</th>
                <th>xu</th>
                <th>xr</th>
                <th>f(xl)</th>
                <th>f(xu)</th>
                <th>f(xr)</th>
                <th>ea (%)</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in table.falsapos" :key="'rf'+row.i">
                <td>{{ row.i }}</td>
                <td>{{ fx6(row.xl) }}</td>
                <td>{{ fx6(row.xu) }}</td>
                <td>{{ fx6(row.xr) }}</td>
                <td>{{ fx6(row.fl) }}</td>
                <td>{{ fx6(row.fu) }}</td>
                <td>{{ fx6(row.fr) }}</td>
                <td>{{ row.ea == null ? '-' : fx6(row.ea) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
      <p v-else>Corre Regla Falsa para ver la tabla.</p>
    </div>
  </div>
</template>
