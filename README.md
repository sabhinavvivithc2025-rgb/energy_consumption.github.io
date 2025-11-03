from flask import Flask, render_template_string, request, redirect, url_for
import json, os
from datetime import datetime

app = Flask(__name__)
DATA_FILE = 'data.json'
RATE_PER_KWH = 6.5
LIMIT_FACTOR = 2.5  # kWh per cubic meter (adjustable)

def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, 'r') as f:
            return json.load(f)
    return {}

def save_data(data):
    with open(DATA_FILE, 'w') as f:
        json.dump(data, f, indent=4)

MONTHS = ["January","February","March","April","May","June",
          "July","August","September","October","November","December"]

# -------------------- HTML Templates --------------------

home_html = """
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Smart Energy Tracker</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css">
</head>
<body class="bg-gray-100 py-10 text-center">
  <h1 class="text-3xl font-bold mb-6">⚡ Smart Energy Tracker</h1>
  <a href="/enter" class="bg-blue-500 text-white px-6 py-2 rounded-lg hover:bg-blue-600">Enter Data</a>
  <a href="/dashboard" class="bg-green-500 text-white px-6 py-2 rounded-lg hover:bg-green-600 ml-3">View Dashboard</a>
</body>
</html>
"""

enter_html = """
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Enter Energy Data</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css">
  <style>
    input[type=number]::-webkit-outer-spin-button,
    input[type=number]::-webkit-inner-spin-button {
      -webkit-appearance: none;
      margin: 0;
    }
    input[type=number] { -moz-appearance: textfield; }
  </style>
</head>
<body class="bg-gray-100 py-10">
  <div class="max-w-lg mx-auto bg-white shadow-lg rounded-xl p-6">
    <h2 class="text-2xl font-bold mb-6 text-center">🏠 Enter Energy Data</h2>

    {% if msg %}
      <div class="mb-4 text-red-600 text-sm">{{ msg }}</div>
    {% endif %}

    <form action="/save" method="POST" class="space-y-4">
      <select name="month" class="border p-2 w-full rounded" required>
        <option value="">Select Month</option>
        {% for m in months %}
        <option value="{{ m }}">{{ m }}</option>
        {% endfor %}
      </select>

      <input type="number" step="0.01" name="energy" placeholder="Energy Consumed (kWh)" class="border p-2 w-full rounded" required>
      <input type="number" step="0.01" name="length" placeholder="Room Length (m)" class="border p-2 w-full rounded" required>
      <input type="number" step="0.01" name="width" placeholder="Room Width (m)" class="border p-2 w-full rounded" required>
      <input type="number" step="0.01" name="height" placeholder="Room Height (m)" class="border p-2 w-full rounded" required>

      <button type="submit" class="bg-blue-500 text-white px-4 py-2 rounded-lg hover:bg-blue-600 w-full">Save Data</button>
    </form>

    <div class="text-center mt-6">
      <a href="/" class="text-blue-600 hover:underline">🏠 Back to Home</a>
    </div>
  </div>
</body>
</html>
"""

confirm_html = """
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Confirm Overwrite</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css">
</head>
<body class="bg-gray-100 py-10">
  <div class="max-w-lg mx-auto bg-white shadow-lg rounded-xl p-6 text-center">
    <h2 class="text-xl font-bold mb-4">⚠️ Data for {{ month }} already exists</h2>
    <p class="mb-4">Existing usage: <strong>{{ existing.usage }} kWh</strong> | Bill: ₹{{ existing.bill }}</p>
    <p class="mb-6">Do you want to overwrite it?</p>

    <form action="/save" method="POST" style="display:inline-block;">
      <input type="hidden" name="month" value="{{ month }}">
      <input type="hidden" name="energy" value="{{ new.energy }}">
      <input type="hidden" name="length" value="{{ new.length }}">
      <input type="hidden" name="width" value="{{ new.width }}">
      <input type="hidden" name="height" value="{{ new.height }}">
      <input type="hidden" name="overwrite" value="1">
      <button type="submit" class="bg-red-500 text-white px-4 py-2 rounded-lg hover:bg-red-600 mr-2">Overwrite</button>
    </form>

    <a href="/enter" class="bg-gray-200 px-4 py-2 rounded-lg hover:bg-gray-300">Cancel</a>
  </div>
</body>
</html>
"""

dashboard_html = """
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Dashboard</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css">
</head>
<body class="bg-gray-100 py-10">
  <div class="max-w-5xl mx-auto bg-white shadow-lg rounded-xl p-6">
    <h2 class="text-3xl font-bold mb-6 text-center">📊 Energy Dashboard</h2>

    {% if data %}
    <table class="w-full border mb-6">
      <thead>
        <tr class="bg-gray-200">
          <th class="p-2 border">Month</th>
          <th class="p-2 border">Usage (kWh)</th>
          <th class="p-2 border">Limit (kWh)</th>
          <th class="p-2 border">Bill (₹)</th>
          <th class="p-2 border">Status</th>
          <th class="p-2 border">Saved</th>
          <th class="p-2 border">Action</th>
        </tr>
      </thead>
      <tbody>
        {% for m in months_ordered %}
        <tr>
          <td class="border p-2">{{ m }}</td>
          <td class="border p-2">{{ data[m].usage }}</td>
          <td class="border p-2">{{ data[m].limit }}</td>
          <td class="border p-2">₹{{ data[m].bill }}</td>
          <td class="border p-2 {% if data[m].status == '⚠️ Over Limit' %}text-red-600{% else %}text-green-600{% endif %}">{{ data[m].status }}</td>
          <td class="border p-2">{{ data[m].timestamp }}</td>
          <td class="border p-2 text-center">
            <a href="/delete/{{ m }}" class="text-red-500 hover:underline" onclick="return confirm('Delete data for {{ m }}?')">Delete</a>
          </td>
        </tr>
        {% endfor %}
      </tbody>
    </table>

    <div class="text-center mb-6">
      <a href="/reset" class="bg-red-500 text-white px-6 py-2 rounded-lg hover:bg-red-600"
         onclick="return confirm('Delete all yearly data?')">Delete All Data</a>
    </div>

    <canvas id="usageChart" height="120"></canvas>
    <script>
      const ctx = document.getElementById('usageChart').getContext('2d');
      new Chart(ctx, {
        type: 'bar',
        data: {
          labels: {{ months_ordered | safe }},
          datasets: [{
            label: 'Usage (kWh)',
            data: {{ usage_values | safe }},
            backgroundColor: {{ colors | safe }},
          }]
        },
        options: { scales: { y: { beginAtZero: true } } }
      });
    </script>

    <p class="text-center mt-6 text-lg font-semibold">
      🔼 Max: {{ max_usage }} kWh | 🔽 Min: {{ min_usage }} kWh
    </p>
    {% else %}
    <p class="text-center text-gray-600">No data added yet.</p>
    {% endif %}

    <div class="text-center mt-6">
      <a href="/enter" class="text-blue-600 hover:underline">➕ Add More Data</a>
    </div>
  </div>
</body>
</html>
"""

# -------------------- Routes --------------------

@app.route('/')
def home():
    return render_template_string(home_html)

@app.route('/enter')
def enter():
    return render_template_string(enter_html, months=MONTHS, msg=request.args.get('msg', ''))

@app.route('/save', methods=['POST'])
def save():
    try:
        month = request.form['month']
        energy = float(request.form['energy'])
        length = float(request.form['length'])
        width = float(request.form['width'])
        height = float(request.form['height'])
        overwrite = request.form.get('overwrite', '')

        room_volume = length * width * height
        limit = round(room_volume * LIMIT_FACTOR, 2)

        data = load_data()
        if month in data and not overwrite:
            existing = data[month]
            return render_template_string(confirm_html, month=month, existing=existing,
                                          new={'energy': energy, 'length': length, 'width': width, 'height': height})

        bill = round(energy * RATE_PER_KWH, 2)
        status = "⚠️ Over Limit" if energy > limit else "✅ Within Limit"

        data[month] = {
            "usage": energy,
            "limit": limit,
            "bill": bill,
            "status": status,
            "length": length,
            "width": width,
            "height": height,
            "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        }
        save_data(data)
        return redirect(url_for('dashboard'))

    except ValueError:
        return redirect(url_for('enter', msg="Invalid number entered."))

@app.route('/dashboard')
def dashboard():
    data = load_data()
    months_ordered = [m for m in MONTHS if m in data]
    usage_values = [data[m]['usage'] for m in months_ordered]

    max_usage = max(usage_values) if usage_values else 0
    min_usage = min(usage_values) if usage_values else 0
    colors = []
    for u in usage_values:
        if u == max_usage:
            colors.append("'rgba(211,47,47,0.8)'")
        elif u == min_usage:
            colors.append("'rgba(56,142,60,0.8)'")
        else:
            colors.append("'rgba(54,162,235,0.6)'")

    return render_template_string(dashboard_html, data=data,
                                  months_ordered=months_ordered,
                                  usage_values=usage_values,
                                  max_usage=max_usage,
                                  min_usage=min_usage,
                                  colors="[" + ",".join(colors) + "]")

@app.route('/delete/<month>')
def delete(month):
    data = load_data()
    if month in data:
        del data[month]
        save_data(data)
    return redirect(url_for('dashboard'))

@app.route('/reset')
def reset():
    if os.path.exists(DATA_FILE):
        os.remove(DATA_FILE)
    return redirect(url_for('dashboard'))

if __name__ == '__main__':
    app.run(debug=True)
