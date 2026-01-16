<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>SET Hardcore - 12 Card Limit</title>
    <style>
        :root {
            --bg: #050505;
            --card: #ffffff;
            --neon: #00f2ff;
            --err: #ff0055;
        }

        body {
            background: var(--bg);
            color: #fff;
            font-family: 'Segoe UI', system-ui, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            padding: 20px;
            overflow-x: hidden;
        }

        .status-bar {
            display: flex;
            justify-content: space-between;
            width: 540px;
            margin-bottom: 20px;
            padding: 10px;
            border: 1px solid var(--neon);
            box-shadow: 0 0 10px var(--neon);
        }

        #grid {
            display: grid;
            grid-template-columns: repeat(4, 130px);
            grid-gap: 10px;
        }

        .card {
            width: 130px;
            height: 180px;
            background: var(--card);
            border-radius: 10px;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            cursor: pointer;
            border: 4px solid transparent;
            transition: 0.2s;
        }

        .card.selected { border-color: var(--neon); transform: scale(0.98); box-shadow: 0 0 15px var(--neon); }
        .card.hint { border-color: #ffd700; box-shadow: 0 0 20px #ffd700; }

        .shape { width: 70px; height: 35px; margin: 6px 0; border-width: 4px; border-style: solid; box-sizing: border-box; }
        .shape.oval { border-radius: 20px; }
        .shape.diamond { transform: scaleY(0.8) rotate(45deg); width: 45px; height: 45px; margin: 10px 0; }
        .shape.squiggle { border-radius: 40% 10%; transform: skewX(-15deg); }
        
        .shading-solid { background-color: currentColor; }
        .shading-striped { background-image: repeating-linear-gradient(45deg, transparent, transparent 2px, currentColor 2px, currentColor 4px); background-color: transparent !important; }
        .shading-outlined { background-color: transparent !important; }

        .btns { margin-top: 30px; }
        button {
            padding: 10px 30px;
            background: none;
            color: var(--neon);
            border: 1px solid var(--neon);
            cursor: pointer;
            font-weight: bold;
            margin: 0 10px;
        }
        button:hover { background: var(--neon); color: #000; }
    </style>
</head>
<body>

    <div class="status-bar">
        <span>SCORE: <span id="score">0</span></span>
        <span>MODE: HARDCORE</span>
        <span>DECK: <span id="deck-count">0</span></span>
    </div>
    
    <div id="grid"></div>

    <div class="btns">
        <button onclick="getHint()">HINT</button>
        <button onclick="resetGame()">RESTART</button>
    </div>

    <script>
        const COLORS = ['#FF0055', '#00FF99', '#7700FF'];
        const SHAPES = ['oval', 'diamond', 'squiggle'];
        const SHADINGS = ['solid', 'striped', 'outlined'];
        
        let deck = [], table = [], selected = [], score = 0;

        function createDeck() {
            deck = [];
            for (let c=0; c<3; c++) for (let s=0; s<3; s++) 
                for (let sh=0; sh<3; sh++) for (let n=0; n<3; n++)
                    deck.push({ color: c, shape: s, shading: sh, num: n });
            deck.sort(() => Math.random() - 0.5);
        }

        // 核心：硬核判定 (强制颜色、形状、底纹全异)
        function isHardcoreSet(c1, c2, c3) {
            const props = ['color', 'shape', 'shading', 'num'];
            if (!props.every(p => (c1[p] + c2[p] + c3[p]) % 3 === 0)) return false;
            // 硬核过滤：禁止任何一个属性出现“全同” (除了数量)
            return (c1.color !== c2.color && c1.shape !== c2.shape && c1.shading !== c2.shading);
        }

        function findSets() {
            let found = [];
            for (let i=0; i<table.length; i++)
                for (let j=i+1; j<table.length; j++)
                    for (let k=j+1; k<table.length; k++)
                        if (isHardcoreSet(table[i], table[j], table[k])) found.push([i, j, k]);
            return found;
        }

        function refreshTable() {
            // 如果场上不满12张且牌堆有牌，补齐
            while (table.length < 12 && deck.length > 0) {
                table.push(deck.pop());
            }

            // 检查这12张是否有解，无解则全部洗回重新发
            let sets = findSets();
            if (sets.length === 0 && deck.length > 0) {
                console.log("No hardcore set found. Reshuffling...");
                deck.push(...table);
                deck.sort(() => Math.random() - 0.5);
                table = [];
                refreshTable();
            }
            render();
        }

        function render() {
            const grid = document.getElementById('grid');
            grid.innerHTML = '';
            table.forEach((card, idx) => {
                const el = document.createElement('div');
                el.className = 'card';
                el.onclick = () => {
                    document.querySelectorAll('.card').forEach(c => c.classList.remove('hint'));
                    if (el.classList.contains('selected')) {
                        el.classList.remove('selected');
                        selected = selected.filter(s => s.idx !== idx);
                    } else {
                        el.classList.add('selected');
                        selected.push({ el, data: card, idx });
                    }
                    if (selected.length === 3) check();
                };
                for (let i=0; i<=card.num; i++) {
                    const s = document.createElement('div');
                    s.className = `shape ${SHAPES[card.shape]} shading-${SHADINGS[card.shading]}`;
                    s.style.color = COLORS[card.color];
                    s.style.borderColor = COLORS[card.color];
                    el.appendChild(s);
                }
                grid.appendChild(el);
            });
            document.getElementById('deck-count').innerText = deck.length;
        }

        function check() {
            const [a, b, c] = selected;
            if (isHardcoreSet(a.data, b.data, c.data)) {
                score += 10;
                document.getElementById('score').innerText = score;
                // 移除选中的牌
                const indices = [a.idx, b.idx, c.idx].sort((x, y) => y - x);
                indices.forEach(i => table.splice(i, 1));
            } else {
                alert("ERROR: 属性全同。硬核规则要求颜色、形状、底纹必须各不相同！");
            }
            selected = [];
            refreshTable();
        }

        function getHint() {
            const sets = findSets();
            if (sets.length > 0) {
                const els = document.querySelectorAll('.card');
                sets[0].forEach(i => els[i].classList.add('hint'));
            }
        }

        function resetGame() {
            score = 0;
            document.getElementById('score').innerText = 0;
            createDeck();
            table = [];
            refreshTable();
        }

        createDeck();
        refreshTable();
    </script>
</body>
</html>
