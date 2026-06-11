<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simpler Integration Calculator</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.8.0/math.js"></script>
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: #f5f5f7;
            color: #333;
            margin: 0;
            padding: 20px;
            display: flex;
            justify-content: center;
        }
        .container {
            background: #ffffff;
            padding: 25px;
            border-radius: 12px;
            box-shadow: 0 4px 20px rgba(0,0,0,0.08);
            max-width: 900px;
            width: 100%;
        }
        h1 {
            font-size: 24px;
            margin-top: 0;
            color: #1d1d1f;
            text-align: center;
        }
        .calculator-layout {
            display: flex;
            gap: 25px;
            flex-wrap: wrap;
        }
        .controls {
            flex: 1;
            min-width: 280px;
            display: flex;
            flex-direction: column;
            gap: 15px;
        }
        .graph-container {
            flex: 2;
            min-width: 320px;
            position: relative;
            height: 400px;
        }
        .input-group {
            display: flex;
            flex-direction: column;
            gap: 5px;
        }
        label {
            font-weight: 600;
            font-size: 14px;
            color: #666;
        }
        input {
            padding: 10px;
            border: 1px solid #ccc;
            border-radius: 6px;
            font-size: 16px;
            outline: none;
            transition: border-color 0.2s;
        }
        input:focus {
            border-color: #0071e3;
        }
        .bounds-group {
            display: flex;
            gap: 10px;
        }
        .bounds-group .input-group {
            flex: 1;
        }
        .result-box {
            background-color: #f0f7ff;
            border-left: 4px solid #0071e3;
            padding: 15px;
            border-radius: 4px;
            margin-top: 10px;
        }
        .result-title {
            font-size: 14px;
            color: #555;
            margin: 0 0 5px 0;
        }
        .result-value {
            font-size: 22px;
            font-weight: bold;
            color: #0071e3;
            margin: 0;
        }
        .error {
            color: #ff3b30;
            font-size: 14px;
            margin-top: 5px;
            display: none;
        }
    </style>
</head>
<body>

<div class="container">
    <h1>Definite Integration Calculator</h1>
    
    <div class="calculator-layout">
        <div class="controls">
            <div class="input-group">
                <label for="functionInput">Function f(x)</label>
                <input type="text" id="functionInput" value="x^2 - 2x">
                <div id="errorMsg" class="error">Invalid expression</div>
            </div>
            
            <div class="bounds-group">
                <div class="input-group">
                    <label for="lowerBound">Lower Bound (a)</label>
                    <input type="number" id="lowerBound" value="0" step="any">
                </div>
                <div class="input-group">
                    <label for="upperBound">Upper Bound (b)</label>
                    <input type="number" id="upperBound" value="3" step="any">
                </div>
            </div>
            
            <div class="result-box">
                <p class="result-title">Definite Integral Value</p>
                <p class="result-value" id="integralResult">0.00</p>
            </div>
        </div>
        
        <div class="graph-container">
            <canvas id="graphCanvas"></canvas>
        </div>
    </div>
</div>

<script>
    const ctx = document.getElementById('graphCanvas').getContext('2d');
    let chartInstance = null;

    // Numerical integration using Simpson's Rule
    function simpsonsRule(f, a, b, n = 1000) {
        if (n % 2 !== 0) n++; // n must be even
        const h = (b - a) / n;
        let sum = f(a) + f(b);

        for (let i = 1; i < n; i++) {
            const x = a + i * h;
            sum += i % 2 === 0 ? 2 * f(x) : 4 * f(x);
        }
        return (h / 3) * sum;
    }

    function updateCalculator() {
        const exprString = document.getElementById('functionInput').value;
        const a = parseFloat(document.getElementById('lowerBound').value);
        const b = parseFloat(document.getElementById('upperBound').value);
        const errorMsg = document.getElementById('errorMsg');
        const resultValue = document.getElementById('integralResult');

        try {
            // Compile user input using math.js
            const expr = math.compile(exprString);
            const f = (x) => expr.evaluate({ x: x });
            
            // Validate math output on a test number
            if (typeof f(1) !== 'number' || isNaN(f(1))) throw new Error();
            errorMsg.style.display = 'none';

            // 1. Calculate the Integral
            const minBound = Math.min(a, b);
            const maxBound = Math.max(a, b);
            let integral = simpsonsRule(f, minBound, maxBound);
            if (a > b) integral = -integral; // Handle reverse bounds orientation
            
            resultValue.textContent = isNaN(integral) ? "Undefined" : integral.toFixed(4);

            // 2. Generate Graph Data
            // We expand the padding view outside bounds slightly just like Desmos does
            const pad = Math.max((maxBound - minBound) * 0.5, 2);
            const viewMin = minBound - pad;
            const viewMax = maxBound + pad;
            const steps = 200;
            const stepSize = (viewMax - viewMin) / steps;

            const lineData = [];
            const fillData = [];

            for (let i = 0; i <= steps; i++) {
                const x = viewMin + (i * stepSize);
                let y = f(x);
                
                // Keep numbers sane for graphing canvas bounds
                if (!isFinite(y)) y = null; 

                lineData.push({ x: x, y: y });

                // Fill logic: only map data inside the integrated bounds [a, b]
                if (x >= minBound && x <= maxBound) {
                    fillData.push({ x: x, y: y });
                }
            }

            renderChart(lineData, fillData, minBound, maxBound);

        } catch (e) {
            errorMsg.style.display = 'block';
            resultValue.textContent = "Error";
        }
    }

    function renderChart(lineData, fillData, a, b) {
        if (chartInstance) {
            chartInstance.destroy();
        }

        chartInstance = new Chart(ctx, {
            type: 'scatter',
            data: {
                datasets: [
                    {
                        label: 'f(x)',
                        data: lineData,
                        showLine: true,
                        borderColor: '#0071e3',
                        borderWidth: 2.5,
                        pointRadius: 0,
                        fill: false,
                        tension: 0.1
                    },
                    {
                        label: 'Integrated Area',
                        data: fillData,
                        showLine: true,
                        borderColor: 'transparent',
                        pointRadius: 0,
                        backgroundColor: 'rgba(0, 113, 227, 0.2)',
                        fill: 'origin' // Fills to the x-axis (y=0)
                    }
                ]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                    legend: { display: false }
                },
                scales: {
                    x: {
                        type: 'linear',
                        position: 'bottom',
                        grid: { color: '#e5e5e5' },
                        title: { display: true, text: 'X' }
                    },
                    y: {
                        type: 'linear',
                        grid: { color: '#e5e5e5' },
                        title: { display: true, text: 'Y' }
                    }
                }
            }
        });
    }

    // Attach real-time event listeners to all control fields
    document.getElementById('functionInput').addEventListener('input', updateCalculator);
    document.getElementById('lowerBound').addEventListener('input', updateCalculator);
    document.getElementById('upperBound').addEventListener('input', updateCalculator);

    // Initial load execution
    updateCalculator();
</script>

</body>
</html>
