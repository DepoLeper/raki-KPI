<!DOCTYPE html>
<html>
<head>
    <title>Raktári KPI Dashboard</title>
    <meta charset="UTF-8">
    <script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            background-color: #f0f2f5;
            color: #333;
        }
        h1 {
            color: #2c3e50;
            margin-bottom: 20px;
        }
        #gauge_div {
            width: 450px; /* Növelt méret a jobb láthatóságért */
            height: 450px;
            box-shadow: 0 4px 12px rgba(0,0,0,0.15); /* Finomabb árnyék */
            border-radius: 10px; /* Lekerekített sarkok */
            background-color: #ffffff; /* Háttér a gauge-nak */
            padding:10px;
        }
        #info_div {
            text-align: center;
            margin-top: 25px;
            font-size: 1.6em; /* Nagyobb betűméret */
            color: #555;
            font-weight: 500; /* Kicsit vastagabb betű */
        }
        #current_value_display, #target_value_display {
            font-weight: 600; /* Még vastagabb az értékeknek */
            color: #2c3e50;
        }
        #error_div {
            color: #e74c3c; /* Jól látható piros a hibáknak */
            margin-top: 15px;
            text-align: center;
            font-weight: bold;
            border: 1px solid #e74c3c;
            padding: 10px;
            border-radius: 5px;
            background-color: #fbeae5;
            max-width: 600px; /* Ne legyen túl széles a hibaüzenet */
        }
        /* Stílus a betöltési animációnak (opcionális) */
        #gauge_div[data-loading="true"]::before {
            content: "Adatok betöltése...";
            position: absolute;
            left: 50%;
            top: 50%;
            transform: translate(-50%, -50%);
            font-size: 1.2em;
            color: #7f8c8d;
        }
    </style>
</head>
<body>
    <h1>Raktári Teljesítmény</h1>
    <div id="gauge_div" data-loading="true"></div>
    <div id="info_div">
        <span id="current_value_display_label">Aktuális: </span><span id="current_value_display">Betöltés...</span> /
        <span id="target_value_display_label">Cél: </span><span id="target_value_display">Betöltés...</span>
    </div>
    <div id="error_div"></div>

    <script type="text/javascript">
        // --- Google Charts Setup ---
        google.charts.load('current', {'packages':['gauge']});
        google.charts.setOnLoadCallback(initializeDashboard);

        // --- Configuration ---
        // FONTOS: Ide illeszd be a clouderp.hu-s teljes URL-t!
const DATA_SOURCE_URL = 'https://app.clouderp.hu/api/1/automatism/file-share/?s=Z0FBQUFBQm9JNFh1QmhYOENtaWpsNzJRd1Z5MURsNXNVRzc3emJNbU0zMUZQWTdWQkp5Z3d4TDFyVktCbEhyRnRrQ1NzVnF0Zk9MVnp6clNMaE9IMGRfWldsbmh6NzV5Y3ptMDNvMGlVZnJub2xPZlZNY0lEc0k9'; // <<<--- CSERÉLD ERRE A VALÓDI URL-T!
const FIXED_TARGET_VALUE = 300; // Fix célérték

        let gaugeChart;
        let gaugeDataTable;
        let gaugeOptions;
        const refreshIntervalMilliseconds = 5 * 60 * 1000; // 5 perc

        function initializeDashboard() {
         const gaugeElement = document.getElementById('gauge_div');
         // ... (a gaugeDataTable és gaugeOptions definíciója változatlan marad) ...

         gaugeChart = new google.visualization.Gauge(gaugeElement);
         gaugeChart.draw(gaugeDataTable, gaugeOptions); // Első rajzolás

         fetchDataFromSource(); // <<-- ITT VÁLTOZOTT A NÉV
         setInterval(fetchDataFromSource, refreshIntervalMilliseconds); // <<-- ÉS ITT IS
     }

            gaugeOptions = {
                width: 450, height: 450,

                // Új színtéma: Semlegesből a megadott mélylila (#502785) felé
                redColor: '#E0E0E0',    // Világosszürke - ez jelöli a még "üres", célértéktől távoli részt (pl. 0-40% tartomány)
                yellowColor: '#9575CD', // Közepes lila (pl. Material Design "Deep Purple 300") - ez jelzi a haladást (pl. 40-80% tartomány)
                greenColor: '#502785',   // A megadott mélylila - ez jelzi a cél elérését vagy túlteljesítését (pl. 80-100%+ tartomány)

                // Ezen színzónák határértékei (ezeket a fetchDataAndUpdateChart funkció dinamikusan állítja be
                // a célérték alapján, pl. 0-40%, 40-80%, 80-100% bontásban)
                redFrom: 0, redTo: 0,       // Kezdeti érték, dinamikusan frissül
                yellowFrom: 0, yellowTo: 0, // Kezdeti érték, dinamikusan frissül
                greenFrom: 0, greenTo: 0,   // Kezdeti érték, dinamikusan frissül
                
                minorTicks: 0, // Eltávolítja a kisebb osztásjeleket a letisztultabb kinézetért
                
                max: 100, // Kezdeti maximum, ez is dinamikusan frissül az adatokból
                animation: {
                    duration: 1000, // Animáció hossza ezredmásodpercben
                    easing: 'out',  // Animáció típusa
                },
                chartArea: { width: '90%', height: '90%' }, // A diagram területe a rendelkezésre álló helyet használja
                
                // A mutató közepén megjelenő érték szövegének stílusa
                textStyle: { 
                    color: '#502785', // Az érték szövegének színe megegyezik a cél mélylila színével
                    fontSize: '28px', // Betűméret igény szerint
                    bold: true        // Félkövér stílus
                }
            };

            gaugeChart = new google.visualization.Gauge(gaugeElement);
            gaugeChart.draw(gaugeDataTable, gaugeOptions);

            fetchDataAndUpdateChart();
            setInterval(fetchDataAndUpdateChart, refreshIntervalMilliseconds);
        

        async function fetchDataFromSource() { // Figyelem: a funkció neve megváltozott!
         const errorDiv = document.getElementById('error_div');
         errorDiv.textContent = ''; // Korábbi hibaüzenetek törlése

         if (DATA_SOURCE_URL === 'IDE_MASOLD_A_TELJES_UJ_URL_CIMET' || DATA_SOURCE_URL.trim() === '') {
             const errorMsg = "Nincs beállítva az adatforrás URL! Kérlek, add meg a DATA_SOURCE_URL változó értékét a kódban.";
             console.error(errorMsg);
             errorDiv.textContent = errorMsg;
             document.getElementById('current_value_display').textContent = "Hiba";
             document.getElementById('target_value_display').textContent = `Cél: ${FIXED_TARGET_VALUE}`;
             throw new Error(errorMsg);
         }

         try {
             const url = new URL(DATA_SOURCE_URL);
             // Cache-busting hozzáadása, hogy mindig friss adatot kérjünk le
             url.searchParams.append('timestamp', new Date().getTime());

             // FONTOS: Ha az új API speciális fejléceket igényel (pl. authentikáció),
             // akkor azokat itt kell hozzáadni a fetch híváshoz. Példa:
             // const apiHeaders = new Headers();
             // apiHeaders.append('Authorization', 'Bearer YOUR_API_KEY_HERE');
             // apiHeaders.append('Accept', 'text/csv'); // Ha tudjuk, hogy CSV-t várunk
             // const response = await fetch(url.toString(), { headers: apiHeaders });
             // Jelenleg feltételezzük, hogy nincsenek extra fejlécek:
             const response = await fetch(url.toString());

             if (!response.ok) {
                 let responseErrorText = '';
                 try {
                     // Próbáljuk meg kiolvasni a szerver válaszát hiba esetén
                     responseErrorText = await response.text(); 
                 } catch (e) { /* hiba a válasz olvasásakor, nem baj */ }
                 throw new Error(`Hiba az adatforrás (${response.status} ${response.statusText}) elérésekor. Szerver válasza: ${responseErrorText}`);
             }

             const csvText = await response.text(); // Feltételezzük, hogy CSV szöveget kapunk
             const lines = csvText.trim().split('\n');

             if (lines.length === 0 || (lines.length === 1 && lines[0].trim() === '')) {
                 throw new Error('Az adatforrásból kapott CSV fájl üres vagy érvénytelen.');
             }

             let sumOfColumnE = 0;
             let firstRowIsHeader = false;
             const columnIndexE = 4; // E oszlop indexe (A=0, B=1, C=2, D=3, E=4)

             // Ellenőrizzük, hogy az első sor valószínűleg fejléc-e
             // (ha az E oszlopban nem szám van az első sorban)
             if (lines.length > 0) {
                 const firstLineValues = lines[0].split(','); // FIGYELEM: Ha az elválasztó nem vessző, itt módosítani kell!
                 if (firstLineValues.length > columnIndexE && isNaN(parseFloat(firstLineValues[columnIndexE].trim()))) {
                     firstRowIsHeader = true;
                 }
             }

             const startIndex = firstRowIsHeader ? 1 : 0; // Ha van fejléc, a 2. sortól kezdjük (index 1)

             for (let i = startIndex; i < lines.length; i++) {
                 const line = lines[i].trim();
                 if (line === '') continue; // Üres sorok kihagyása

                 // FIGYELEM: Ha az CSV elválasztójel NEM vessző (pl. pontosvessző), akkor itt a .split(',') részt módosítani kell!
                 // Például pontosvessző esetén: line.split(';');
                 const values = line.split(','); 

                 if (values.length > columnIndexE) {
                     const cellValueString = values[columnIndexE].trim();
                     // Előfordulhat, hogy a számok formátuma pl. "1 234,56" vagy "1.234,56"
                     // Ezeket a parseFloat helyes kezeléséhez át kell alakítani "1234.56" formátumra.
                     // Egyszerűsített eset: feltételezzük, hogy a szám tizedespontot használ, és nincsenek ezreselválasztók.
                     // Ha bonyolultabb a számformátum, itt további átalakításra lehet szükség.
                     const cellValue = parseFloat(cellValueString.replace(',', '.')); // Tizedesvesszőt pontra cseréljük, ha van
                     if (!isNaN(cellValue)) {
                         sumOfColumnE += cellValue;
                     }
                 }
             }
             
             const currentPickedOrders = sumOfColumnE;
             const targetValue = FIXED_TARGET_VALUE;

             return { 
                 current: currentPickedOrders, 
                 target: targetValue, 
                 displayTargetForGauge: Math.max(targetValue, 1) 
             };

         } catch (error) {
             console.error('Hiba az adatok feldolgozásakor:', error);
             errorDiv.textContent = `Adatfeldolgozási hiba: ${error.message}. Ellenőrizd a konzolt (F12), az adatforrás URL-t és formátumát.`;
             document.getElementById('current_value_display').textContent = "Hiba";
             document.getElementById('target_value_display').textContent = `Cél: ${FIXED_TARGET_VALUE}`; // Cél megjelenítése hiba esetén is
             throw error;
         }
     }

        async function fetchDataAndUpdateChart() {
            const gaugeElement = document.getElementById('gauge_div');
            try {
                gaugeElement.setAttribute('data-loading', 'true'); // Betöltés jelzése
                const data = await fetchDataFromSheet();
                gaugeElement.removeAttribute('data-loading');

                document.getElementById('current_value_display').textContent = data.current;
                document.getElementById('target_value_display').textContent = data.target;

                gaugeDataTable.setValue(0, 1, data.current);

                // A gauge maximumának és színzónáinak dinamikus beállítása
                const effectiveTarget = data.displayTargetForGauge;
                gaugeOptions.max = effectiveTarget;
                gaugeOptions.redFrom = 0;
                gaugeOptions.redTo = Math.max(0, Math.floor(effectiveTarget * 0.4)); // 0-40% piros
                gaugeOptions.yellowFrom = Math.max(0, Math.floor(effectiveTarget * 0.4));
                gaugeOptions.yellowTo = Math.max(0, Math.floor(effectiveTarget * 0.8)); // 40-80% sárga
                gaugeOptions.greenFrom = Math.max(0, Math.floor(effectiveTarget * 0.8));
                gaugeOptions.greenTo = effectiveTarget; // 80-100% zöld

                // Ha az aktuális érték meghaladja a célt, a Google Gauge alapból jól kezeli,
                // a mutató a maximum fölé megy. A zöld zóna a célértékig tart.

                gaugeChart.draw(gaugeDataTable, gaugeOptions);
                document.getElementById('error_div').textContent = ''; // Sikeres frissítés esetén hibaüzenet törlése

            } catch (error) {
                gaugeElement.removeAttribute('data-loading');
                // A hiba már naplózva lett és ki lett írva a fetchDataFromSheet-ben vagy itt.
                console.error('A diagram frissítése nem sikerült.', error.message);
                // A gauge_div-ben megjelenhetne egy hibaüzenet is, de az error_div már tájékoztat.
            }
        }
    </script>
</body>
</html>
