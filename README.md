<!DOCTYPE html>
<html lang="vi">
<head>
<meta charset="UTF-8">
<title>Enterprise Mini WMS</title>
<meta name="viewport" content="width=device-width, initial-scale=1">
<script src="https://unpkg.com/html5-qrcode"></script>
<script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>
<style>
body{margin:0;font-family:Segoe UI;background:#eef2f5}
header{background:#003366;color:#fff;padding:15px}
.container{max-width:1200px;margin:auto;padding:20px}
.card{background:#fff;border-radius:8px;padding:20px;margin-bottom:20px;box-shadow:0 4px 10px rgba(0,0,0,.08)}
h2{border-left:6px solid #003366;padding-left:10px}
input,select,button{padding:10px;width:100%;margin-top:8px}
button{background:#003366;color:white;border:none;border-radius:5px;cursor:pointer}
button:hover{background:#002244}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(180px,1fr));gap:10px}
table{width:100%;border-collapse:collapse;margin-top:10px}
th,td{border:1px solid #ccc;padding:8px;text-align:center}
th{background:#f0f4f8}
.hidden{display:none}
.map{display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:12px}
.bin{background:#cce5ff;padding:12px;border-radius:6px;text-align:center}
.highlight{background:#ffd966 !important}
.result{white-space:pre-line;font-weight:600;margin-top:10px}
</style>
</head>

<body>

<header>
<h1>🏭 ENTERPRISE MINI WMS</h1>
<p>EAN → SKU • Tra cứu vị trí • Bản đồ kho</p>
</header>

<div class="container">

<!-- LOGIN -->
<div class="card" id="login">
<h2>👤 Đăng nhập</h2>

<select id="role">
<option value="admin">Admin</option>
<option value="staff">Nhân viên</option>
</select>

<input id="user" placeholder="Tài khoản">
<input id="pass" type="password" placeholder="Mật khẩu">

<button onclick="doLogin()">Đăng nhập</button>
<p id="err" style="color:red"></p>
</div>

<div id="app" class="hidden">
<div class="card">
  <h2>📊 Tổng quan kho</h2>
  <div class="grid">
    <div><b>Tổng SKU</b><br><span id="totalSKU">0</span></div>
    <div><b>Tổng tồn</b><br><span id="totalQty">0</span></div>
    <div><b>Số vị trí</b><br><span id="totalLoc">0</span></div>
  </div>
</div>
<!-- SCAN -->
<div class="card">
<h2>📷 Quét mã EAN</h2>
<div id="reader"></div>
<input id="ean" placeholder="EAN được quét tự động">
</div>

<!-- NHẬP KHO -->
<div class="card admin">
<h2>📥 Nhập kho</h2>
<div class="grid">
<input id="name" placeholder="Tên sản phẩm">
<input id="qty" type="number" placeholder="Số lượng">
<input id="mfgDate" type="date" placeholder="Ngày nhập">
<input id="expDate" type="date" placeholder="Hạn sử dụng">
<input id="zone" placeholder="Zone (Z1)">
<input id="aisle" placeholder="Aisle (A01)">
<input id="rack" placeholder="Rack (R01)">
</div>
<button onclick="importGoods()">Xác nhận nhập kho</button>
</div>

<!-- TRA CỨU -->
<div class="card">
<h2>🔎 Tra cứu vị trí hàng hóa</h2>
<input id="searchEAN" placeholder="Nhập hoặc quét EAN">
<button onclick="searchLocation()">Tra cứu</button>
<div class="result" id="result"></div>
</div>

<!-- XUẤT KHO -->
<div class="card">
<h2>📤 Xuất kho</h2>
<input id="eanOut" placeholder="Quét EAN tại vị trí">
<input id="qtyOut" type="number" placeholder="Số lượng xuất">
<button onclick="exportGoods()">Xác nhận xuất kho</button>
</div>

<!-- BẢN ĐỒ KHO -->
<div class="card">
<h2>📍 Bản đồ kho</h2>
<div class="map" id="map"></div>
</div>

<!-- TỒN KHO -->
<div class="card">
<h2>📊 Tồn kho</h2>
<button id="btnExportExcel" onclick="exportExcel()">📥 Xuất file Excel</button>
<table>
<thead>
<tr>
<th>EAN</th>
<th>SKU</th>
<th>Sản phẩm</th>
<th>Qty</th>
<th>Vị trí</th>
</tr>
</thead>
<tbody id="table"></tbody>
</table>
</div>

</div>
</div>

<script>
let role="";
let stock = JSON.parse(localStorage.getItem("stock")) || [];
let productMaster = JSON.parse(localStorage.getItem("productMaster")) || [];
let highlightLocations = [];

/* LOGIN */
function doLogin(){
  role = document.getElementById("role").value;
  let user = document.getElementById("user").value;
  let pass = document.getElementById("pass").value;

  if (
    (role === "admin" && user === "admin" && pass === "123") ||
    (role === "staff" && user === "staff" && pass === "123")
  ) {
    document.getElementById("login").classList.add("hidden");
    document.getElementById("app").classList.remove("hidden");

    if(role !== "admin"){
      document.querySelectorAll(".admin").forEach(e=>e.remove());
    }
if(role !== "admin"){
  document.getElementById("btnExportExcel")?.remove();
}

    ean = document.getElementById("ean");
    eanOut = document.getElementById("eanOut");
    searchEAN = document.getElementById("searchEAN");
    qtyOut = document.getElementById("qtyOut");
    nameInput = document.getElementById("name");
    qtyInput = document.getElementById("qty");
    zone = document.getElementById("zone");
    aisle = document.getElementById("aisle");
    rack = document.getElementById("rack");
    table = document.getElementById("table");
    map = document.getElementById("map");
    result = document.getElementById("result");

    startScan();
    render();
  } else {
    document.getElementById("err").innerText = "❌ Sai tài khoản hoặc mật khẩu";
  }
}

/* SCAN */
function startScan(){
  new Html5Qrcode("reader").start(
    { facingMode: "environment" },
    { fps: 10, qrbox: 250 },
    txt => {
      ean.value = txt;
      eanOut.value = txt;
      searchEAN.value = txt;
    }
  );
}

/* CORE: EAN → SKU */
function getOrCreateSKU(eanCode){
  let product = productMaster.find(p => p.ean === eanCode);
  if(!product){
    product = { ean: eanCode, sku: "SKU-" + eanCode.slice(-6) };
    productMaster.push(product);
    localStorage.setItem("productMaster", JSON.stringify(productMaster));
  }
  return product.sku;
}

/* NHẬP */
function importGoods(){
  let sku = getOrCreateSKU(ean.value);
  let location = `${zone.value}-${aisle.value}-${rack.value}`;
  let item = stock.find(i => i.sku === sku && i.location === location);

  if(item) item.qty += +qty.value;
  else stock.push({
  ean: ean.value,
  sku,
  name: name.value,
  qty: +qty.value,
  location,
  mfgDate: mfgDate.value || new Date().toISOString().slice(0,10),
  expDate: expDate.value || null
});

  save();
}

/* TRA CỨU */
function searchLocation(){
  highlightLocations = [];
  let sku = getOrCreateSKU(searchEAN.value);
  let items = stock.filter(i => i.sku === sku && i.qty > 0);

  if(items.length === 0){
    result.innerText = "❌ Không tìm thấy hàng trong kho";
    render();
    return;
  }

  let text = "📍 Vị trí lưu kho:\n";
  items.forEach(i=>{
    text += `- ${i.location} | Qty: ${i.qty}\n`;
    highlightLocations.push(i.location);
  });

  result.innerText = text;
  render();
}

/* XUẤT */
function exportGoods(){
  let sku = getOrCreateSKU(eanOut.value);
  let qtyToExport = +qtyOut.value;

  if(qtyToExport <= 0){
    alert("Số lượng xuất không hợp lệ");
    return;
  }

  let lots = stock.filter(i => i.sku === sku && i.qty > 0);

  if(lots.length === 0){
    alert("Không có tồn kho cho SKU này");
    return;
  }

  lots.sort((a,b)=>{
    if(a.expDate && b.expDate){
      return new Date(a.expDate) - new Date(b.expDate);
    }
    return new Date(a.mfgDate) - new Date(b.mfgDate);
  });

  let remain = qtyToExport;

  for(let lot of lots){
    if(remain <= 0) break;

    let deduct = Math.min(lot.qty, remain);
    lot.qty -= deduct;
    remain -= deduct;
  }

  if(remain > 0){
    alert("❌ Không đủ tồn kho để xuất");
    return;
  }

  save();
  alert(`✅ Xuất ${qtyToExport} ${sku} theo FIFO/FEFO thành công`);
}

/* SAVE & RENDER */
function save(){
  localStorage.setItem("stock", JSON.stringify(stock));
  render();
}

function render(){
  table.innerHTML="";
  map.innerHTML="";

  stock.forEach(i=>{
    table.innerHTML += `
      <tr>
        <td>${i.ean}</td>
        <td>${i.sku}</td>
        <td>${i.name}</td>
        <td>${i.qty}</td>
        <td>${i.location}</td>
      </tr>
    `;

    let highlight = highlightLocations.includes(i.location) ? "highlight" : "";
    map.innerHTML += `
      <div class="bin ${highlight}">
        <b>${i.location}</b><br>
        ${i.sku}<br>
        Qty: ${i.qty}
      </div>
    `;
  });
document.getElementById("totalSKU").innerText =
    new Set(stock.map(i => i.sku)).size;

  document.getElementById("totalQty").innerText =
    stock.reduce((a, b) => a + b.qty, 0);

  document.getElementById("totalLoc").innerText =
    new Set(stock.map(i => i.location)).size;
}
function exportExcel(){
  if(role !== "admin"){
    alert("Chỉ Admin mới được xuất file Excel");
    return;
  }

  const data = stock.map(i => ({
    EAN: i.ean,
    SKU: i.sku,
    "Tên sản phẩm": i.name,
    "Số lượng": i.qty,
    "Vị trí": i.location
  }));
const totalQty = stock.reduce((a,b)=>a+b.qty,0);
  data.push({
    EAN: "",
    SKU: "",
    "Tên sản phẩm": "TỔNG CỘNG",
    "Số lượng": totalQty,
    "Vị trí": ""
  });

  const ws = XLSX.utils.json_to_sheet(data);
  const wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb, ws, "TonKho");

  const today = new Date().toISOString().slice(0,10);
  XLSX.writeFile(wb, "ton_kho_WMS.xlsx");
}
</script>

</body>
</html>

