<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Planet Destroyer</title>
    <style>
        * { margin:0; padding:0; box-sizing:border-box; }
        body {
            font-family: Arial, Helvetica, sans-serif;
            background: linear-gradient(to bottom, #000428, #004e92);
            color: #fff;
            text-align: center;
            min-height: 100vh;
            padding: 15px 10px;
        }
        h1 { font-size: 2.2em; margin: 10px 0; text-shadow: 0 0 15px #0ff; }
        #planetName { font-size: 1.8em; margin: 15px; font-weight: bold; }
        .hp-container {
            width: 90vw;
            max-width: 500px;
            height: 32px;
            background: #111;
            border-radius: 16px;
            margin: 15px auto;
            overflow: hidden;
            box-shadow: inset 0 0 20px #000;
        }
        #hpfill {
            height: 100%;
            width: 100%;
            background: linear-gradient(90deg, #4caf50, #8bc34a);
            transition: width 0.5s ease, background 0.5s;
        }
        #planet {
            width: 48vmin;
            height: 48vmin;
            max-width: 320px;
            max-height: 320px;
            margin: 30px auto;
            border-radius: 50%;
            cursor: pointer;
            position: relative;
            overflow: hidden;
            box-shadow: 0 0 60px rgba(255,255,255,0.4);
            transition: transform 0.15s;
        }
        #planet:active { transform: scale(0.96); }
        #planet::before {
            content: "";
            position: absolute;
            inset: 0;
            border-radius: 50%;
            background: radial-gradient(circle at 30% 30%, rgba(255,255,255,0.4), transparent 50%);
            pointer-events: none;
        }
        #planet.damaged { animation: shake 0.6s infinite; }
        @keyframes shake {
            0%,100% { transform: translateX(0); }
            25% { transform: translateX(-8px); }
            75% { transform: translateX(8px); }
        }

        button {
            padding: 16px;
            margin: 6px;
            background: #2196F3;
            color: white;
            border: none;
            border-radius: 30px;
            font-size: 1em;
            min-height: 60px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.4);
        }
        button:disabled { background:#444; opacity:0.6; }
        #shop, #upgrades {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
            gap: 12px;
            margin: 25px 0;
        }
        #prestige {
            background: #ff1744 !important;
            font-size: 1.2em;
            padding: 18px;
        }
        .popup {
            position: absolute;
            font-weight: bold;
            color: #ff0;
            text-shadow: 2px 2px 8px #000;
            pointer-events: none;
            animation: float 1.2s forwards;
        }
        .big { font-size: 3em; color:#ff4444; animation: explode 2s forwards; }
        @keyframes float { to { transform: translateY(-120px); opacity:0; } }
        @keyframes explode { to { transform: translateY(-150px) scale(2); opacity:0; } }
    </style>
</head>
<body>
    <h1>Planet Destroyer</h1>
    <div id="planetName">Planet 1: Terra</div>
    <div id="hptext">HP: 1,000,000 / 1,000,000</div>
    <div class="hp-container"><div id="hpfill"></div></div>

    <div id="score">Points: 0</div>
    <div id="planet" onclick="clickPlanet()"></div>
    <div id="gps">DPS: 0</div>

    <h2>Annoyers</h2>
    <div id="shop"></div>

    <h2>Upgrades</h2>
    <div id="upgrades"></div>

    <h2>Prestige</h2>
    <button id="prestige" onclick="prestigeReset()">Galactic Reset (need 1T total)</button>
    <div id="passive">Bonus: 1.00x</div>

    <script>
        let points = 0, dps = 0, clickPower = 1, bonus = 1, totalDamage = 0;
        let planetHP = 0, maxHP = 0, currentPlanet = 0;

        const planets = [
            { name:"Terra",      hp:1e6,   gradient:"radial-gradient(#6ab7ff, #0080ff, #003366)" },
            { name:"Pyroclast",  hp:8e6,   gradient:"radial-gradient(#ff6b35, #f7931e, #c0392b)" },
            { name:"Cryon",      hp:5e7,   gradient:"radial-gradient(#a0e7ff, #4fc3f7, #0288d1)" },
            { name:"Gas Giant",  hp:3e8,   gradient:"radial-gradient(#ffb74d, #ff9800, #e65100)" },
            { name:"Void Heart", hp:2e9,   gradient:"radial-gradient(#8e24aa, #4a148c, #000)" }
        ];

        const annoyers = [
            {id:"sat",  name:"Satellite Swarm",     cost:15,     dps:1,     mult:1.12, owned:0, baseCost:15, baseDps:1},
            {id:"moon", name:"Moon Tug",            cost:80,     dps:6,     mult:1.15, owned:0, baseCost:80, baseDps:6},
            {id:"alien",name:"UFO Invasion",        cost:400,    dps:25,    mult:1.18, owned:0, baseCost:400, baseDps:25},
            {id:"storm",name:"Storm Generator",     cost:1500,   dps:100,   mult:1.22, owned:0, baseCost:1500, baseDps:100},
            {id:"comet",name:"Comet Barrage",       cost:8000,   dps:500,   mult:1.28, owned:0, baseCost:8000, baseDps:500},
            {id:"quake",name:"Tectonic Disruptor",  cost:40000,  dps:2500,  mult:1.35, owned:0, baseCost:40000, baseDps:2500},
            {id:"bh",   name:"Mini Black Hole",     cost:250000, dps:15000, mult:1.45, owned:0, baseCost:250000, baseDps:15000}
        ];

        const upgrades = [
            {id:"double", name:"Double Damage",        cost:200,    bought:false},
            {id:"triple", name:"Triple Damage",        cost:1000,   bought:false},
            {id:"mega",   name:"Mega Click (x10)",     cost:15000,  bought:false},
            {id:"dps2",   name:"DPS ×2",               cost:50000,  bought:false},
            {id:"dps3",   name:"DPS ×3",               cost:300000, bought:false}
        ];

        function startPlanet() {
            maxHP = planets[currentPlanet].hp;
            planetHP = maxHP;
            document.getElementById("planetName").textContent = `Planet ${currentPlanet+1}: ${planets[currentPlanet].name}`;
            document.getElementById("planet").style.background = planets[currentPlanet].gradient;
            document.getElementById("planet").className = "";
            update();
        }

        function update() {
            points = Math.floor(points);
            document.getElementById("score").textContent = `Points: ${points.toLocaleString()}`;
            document.getElementById("gps").textContent = `DPS: ${Math.floor(dps).toLocaleString()}`;
            document.getElementById("passive").textContent = `Bonus: ${bonus.toFixed(2)}x`;
            document.getElementById("hptext").textContent = `HP: ${Math.floor(planetHP).toLocaleString()} / ${maxHP.toLocaleString()}`;
            document.getElementById("hpfill").style.width = (planetHP/maxHP*100)+"%";

            const pct = planetHP/maxHP;
            document.getElementById("hpfill").style.background = pct > 0.5 ? 
                "linear-gradient(90deg, #4caf50, #8bc34a)" :
                pct > 0.2 ? "linear-gradient(90deg, #ffc107, #ff9800)" :
                "linear-gradient(90deg, #f44336, #d32f2f)";

            if (pct < 0.3) document.getElementById("planet").classList.add("damaged");
            else document.getElementById("planet").classList.remove("damaged");

            // Annoyers
            document.getElementById("shop").innerHTML = annoyers.map(a=>`
                <button onclick="buyAnnoyer('${a.id}')" ${points < a.cost ? 'disabled' : ''}>
                    ${a.name}<br>Cost: ${Math.floor(a.cost).toLocaleString()}<br>Owned: ${a.owned}
                </button>`).join("");

            // Upgrades - NOW FIXED
            document.getElementById("upgrades").innerHTML = upgrades
                .filter(u => !u.bought)
                .map(u=>`
                <button onclick="buyUpgrade('${u.id}')" ${points < u.cost ? 'disabled' : ''}>
                    ${u.name}<br>Cost: ${u.cost.toLocaleString()}
                </button>`).join("");
        }

        function clickPlanet() {
            const dmg = clickPower * bonus;
            points += dmg;
            totalDamage += dmg;
            planetHP -= dmg;
            popup("+" + Math.floor(dmg).toLocaleString());
            if (planetHP <= 0) killPlanet();
            update();
        }

        function buyAnnoyer(id) {
            const a = annoyers.find(x=>x.id===id);
            if (points >= a.cost) {
                points -= a.cost;
                a.owned++;
                a.cost = a.baseCost * Math.pow(a.mult, a.owned);
                dps += a.baseDps * bonus;
                update();
            }
        }

        function buyUpgrade(id) {
            const u = upgrades.find(x=>x.id===id);
            if (u.bought || points < u.cost) return;
            points -= u.cost;
            u.bought = true;

            if (id === "double") clickPower *= 2;
            if (id === "triple") clickPower *= 3;
            if (id === "mega")   clickPower *= 10;
            if (id === "dps2")   annoyers.forEach(a => a.baseDps *= 2);
            if (id === "dps3")   annoyers.forEach(a => a.baseDps *= 3);

            // Recalculate current DPS after multiplier upgrades
            dps = annoyers.reduce((sum,a) => sum + a.owned * a.baseDps * bonus, 0);

            update();
        }

        function killPlanet() {
            popup("PLANET DESTROYED!", true);
            currentPlanet++;
            if (currentPlanet >= planets.length) {
                currentPlanet = 0;
                bonus *= 2;
                popup("Cycle Complete ×2 Bonus!", true);
            }
            startPlanet();
        }

        function prestigeReset() {
            if (totalDamage >= 1e12) {
                bonus *= Math.floor(Math.sqrt(totalDamage/1e10)) + 3;
                points = 0; dps = 0; clickPower = 1; totalDamage = 0;
                annoyers.forEach(a=>{a.owned=0; a.cost=a.baseCost; a.baseDps=a.dps;});
                upgrades.forEach(u=>u.bought=false);
                currentPlanet = 0;
                startPlanet();
            }
        }

        function popup(text, big=false) {
            const el = document.createElement("div");
            el.className = "popup" + (big?" big":"");
            el.textContent = text;
            el.style.left = (45 + Math.random()*10)+"%";
            document.getElementById("planet").appendChild(el);
            setTimeout(()=>el.remove(), big?2500:1300);
        }

        setInterval(()=>{
            if (dps > 0) {
                const dmg = dps/10;
                points += dmg;
                totalDamage += dmg;
                planetHP -= dmg;
                if (planetHP <= 0) killPlanet();
                update();
            }
        },100);

        startPlanet();
    </script>
</body>
</html>
