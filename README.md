<!DOCTYPE html>
<html lang="te">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>స్మార్ట్ బడ్జెట్ క్యాలిక్యులేటర్</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/FileSaver.js/2.0.5/FileSaver.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <style>
        body { font-family: 'Arial', sans-serif; background-color: #f5f5f5; padding: 20px; }
        .container { max-width: 600px; margin: auto; background: white; padding: 20px; border-radius: 10px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        .summary { text-align: center; margin-bottom: 15px; }
        .summary h2 { color: #4CAF50; }
        .input-group { margin-bottom: 10px; }
        input, select, button { width: 100%; padding: 10px; margin-top: 5px; border: 1px solid #ddd; border-radius: 5px; }
        button { background-color: #4CAF50; color: white; border: none; cursor: pointer; }
        button:hover { background-color: #45a049; }
        .entry-item { display: flex; justify-content: space-between; padding: 10px; margin: 5px 0; background: #f8f8f8; border-radius: 5px; }
        .income { border-left: 4px solid #4CAF50; }
        .expense { border-left: 4px solid #f44336; }
        .delete-btn, .edit-btn, .delete-category-btn { color: #888; cursor: pointer; margin-left: 10px; }
        .delete-btn:hover, .edit-btn:hover, .delete-category-btn:hover { color: red; }
        .chart-container { text-align: center; margin-top: 20px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="summary">
            <h2>మీ బడ్జెట్</h2>
            <p>మొత్తం ఆదాయం: ₹<span id="total-income">0</span></p>
            <p>మొత్తం ఖర్చు: ₹<span id="total-expense">0</span></p>
            <p>మిగిలిన మొత్తం: ₹<span id="remaining-budget">0</span></p>
        </div>

        <div class="input-group">
            <label for="category-filter">కేటగిరీ ఫిల్టర్:</label>
            <select id="category-filter">
                <option value="">అన్ని</option>
                <option value="Food">భోజనం</option>
                <option value="Utilities">ఉపకరణాలు</option>
                <option value="Entertainment">వినోదం</option>
                <option value="General">సాధారణ</option>
                <option value="Custom">కస్టమ్</option>
            </select>
        </div>

        <div class="input-group">
            <label for="type">వర్గం:</label>
            <select id="type">
                <option value="income">ఆదాయం</option>
                <option value="expense">ఖర్చు</option>
            </select>
        </div>

        <div class="input-group">
            <label for="category">కేటగిరీ:</label>
            <select id="category">
                <option value="General">సాధారణ</option>
                <option value="Food">భోజనం</option>
                <option value="Utilities">ఉపకరణాలు</option>
                <option value="Entertainment">వినోదం</option>
                <option value="Custom">కస్టమ్</option>
            </select>
        </div>

        <div class="input-group">
            <label for="name">పేరు:</label>
            <input type="text" id="name">
        </div>

        <div class="input-group">
            <label for="amount">మొత్తం (₹):</label>
            <input type="number" id="amount">
        </div>

        <div class="input-group">
            <label for="note">గమనిక (Optional):</label>
            <input type="text" id="note">
        </div>

        <div class="input-group">
            <label for="custom-category">కొత్త కస్టమ్ కేటగిరీ జోడించు:</label>
            <input type="text" id="custom-category" placeholder="కొత్త కేటగిరీ పేరు">
            <button onclick="addCustomCategory()">జోడించు</button>
        </div>

        <button onclick="addEntry()">జోడించు</button>

        <div id="entry-list"></div>

        <div class="chart-container">
            <canvas id="chart"></canvas>
        </div>

        <button onclick="exportData()">CSV ఎగుమతి చేయండి</button>
        <button onclick="saveToPDF()">PDFగా సేవ్ చేయండి</button>
        <button onclick="clearAllEntries()">అన్ని ఎంట్రీలు క్లియర్ చేయండి</button>

        <!-- Custom Category List with Delete -->
        <div>
            <h3>కస్టమ్ కేటగిరీలు:</h3>
            <ul id="custom-category-list"></ul>
        </div>
    </div>

    <script>
        let budgetData = JSON.parse(localStorage.getItem('budgetData')) || { income: [], expenses: [] };
        let customCategories = JSON.parse(localStorage.getItem('customCategories')) || [];
        const defaultCategories = ["Food", "Utilities", "Entertainment", "General"];
        let chartInstance = null;

        function saveData() {
            localStorage.setItem('budgetData', JSON.stringify(budgetData));
            localStorage.setItem('customCategories', JSON.stringify(customCategories));
        }

        function addCustomCategory() {
            const customCategory = document.getElementById('custom-category').value.trim();
            if (customCategory && !customCategories.includes(customCategory) && !defaultCategories.includes(customCategory)) {
                customCategories.push(customCategory);
                saveData();
                updateCategoryList();
                document.getElementById('custom-category').value = '';
            } else {
                alert('ఈ కేటగిరీ ఇప్పటికే ఉంది లేదా ఖాళీగా ఉంది.');
            }
        }

        function updateCategoryList() {
            const categorySelect = document.getElementById('category');
            const filterSelect = document.getElementById('category-filter');
            const customCategoryList = document.getElementById('custom-category-list');

            // Add default categories and custom categories dynamically
            const allCategories = [...defaultCategories, ...customCategories];
            const options = allCategories.map(category => `<option value="${category}">${category}</option>`).join('');
            categorySelect.innerHTML = options;
            filterSelect.innerHTML = `
                <option value="">అన్ని</option>
                ${allCategories.map(category => `<option value="${category}">${category}</option>`).join('')}
            `;

            // Display custom categories with delete option
            customCategoryList.innerHTML = customCategories.map(category => `
                <li>
                    ${category} 
                    <span class="delete-category-btn" onclick="deleteCategory('${category}')">❌</span>
                </li>
            `).join('');
        }

        function deleteCategory(category) {
            if (defaultCategories.includes(category)) {
                alert('డిఫాల్ట్ కేటగిరీలను మీరు తొలగించలేరు.');
            } else {
                customCategories = customCategories.filter(item => item !== category);
                saveData();
                updateCategoryList();
            }
        }

        function addEntry() {
            const type = document.getElementById('type').value;
            const category = document.getElementById('category').value;
            const name = document.getElementById('name').value.trim();
            const amount = parseFloat(document.getElementById('amount').value);
            const note = document.getElementById('note').value.trim();
            const date = new Date().toLocaleString();

            if (!name || isNaN(amount) || amount <= 0) {
                alert('దయచేసి సరైన వివరాలు నమోదు చేయండి');
                return;
            }

            const entry = { 
                id: Date.now(), 
                name, 
                amount, 
                type, 
                category, 
                note, 
                date 
            };

            if (type === 'income') {
                budgetData.income.push(entry);
            } else {
                budgetData.expenses.push(entry);
            }

            saveData();
            updateUI();
        }

        function deleteEntry(id, type) {
            if (type === 'income') {
                budgetData.income = budgetData.income.filter(entry => entry.id !== id);
            } else {
                budgetData.expenses = budgetData.expenses.filter(entry => entry.id !== id);
            }
            saveData();
            updateUI();
        }

        function editEntry(id, type) {
            const entry = (type === 'income' ? budgetData.income : budgetData.expenses).find(e => e.id === id);
            document.getElementById('name').value = entry.name;
            document.getElementById('amount').value = entry.amount;
            document.getElementById('note').value = entry.note;
            document.getElementById('category').value = entry.category;
            document.getElementById('type').value = entry.type;
        }

        function calculateTotals() {
            const totalIncome = budgetData.income.reduce((sum, entry) => sum + entry.amount, 0);
            const totalExpense = budgetData.expenses.reduce((sum, entry) => sum + entry.amount, 0);
            return { totalIncome, totalExpense, remaining: totalIncome - totalExpense };
        }

        function updateUI() {
            const list = document.getElementById('entry-list');
            const categoryFilter = document.getElementById('category-filter').value;

            const filteredEntries = [...budgetData.income, ...budgetData.expenses].filter(entry => 
                !categoryFilter || entry.category === categoryFilter
            );

            list.innerHTML = filteredEntries.map(entry => `
                <div class="entry-item ${entry.type}">
                    <span>${entry.name} - ₹${entry.amount.toFixed(2)} (${entry.category})</span>
                    <span>${entry.date}</span>
                    <span class="edit-btn" onclick="editEntry(${entry.id}, '${entry.type}')">✎</span>
                    <span class="delete-btn" onclick="deleteEntry(${entry.id}, '${entry.type}')">🗑️</span>
                </div>
            `).join('');

            const totals = calculateTotals();
            document.getElementById('total-income').textContent = totals.totalIncome.toFixed(2);
            document.getElementById('total-expense').textContent = totals.totalExpense.toFixed(2);
            document.getElementById('remaining-budget').textContent = totals.remaining.toFixed(2);

            updateChart();
        }

        function updateChart() {
            const totals = calculateTotals();
            if (chartInstance) {
                chartInstance.destroy();
            }
            const ctx = document.getElementById('chart').getContext('2d');
            chartInstance = new Chart(ctx, {
                type: 'pie',
                data: {
                    labels: ['ఆదాయం', 'ఖర్చు', 'మిగిలిన మొత్తం'],
                    datasets: [{
                        data: [totals.totalIncome, totals.totalExpense, totals.remaining],
                        backgroundColor: ['#4CAF50', '#f44336', '#FFC107'],
                    }]
                },
                options: {
                    responsive: true,
                    plugins: {
                        legend: { position: 'top' },
                        tooltip: { callbacks: { label: (tooltipItem) => `₹${tooltipItem.raw.toFixed(2)}` } }
                    }
                }
            });
        }

        document.getElementById('category-filter').addEventListener('change', updateUI);
        updateCategoryList();
        updateUI();
    </script>
</body>
</html>
