<!DOCTYPE html>
<html lang="ko">
<head>
   <meta charset="UTF-8">
   <meta name="viewport" content="width=device-width, initial-scale=1.0">
   <title>물 몬스터 잡기</title>
   <script src="https://cdn.tailwindcss.com"></script>
   <style>
       @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap');
       body {
           font-family: 'Inter', sans-serif;
           background-color: #f0f9ff;
       }
       canvas {
           background-color: #87ceeb; /* 하늘색 배경 */
           border-radius: 1rem;
           box-shadow: 0 4px 14px rgba(0, 0, 0, 0.2);
           touch-action: none;
           cursor: crosshair;
       }
       .game-container {
           width: 95%;
           max-width: 800px;
           height: auto;
           aspect-ratio: 16 / 9;
           margin: 2rem auto;
           display: flex;
           flex-direction: column;
           align-items: center;
       }
   </style>
</head>
<body class="flex items-center justify-center min-h-screen bg-gray-100 p-4">


   <div class="game-container bg-white p-6 rounded-3xl shadow-2xl flex flex-col items-center justify-between">
       <h1 class="text-3xl font-bold text-gray-800 mb-4 text-center">🌊 물 몬스터 잡기!</h1>
       <div class="w-full flex justify-between items-center text-lg font-semibold text-gray-700 mb-4">
           <span id="score-display">점수: 0</span>
           <span id="round-display">라운드: 1</span>
           <span id="time-display">시간: 60s</span>
       </div>
       <canvas id="gameCanvas" class="w-full"></canvas>
       <div id="message-box" class="mt-4 p-4 bg-yellow-200 text-yellow-800 rounded-lg hidden text-center w-full"></div>
       <button id="startButton" class="mt-4 px-8 py-3 bg-blue-500 hover:bg-blue-600 text-white font-bold rounded-full shadow-lg transition-transform transform hover:scale-105">게임 시작</button>
   </div>


   <script>
       const canvas = document.getElementById('gameCanvas');
       const ctx = canvas.getContext('2d');
       const scoreDisplay = document.getElementById('score-display');
       const timeDisplay = document.getElementById('time-display');
       const roundDisplay = document.getElementById('round-display');
       const startButton = document.getElementById('startButton');
       const messageBox = document.getElementById('message-box');


       let score = 0;
       let gameTime = 60;
       let monsters = [];
       let isGameActive = false;
       let monsterInterval;
       let timerInterval;
       let round = 1;
      
       let waterLevel = 0;
       const maxWaterLevel = 100;
       const waterPerMonster = 5;
       const roundBonus = 50;
       let bucket = {};


       // 캔버스 크기 조절
       function resizeCanvas() {
           canvas.width = canvas.offsetWidth;
           canvas.height = canvas.offsetWidth * (9 / 16);
          
           // 물통 위치 및 크기 계산
           bucket.width = canvas.width * 0.05;
           bucket.height = canvas.height * 0.5;
           bucket.x = canvas.width * 0.03;
           bucket.y = canvas.height - bucket.height - canvas.height * 0.03;
       }
       window.addEventListener('resize', resizeCanvas);
       resizeCanvas();


       // 메시지 박스 표시
       function showMessage(msg, duration = 3000) {
           messageBox.textContent = msg;
           messageBox.classList.remove('hidden');
           setTimeout(() => {
               messageBox.classList.add('hidden');
           }, duration);
       }


       // 몬스터 클래스
       class Monster {
           constructor(x, y, radius, dx, dy) {
               this.x = x;
               this.y = y;
               this.radius = radius;
               this.dx = dx;
               this.dy = dy;
           }


           draw() {
               // 물귀신 몸통 (불투명한 흰색)
               ctx.beginPath();
               ctx.moveTo(this.x - this.radius, this.y);
               ctx.lineTo(this.x - this.radius, this.y + this.radius * 0.8);
               ctx.arcTo(this.x - this.radius * 0.5, this.y + this.radius * 1.5, this.x, this.y + this.radius * 1.5, this.radius * 0.5);
               ctx.arcTo(this.x + this.radius * 0.5, this.y + this.radius * 1.5, this.x + this.radius, this.y + this.radius * 0.8, this.radius * 0.5);
               ctx.lineTo(this.x + this.radius, this.y);
               ctx.arc(this.x, this.y, this.radius, 0, Math.PI, true);
               ctx.fillStyle = 'rgba(255, 255, 255, 0.8)';
               ctx.fill();
               ctx.closePath();


               // 물귀신 눈 (검은색)
               ctx.beginPath();
               ctx.arc(this.x - this.radius * 0.3, this.y - this.radius * 0.2, this.radius * 0.1, 0, Math.PI * 2);
               ctx.arc(this.x + this.radius * 0.3, this.y - this.radius * 0.2, this.radius * 0.1, 0, Math.PI * 2);
               ctx.fillStyle = '#000';
               ctx.fill();
               ctx.closePath();
           }


           update() {
               // 경계선 충돌 처리
               if (this.x + this.radius > canvas.width || this.x - this.radius < 0) {
                   this.dx = -this.dx;
               }
               if (this.y + this.radius > canvas.height || this.y - this.radius < 0) {
                   this.dy = -this.dy;
               }


               // 위치 업데이트
               this.x += this.dx;
               this.y += this.dy;


               this.draw();
           }
       }


       // 물통 그리기
       function drawWaterBucket() {
           // 물통 테두리 (철 양동이 느낌을 위해 회색으로 변경)
           ctx.strokeStyle = '#7f7f7f';
           ctx.lineWidth = 4;
           ctx.strokeRect(bucket.x, bucket.y, bucket.width, bucket.height);


           // 물통 내부의 물
           const filledHeight = (waterLevel / maxWaterLevel) * bucket.height;
           const waterY = bucket.y + bucket.height - filledHeight;
           ctx.fillStyle = '#00bfff';
           ctx.fillRect(bucket.x, waterY, bucket.width, filledHeight);
       }
      
       // 다음 라운드로 이동하는 함수
       function nextRound() {
           round++;
           score += roundBonus;
           monsters = [];
           waterLevel = 0;
          
           roundDisplay.textContent = `라운드: ${round}`;
           scoreDisplay.textContent = `점수: ${score}`;
          
           showMessage(`라운드 ${round} 시작! 보너스 점수 ${roundBonus}점 획득!`);
       }


       // 새로운 몬스터 생성
       function createMonster() {
           const radius = Math.random() * 10 + 15; // 15 to 25
           const x = Math.random() * (canvas.width - radius * 2) + radius;
           const y = Math.random() * (canvas.height - radius * 2) + radius;
           // 몬스터의 초기 속도를 라운드에 따라 조정
           const speedFactor = 1 + round * 0.2;
           const dx = (Math.random() - 0.5) * 0.8 * speedFactor;
           const dy = (Math.random() - 0.5) * 0.8 * speedFactor;


           monsters.push(new Monster(x, y, radius, dx, dy));
       }


       // 게임 루프
       function gameLoop() {
           if (!isGameActive) return;


           ctx.clearRect(0, 0, canvas.width, canvas.height);
           drawWaterBucket(); // 물통을 몬스터보다 먼저 그려서 몬스터가 위에 보이도록 함
           monsters.forEach(monster => {
               monster.update();
           });


           requestAnimationFrame(gameLoop);
       }


       // 점수 및 물통 업데이트
       function updateScore(points) {
           score += points;
           scoreDisplay.textContent = `점수: ${score}`;
          
           // 물통 레벨 업데이트
           waterLevel += waterPerMonster;
           if (waterLevel >= maxWaterLevel) {
               nextRound();
           }
       }


       // 게임 초기화
       function initializeGame() {
           score = 0;
           gameTime = 60;
           monsters = [];
           waterLevel = 0;
           round = 1;
           scoreDisplay.textContent = `점수: 0`;
           roundDisplay.textContent = `라운드: 1`;
           timeDisplay.textContent = `시간: 60s`;
           showMessage('게임을 시작하려면 "게임 시작" 버튼을 누르세요!');
          
           // 초기 화면에 물통 그리기
           ctx.clearRect(0, 0, canvas.width, canvas.height);
           drawWaterBucket();
       }


       // 게임 시작
       function startGame() {
           if (isGameActive) return;
           isGameActive = true;
           startButton.textContent = '게임 중...';
           startButton.disabled = true;
           initializeGame();


           // 몬스터 생성 간격
           monsterInterval = setInterval(() => {
               createMonster();
           }, 1000);


           // 타이머
           timerInterval = setInterval(() => {
               gameTime--;
               timeDisplay.textContent = `시간: ${gameTime}s`;
               if (gameTime <= 0) {
                   endGame();
               }
           }, 1000);


           gameLoop();
       }


       // 게임 종료
       function endGame() {
           isGameActive = false;
           clearInterval(monsterInterval);
           clearInterval(timerInterval);
           startButton.textContent = '다시 시작';
           startButton.disabled = false;
           showMessage(`게임 종료! 최종 점수: ${score}점`, 5000);
       }


       // 클릭 이벤트 핸들러
       canvas.addEventListener('mousedown', (event) => {
           if (!isGameActive) return;


           const rect = canvas.getBoundingClientRect();
           const mouseX = event.clientX - rect.left;
           const mouseY = event.clientY - rect.top;


           for (let i = monsters.length - 1; i >= 0; i--) {
               const monster = monsters[i];
               const distance = Math.sqrt((mouseX - monster.x)**2 + (mouseY - monster.y)**2);
               if (distance < monster.radius) {
                   updateScore(10);
                   monsters.splice(i, 1);
                   break;
               }
           }
       });


       // 터치 이벤트 핸들러 (모바일)
       canvas.addEventListener('touchstart', (event) => {
           if (!isGameActive) return;
           event.preventDefault(); // 기본 스크롤 동작 방지


           const rect = canvas.getBoundingClientRect();
           const touch = event.touches[0];
           const touchX = touch.clientX - rect.left;
           const touchY = touch.clientY - rect.top;


           for (let i = monsters.length - 1; i >= 0; i--) {
               const monster = monsters[i];
               const distance = Math.sqrt((touchX - monster.x)**2 + (touchY - monster.y)**2);
               if (distance < monster.radius) {
                   updateScore(10);
                   monsters.splice(i, 1);
                   break;
               }
           }
       });


       startButton.addEventListener('click', startGame);


       // 페이지 로드 시 게임 초기화
       window.onload = initializeGame;
   </script>


</body>
</html>



