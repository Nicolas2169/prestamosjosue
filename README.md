<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<title>Registro de Préstamos</title>
<style>
body{font-family:Arial;margin:20px;background:#f4f6f8;}
h1,h2{text-align:center;color:#333;}
.section{margin-bottom:15px;padding:10px;border:1px solid #ccc;border-radius:5px;background:#fff;}
button{margin:3px;padding:5px 10px;border:none;border-radius:4px;background:#4CAF50;color:white;font-size:12px;cursor:pointer;}
button:hover{background:#45a049;}
select{padding:3px;border-radius:4px;margin:3px;font-size:12px;}
table{border-collapse:collapse;width:100%;background:#fff;box-shadow:0 2px 5px rgba(0,0,0,0.1);font-size:12px;}
th,td{border:1px solid #ddd;padding:4px;text-align:center;}
th{background:#3498db;color:white;}
.cliente{cursor:pointer;font-weight:bold;color:#2c3e50;}
.detalle{display:none;background:#eef1f5;margin-top:2px;padding:5px;border-radius:4px;text-align:left;}
.resumen-box{margin-top:5px;padding:5px;background:#dfe6e9;border-radius:4px;font-size:12px;line-height:1.5;}
.resumen-box b{color:#2c3e50;}
input[type="number"],input[type="date"],input[type="text"]{font-size:11px;padding:2px;}
input[type="checkbox"]{cursor:pointer;}
.pagado{background:#ccffcc;}
.incompleto{background:#fff6a3;}
</style>
</head>
<body>

<h1>Registro de Préstamos</h1>

<div class="section">
<h2>Clientes - Préstamos</h2>
<button onclick="agregarNombre()">Agregar Nombre</button>
<select id="selectNombresComunes"><option disabled selected>Selecciona un nombre</option></select>
<button onclick="eliminarNombre()">Eliminar Nombre</button>
</div>

<table id="tablaPrestamos">
  <thead><tr><th>Nombre</th><th>Resumen del Préstamo</th></tr></thead>
  <tbody id="cuerpoTabla"></tbody>
</table>

<script>
const currentYear = new Date().getFullYear();
let nombresComunes = [];
const meses = ["Ene","Feb","Mar","Abr","May","Jun","Jul","Ago","Sep","Oct","Nov","Dic"];
let dataClientes = JSON.parse(localStorage.getItem("clientes")) || {};

function inicializarDatos(){
  const stored = JSON.parse(localStorage.getItem("clientes"));
  if(stored) nombresComunes = Object.keys(stored);
  actualizarSelect();
}

function actualizarSelect(){
  const select = document.getElementById("selectNombresComunes");
  select.innerHTML = "<option disabled selected>Selecciona un nombre</option>";
  nombresComunes.forEach(n => {
    const opt = document.createElement("option");
    opt.value = n;
    opt.textContent = n;
    select.appendChild(opt);
  });
}

function renderizarTabla(){
  const cuerpo = document.getElementById("cuerpoTabla");
  cuerpo.innerHTML = "";

  nombresComunes.forEach(nombre=>{
    if(!dataClientes[nombre])
      dataClientes[nombre]={valor:0,interes:0,cuotas:1,pagos:{},fechaInicio:`${currentYear}-01`};

    const tr=document.createElement("tr");
    const tdNombre=document.createElement("td");
    tdNombre.classList.add("cliente");
    tdNombre.onclick=()=>toggleDetalle(tr.nextSibling);
    tdNombre.innerHTML=`<b>${nombre}</b><br><span id="totales_${escapeId(nombre)}" class="restante-total"></span>`;
    tr.appendChild(tdNombre);

    const tdResumen=document.createElement("td");
    tdResumen.id=`resumen_${escapeId(nombre)}`;
    tdResumen.innerHTML="Sin datos";
    tr.appendChild(tdResumen);
    cuerpo.appendChild(tr);

    const detalleTr=document.createElement("tr");
    const tdDetalle=document.createElement("td");
    tdDetalle.colSpan=2;
    tdDetalle.innerHTML=`
      <div class="detalle" style="display:none;">
        <b>Configuración de ${nombre}</b><br>
        Fecha de Inicio: <input type="month" id="inicio_${escapeId(nombre)}" value="${dataClientes[nombre].fechaInicio}" onchange="actualizarCliente('${escapeJs(nombre)}')">
        Valor: <input type="number" id="valor_${escapeId(nombre)}" style="width:80px;" value="${dataClientes[nombre].valor}" oninput="actualizarCliente('${escapeJs(nombre)}')">
        Interés anual (%): <input type="number" id="interes_${escapeId(nombre)}" style="width:60px;" value="${dataClientes[nombre].interes}" oninput="actualizarCliente('${escapeJs(nombre)}')">
        Cuotas: <input type="number" id="cuotas_${escapeId(nombre)}" style="width:50px;" value="${dataClientes[nombre].cuotas}" oninput="actualizarCliente('${escapeJs(nombre)}')">
        <div id="tablaPagos_${escapeId(nombre)}"></div>
      </div>`;
    detalleTr.appendChild(tdDetalle);
    cuerpo.appendChild(detalleTr);

    generarTablaPagos(nombre);
  });

  actualizarSelect();
}

function actualizarCliente(nombre){
  const c=dataClientes[nombre];
  c.valor=parseFloat(document.getElementById(`valor_${escapeId(nombre)}`).value)||0;
  c.interes=parseFloat(document.getElementById(`interes_${escapeId(nombre)}`).value)||0;
  c.cuotas=parseInt(document.getElementById(`cuotas_${escapeId(nombre)}`).value)||1;
  c.fechaInicio=document.getElementById(`inicio_${escapeId(nombre)}`).value;
  guardarData();
  generarTablaPagos(nombre);
}

// ====================================================
// NUEVA LÓGICA DE INTERÉS DECRECIENTE (SOBRE SALDO)
// ====================================================
function generarTablaPagos(nombre, focusId = null){
  const c = dataClientes[nombre];
  const tasaMensual = c.interes / 12 / 100;
  const valor = c.valor;
  const n = c.cuotas;

  const cuotaFija = (tasaMensual > 0 && n > 0)
    ? (valor * tasaMensual) / (1 - Math.pow(1 + tasaMensual, -n))
    : (n > 0 ? valor / n : 0);

  let saldo = valor;
  let totalPagado = 0, totalIntereses = 0, cuotasPagadas = 0;

  const contenedor = document.getElementById(`tablaPagos_${escapeId(nombre)}`);
  let html = `<div class='resumen-box'>
    <b>Valor préstamo:</b> $${valor.toFixed(2)} |
    <b>Interés anual:</b> ${c.interes.toFixed(2)}% |
    <b>Cuotas:</b> ${c.cuotas} |
    <b>Cuota fija:</b> $${cuotaFija.toFixed(2)}
  </div>`;

  html += `<table style="width:100%;margin-top:5px;">
  <tr>
  <th>Mes</th><th>Año</th><th>Fecha</th>
  <th>Interés</th><th>Capital</th><th>Saldo</th>
  <th>Pagado</th><th>Pago Incompleto</th>
  <th>Abono Extra</th><th>Pendiente</th><th>Observaciones</th></tr>`;

  const [anioInicio, mesInicio] = c.fechaInicio.split('-').map(Number);
  let mes = (isNaN(mesInicio)?0:mesInicio - 1),
      anio = (isNaN(anioInicio)?currentYear:anioInicio);
  let abonoExtraPendiente = 0;

  for (let i = 0; i < n; i++) {
    const key = `${anio}_${meses[mes]}`;
    if (!c.pagos[key]) c.pagos[key] = { fecha: "", pagado: false, incompleto: 0, extra: 0, obs: "" };
    const p = c.pagos[key];

    const interesMes = saldo * tasaMensual;
    let capitalMes = cuotaFija - interesMes;
    if (capitalMes < 0) capitalMes = 0;

    capitalMes += abonoExtraPendiente;
    abonoExtraPendiente = 0;

    let nuevoSaldo = saldo - capitalMes;
    if (nuevoSaldo < 0) nuevoSaldo = 0;

    totalIntereses += interesMes;

    const pagoCompleto = p.pagado ? cuotaFija : 0;
    const pagoIncompleto = parseFloat(p.incompleto) || 0;
    const abonoExtra = parseFloat(p.extra) || 0;
    const pendiente = Math.max(0, cuotaFija - (pagoCompleto + pagoIncompleto + abonoExtra));

    totalPagado += pagoCompleto + pagoIncompleto + abonoExtra;
    if (p.pagado) cuotasPagadas++;
    abonoExtraPendiente += abonoExtra;

    const clase = p.pagado ? 'pagado' : (pagoIncompleto > 0 ? 'incompleto' : '');
    const idKey = escapeId(key);

    html += `<tr class="${clase}">
      <td>${meses[mes]}</td>
      <td>${anio}</td>
      <td><input type="date" id="fecha_${escapeId(nombre)}_${idKey}" value="${p.fecha}" onchange="editarCampo('${escapeJs(nombre)}','${escapeJs(key)}','fecha',this.value, this.id)"></td>
      <td>$${interesMes.toFixed(2)}</td>
      <td>$${capitalMes.toFixed(2)}</td>
      <td>$${nuevoSaldo.toFixed(2)}</td>
      <td><input type="checkbox" id="check_${escapeId(nombre)}_${idKey}" ${p.pagado ? "checked" : ""} onchange="editarCampo('${escapeJs(nombre)}','${escapeJs(key)}','pagado',this.checked)"></td>
      <td><input type="number" id="incompleto_${escapeId(nombre)}_${idKey}" min="0" value="${pagoIncompleto}" oninput="editarCampo('${escapeJs(nombre)}','${escapeJs(key)}','incompleto',this.value, this.id)" style="width:70px;text-align:right;"></td>
      <td><input type="number" id="extra_${escapeId(nombre)}_${idKey}" min="0" value="${abonoExtra}" oninput="editarCampo('${escapeJs(nombre)}','${escapeJs(key)}','extra',this.value, this.id)" style="width:70px;text-align:right;"></td>
      <td id="pendiente_${escapeId(nombre)}_${idKey}">$${pendiente.toFixed(2)}</td>
      <td><input type="text" id="obs_${escapeId(nombre)}_${idKey}" value="${p.obs}" oninput="editarCampo('${escapeJs(nombre)}','${escapeJs(key)}','obs',this.value, this.id)" style="width:100px;"></td>
    </tr>`;

    saldo = nuevoSaldo;
    mes++; if (mes > 11) { mes = 0; anio++; }
  }

  html += "</table>";
  contenedor.innerHTML = html;

  const resumen = document.getElementById(`resumen_${escapeId(nombre)}`);
  const totalPendiente = Math.max(0, valor + totalIntereses - totalPagado);
  resumen.innerHTML = `
    <div class="resumen-box">
    <b>Valor préstamo:</b> $${valor.toFixed(2)} |
    <b>Interés anual:</b> ${c.interes.toFixed(2)}%<br>
    <b>Total intereses:</b> $${totalIntereses.toFixed(2)} |
    <b>Total pagado:</b> $${totalPagado.toFixed(2)}<br>
    <b>Cuotas:</b> ${c.cuotas} | <b>Pagadas:</b> ${cuotasPagadas} |
    <b>Pendiente:</b> $${totalPendiente.toFixed(2)}
    </div>`;
  guardarData();
  if (focusId) focusElementById(focusId);
}

function editarCampo(nombre,key,campo,valor, elementId = null){
  const c=dataClientes[nombre];
  if(!c.pagos[key]) c.pagos[key]={fecha:"",pagado:false,incompleto:0,extra:0,obs:""};
  if(campo==="pagado") c.pagos[key].pagado=!!valor;
  else if(["fecha","obs"].includes(campo)) c.pagos[key][campo]=valor;
  else c.pagos[key][campo]=parseFloat(valor)||0;
  guardarData();
  generarTablaPagos(nombre, elementId);
}

function toggleDetalle(tr){
  const d=tr.querySelector(".detalle");
  d.style.display=d.style.display==="none"?"block":"none";
}

function guardarData(){localStorage.setItem("clientes",JSON.stringify(dataClientes));}
function agregarNombre(){
  const n=prompt("Ingrese nombre:");
  if(n && !nombresComunes.includes(n)){
    nombresComunes.push(n);
    dataClientes[n]={valor:0,interes:0,cuotas:1,pagos:{},fechaInicio:`${currentYear}-01`};
    guardarData();
    renderizarTabla();
  }
}
function eliminarNombre(){
  const sel=document.getElementById("selectNombresComunes");
  const n=sel.value;
  if(n){
    delete dataClientes[n];
    nombresComunes=nombresComunes.filter(x=>x!==n);
    guardarData();
    renderizarTabla();
  } else {
    alert("Selecciona un nombre para eliminar.");
  }
}

function escapeId(s){return String(s).replace(/\s+/g,"_").replace(/[^a-zA-Z0-9_\-]/g,"");}
function escapeJs(s){return String(s).replace(/\\/g,"\\\\").replace(/'/g,"\\'");}

inicializarDatos();
renderizarTabla();
git remote add origin https://github.com/Nicolas2169/Prestamos-Registro.git
git branch -M main
git push -u origin main
</script>

</body>
</html>

