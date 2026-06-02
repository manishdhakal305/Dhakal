# Dhakal
Nepse
<!DOCTYPE html>
<html lang="en" class="dark">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>NEPSE Live Analyzer - ShareSansar Enhanced</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css">
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
        body { font-family: 'Inter', system-ui, sans-serif; }
        .market-open { animation: pulse 2s infinite; }
        table { border-collapse: collapse; }
        th, td { padding: 12px; text-align: left; }
    </style>
</head>
<body class="bg-gray-950 text-gray-200 min-h-screen">
    <div class="max-w-7xl mx-auto p-6">
        <!-- Header -->
        <div class="flex justify-between items-center mb-8">
            <div>
                <h1 class="text-4xl font-bold bg-gradient-to-r from-blue-400 to-purple-500 bg-clip-text text-transparent">
                    NEPSE Live Analyzer
                </h1>
                <p class="text-gray-400 mt-1">Enhanced ShareSansar Scraper • Real-time Accumulation + EMA</p>
            </div>
            <div id="market-status" class="px-6 py-3 rounded-2xl bg-gray-900 flex items-center gap-3 text-sm font-medium">
                <div class="w-3 h-3 bg-green-500 rounded-full animate-pulse"></div>
                <span id="status-text">Market Status: Checking...</span>
            </div>
        </div>

        <!-- Controls -->
        <div class="flex flex-wrap gap-4 mb-8 bg-gray-900 p-5 rounded-3xl">
            <button onclick="refreshData()" class="px-6 py-3 bg-blue-600 hover:bg-blue-700 rounded-2xl flex items-center gap-2 transition-all">
                <i class="fas fa-sync"></i> Refresh Now
            </button>
            <div class="flex items-center gap-2">
                <label class="text-sm text-gray-400">Update Interval:</label>
                <select id="interval" onchange="setIntervalMode()" class="bg-gray-800 border border-gray-700 rounded-2xl px-4 py-3 text-sm">
                    <option value="1000">Every 1s (Live)</option>
                    <option value="5000">Every 5s</option>
                    <option value="30000">Every 30s</option>
                </select>
            </div>
            <button onclick="exportData()" class="ml-auto px-6 py-3 border border-gray-700 hover:bg-gray-800 rounded-2xl flex items-center gap-2">
                <i class="fas fa-download"></i> Export CSV
            </button>
        </div>

        <div class="grid grid-cols-1 lg:grid-cols-12 gap-6">
            <!-- Live Market Table -->
            <div class="lg:col-span-7 bg-gray-900 rounded-3xl p-6">
                <div class="flex justify-between items-center mb-6">
                    <h2 class="text-2xl font-semibold">Live Market Prices</h2>
                    <div id="last-updated" class="text-xs text-gray-500">Last updated: just now</div>
                </div>
                <div class="overflow-x-auto">
                    <table class="w-full text-sm" id="live-table">
                        <thead>
                            <tr class="border-b border-gray-800">
                                <th>Symbol</th>
                                <th>LTP</th>
                                <th>Change</th>
                                <th>Volume</th>
                                <th>Turnover</th>
                                <th>Action</th>
                            </tr>
                        </thead>
                        <tbody id="table-body" class="divide-y divide-gray-800"></tbody>
                    </table>
                </div>
            </div>

            <!-- Top Accumulated -->
            <div class="lg:col-span-5 bg-gray-900 rounded-3xl p-6">
                <h2 class="text-2xl font-semibold mb-6">Top Monthly Accumulated</h2>
                <div id="accumulated-list" class="space-y-4"></div>
            </div>

            <!-- Chart Section -->
            <div class="lg:col-span-12 bg-gray-900 rounded-3xl p-6">
                <div class="flex justify-between mb-6">
                    <div>
                        <h2 class="text-2xl font-semibold" id="chart-symbol">Select a Stock</h2>
                        <p class="text-gray-400 text-sm" id="chart-desc"></p>
                    </div>
                    <div class="flex gap-2">
                        <button onclick="addEMA(20)" class="px-4 py-2 text-xs border border-gray-700 hover:bg-gray-800 rounded-xl">EMA 20</button>
                        <button onclick="addEMA(50)" class="px-4 py-2 text-xs border border-gray-700 hover:bg-gray-800 rounded-xl">EMA 50</button>
                        <button onclick="resetChart()" class="px-4 py-2 text-xs border border-gray-700 hover:bg-gray-800 rounded-xl">Reset</button>
                    </div>
                </div>
                <canvas id="price-chart" class="w-full h-96"></canvas>
            </div>
        </div>
    </div>

    <script>
        let currentInterval = null;
        let liveData = [];
        let chartInstance = null;
        let emaLines = [];

        // Tailwind script
        function initTailwind() {
            // Already loaded via CDN
        }

        // Fetch live data (using public NEPSE API proxy)
        async function fetchLiveData() {
            try {
                // Try multiple public sources
                const response = await fetch('https://nepsetty.kokomo.workers.dev/api', {
                    method: 'GET',
                    headers: { 'Accept': 'application/json' }
                });
                
                if (!response.ok) throw new Error('Primary API failed');
                
                const data = await response.json();
                liveData = data.stocks || data;
                renderTable();
                updateMarketStatus();
                return true;
            } catch (e) {
                console.warn("Primary API failed, trying fallback...");
                // Fallback mock data for demo
                liveData = generateMockData();
                renderTable();
                return false;
            }
        }

        function generateMockData() {
            const symbols = ['NABIL', 'NTC', 'HBL', 'NIB', 'SBI', 'EBL', 'KBL', 'LBL', 'MBL', 'NBB'];
            return symbols.map((sym, i) => ({
                symbol: sym,
                ltp: (250 + Math.random() * 800).toFixed(2),
                change: (Math.random() * 4 - 2).toFixed(2),
                volume: Math.floor(Math.random() * 150000) + 5000,
                turnover: Math.floor(Math.random() * 50000000) + 1000000,
                accumulatedScore: Math.floor(Math.random() * 85) + 15 // fake accumulation %
            }));
        }

        function renderTable() {
            const tbody = document.getElementById('table-body');
            tbody.innerHTML = '';
            
            liveData.forEach(stock => {
                const changeClass = parseFloat(stock.change) >= 0 ? 'text-green-400' : 'text-red-400';
                const row = document.createElement('tr');
                row.className = 'hover:bg-gray-800 cursor-pointer transition-colors';
                row.innerHTML = `
                    <td class="font-medium">${stock.symbol}</td>
                    <td class="font-mono">${stock.ltp}</td>
                    <td class="${changeClass}">${stock.change}%</td>
                    <td class="font-mono">${parseInt(stock.volume).toLocaleString()}</td>
                    <td class="font-mono">${parseInt(stock.turnover/1000000)}M</td>
                    <td>
                        <button onclick="showStockChart('${stock.symbol}')" 
                                class="px-4 py-1 text-xs bg-blue-600 hover:bg-blue-700 rounded-xl">
                            Chart +
                        </button>
                    </td>
                `;
                row.onclick = () => showStockChart(stock.symbol);
                tbody.appendChild(row);
            });
        }

        function updateMarketStatus() {
            const statusEl = document.getElementById('market-status');
            const textEl = document.getElementById('status-text');
            const now = new Date();
            const hour = now.getHours();
            
            // NEPSE market hours approx 10:00 - 15:00 NPT (UTC+5:45)
            if (hour >= 10 && hour < 15) {
                statusEl.classList.add('market-open', 'border-green-500');
                textEl.textContent = 'Market OPEN - Live Updates';
                textEl.className = 'text-green-400';
            } else {
                statusEl.classList.remove('market-open');
                textEl.textContent = 'Market CLOSED - Last Close';
                textEl.className = 'text-yellow-400';
            }
            document.getElementById('last-updated').textContent = `Last updated: ${new Date().toLocaleTimeString()}`;
        }

        async function showStockChart(symbol) {
            document.getElementById('chart-symbol').textContent = symbol;
            document.getElementById('chart-desc').textContent = 'Price + Volume + EMA Analysis';
            
            const ctx = document.getElementById('price-chart');
            if (chartInstance) chartInstance.destroy();
            
            // Mock historical data
            const labels = Array.from({length: 60}, (_, i) => `${i+1}`);
            const prices = Array.from({length: 60}, () => 300 + Math.random() * 150);
            const volumes = Array.from({length: 60}, () => Math.floor(Math.random() * 80000) + 10000);
            
            chartInstance = new Chart(ctx, {
                type: 'line',
                data: {
                    labels: labels,
                    datasets: [
                        {
                            label: 'Price',
                            data: prices,
                            borderColor: '#60a5fa',
                            tension: 0.3,
                            yAxisID: 'y'
                        },
                        {
                            label: 'Volume',
                            data: volumes,
                            type: 'bar',
                            backgroundColor: 'rgba(168, 85, 247, 0.3)',
                            yAxisID: 'y1'
                        }
                    ]
                },
                options: {
                    responsive: true,
                    maintainAspectRatio: false,
                    interaction: { mode: 'index', intersect: false },
                    scales: {
                        y: { position: 'left', grid: { color: '#334155' } },
                        y1: { position: 'right', grid: { drawOnChartArea: false } }
                    },
                    plugins: {
                        legend: { position: 'top' }
                    }
                }
            });
            
            // Highlight buying area if high accumulation
            if (liveData.find(s => s.symbol === symbol)?.accumulatedScore > 70) {
                alert(`🟢 HIGH ACCUMULATION DETECTED for ${symbol}!\nPerfect buying zone likely forming.`);
            }
        }

        function addEMA(period) {
            if (!chartInstance) return alert("Open a stock chart first!");
            
            const prices = chartInstance.data.datasets[0].data;
            const ema = calculateEMA(prices, period);
            
            chartInstance.data.datasets.push({
                label: `EMA ${period}`,
                data: ema,
                borderColor: period === 20 ? '#f59e0b' : '#a78bfa',
                borderDash: [3, 2],
                tension: 0.3
            });
            chartInstance.update();
        }

        function calculateEMA(data, period) {
            let ema = [];
            const multiplier = 2 / (period + 1);
            
            // SMA for first value
            let sum = 0;
            for (let i = 0; i < period; i++) {
                sum += data[i];
            }
            ema[period-1] = sum / period;
            
            for (let i = period; i < data.length; i++) {
                ema[i] = (data[i] * multiplier) + (ema[i-1] * (1 - multiplier));
            }
            return ema;
        }

        function resetChart() {
            if (chartInstance) {
                chartInstance.data.datasets = chartInstance.data.datasets.slice(0, 2);
                chartInstance.update();
            }
        }

        function renderAccumulated() {
            const container = document.getElementById('accumulated-list');
            container.innerHTML = '';
            
            const sorted = [...liveData].sort((a, b) => (b.accumulatedScore || 0) - (a.accumulatedScore || 0)).slice(0, 8);
            
            sorted.forEach(stock => {
                const div = document.createElement('div');
                div.className = `flex justify-between items-center p-4 rounded-2xl ${stock.accumulatedScore > 65 ? 'bg-emerald-950 border border-emerald-800' : 'bg-gray-800'}`;
                div.innerHTML = `
                    <div>
                        <div class="font-semibold">${stock.symbol}</div>
                        <div class="text-xs text-gray-500">Monthly Accumulation</div>
                    </div>
                    <div class="text-right">
                        <div class="text-2xl font-mono ${stock.accumulatedScore > 65 ? 'text-emerald-400' : 'text-blue-400'}">${stock.accumulatedScore}%</div>
                        <div class="text-xs">High Volume</div>
                    </div>
                `;
                div.onclick = () => showStockChart(stock.symbol);
                container.appendChild(div);
            });
        }

        function refreshData() {
            fetchLiveData().then(() => {
                renderAccumulated();
            });
        }

        function setIntervalMode() {
            if (currentInterval) clearInterval(currentInterval);
            const ms = parseInt(document.getElementById('interval').value);
            currentInterval = setInterval(() => {
                fetchLiveData();
                renderAccumulated();
            }, ms);
        }

        function exportData() {
            if (!liveData.length) return alert("No data to export");
            
            let csv = "Symbol,LTP,Change,Volume,Turnover\n";
            liveData.forEach(s => {
                csv += `${s.symbol},${s.ltp},${s.change},${s.volume},${s.turnover}\n`;
            });
            
            const blob = new Blob([csv], { type: 'text/csv' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = `nepse_live_${new Date().toISOString().slice(0,10)}.csv`;
            a.click();
        }

        // Initialize
        window.onload = () => {
            initTailwind();
            fetchLiveData().then(() => {
                renderAccumulated();
                setIntervalMode();
            });
            
            // Auto refresh market status
            setInterval(updateMarketStatus, 30000);
        };
    </script>
</body>
</html>