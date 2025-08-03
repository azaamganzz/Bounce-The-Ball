<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>bounce the ball</title>
  <style>
    body {
      margin: 0;
      background: linear-gradient(135deg, #1e3c72, #2a5298);
      color: #fff;
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      user-select: none;
    }
    h1 {
      margin-top: 1rem;
      font-weight: 700;
      letter-spacing: 2px;
      text-shadow: 0 2px 8px rgba(0,0,0,0.6);
    }
    #gameCanvas {
      background: #15294b;
      border: 3px solid #00baff;
      border-radius: 12px;
      margin-top: 1rem;
      display: block;
      box-shadow: 0 0 25px #00baffaa;
    }
    #scoreBoard {
      margin-top: 1rem;
      font-size: 1.25rem;
      font-weight: 600;
      text-shadow: 0 1px 3px rgba(0,0,0,0.6);
    }
    #message {
      margin-top: 1rem;
      font-size: 1.5rem;
      font-weight: 700;
      color: #00ffd1;
      letter-spacing: 1.5px;
      text-shadow: 0 0 8px #00ffd1;
    }
    button#restartBtn {
      margin-top: 1rem;
      padding: 0.75rem 1.5rem;
      font-size: 1rem;
      font-weight: 700;
      border-radius: 8px;
      border: none;
      background: #00baff;
      color: #15294b;
      cursor: pointer;
      box-shadow: 0 0 12px #00baffaa;
      transition: background-color 0.3s cubic-bezier(0.4,0,0.2,1);
    }
    button#restartBtn:hover {
      background: #008fb3;
    }
  </style>
</head>
<body>
  <h1>Bounce The Ball</h1>
  <canvas id="gameCanvas" width="800" height="500" aria-label="Block Bash game canvas"></canvas>
  <div id="scoreBoard">Score: 0</div>
  <div id="message"></div>
  <button id="restartBtn" style="display:none;">Restart</button>

  <script>
    (() => {
      const canvas = document.getElementById('gameCanvas');
      const ctx = canvas.getContext('2d');
      const scoreBoard = document.getElementById('scoreBoard');
      const message = document.getElementById('message');
      const restartBtn = document.getElementById('restartBtn');

      const paddle = {
        height: 15,
        width: 120,
        x: (canvas.width - 120) / 2,
        y: canvas.height - 30,
        speed: 10,
        dx: 0,
        color: '#00e4ff',
        radius: 8,
      };

      const ball = {
        x: canvas.width / 2,
        y: canvas.height - 40,
        radius: 12,
        speed: 6,
        dx: 4,
        dy: -4,
        color: '#00fff7',
        trail: [],
        trailMaxLength: 15,
      };

      const blockOptions = {
        rowCount: 6,
        columnCount: 11,
        width: 60,
        height: 25,
        padding: 8,
        offsetTop: 50,
        offsetLeft: 35,
        colors: ['#ff6e40', '#fbc02d', '#42a5f5', '#66bb6a', '#ba68c8', '#ff8a65'],
      };

      let blocks = [];
      let score = 0;
      let isGameOver = false;
      let isWin = false;

      function createBlocks() {
        blocks = [];
        for(let r=0; r < blockOptions.rowCount; r++) {
          for(let c=0; c < blockOptions.columnCount; c++) {
            const blockX = blockOptions.offsetLeft + c * (blockOptions.width + blockOptions.padding);
            const blockY = blockOptions.offsetTop + r * (blockOptions.height + blockOptions.padding);
            blocks.push({
              x: blockX,
              y: blockY,
              width: blockOptions.width,
              height: blockOptions.height,
              color: blockOptions.colors[r % blockOptions.colors.length],
              destroyed: false,
            });
          }
        }
      }

      function drawPaddle() {
        ctx.beginPath();
        ctx.fillStyle = paddle.color;
        ctx.shadowColor = '#00baff';
        ctx.shadowBlur = 12;
        // rounded paddle
        let rad = paddle.height / 2;
        ctx.moveTo(paddle.x + rad, paddle.y);
        ctx.lineTo(paddle.x + paddle.width - rad, paddle.y);
        ctx.quadraticCurveTo(paddle.x + paddle.width, paddle.y, paddle.x + paddle.width, paddle.y + rad);
        ctx.lineTo(paddle.x + paddle.width, paddle.y + paddle.height - rad);
        ctx.quadraticCurveTo(paddle.x + paddle.width, paddle.y + paddle.height, paddle.x + paddle.width - rad, paddle.y + paddle.height);
        ctx.lineTo(paddle.x + rad, paddle.y + paddle.height);
        ctx.quadraticCurveTo(paddle.x, paddle.y + paddle.height, paddle.x, paddle.y + paddle.height - rad);
        ctx.lineTo(paddle.x, paddle.y + rad);
        ctx.quadraticCurveTo(paddle.x, paddle.y, paddle.x + rad, paddle.y);
        ctx.fill();
        ctx.shadowBlur = 0;
        ctx.closePath();
      }

      function drawBall() {
        // Draw ball glow and trail
        ctx.save();
        ctx.shadowColor = '#00fff7';
        ctx.shadowBlur = 18;
        // Draw trails
        for(let i = 0; i < ball.trail.length; i++) {
          const t = ball.trail[i];
          ctx.beginPath();
          const alpha = (i + 1) / ball.trail.length * 0.4;
          ctx.fillStyle = `rgba(0,255,247,${alpha})`;
          ctx.arc(t.x, t.y, ball.radius - i * 0.7, 0, Math.PI * 2);
          ctx.fill();
          ctx.closePath();
        }
        ctx.restore();

        ctx.beginPath();
        ctx.fillStyle = ball.color;
        ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
        ctx.fill();
        ctx.closePath();
      }

      function drawBlocks() {
        blocks.forEach(block => {
          if (!block.destroyed) {
            ctx.beginPath();
            ctx.fillStyle = block.color;
            ctx.shadowColor = block.color;
            ctx.shadowBlur = 8;
            ctx.fillRect(block.x, block.y, block.width, block.height);
            ctx.shadowBlur = 0;
            ctx.strokeStyle = '#fff';
            ctx.lineWidth = 1;
            ctx.strokeRect(block.x, block.y, block.width, block.height);
            ctx.closePath();
          }
        });
      }

      function drawScore() {
        scoreBoard.textContent = `Score: ${score}`;
      }

      function draw() {
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        drawBlocks();
        drawPaddle();
        drawBall();
        drawScore();
      }

      function updateBallPosition() {
        if (isGameOver || isWin) return;

        ball.x += ball.dx;
        ball.y += ball.dy;

        // Save trail points
        ball.trail.unshift({x: ball.x, y: ball.y});
        if (ball.trail.length > ball.trailMaxLength) ball.trail.pop();

        // Wall collision (left/right)
        if(ball.x + ball.radius > canvas.width) {
          ball.x = canvas.width - ball.radius;
          ball.dx = -ball.dx;
        } else if(ball.x - ball.radius < 0) {
          ball.x = ball.radius;
          ball.dx = -ball.dx;
        }

        // Ceiling collision
        if(ball.y - ball.radius < 0) {
          ball.y = ball.radius;
          ball.dy = -ball.dy;
        }

        // Paddle collision
        if(ball.y + ball.radius > paddle.y &&
           ball.x > paddle.x &&
           ball.x < paddle.x + paddle.width) {
          ball.y = paddle.y - ball.radius;
          ball.dy = -ball.dy;

          // Change direction depending on where it hit the paddle
          const hitPos = ball.x - (paddle.x + paddle.width / 2);
          ball.dx = hitPos * 0.15;
        }

        // Bottom collision - lose condition
        if(ball.y - ball.radius > canvas.height) {
          endGame(false);
        }
      }

      function checkBlockCollision() {
        blocks.forEach(block => {
          if (!block.destroyed) {
            if (ball.x + ball.radius > block.x &&
                ball.x - ball.radius < block.x + block.width &&
                ball.y + ball.radius > block.y &&
                ball.y - ball.radius < block.y + block.height) {
              block.destroyed = true;

              // Bounce ball
              ball.dy = -ball.dy;

              score += 10;

              if (score === blockOptions.rowCount * blockOptions.columnCount * 10) {
                endGame(true);
              }
            }
          }
        });
      }

      function updatePaddle() {
        if (isGameOver || isWin) return;
        paddle.x += paddle.dx;

        if(paddle.x < 0) paddle.x = 0;
        if(paddle.x + paddle.width > canvas.width) paddle.x = canvas.width - paddle.width;
      }

      function clearMessage() {
        message.textContent = '';
        restartBtn.style.display = 'none';
      }

      function endGame(won) {
        isGameOver = true;
        isWin = won;
        message.textContent = won ? 'You Win! 🎉' : 'Game Over! ☹️';
        restartBtn.style.display = 'inline-block';
      }

      function restartGame() {
        score = 0;
        isGameOver = false;
        isWin = false;
        clearMessage();
        createBlocks();
        paddle.x = (canvas.width - paddle.width) / 2;
        ball.x = canvas.width/2;
        ball.y = canvas.height - 40;
        ball.dx = 4;
        ball.dy = -4;
        ball.trail = [];
      }

      function gameLoop() {
        updatePaddle();
        updateBallPosition();
        checkBlockCollision();
        draw();
        requestAnimationFrame(gameLoop);
      }

      // Controls
      document.addEventListener('keydown', e => {
        if (e.key === 'ArrowLeft' || e.key === 'a') {
          paddle.dx = -paddle.speed;
        }
        else if (e.key === 'ArrowRight' || e.key === 'd') {
          paddle.dx = paddle.speed;
        }
      });
      document.addEventListener('keyup', e => {
        if ((e.key === 'ArrowLeft' || e.key === 'a') && paddle.dx < 0) {
          paddle.dx = 0;
        }
        else if ((e.key === 'ArrowRight' || e.key === 'd') && paddle.dx > 0) {
          paddle.dx = 0;
        }
      });

      // Mouse control for paddle
      canvas.addEventListener('mousemove', e => {
        if(isGameOver || isWin) return;
        const rect = canvas.getBoundingClientRect();
        paddle.x = e.clientX - rect.left - paddle.width / 2;

        if(paddle.x < 0) paddle.x = 0;
        if(paddle.x + paddle.width > canvas.width) paddle.x = canvas.width - paddle.width;
      });

      restartBtn.addEventListener('click', () => {
        restartGame();
      });

      createBlocks();
      gameLoop();
    })();
  </script>
</body>
</html>
