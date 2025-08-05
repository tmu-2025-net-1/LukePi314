<script>
	import { onMount } from 'svelte';
	
	let canvas;
	let ctx;
	let gameRunning = false;
	let gameObjects = [];
	let currentKanji = null;
	let mouseX = 0;
	let gravity = 0.3;
	let friction = 0.98;
	let bounceMultiplier = 0.6;
	let groundY = 0;
	let gameHeight = 600;
	let gameWidth = 800;
	let dropSpeed = 2;
	let lastTime = 0;
	let animationId = null;
	
	// 積みやすい・引っ掛かりやすい漢字のリスト
	const kanjiList = [
		{ char: "山", width: 60, height: 50, density: 0.8, shape: "triangle" },
		{ char: "川", width: 50, height: 60, density: 0.7, shape: "vertical" },
		{ char: "木", width: 55, height: 65, density: 0.6, shape: "tree" },
		{ char: "田", width: 50, height: 50, density: 0.9, shape: "square" },
		{ char: "口", width: 45, height: 45, density: 0.5, shape: "square" },
		{ char: "人", width: 55, height: 60, density: 0.4, shape: "triangle" },
		{ char: "大", width: 60, height: 55, density: 0.5, shape: "spread" },
		{ char: "小", width: 40, height: 45, density: 0.3, shape: "compact" },
		{ char: "上", width: 50, height: 45, density: 0.6, shape: "horizontal" },
		{ char: "下", width: 50, height: 45, density: 0.6, shape: "horizontal" },
		{ char: "中", width: 45, height: 55, density: 0.7, shape: "vertical" },
		{ char: "十", width: 50, height: 50, density: 0.4, shape: "cross" },
		{ char: "工", width: 55, height: 40, density: 0.7, shape: "horizontal" },
		{ char: "土", width: 50, height: 50, density: 0.8, shape: "base" },
		{ char: "王", width: 50, height: 55, density: 0.8, shape: "vertical" },
		{ char: "火", width: 45, height: 60, density: 0.5, shape: "flame" },
		{ char: "水", width: 40, height: 65, density: 0.6, shape: "flowing" },
		{ char: "金", width: 55, height: 60, density: 1.0, shape: "complex" },
		{ char: "石", width: 50, height: 55, density: 1.2, shape: "solid" },
		{ char: "竹", width: 35, height: 70, density: 0.4, shape: "tall" }
	];
	
	class KanjiObject {
		constructor(kanjiData, x, y) {
			this.kanjiData = kanjiData;
			this.char = kanjiData.char;
			this.x = x;
			this.y = y;
			this.width = kanjiData.width;
			this.height = kanjiData.height;
			this.vx = 0;
			this.vy = 0;
			this.rotation = 0;
			this.angularVelocity = 0;
			this.density = kanjiData.density;
			this.mass = this.width * this.height * this.density * 0.001;
			this.isGrounded = false;
			this.isControlled = false;
			this.color = this.getKanjiColor();
			this.fontSize = Math.max(this.width * 0.8, 24);
		}
		
		getKanjiColor() {
			// 漢字の種類に応じて色を決定
			const colorMap = {
				'山': '#8B4513',
				'川': '#4169E1',
				'木': '#228B22',
				'火': '#DC143C',
				'水': '#00CED1',
				'金': '#FFD700',
				'土': '#DEB887',
				'石': '#696969'
			};
			return colorMap[this.char] || '#333333';
		}
		
		update() {
			if (!this.isControlled) {
				// 重力を適用
				this.vy += gravity * this.mass;
				
				// 速度制限
				this.vy = Math.min(this.vy, 15);
				this.vx = Math.max(-10, Math.min(10, this.vx));
				
				// 位置更新
				this.x += this.vx;
				this.y += this.vy;
				this.rotation += this.angularVelocity;
				
				// 摩擦
				this.vx *= friction;
				this.angularVelocity *= 0.99;
				
				// 地面との衝突
				if (this.y + this.height > groundY) {
					this.y = groundY - this.height;
					this.vy *= -bounceMultiplier;
					this.vx *= friction;
					this.isGrounded = true;
					
					if (Math.abs(this.vy) < 1) {
						this.vy = 0;
					}
				}
				
				// 左右の壁との衝突
				if (this.x < 0) {
					this.x = 0;
					this.vx *= -bounceMultiplier;
				} else if (this.x + this.width > gameWidth) {
					this.x = gameWidth - this.width;
					this.vx *= -bounceMultiplier;
				}
			}
		}
		
		checkCollision(other) {
			return this.x < other.x + other.width &&
				   this.x + this.width > other.x &&
				   this.y < other.y + other.height &&
				   this.y + this.height > other.y;
		}
		
		resolveCollision(other) {
			if (!this.checkCollision(other)) return;
			
			// 重心計算
			const centerX1 = this.x + this.width / 2;
			const centerY1 = this.y + this.height / 2;
			const centerX2 = other.x + other.width / 2;
			const centerY2 = other.y + other.height / 2;
			
			const dx = centerX1 - centerX2;
			const dy = centerY1 - centerY2;
			const distance = Math.sqrt(dx * dx + dy * dy);
			
			if (distance > 0) {
				// 分離
				const overlapX = (this.width + other.width) / 2 - Math.abs(dx);
				const overlapY = (this.height + other.height) / 2 - Math.abs(dy);
				
				if (overlapX < overlapY) {
					// 水平方向の分離
					const separation = overlapX / 2;
					if (dx > 0) {
						this.x += separation;
						other.x -= separation;
					} else {
						this.x -= separation;
						other.x += separation;
					}
					// 速度交換（簡易）
					const tempVx = this.vx;
					this.vx = other.vx * 0.8;
					other.vx = tempVx * 0.8;
				} else {
					// 垂直方向の分離
					const separation = overlapY / 2;
					if (dy > 0) {
						this.y += separation;
						other.y -= separation;
					} else {
						this.y -= separation;
						other.y += separation;
					}
					// 速度交換（簡易）
					const tempVy = this.vy;
					this.vy = other.vy * 0.6;
					other.vy = tempVy * 0.6;
				}
				
				// 回転速度追加
				this.angularVelocity += (Math.random() - 0.5) * 0.1;
				other.angularVelocity += (Math.random() - 0.5) * 0.1;
			}
		}
		
		draw() {
			ctx.save();
			ctx.translate(this.x + this.width / 2, this.y + this.height / 2);
			ctx.rotate(this.rotation);
			
			// 漢字の背景（影のような効果）
			ctx.fillStyle = 'rgba(0, 0, 0, 0.2)';
			ctx.fillRect(-this.width / 2 + 2, -this.height / 2 + 2, this.width, this.height);
			
			// 漢字の背景
			ctx.fillStyle = 'rgba(255, 255, 255, 0.9)';
			ctx.fillRect(-this.width / 2, -this.height / 2, this.width, this.height);
			
			// 境界線
			ctx.strokeStyle = this.color;
			ctx.lineWidth = 2;
			ctx.strokeRect(-this.width / 2, -this.height / 2, this.width, this.height);
			
			// 漢字テキスト
			ctx.fillStyle = this.color;
			ctx.font = `bold ${this.fontSize}px 'Noto Sans CJK JP', serif`;
			ctx.textAlign = 'center';
			ctx.textBaseline = 'middle';
			ctx.fillText(this.char, 0, 0);
			
			ctx.restore();
		}
	}
	
	function createNewKanji() {
		const kanjiData = kanjiList[Math.floor(Math.random() * kanjiList.length)];
		const x = mouseX - kanjiData.width / 2;
		const kanji = new KanjiObject(kanjiData, x, 50);
		kanji.isControlled = true;
		return kanji;
	}
	
	function dropCurrentKanji() {
		if (currentKanji) {
			currentKanji.isControlled = false;
			gameObjects.push(currentKanji);
			currentKanji = null;
			
			// 新しい漢字を生成
			setTimeout(() => {
				if (gameRunning) {
					currentKanji = createNewKanji();
				}
			}, 1000);
		}
	}
	
	function updatePhysics() {
		// 全オブジェクトの更新
		gameObjects.forEach(obj => obj.update());
		
		// 衝突検出と解決
		for (let i = 0; i < gameObjects.length; i++) {
			for (let j = i + 1; j < gameObjects.length; j++) {
				gameObjects[i].resolveCollision(gameObjects[j]);
			}
		}
		
		// 現在の漢字の位置更新
		if (currentKanji) {
			currentKanji.x = mouseX - currentKanji.width / 2;
			// 境界チェック
			currentKanji.x = Math.max(0, Math.min(gameWidth - currentKanji.width, currentKanji.x));
		}
	}
	
	function render() {
		// 背景をクリア
		ctx.fillStyle = '#f0f8ff';
		ctx.fillRect(0, 0, gameWidth, gameHeight);
		
		// 地面を描画
		ctx.fillStyle = '#8B4513';
		ctx.fillRect(0, groundY, gameWidth, gameHeight - groundY);
		
		// ガイドライン
		if (currentKanji) {
			ctx.strokeStyle = 'rgba(255, 0, 0, 0.5)';
			ctx.lineWidth = 2;
			ctx.setLineDash([5, 5]);
			ctx.beginPath();
			ctx.moveTo(currentKanji.x + currentKanji.width / 2, 0);
			ctx.lineTo(currentKanji.x + currentKanji.width / 2, gameHeight);
			ctx.stroke();
			ctx.setLineDash([]);
		}
		
		// 全オブジェクトを描画
		gameObjects.forEach(obj => obj.draw());
		
		// 現在の漢字を描画
		if (currentKanji) {
			currentKanji.draw();
		}
		
		// UI情報
		ctx.fillStyle = '#333';
		ctx.font = '16px Arial';
		ctx.textAlign = 'left';
		ctx.fillText(`漢字の数: ${gameObjects.length}`, 10, 30);
		ctx.fillText('クリックで漢字を落下させる', 10, 50);
		ctx.fillText('マウスで左右に移動', 10, 70);
	}
	
	function gameLoop(currentTime) {
		if (!gameRunning) return;
		
		const deltaTime = currentTime - lastTime;
		if (deltaTime > 16) { // 約60FPS
			updatePhysics();
			render();
			lastTime = currentTime;
		}
		
		animationId = requestAnimationFrame(gameLoop);
	}
	
	function startGame() {
		gameRunning = true;
		gameObjects = [];
		currentKanji = createNewKanji();
		lastTime = 0;
		animationId = requestAnimationFrame(gameLoop);
	}
	
	function stopGame() {
		gameRunning = false;
		if (animationId) {
			cancelAnimationFrame(animationId);
		}
	}
	
	function resetGame() {
		stopGame();
		gameObjects = [];
		currentKanji = null;
	}
	
	function handleMouseMove(event) {
		const rect = canvas.getBoundingClientRect();
		mouseX = event.clientX - rect.left;
	}
	
	function handleClick() {
		if (gameRunning && currentKanji) {
			dropCurrentKanji();
		}
	}
	
	onMount(() => {
		ctx = canvas.getContext('2d');
		groundY = gameHeight - 50;
		
		canvas.addEventListener('mousemove', handleMouseMove);
		canvas.addEventListener('click', handleClick);
		
		return () => {
			stopGame();
			canvas.removeEventListener('mousemove', handleMouseMove);
			canvas.removeEventListener('click', handleClick);
		};
	});
</script>

<div class="game-container">
	<h1>漢字タワーバトル</h1>
	
	<div class="game-controls">
		<button on:click={startGame} disabled={gameRunning}>ゲーム開始</button>
		<button on:click={stopGame} disabled={!gameRunning}>ゲーム停止</button>
		<button on:click={resetGame}>リセット</button>
	</div>
	
	<div class="game-info">
		<p>🎮 <strong>遊び方:</strong></p>
		<ul>
			<li>マウスを動かして漢字の落下位置を調整</li>
			<li>クリックして漢字を落下させる</li>
			<li>漢字を積み重ねてタワーを作ろう！</li>
			<li>形の異なる漢字の特性を活かして安定したタワーを目指そう</li>
		</ul>
	</div>
	
	<canvas 
		bind:this={canvas}
		width={gameWidth}
		height={gameHeight}
		class="game-canvas"
	></canvas>
	
	<div class="kanji-info">
		<h3>登場する漢字たち</h3>
		<div class="kanji-grid">
			{#each kanjiList as kanji}
				<div class="kanji-card" style="color: {kanji.char === '山' ? '#8B4513' : kanji.char === '川' ? '#4169E1' : kanji.char === '木' ? '#228B22' : kanji.char === '火' ? '#DC143C' : kanji.char === '水' ? '#00CED1' : kanji.char === '金' ? '#FFD700' : kanji.char === '土' ? '#DEB887' : kanji.char === '石' ? '#696969' : '#333333'}">
					<span class="kanji-char">{kanji.char}</span>
					<span class="kanji-shape">{kanji.shape}</span>
				</div>
			{/each}
		</div>
	</div>
</div>

<style>
	.game-container {
		display: flex;
		flex-direction: column;
		align-items: center;
		padding: 2rem;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		min-height: 100vh;
		color: white;
		font-family: 'Arial', sans-serif;
	}
	
	h1 {
		font-size: 2.5rem;
		margin-bottom: 1rem;
		text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
	}
	
	.game-controls {
		display: flex;
		gap: 1rem;
		margin-bottom: 1rem;
	}
	
	.game-controls button {
		padding: 0.75rem 1.5rem;
		font-size: 1rem;
		border: none;
		border-radius: 8px;
		background: rgba(255, 255, 255, 0.9);
		color: #333;
		cursor: pointer;
		transition: all 0.3s ease;
		font-weight: bold;
	}
	
	.game-controls button:hover:not(:disabled) {
		background: rgba(255, 255, 255, 1);
		transform: translateY(-2px);
		box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
	}
	
	.game-controls button:disabled {
		opacity: 0.5;
		cursor: not-allowed;
	}
	
	.game-info {
		background: rgba(255, 255, 255, 0.1);
		padding: 1rem;
		border-radius: 8px;
		margin-bottom: 1rem;
		max-width: 600px;
		backdrop-filter: blur(10px);
	}
	
	.game-info ul {
		margin: 0.5rem 0;
		padding-left: 1.5rem;
	}
	
	.game-info li {
		margin: 0.25rem 0;
	}
	
	.game-canvas {
		border: 3px solid rgba(255, 255, 255, 0.3);
		border-radius: 8px;
		background: white;
		cursor: crosshair;
		box-shadow: 0 8px 16px rgba(0, 0, 0, 0.3);
	}
	
	.kanji-info {
		margin-top: 2rem;
		max-width: 800px;
		text-align: center;
	}
	
	.kanji-info h3 {
		margin-bottom: 1rem;
		font-size: 1.5rem;
	}
	
	.kanji-grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(80px, 1fr));
		gap: 0.5rem;
		padding: 1rem;
		background: rgba(255, 255, 255, 0.1);
		border-radius: 8px;
		backdrop-filter: blur(10px);
	}
	
	.kanji-card {
		display: flex;
		flex-direction: column;
		align-items: center;
		padding: 0.5rem;
		background: rgba(255, 255, 255, 0.9);
		border-radius: 6px;
		color: #333;
	}
	
	.kanji-char {
		font-size: 1.5rem;
		font-weight: bold;
		margin-bottom: 0.25rem;
	}
	
	.kanji-shape {
		font-size: 0.7rem;
		opacity: 0.7;
		text-transform: capitalize;
	}
	
	@media (max-width: 768px) {
		.game-container {
			padding: 1rem;
		}
		
		h1 {
			font-size: 2rem;
		}
		
		.game-canvas {
			width: 100%;
			max-width: 400px;
			height: 300px;
		}
		
		.kanji-grid {
			grid-template-columns: repeat(auto-fit, minmax(60px, 1fr));
		}
	}
</style>
