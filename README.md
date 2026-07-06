<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Snake Game</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background: #1a1a2e;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            flex-direction: column;
        }
        .game-container {
            text-align: center;
        }
        h1 {
            color: #4ecca3;
            margin-bottom: 10px;
            font-size: 2.5rem;
            text-shadow: 0 0 10px rgba(78, 204, 163, 0.5);
        }
        .score-board {
            display: flex;
            justify-content: center;
            gap: 30px;
            margin-bottom: 15px;
            color: #fff;
            font-size: 1.2rem;
        }
        .score-board span {
            color: #4ecca3;
            font-weight: bold;
        }
        canvas {
            border: 3px solid #4ecca3;
            border-radius: 8px;
            box-shadow: 0 0 20px rgba(78, 204, 163, 0.3);
            background: #0f0f1a;
        }
        .controls {
            margin-top: 20px;
            color: #888;
            font-size: 0.9rem;
        }
        .controls p {
            margin: 5px 0;
        }
        .btn {
            background: #4ecca3;
            color: #1a1a2e;
            border: none;
            padding: 10px 30px;
            font-size: 1rem;
            font-weight: bold;
            border-radius: 25px;
            cursor: pointer;
            margin-top: 15px;
            transition: all 0.3s;
        }
        .btn:hover {
            background: #3db892;
            transform: scale(1.05);
        }
        .game-over {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.9);
            padding: 30px 50px;
            border-radius: 15px;
            border: 2px solid #e74c3c;
            display: none;
            text-align: center;
        }
        .game-over h2 {
            color: #e74c3c;
            font-size: 2rem;
            margin-bottom: 10px;
        }
        .game-over p {
            color: #fff;
            margin-bottom: 20px;
        }
        .canvas-wrapper {
            position: relative;
            display: inline-block;
        }
    </style>
<base target="_blank">
</head>
<body>
    <div class="game-container">
        <h1>🐍 Snake Game</h1>
        <div class="score-board">
            <div>Score: <span id="score">0</span></div>
            <div>High Score: <span id="highScore">0</span></div>
        </div>
        <div class="canvas-wrapper">
            <canvas id="gameCanvas" width="400" height="400"></canvas>
            <div class="game-over" id="gameOver">
                <h2>Game Over!</h2>
                <p>Final Score: <span id="finalScore">0</span></p>
                <button class="btn" onclick="restartGame()">Play Again</button>
            </div>
        </div>
        <div class="controls">
            <p>Use <strong>Arrow Keys</strong> or <strong>WASD</strong> to move</p>
            <p>Press <strong>Space</strong> to pause/resume</p>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreElement = document.getElementById('score');
        const highScoreElement = document.getElementById('highScore');
        const gameOverElement = document.getElementById('gameOver');
        const finalScoreElement = document.getElementById('finalScore');

        // Game settings
        const gridSize = 20;
        const tileCount = canvas.width / gridSize;

        let snake = [{x: 10, y: 10}];
        let food = {x: 15, y: 15};
        let dx = 0;
        let dy = 0;
        let score = 0;
        let highScore = localStorage.getItem('snakeHighScore') || 0;
        let gameRunning = true;
        let gamePaused = false;
        let speed = 100;
        let gameLoop;

        highScoreElement.textContent = highScore;

        // Draw game
        function drawGame() {
            // Clear canvas
            ctx.fillStyle = '#0f0f1a';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Draw grid (subtle)
            ctx.strokeStyle = '#1a1a2e';
            ctx.lineWidth = 0.5;
            for (let i = 0; i < tileCount; i++) {
                ctx.beginPath();
                ctx.moveTo(i * gridSize, 0);
                ctx.lineTo(i * gridSize, canvas.height);
                ctx.stroke();
                ctx.beginPath();
                ctx.moveTo(0, i * gridSize);
                ctx.lineTo(canvas.width, i * gridSize);
                ctx.stroke();
            }

            // Draw snake
            snake.forEach((segment, index) => {
                if (index === 0) {
                    // Head
                    ctx.fillStyle = '#4ecca3';
                    ctx.shadowColor = '#4ecca3';
                    ctx.shadowBlur = 10;
                } else {
                    // Body
                    const gradient = 1 - (index / snake.length) * 0.5;
                    ctx.fillStyle = `rgba(78, 204, 163, ${gradient})`;
                    ctx.shadowBlur = 0;
                }
                ctx.fillRect(segment.x * gridSize + 1, segment.y * gridSize + 1, gridSize - 2, gridSize - 2);
                ctx.shadowBlur = 0;

                // Draw eyes on head
                if (index === 0) {
                    ctx.fillStyle = '#1a1a2e';
                    const eyeSize = 3;
                    if (dx === 1) { // Moving right
                        ctx.fillRect(segment.x * gridSize + 12, segment.y * gridSize + 5, eyeSize, eyeSize);
                        ctx.fillRect(segment.x * gridSize + 12, segment.y * gridSize + 12, eyeSize, eyeSize);
                    } else if (dx === -1) { // Moving left
                        ctx.fillRect(segment.x * gridSize + 5, segment.y * gridSize + 5, eyeSize, eyeSize);
                        ctx.fillRect(segment.x * gridSize + 5, segment.y * gridSize + 12, eyeSize, eyeSize);
                    } else if (dy === -1) { // Moving up
                        ctx.fillRect(segment.x * gridSize + 5, segment.y * gridSize + 5, eyeSize, eyeSize);
                        ctx.fillRect(segment.x * gridSize + 12, segment.y * gridSize + 5, eyeSize, eyeSize);
                    } else if (dy === 1) { // Moving down
                        ctx.fillRect(segment.x * gridSize + 5, segment.y * gridSize + 12, eyeSize, eyeSize);
                        ctx.fillRect(segment.x * gridSize + 12, segment.y * gridSize + 12, eyeSize, eyeSize);
                    } else { // Default
                        ctx.fillRect(segment.x * gridSize + 12, segment.y * gridSize + 5, eyeSize, eyeSize);
                        ctx.fillRect(segment.x * gridSize + 12, segment.y * gridSize + 12, eyeSize, eyeSize);
                    }
                }
            });

            // Draw food
            ctx.fillStyle = '#e74c3c';
            ctx.shadowColor = '#e74c3c';
            ctx.shadowBlur = 15;
            ctx.beginPath();
            ctx.arc(
                food.x * gridSize + gridSize / 2,
                food.y * gridSize + gridSize / 2,
                gridSize / 2 - 2,
                0,
                Math.PI * 2
            );
            ctx.fill();
            ctx.shadowBlur = 0;

            // Draw food shine
            ctx.fillStyle = '#ff6b6b';
            ctx.beginPath();
            ctx.arc(
                food.x * gridSize + gridSize / 2 - 3,
                food.y * gridSize + gridSize / 2 - 3,
                3,
                0,
                Math.PI * 2
            );
            ctx.fill();
        }

        // Update game state
        function updateGame() {
            if (!gameRunning || gamePaused) return;

            const head = {x: snake[0].x + dx, y: snake[0].y + dy};

            // Check wall collision
            if (head.x < 0 || head.x >= tileCount || head.y < 0 || head.y >= tileCount) {
                gameOver();
                return;
            }

            // Check self collision
            for (let segment of snake) {
                if (head.x === segment.x && head.y === segment.y) {
                    gameOver();
                    return;
                }
            }

            snake.unshift(head);

            // Check food collision
            if (head.x === food.x && head.y === food.y) {
                score += 10;
                scoreElement.textContent = score;
                if (score > highScore) {
                    highScore = score;
                    highScoreElement.textContent = highScore;
                    localStorage.setItem('snakeHighScore', highScore);
                }
                // Increase speed slightly
                if (speed > 50) speed -= 2;
                placeFood();
            } else {
                snake.pop();
            }

            drawGame();
        }

        // Place food randomly
        function placeFood() {
            do {
                food = {
                    x: Math.floor(Math.random() * tileCount),
                    y: Math.floor(Math.random() * tileCount)
                };
            } while (snake.some(segment => segment.x === food.x && segment.y === food.y));
        }

        // Game over
        function gameOver() {
            gameRunning = false;
            clearInterval(gameLoop);
            finalScoreElement.textContent = score;
            gameOverElement.style.display = 'block';
        }

        // Restart game
        function restartGame() {
            snake = [{x: 10, y: 10}];
            dx = 0;
            dy = 0;
            score = 0;
            speed = 100;
            scoreElement.textContent = score;
            gameRunning = true;
            gamePaused = false;
            gameOverElement.style.display = 'none';
            placeFood();
            drawGame();
            gameLoop = setInterval(updateGame, speed);
        }

        // Keyboard controls
        document.addEventListener('keydown', (e) => {
            if (!gameRunning && e.key !== ' ') return;

            switch(e.key) {
                case 'ArrowUp':
                case 'w':
                case 'W':
                    if (dy === 0) { dx = 0; dy = -1; }
                    break;
                case 'ArrowDown':
                case 's':
                case 'S':
                    if (dy === 0) { dx = 0; dy = 1; }
                    break;
                case 'ArrowLeft':
                case 'a':
                case 'A':
                    if (dx === 0) { dx = -1; dy = 0; }
                    break;
                case 'ArrowRight':
                case 'd':
                case 'D':
                    if (dx === 0) { dx = 1; dy = 0; }
                    break;
                case ' ':
                    if (gameRunning) {
                        gamePaused = !gamePaused;
                    }
                    break;
            }
        });

        // Start game
        placeFood();
        drawGame();
        gameLoop = setInterval(updateGame, speed);
    </script>
</body>
</html>
