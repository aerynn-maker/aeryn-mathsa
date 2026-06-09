<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Definite Integration Calculator & Visualizer</title>
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.8.0/math.js"></script>
    <style>
        body {
            font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
        }
    </style>
</head>
<body class="bg-slate-900 text-slate-100 min-h-screen flex flex-col md:flex-row">

    <main class="w-full md:w-96 bg-slate-800 p-6 flex flex-col justify-between border-b md:border-b-0 md:border-r border-slate-700 shadow-xl z-10">
        <div>
            <div class="mb-6">
                <h1 class="text-xl font-bold tracking-tight text-emerald-400">∫ Definite Integrator</h1>
                <p class="text-xs text-slate-400 mt-1">Visualize and calculate the area under a curve.</p>
            </div>

            <div class="space-y-4">
                <div>
                    <label class="block text-xs font-semibold uppercase tracking-wider text-slate-400 mb-1">Function f(x)</label>
                    <div class="relative">
                        <span class="absolute left-3 top-2 text-slate-500 font-mono">f(x) =</span>
                        <input id="function-input" type="text" value="x^2 - 2x + 1" 
                            class="w-full bg-slate-950 border border-slate-700 rounded-lg py-2 pl-14 pr-3 text-emerald-300 font-mono focus:outline-none focus:border-emerald-500 transition shadow-inner">
                    </div>
                    <p id="error-msg" class="text-xs text-rose-400 mt-1 hidden">Invalid mathematical expression.</p>
                </div>

                <div class="grid grid-cols-2 gap-4">
                    <div>
                        <label class="block text-xs font-semibold uppercase tracking-wider text-slate-400 mb-1">Lower Bound (a)</label>
                        <input id="bound-a" type="number" value="0" step="0.5"
                            class="w-full bg-slate-950 border border-slate-700 rounded-lg p-2 font-mono text-center focus:outline-none focus:border-emerald-500 transition">
                    </div>
                    <div>
                        <label class="block text-xs font-semibold uppercase tracking-wider text-slate-400 mb-1">Upper Bound (b)</label>
                        <input id="bound-b" type="number" value="3" step="0.5"
                            class="w-full bg-slate-950 border border-slate-700 rounded-lg p-2 font-mono text-center focus:outline-none focus:border-emerald-500 transition">
                    </div>
                </div>

                <div>
                    <label class="block text-xs font-semibold uppercase tracking-wider text-slate-400 mb-1">
                        Riemann Intervals: <span id="intervals-val" class="text-emerald-400 font-mono">100</span>
                    </label>
                    <input id="intervals-input" type="range" min="10" max="500" step="10" value="100" 
                        class="w-full accent-emerald-500 cursor-pointer bg-slate-950 rounded-lg appearance-none h-2">
                    <div class="flex justify-between text-[10px] text-slate-500 px-1 mt-0.5">
                        <span>Fast (10)</span>
                        <span>Precise (500)</span>
                    </div>
                </div>
            </div>
        </div>

        <div class="mt-8 bg-slate-950 border border-slate-700 rounded-xl p-4 shadow-inner">
            <div class="text-xs uppercase tracking-wider text-slate-500 font-semibold mb-1">Calculated Area</div>
            <div class="flex items-baseline space-x-2 font-mono">
                <span class="text-slate-400 text-lg">∫</span>
                <span class="text-2xl font-bold text-emerald-400" id="result-display">3.0000</span>
            </div>
            <div class="text-[10px] text-slate-500 mt-2 italic">Computed using Simpson's Rule.</div>
        </div>
    </main>

    <section class="flex-1 relative p-4 md:p-8 flex items-center justify-center min-h-[400px]">
        <div class="w-full h-full max-w-4xl bg-slate-950 rounded-2xl border border-slate-800 p-4 shadow-2xl relative">
            <canvas id="graphCanvas"></canvas>
        </div>
    </section>

    <script>
        // DOM Element Cache
        const fnInput = document.getElementById('function-input');
        const boundAInput = document.getElementById('bound-a');
        const boundBInput = document.getElementById('bound-b');
        const intervalsInput = document.getElementById('intervals-input');
        const intervalsVal = document.getElementById('intervals-val');
        const resultDisplay = document.getElementById('result-display');
        const errorMsg = document.getElementById('error-msg');
        const ctx = document.getElementById('graphCanvas').getContext('2d');

        let chart;

        // Numerical Integration using Simpson's Rule for higher precision
        function calculateIntegral(f, a, b, n) {
            if (n % 2 !== 0) n++; // Simpson's rule requires an even number of intervals
            const h = (b - a) / n;
            let sum = f(a) + f(b);

            for (let i = 1; i < n; i++) {
                const x = a + i * h;
                sum += i % 2 === 0 ? 2 * f(x) : 4 * f(x);
            }
            return (h / 3) * sum;
        }

        // Main Draw and Calculate Cycle
        function updateGraph() {
            const exprString = fnInput.value;
            const a = parseFloat(boundAInput.value);
            const b = parseFloat(boundBInput.value);
            const intervals = parseInt(intervalsInput.value);

            intervalsVal.textContent = intervals;

            try {
                // Compile the math expression safely using math.js
                const expr = math.compile(exprString);
                const f = (x) => expr.evaluate({ x: x });

                // Test evaluation to flag syntax errors early
                f(a);
                errorMsg.classList.add('hidden');
                fnInput.classList.remove('border-rose-500');

                // Dynamic viewport padding beyond bounds [a, b]
                const margin = Math.max(1, Math.abs(b - a) * 0.4);
                const minX = Math.min(a, b) - margin;
                const maxX = Math.max(a, b) + margin;
                const totalPoints = 200; 

                const lineData = [];
                const fillData = [];

                // Generate points across the graph view window
                for (let i = 0; i <= totalPoints; i++) {
                    const x = minX + (i / totalPoints) * (maxX - minX);
                    let y;
                    try {
                        y = f(x);
                    } catch(e) { y = null; }

                    if (typeof y === 'number' && !isNaN(y) && isFinite(y)) {
                        lineData.push({ x, y });
                        
                        // Capture data specifically inside the integral boundaries [a, b]
                        if (x >= Math.min(a, b) && x <= Math.max(a, b)) {
                            fillData.push({ x, y });
                        }
                    }
                }

                // Numerical Integration computation
                const area = calculateIntegral(f, a, b, intervals);
                resultDisplay.textContent = isNaN(area) ? "Undefined" : area.toFixed(4);

                // Re-render Chart instance
                if (chart) chart.destroy();

                chart = new Chart(ctx, {
                    type: 'line',
                    data: {
                        datasets: [
                            {
                                label: `f(x) = ${exprString}`,
                                data: lineData,
                                borderColor: '#10b981', // emerald-500
                                borderWidth: 3,
                                pointRadius: 0,
                                fill: false,
                                order: 1
                            },
                            {
                                label: 'Integrated Area',
                                data: fillData,
                                borderColor: 'transparent',
                                backgroundColor: 'rgba(16, 185, 129, 0.25)', // Shaded integration region
                                pointRadius: 0,
                                fill: 'origin', // Fills dynamically to the x-axis zero line
                                order: 2
                            }
                        ]
                    },
                    options: {
                        responsive: true,
                        maintainAspectRatio: false,
                        scales: {
                            x: {
                                type: 'linear',
                                position: 'bottom',
                                grid: { color: '#334155' }, // slate-700
                                ticks: { color: '#94a3b8' }  // slate-400
                            },
                            y: {
                                type: 'linear',
                                grid: { color: '#334155' },
                                ticks: { color: '#94a3b8' }
                            }
                        },
                        plugins: {
                            legend: { display: false } // Custom sidebar UI replaces default legends
                        },
                        animation: { duration: 150 } // Smooth quick updates on slider/input changes
                    }
                });

            } catch (err) {
                // Handle parsing errors smoothly without crashing UI
                errorMsg.classList.remove('hidden');
                fnInput.classList.add('border-rose-500');
            }
        }

        // Attach interactive event listeners
        [fnInput, boundAInput, boundBInput, intervalsInput].forEach(element => {
            element.addEventListener('input', updateGraph);
        });

        // Initialize setup on load
        window.addEventListener('DOMContentLoaded', updateGraph);
    </script>
</body>
</html>
