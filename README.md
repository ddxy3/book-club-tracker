<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Book Club Reading Tracker</title>
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
        Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
      max-width: 480px;
      margin: auto;
      padding: 1rem;
      background: #fefefe;
      color: #222;
    }
    h1 {
      text-align: center;
      margin-bottom: 0.5rem;
    }
    form {
      display: flex;
      gap: 0.5rem;
      margin-bottom: 1rem;
      flex-wrap: wrap;
    }
    label {
      flex: 1 1 40%;
      min-width: 120px;
      font-weight: 600;
      display: flex;
      flex-direction: column;
      font-size: 0.9rem;
    }
    input[type="date"],
    input[type="number"] {
      padding: 0.5rem;
      font-size: 1rem;
      border: 1px solid #ccc;
      border-radius: 4px;
      margin-top: 0.25rem;
    }
    button {
      flex: 1 1 100%;
      padding: 0.6rem;
      font-size: 1rem;
      background-color: #0077cc;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
    button:disabled {
      background-color: #999;
      cursor: not-allowed;
    }
    #status {
      margin: 0.5rem 0;
      font-style: italic;
      color: #555;
      text-align: center;
    }
    canvas {
      max-width: 100%;
      height: 250px;
    }
    footer {
      font-size: 0.8rem;
      text-align: center;
      margin-top: 2rem;
      color: #888;
    }
  </style>
</head>
<body>
  <h1>Reading Tracker</h1>
  <form id="reading-form">
    <label>
      Date
      <input type="date" id="date-input" required />
    </label>
    <label>
      Pages Read
      <input type="number" id="pages-input" min="1" required />
    </label>
    <button type="submit">Add Entry</button>
  </form>
  <div id="status">Loading...</div>
  <canvas id="readingChart"></canvas>
  <footer>Book Club Tracker &mdash; Offline & Online Sync</footer>

  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
  <script>
    const SUPABASE_URL = "https://kglbesztcnnlmjbgialt.supabase.co";
    const SUPABASE_ANON_KEY =
      "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImtnbGJlc3p0Y25ubG1qYmdpYWx0Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NDg3MzYwMzUsImV4cCI6MjA2NDMxMjAzNX0.9kCJa839uBL9HYGHHfUEmiaDRisncPdL-01vIAb_D8c";

    const supabase = supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

    // Generate or get unique member ID stored locally
    function getMemberId() {
      let id = localStorage.getItem("member_id");
      if (!id) {
        id = crypto.randomUUID();
        localStorage.setItem("member_id", id);
      }
      return id;
    }
    const memberId = getMemberId();

    const form = document.getElementById("reading-form");
    const dateInput = document.getElementById("date-input");
    const pagesInput = document.getElementById("pages-input");
    const status = document.getElementById("status");
    const ctx = document.getElementById("readingChart").getContext("2d");

    let readingData = []; // {date: "YYYY-MM-DD", pages: number, synced: bool, id: uuid|null}

    // Load local data
    function loadLocalData() {
      const data = localStorage.getItem("reading_data");
      return data ? JSON.parse(data) : [];
    }
    // Save local data
    function saveLocalData(data) {
      localStorage.setItem("reading_data", JSON.stringify(data));
    }

    // Sync unsynced entries to Supabase
    async function syncData() {
      status.textContent = "Syncing data...";
      const unsynced = readingData.filter((r) => !r.synced);
      for (const entry of unsynced) {
        try {
          const { data, error } = await supabase
            .from("reading_logs")
            .insert([
              {
                member_id: memberId,
                date: entry.date,
                pages: entry.pages,
              },
            ])
            .select()
            .single();

          if (error) {
            console.error("Insert error:", error);
            status.textContent = "Sync error! Will retry later.";
            return;
          }
          // Mark as synced with id from DB
          entry.synced = true;
          entry.id = data.id;
          saveLocalData(readingData);
          status.textContent = "All data synced ✅";
        } catch (err) {
          console.error("Sync failed:", err);
          status.textContent = "Sync error! Will retry later.";
        }
      }
    }

    // Load synced data from backend + merge with local (to show latest)
    async function loadBackendData() {
      status.textContent = "Loading data from server...";
      const { data, error } = await supabase
        .from("reading_logs")
        .select("*")
        .eq("member_id", memberId)
        .order("date", { ascending: true });

      if (error) {
        console.error("Load error:", error);
        status.textContent = "Failed to load server data.";
        return;
      }
      // Merge backend data with local, preferring backend for synced entries
      const backendDates = new Set(data.map((d) => d.date));
      readingData = readingData.filter((entry) => !backendDates.has(entry.date));
      readingData = readingData.concat(
        data.map((d) => ({
          date: d.date,
          pages: d.pages,
          synced: true,
          id: d.id,
        }))
      );
      saveLocalData(readingData);
      status.textContent = "Data loaded successfully.";
      drawChart();
    }

    // Draw Chart.js graph
    let chart;
    function drawChart() {
      const sortedData = [...readingData].sort((a, b) =>
        a.date.localeCompare(b.date)
      );
      const labels = sortedData.map((r) => r.date);
      const pages = sortedData.map((r) => r.pages);

      if (chart) chart.destroy();
      chart = new Chart(ctx, {
        type: "line",
        data: {
          labels,
          datasets: [
            {
              label: "Pages Read",
              data: pages,
              borderColor: "#0077cc",
              backgroundColor: "rgba(0,119,204,0.2)",
              fill: true,
              tension: 0.3,
            },
          ],
        },
        options: {
          responsive: true,
          scales: {
            y: { beginAtZero: true, stepSize: 1 },
          },
          interaction: {
            mode: "nearest",
            axis: "x",
            intersect: false,
          },
        },
      });
    }

    // Handle form submission
    form.addEventListener("submit", (e) => {
      e.preventDefault();
      const date = dateInput.value;
      const pages = parseInt(pagesInput.value);
      if (!date || !pages || pages < 1) return;

      // Check if date already exists locally; if yes, add pages
      const existingIndex = readingData.findIndex((r) => r.date === date);
      if (existingIndex >= 0) {
        readingData[existingIndex].pages += pages;
        readingData[existingIndex].synced = false; // mark as unsynced after update
      } else {
        readingData.push({ date, pages, synced: false, id: null });
      }

      saveLocalData(readingData);
      drawChart();
      form.reset();
      status.textContent = "Entry added. Syncing...";
      syncData();
    });

    // On page load:
    readingData = loadLocalData();
    drawChart();
    loadBackendData();

    // Try syncing data every 30 seconds if online
    setInterval(() => {
      if (navigator.onLine) syncData();
    }, 30000);

  </script>
</body>
</html>
