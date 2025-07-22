# Jogo-de-guerra-

<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Guerra Tática 2D - Implementação Completa</title>
    <style>
        /* (Manter os estilos anteriores) */
        #game-ui {
            display: flex;
            margin-top: 10px;
        }
        #unit-info {
            background-color: #0f3460;
            padding: 10px;
            border-radius: 8px;
            margin-left: 20px;
            width: 200px;
        }
        .tile {
            width: 40px;
            height: 40px;
            display: inline-block;
            margin: 2px;
        }
        .terrain-grass { background-color: #4CAF50; }
        .terrain-water { background-color: #2196F3; }
        .terrain-mountain { background-color: #795548; }
    </style>
</head>
<body>
    <!-- (Manter a estrutura HTML anterior) -->

    <script>
        // ===== VARIÁVEIS GLOBAIS =====
        const ADMIN_PASSWORD = "senha123";
        let isAdmin = false;
        let selectedUnit = null;
        let currentPlayer = 'blue';
        let gameState = 'selecting'; // 'selecting', 'moving', 'attacking'
        
        // Dados do jogo
        const weapons = [
            { id: 1, name: "Rifle", damage: 20, range: 5, icon: "🔫" },
            { id: 2, name: "Bazuca", damage: 50, range: 3, icon: "💥" },
            { id: 3, name: "Sniper", damage: 35, range: 8, icon: "🎯" }
        ];

        const units = [
            { id: 1, x: 2, y: 2, type: 'soldier', team: 'blue', weapon: 1, health: 100, moves: 3 },
            { id: 2, x: 7, y: 5, type: 'tank', team: 'blue', weapon: 2, health: 150, moves: 2 },
            { id: 3, x: 8, y: 2, type: 'soldier', team: 'red', weapon: 1, health: 100, moves: 3 }
        ];

        const map = [
            [0, 0, 0, 1, 0, 0, 0, 0, 0, 0],
            [0, 0, 0, 1, 0, 2, 0, 0, 0, 0],
            [0, 1, 1, 1, 0, 0, 0, 2, 0, 0],
            [0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
            [0, 0, 2, 0, 0, 1, 1, 0, 0, 0],
            [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
        ];
        // 0 = grama, 1 = água, 2 = montanha

        // ===== FUNÇÕES DO JOGO =====
        function initGame() {
            const canvas = document.getElementById('gameCanvas');
            canvas.addEventListener('click', handleCanvasClick);
            
            renderGame();
        }

        function renderGame() {
            const canvas = document.getElementById('gameCanvas');
            const ctx = canvas.getContext('2d');
            
            // Limpar canvas
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Desenhar mapa
            const tileSize = 40;
            for (let y = 0; y < map.length; y++) {
                for (let x = 0; x < map[y].length; x++) {
                    // Terreno
                    switch(map[y][x]) {
                        case 0: ctx.fillStyle = '#4CAF50'; break; // Grama
                        case 1: ctx.fillStyle = '#2196F3'; break; // Água
                        case 2: ctx.fillStyle = '#795548'; break; // Montanha
                    }
                    ctx.fillRect(x * tileSize, y * tileSize, tileSize, tileSize);
                    ctx.strokeStyle = '#333';
                    ctx.strokeRect(x * tileSize, y * tileSize, tileSize, tileSize);
                }
            }
            
            // Desenhar unidades
            units.forEach(unit => {
                const x = unit.x * tileSize;
                const y = unit.y * tileSize;
                
                // Cor da unidade
                ctx.fillStyle = unit.team === 'blue' ? '#3498db' : '#e74c3c';
                ctx.beginPath();
                ctx.arc(x + tileSize/2, y + tileSize/2, tileSize/2 - 2, 0, Math.PI * 2);
                ctx.fill();
                
                // Destaque para unidade selecionada
                if (selectedUnit && selectedUnit.id === unit.id) {
                    ctx.strokeStyle = '#f1c40f';
                    ctx.lineWidth = 3;
                    ctx.beginPath();
                    ctx.arc(x + tileSize/2, y + tileSize/2, tileSize/2, 0, Math.PI * 2);
                    ctx.stroke();
                }
                
                // Barra de saúde
                const healthPercent = unit.health / (unit.type === 'tank' ? 150 : 100);
                ctx.fillStyle = healthPercent > 0.6 ? '#2ecc71' : healthPercent > 0.3 ? '#f39c12' : '#e74c3c';
                ctx.fillRect(x + 2, y - 10, (tileSize - 4) * healthPercent, 5);
            });
            
            // Atualizar UI
            updateUnitInfo();
        }

        function handleCanvasClick(event) {
            const canvas = document.getElementById('gameCanvas');
            const rect = canvas.getBoundingClientRect();
            const x = Math.floor((event.clientX - rect.left) / 40);
            const y = Math.floor((event.clientY - rect.top) / 40);
            
            // Verificar se clicou em uma unidade
            const clickedUnit = units.find(u => u.x === x && u.y === y);
            
            switch(gameState) {
                case 'selecting':
                    if (clickedUnit && clickedUnit.team === currentPlayer) {
                        selectedUnit = clickedUnit;
                        gameState = 'moving';
                    }
                    break;
                    
                case 'moving':
                    if (isValidMove(x, y)) {
                        moveUnit(x, y);
                        gameState = 'attacking';
                    } else if (clickedUnit === selectedUnit) {
                        // Cancelar seleção
                        selectedUnit = null;
                        gameState = 'selecting';
                    }
                    break;
                    
                case 'attacking':
                    if (clickedUnit && clickedUnit.team !== currentPlayer) {
                        attackUnit(clickedUnit);
                        endTurn();
                    } else if (clickedUnit === selectedUnit) {
                        // Pular ataque
                        endTurn();
                    }
                    break;
            }
            
            renderGame();
        }

        function isValidMove(x, y) {
            if (!selectedUnit) return false;
            
            // Verificar se está dentro do mapa
            if (x < 0 || y < 0 || y >= map.length || x >= map[0].length) return false;
            
            // Verificar terreno (unidades não podem se mover para água ou montanhas)
            if (map[y][x] === 1 || map[y][x] === 2) return false;
            
            // Verificar se já tem uma unidade no local
            if (units.some(u => u.x === x && u.y === y)) return false;
            
            // Verificar distância do movimento
            const dx = Math.abs(x - selectedUnit.x);
            const dy = Math.abs(y - selectedUnit.y);
            const distance = dx + dy;
            
            return distance <= selectedUnit.moves;
        }

        function moveUnit(x, y) {
            selectedUnit.x = x;
            selectedUnit.y = y;
            selectedUnit.moves = 0; // Gasta todos os movimentos
        }

        function attackUnit(target) {
            const weapon = weapons.find(w => w.id === selectedUnit.weapon);
            target.health -= weapon.damage;
            
            // Verificar se a unidade foi destruída
            if (target.health <= 0) {
                const index = units.findIndex(u => u.id === target.id);
                units.splice(index, 1);
            }
        }

        function endTurn() {
            selectedUnit = null;
            currentPlayer = currentPlayer === 'blue' ? 'red' : 'blue';
            
            // Resetar movimentos para todas as unidades do jogador atual
            units.filter(u => u.team === currentPlayer).forEach(u => {
                u.moves = u.type === 'tank' ? 2 : 3;
            });
            
            gameState = 'selecting';
            renderGame();
        }

        function updateUnitInfo() {
            const infoDiv = document.getElementById('unit-info');
            
            if (!selectedUnit) {
                infoDiv.innerHTML = `<h3>Time ${currentPlayer.toUpperCase()}</h3>
                                   <p>Selecione uma unidade</p>`;
                return;
            }
            
            const weapon = weapons.find(w => w.id === selectedUnit.weapon);
            infoDiv.innerHTML = `
                <h3>${selectedUnit.type.toUpperCase()} (${selectedUnit.team})</h3>
                <p>Saúde: ${selectedUnit.health}</p>
                <p>Arma: ${weapon.name} (${weapon.damage} dano)</p>
                <p>Alcance: ${weapon.range} quadrados</p>
                ${gameState === 'moving' ? '<p>Selecione um local para mover</p>' : ''}
                ${gameState === 'attacking' ? '<p>Selecione um alvo para atacar</p>' : ''}
            `;
        }

        // ===== FUNÇÕES DE ADMIN (manter as anteriores) =====
        function login() { /* ... */ }
        function playAsGuest() { /* ... */ }
        function addWeapon() { /* ... */ }
        function updateWeaponsList() { /* ... */ }
        function editWeapon(index) { /* ... */ }
        function deleteWeapon(index) { /* ... */ }

        // ===== INICIALIZAÇÃO =====
        window.onload = function() {
            // (Manter lógica de login anterior)
            updateWeaponsList();
        };
    </script>
</body>
</html>
