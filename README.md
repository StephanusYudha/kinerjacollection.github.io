<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BKK Monitoring - Advanced Admin</title>
    
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.1/font/bootstrap-icons.css">
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>

    <style>
        :root { --primary-dark: #003366; --gold: #FFD700; }
        body { background-color: #f4f7f9; font-family: 'Segoe UI', sans-serif; }
        .navbar { background: var(--primary-dark); border-bottom: 3px solid var(--gold); }
        .card { border: none; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.08); }
        #map { height: 400px; border-radius: 12px; z-index: 1; }
        .stat-val { font-size: 1.8rem; font-weight: 800; }
        .table-responsive { max-height: 500px; }
        .filter-box { background: #fff; padding: 15px; border-radius: 12px; margin-bottom: 20px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); }
    </style>
</head>
<body>

<nav class="navbar navbar-dark shadow-sm sticky-top mb-4">
    <div class="container-fluid px-4">
        <span class="navbar-brand fw-bold text-white">
            <i class="bi bi-shield-check me-2 text-warning"></i>MONITORING BKK
        </span>
        <div class="d-flex gap-2">
            <button onclick="exportToExcel()" class="btn btn-success btn-sm px-3 shadow-sm">
                <i class="bi bi-file-earmark-spreadsheet me-1"></i> Export Excel
            </button>
            <button onclick="loadAdminData()" id="refreshBtn" class="btn btn-warning btn-sm px-3 shadow-sm fw-bold">
                <i class="bi bi-arrow-clockwise me-1"></i> REFRESH DATA
            </button>
        </div>
    </div>
</nav>

<div class="container-fluid px-4">
    <div class="filter-box">
        <div class="row g-3 align-items-end">
            <div class="col-md-3">
                <label class="form-label small fw-bold">Pilih Petugas</label>
                <select id="filterPetugas" class="form-select" onchange="applyFilters()">
                    <option value="semua">Semua Petugas</option>
                </select>
            </div>
            <div class="col-md-3">
                <label class="form-label small fw-bold">Dari Tanggal</label>
                <input type="date" id="dateStart" class="form-control" onchange="applyFilters()">
            </div>
            <div class="col-md-3">
                <label class="form-label small fw-bold">Sampai Tanggal</label>
                <input type="date" id="dateEnd" class="form-control" onchange="applyFilters()">
            </div>
            <div class="col-md-3">
                <button class="btn btn-outline-secondary w-100" onclick="resetFilters()">Reset Filter</button>
            </div>
        </div>
    </div>

    <div class="row g-3 mb-4 text-center">
        <div class="col-md-6">
            <div class="card p-3 border-start border-primary border-5">
                <small class="text-muted fw-bold text-uppercase">Total Setoran Terfilter</small>
                <div id="totalRp" class="stat-val text-primary">Rp 0</div>
            </div>
        </div>
        <div class="col-md-6">
            <div class="card p-3 border-start border-success border-5">
                <small class="text-muted fw-bold text-uppercase">Jumlah Laporan Terfilter</small>
                <div id="totalLaporan" class="stat-val text-success">0</div>
            </div>
        </div>
    </div>

    <div class="row g-4 mb-4">
        <div class="col-lg-7">
            <div class="card p-3">
                <h6 class="fw-bold mb-3"><i class="bi bi-geo-alt-fill text-danger me-2"></i>Sebaran Lokasi</h6>
                <div id="map"></div>
            </div>
        </div>
        <div class="col-lg-5">
            <div class="card p-3 h-100">
                <h6 class="fw-bold mb-3"><i class="bi bi-bar-chart-fill text-primary me-2"></i>Performa Per Petugas</h6>
                <div style="height: 320px;"><canvas id="chartPetugas"></canvas></div>
            </div>
        </div>
    </div>

    <div class="card p-4 mb-5">
        <div class="table-responsive">
            <table class="table table-hover align-middle">
                <thead class="table-dark">
                    <tr>
                        <th>Tanggal/Waktu</th>
                        <th>Petugas</th>
                        <th>Nasabah</th>
                        <th>Status</th>
                        <th>Nominal</th>
                        <th class="text-center">Foto</th>
                        <th class="text-center">Lokasi</th>
                    </tr>
                </thead>
                <tbody id="tableBody"></tbody>
            </table>
        </div>
    </div>
</div>

<script>
    const SCRIPT_URL = "https://script.google.com/macros/s/AKfycbzCKxtdESkpIDN07_jiX9-gW0DrQ0wDFHYSILdq4R4zO5UGYX19iXtFszbhtJ4Vx_pL/exec";
    
    let rawData = [];
    let map = L.map('map').setView([-7.0, 110.4], 10);
    let markers = L.layerGroup().addTo(map);
    let chartObj;

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

    async function loadAdminData() {
        const btn = document.getElementById('refreshBtn');
        btn.disabled = true;
        btn.innerHTML = '<span class="spinner-border spinner-border-sm"></span> LOADING...';
        
        try {
            const response = await fetch(SCRIPT_URL + "?action=getAllData");
            const res = await response.json();
            if (res.laporan) {
                rawData = res.laporan;
                updateFilterDropdown(rawData);
                applyFilters(); // Menjalankan filter saat data baru dimuat
            }
        } catch(e) { alert("Gagal mengambil data dari Google Sheets!"); }
        finally { btn.disabled = false; btn.innerHTML = '<i class="bi bi-arrow-clockwise"></i> REFRESH DATA'; }
    }

    function updateFilterDropdown(data) {
        const select = document.getElementById('filterPetugas');
        const listPetugas = [...new Set(data.map(item => item.petugas))];
        select.innerHTML = '<option value="semua">Semua Petugas</option>';
        listPetugas.forEach(p => { select.innerHTML += `<option value="${p}">${p}</option>`; });
    }

    function applyFilters() {
        const selectedPetugas = document.getElementById('filterPetugas').value;
        const startDate = document.getElementById('dateStart').value;
        const endDate = document.getElementById('dateEnd').value;

        let filtered = rawData.filter(item => {
            // Filter Nama
            const matchNama = (selectedPetugas === 'semua' || item.petugas === selectedPetugas);
            
            // Filter Tanggal
            let matchTanggal = true;
            if (item.waktu) {
                const itemDate = new Date(item.waktu.split(',')[0].split('/').reverse().join('-')); // Format penyesuaian
                const start = startDate ? new Date(startDate) : null;
                const end = endDate ? new Date(endDate) : null;
                
                if (start) itemDate.setHours(0,0,0,0), start.setHours(0,0,0,0);
                if (end) end.setHours(23,59,59,999);

                if (start && itemDate < start) matchTanggal = false;
                if (end && itemDate > end) matchTanggal = false;
            }
            
            return matchNama && matchTanggal;
        });

        renderUI(filtered);
    }

    function renderUI(data) {
        let totalUang = 0;
        let htmlTable = "";
        let rekapPetugas = {};
        markers.clearLayers();

        data.slice().reverse().forEach(row => {
            const nom = parseInt(row.nominal) || 0;
            totalUang += nom;
            rekapPetugas[row.petugas] = (rekapPetugas[row.petugas] || 0) + nom;

            if(row.lokasi && row.lokasi !== "0,0") {
                const coord = row.lokasi.split(',');
                L.marker([parseFloat(coord[0]), parseFloat(coord[1])]).addTo(markers)
                    .bindPopup(`<b>${row.petugas}</b><br>${row.nasabah}<br>Rp ${nom.toLocaleString('id-ID')}`);
            }

            htmlTable += `
                <tr>
                    <td><small>${row.waktu}</small></td>
                    <td><span class="fw-bold">${row.petugas}</span></td>
                    <td>${row.nasabah}</td>
                    <td><span class="badge bg-info text-dark">${row.status}</span></td>
                    <td class="fw-bold text-success">Rp ${nom.toLocaleString('id-ID')}</td>
                    <td class="text-center"><img src="${row.foto || 'https://via.placeholder.com/50'}" style="width:40px;height:40px;object-fit:cover;border-radius:5px;cursor:pointer" onclick="window.open(this.src)"></td>
                    <td class="text-center">
                        <a href="https://www.google.com/maps?q=${row.lokasi}" target="_blank" class="btn btn-sm btn-outline-danger">
                            <i class="bi bi-geo-alt"></i>
                        </a>
                    </td>
                </tr>`;
        });

        document.getElementById('totalRp').innerText = "Rp " + totalUang.toLocaleString('id-ID');
        document.getElementById('totalLaporan').innerText = data.length;
        document.getElementById('tableBody').innerHTML = htmlTable || '<tr><td colspan="7" class="text-center">Data tidak ditemukan pada periode ini</td></tr>';
        updateChart(rekapPetugas);
    }

    function updateChart(rekap) {
        const ctx = document.getElementById('chartPetugas').getContext('2d');
        if(chartObj) chartObj.destroy();
        chartObj = new Chart(ctx, {
            type: 'bar',
            data: {
                labels: Object.keys(rekap),
                datasets: [{ label: 'Total Setoran', data: Object.values(rekap), backgroundColor: '#003366', borderRadius: 5 }]
            },
            options: { indexAxis: 'y', responsive: true, maintainAspectRatio: false }
        });
    }

    function resetFilters() {
        document.getElementById('filterPetugas').value = 'semua';
        document.getElementById('dateStart').value = '';
        document.getElementById('dateEnd').value = '';
        applyFilters();
    }

    function exportToExcel() {
        const table = document.getElementById('tableBody');
        if (table.rows.length === 0 || table.rows[0].cells.length < 2) return alert("Tidak ada data untuk di-export");
        const ws = XLSX.utils.json_to_sheet(rawData);
        const wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, "Laporan BKK");
        XLSX.writeFile(wb, `Laporan_BKK_Export.xlsx`);
    }

    window.onload = loadAdminData;
</script>
</body>
</html>
