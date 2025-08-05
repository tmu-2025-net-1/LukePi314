<script>
	import { onMount } from 'svelte';
	
	let container;
	let canvas;
	let ctx;
	let player1Input = '';
	let player2Input = '';
	let player1Kanji = null;
	let player2Kanji = null;
	let gameStarted = false;
	let winner = null;
	let animationId;
	
	// ゲームオブジェクト
	let gameObjects = [];
	
	onMount(() => {
		initializeCanvas();
		gameLoop();
	});
	
	function initializeCanvas() {
		canvas.width = 800;
		canvas.height = 600;
		ctx = canvas.getContext('2d');
	}
	
	async function getKanjiData(kanji) {
		// 漢字のUnicodeコードポイントを取得
		const codePoint = kanji.codePointAt(0);
		const hexCode = codePoint.toString(16).padStart(4, '0');
		
		try {
			// GlyphWikiから漢字データを取得を試行
			// 実際のAPIが利用できない場合はダミーデータを生成
			return generateKanjiData(kanji, codePoint);
		} catch (error) {
			console.error('API error:', error);
			return generateKanjiData(kanji, codePoint);
		}
	}
	
	function generateKanjiData(kanji, codePoint) {
		// 漢字の複雑さを文字コードから推定
		const complexity = (codePoint % 10) + 5;
		const strokeCount = (codePoint % 20) + 3;
		
		// 漢字の形状を近似する頂点を生成
		const vertices = generateKanjiVertices(kanji, complexity);
		
		return {
			character: kanji,
			complexity: complexity,
			strokeCount: strokeCount,
			vertices: vertices,
			codePoint: codePoint
		};
	}
	
	function generateKanjiVertices(kanji, complexity) {
		const vertices = [];
		const baseRadius = 40;
		const variations = 8 + complexity * 2;
		
		// 漢字の特徴に基づいて頂点を生成
		for (let i = 0; i < variations; i++) {
			const angle = (i / variations) * Math.PI * 2;
			const radiusVariation = baseRadius + Math.sin(angle * complexity) * 15;
			const noise = (Math.sin(angle * 7) + Math.cos(angle * 3)) * 5;
			
			vertices.push({
				x: Math.cos(angle) * (radiusVariation + noise),
				y: Math.sin(angle) * (radiusVariation + noise)
			});
		}
		
		return vertices;
	}
	
	function calculateSharpness(vertices) {
		let totalSharpness = 0;
		let sharpPoints = 0;
		
		for (let i = 0; i < vertices.length; i++) {
			const prev = vertices[(i - 1 + vertices.length) % vertices.length];
			const curr = vertices[i];
			const next = vertices[(i + 1) % vertices.length];
			
			// ベクトルを計算
			const v1 = { x: curr.x - prev.x, y: curr.y - prev.y };
			const v2 = { x: next.x - curr.x, y: next.y - curr.y };
			
			// 内積を使って角度を計算
			const dot = v1.x * v2.x + v1.y * v2.y;
			const mag1 = Math.sqrt(v1.x * v1.x + v1.y * v1.y);
			const mag2 = Math.sqrt(v2.x * v2.x + v2.y * v2.y);
			
			if (mag1 > 0 && mag2 > 0) {
				const cosAngle = dot / (mag1 * mag2);
				const angle = Math.acos(Math.max(-1, Math.min(1, cosAngle)));
				
				// 鋭い角度ほど高いスコア（π - angle）
				const sharpness = Math.PI - angle;
				if (sharpness > Math.PI / 3) { // 60度以下の角度を鋭いとみなす
					totalSharpness += sharpness;
					sharpPoints++;
				}
			}
		}
		
		return {
			total: totalSharpness,
			points: sharpPoints,
			average: sharpPoints > 0 ? totalSharpness / sharpPoints : 0
		};
	}
	
	async function prepareKanji(kanji, isPlayer1 = true) {
		const kanjiData = await getKanjiData(kanji);
		const sharpnessData = calculateSharpness(kanjiData.vertices);
		
		const kanjiObject = {
			character: kanji,
			data: kanjiData,
			vertices: kanjiData.vertices,
			sharpness: sharpnessData,
			attackPower: Math.floor(sharpnessData.total * 20 + kanjiData.strokeCount * 5),
			defense: Math.floor(kanjiData.complexity * 3),
			// 物理プロパティ
			x: isPlayer1 ? 150 : 650,
			y: 300,
			vx: isPlayer1 ? 3 : -3,
			vy: 0,
			rotation: 0,
			angularVelocity: (Math.random() - 0.5) * 0.2,
			color: isPlayer1 ? '#ff6b6b' : '#4ecdc4',
			health: 100
		};
		
		if (isPlayer1) {
			player1Kanji = kanjiObject;
		} else {
			player2Kanji = kanjiObject;
		}
		
		return kanjiObject;
	}
	
	function drawKanji(kanji) {
		ctx.save();
		ctx.translate(kanji.x, kanji.y);
		ctx.rotate(kanji.rotation);
		
		// 頂点を描画
		ctx.beginPath();
		ctx.strokeStyle = kanji.color;
		ctx.fillStyle = kanji.color + '40'; // 半透明
		ctx.lineWidth = 3;
		
		if (kanji.vertices.length > 0) {
			ctx.moveTo(kanji.vertices[0].x, kanji.vertices[0].y);
			for (let i = 1; i < kanji.vertices.length; i++) {
				ctx.lineTo(kanji.vertices[i].x, kanji.vertices[i].y);
			}
			ctx.closePath();
		}
		
		ctx.fill();
		ctx.stroke();
		
		// 漢字を中央に描画
		ctx.fillStyle = '#ffffff';
		ctx.font = 'bold 32px serif';
		ctx.textAlign = 'center';
		ctx.textBaseline = 'middle';
		ctx.fillText(kanji.character, 0, 0);
		
		// 鋭い頂点を強調
		ctx.fillStyle = '#ffff00';
		kanji.vertices.forEach((vertex, i) => {
			const prev = kanji.vertices[(i - 1 + kanji.vertices.length) % kanji.vertices.length];
			const next = kanji.vertices[(i + 1) % kanji.vertices.length];
			
			const v1 = { x: vertex.x - prev.x, y: vertex.y - prev.y };
			const v2 = { x: next.x - vertex.x, y: next.y - vertex.y };
			
			const dot = v1.x * v2.x + v1.y * v2.y;
			const mag1 = Math.sqrt(v1.x * v1.x + v1.y * v1.y);
			const mag2 = Math.sqrt(v2.x * v2.x + v2.y * v2.y);
			
			if (mag1 > 0 && mag2 > 0) {
				const cosAngle = dot / (mag1 * mag2);
				const angle = Math.acos(Math.max(-1, Math.min(1, cosAngle)));
				
				if (Math.PI - angle > Math.PI / 3) { // 鋭い角度
					ctx.beginPath();
					ctx.arc(vertex.x, vertex.y, 4, 0, Math.PI * 2);
					ctx.fill();
				}
			}
		});
		
		ctx.restore();
	}
	
	function updatePhysics(kanji) {
		// 位置更新
		kanji.x += kanji.vx;
		kanji.y += kanji.vy;
		kanji.rotation += kanji.angularVelocity;
		
		// 重力
		kanji.vy += 0.2;
		
		// 境界での反発
		if (kanji.x < 50 || kanji.x > canvas.width - 50) {
			kanji.vx *= -0.8;
			kanji.x = Math.max(50, Math.min(canvas.width - 50, kanji.x));
		}
		if (kanji.y < 50 || kanji.y > canvas.height - 50) {
			kanji.vy *= -0.8;
			kanji.y = Math.max(50, Math.min(canvas.height - 50, kanji.y));
		}
		
		// 空気抵抗
		kanji.vx *= 0.99;
		kanji.vy *= 0.99;
		kanji.angularVelocity *= 0.98;
	}
	
	function checkCollision(kanji1, kanji2) {
		const dx = kanji1.x - kanji2.x;
		const dy = kanji1.y - kanji2.y;
		const distance = Math.sqrt(dx * dx + dy * dy);
		
		return distance < 80; // 衝突判定の距離
	}
	
	function handleCollision(kanji1, kanji2) {
		// 速度ベクトルの大きさを計算
		const speed1 = Math.sqrt(kanji1.vx * kanji1.vx + kanji1.vy * kanji1.vy);
		const speed2 = Math.sqrt(kanji2.vx * kanji2.vx + kanji2.vy * kanji2.vy);
		
		// ダメージ計算
		const damage1 = kanji1.attackPower * speed1 * 0.1;
		const damage2 = kanji2.attackPower * speed2 * 0.1;
		
		// ヘルス減少
		kanji2.health -= damage1;
		kanji1.health -= damage2;
		
		// 反発
		const dx = kanji2.x - kanji1.x;
		const dy = kanji2.y - kanji1.y;
		const distance = Math.sqrt(dx * dx + dy * dy);
		
		if (distance > 0) {
			const normalX = dx / distance;
			const normalY = dy / distance;
			
			const relativeVelocityX = kanji2.vx - kanji1.vx;
			const relativeVelocityY = kanji2.vy - kanji1.vy;
			
			const impulse = 2 * (normalX * relativeVelocityX + normalY * relativeVelocityY) / 2;
			
			kanji1.vx += impulse * normalX;
			kanji1.vy += impulse * normalY;
			kanji2.vx -= impulse * normalX;
			kanji2.vy -= impulse * normalY;
		}
		
		// 勝者判定
		if (kanji1.health <= 0 && kanji2.health <= 0) {
			winner = 'Draw';
		} else if (kanji1.health <= 0) {
			winner = kanji1 === player1Kanji ? 'Player 2' : 'Player 1';
		} else if (kanji2.health <= 0) {
			winner = kanji2 === player1Kanji ? 'Player 2' : 'Player 1';
		}
		
		if (winner) {
			gameStarted = false;
		}
	}
	
	function gameLoop() {
		// キャンバスをクリア
		ctx.fillStyle = '#1a1a2e';
		ctx.fillRect(0, 0, canvas.width, canvas.height);
		
		if (gameStarted && player1Kanji && player2Kanji) {
			// 物理更新
			updatePhysics(player1Kanji);
			updatePhysics(player2Kanji);
			
			// 衝突判定
			if (checkCollision(player1Kanji, player2Kanji)) {
				handleCollision(player1Kanji, player2Kanji);
			}
			
			// 描画
			drawKanji(player1Kanji);
			drawKanji(player2Kanji);
			
			// ヘルスバー描画
			drawHealthBar(player1Kanji, 50, 50, 'Player 1');
			drawHealthBar(player2Kanji, canvas.width - 250, 50, 'Player 2');
		}
		
		animationId = requestAnimationFrame(gameLoop);
	}
	
	function drawHealthBar(kanji, x, y, label) {
		ctx.fillStyle = '#ffffff';
		ctx.font = '16px Arial';
		ctx.fillText(label, x, y);
		
		ctx.fillStyle = '#333333';
		ctx.fillRect(x, y + 10, 200, 20);
		
		ctx.fillStyle = kanji.health > 50 ? '#4ecdc4' : kanji.health > 25 ? '#f39c12' : '#e74c3c';
		ctx.fillRect(x, y + 10, (kanji.health / 100) * 200, 20);
		
		ctx.strokeStyle = '#ffffff';
		ctx.strokeRect(x, y + 10, 200, 20);
		
		ctx.fillStyle = '#ffffff';
		ctx.font = '12px Arial';
		ctx.fillText(`HP: ${Math.max(0, Math.floor(kanji.health))}`, x + 5, y + 25);
		ctx.fillText(`攻撃力: ${kanji.attackPower}`, x, y + 45);
		ctx.fillText(`鋭さ: ${kanji.sharpness.points}点`, x, y + 60);
	}
	
	async function startGame() {
		if (!player1Input || !player2Input) {
			alert('両方の漢字を入力してください');
			return;
		}
		
		// 漢字データを準備
		await prepareKanji(player1Input, true);
		await prepareKanji(player2Input, false);
		
		gameStarted = true;
		winner = null;
	}
	
	function resetGame() {
		winner = null;
		gameStarted = false;
		player1Input = '';
		player2Input = '';
		player1Kanji = null;
		player2Kanji = null;
	}
</script>

<div class="game-container">
	<h1>🥋 漢字バトルゲーム ⚔️</h1>
	<p class="subtitle">漢字の鋭さが攻撃力に！角張った漢字ほど強力です</p>
	
	{#if !gameStarted && !winner}
		<div class="input-section">
			<div class="player-input">
				<h3>🔴 Player 1</h3>
				<input 
					bind:value={player1Input} 
					placeholder="漢字を入力" 
					maxlength="1"
					class="kanji-input player1"
					on:input={() => prepareKanji(player1Input, true)}
				/>
				{#if player1Kanji}
					<div class="kanji-stats">
						<p><strong>攻撃力:</strong> {player1Kanji.attackPower}</p>
						<p><strong>鋭い頂点:</strong> {player1Kanji.sharpness.points}個</p>
						<p><strong>複雑度:</strong> {player1Kanji.data.complexity}</p>
					</div>
				{/if}
			</div>
			
			<div class="vs-divider">
				<span>VS</span>
			</div>
			
			<div class="player-input">
				<h3>🔵 Player 2</h3>
				<input 
					bind:value={player2Input} 
					placeholder="漢字を入力" 
					maxlength="1"
					class="kanji-input player2"
					on:input={() => prepareKanji(player2Input, false)}
				/>
				{#if player2Kanji}
					<div class="kanji-stats">
						<p><strong>攻撃力:</strong> {player2Kanji.attackPower}</p>
						<p><strong>鋭い頂点:</strong> {player2Kanji.sharpness.points}個</p>
						<p><strong>複雑度:</strong> {player2Kanji.data.complexity}</p>
					</div>
				{/if}
			</div>
		</div>
		
		<button on:click={startGame} class="start-button" disabled={!player1Input || !player2Input}>
			⚔️ バトル開始！ ⚔️
		</button>
	{/if}
	
	{#if winner}
		<div class="result">
			<h2>🏆 {winner === 'Draw' ? '引き分け！' : `${winner} の勝利！`} 🏆</h2>
			<p>{winner === 'Draw' ? '両者相打ち！' : `より鋭い漢字が勝利しました！`}</p>
			<button on:click={resetGame} class="reset-button">
				🔄 もう一度プレイ
			</button>
		</div>
	{/if}
	
	<div class="canvas-container">
		<canvas bind:this={canvas}></canvas>
	</div>
	
	{#if gameStarted}
		<div class="game-info">
			<p>🔥 漢字同士がぶつかり合っています...</p>
			<p>⚡ 鋭い角度の漢字ほど攻撃力が高くなります！</p>
			<p>💨 速度も威力に影響します！</p>
		</div>
	{/if}
	
	<div class="instructions">
		<h3>📖 ゲームルール</h3>
		<ul>
			<li>🔺 漢字の鋭い角度が攻撃力を決定</li>
			<li>💥 衝突時の速度でダメージが変化</li>
			<li>❤️ ヘルスが0になったら敗北</li>
			<li>⚖️ 10秒後に残りヘルスで判定</li>
		</ul>
	</div>
</div>

<style>
	.game-container {
		max-width: 1000px;
		margin: 0 auto;
		padding: 2rem;
		text-align: center;
		font-family: 'Arial', sans-serif;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		min-height: 100vh;
		color: white;
	}
	
	h1 {
		font-size: 3rem;
		margin-bottom: 0.5rem;
		text-shadow: 2px 2px 4px rgba(0,0,0,0.5);
	}
	
	.subtitle {
		font-size: 1.2rem;
		margin-bottom: 2rem;
		opacity: 0.9;
	}
	
	.input-section {
		display: flex;
		justify-content: space-around;
		align-items: center;
		margin-bottom: 2rem;
		gap: 2rem;
	}
	
	.player-input {
		background: rgba(255, 255, 255, 0.1);
		padding: 2rem;
		border-radius: 15px;
		backdrop-filter: blur(10px);
		border: 1px solid rgba(255, 255, 255, 0.2);
		box-shadow: 0 8px 32px rgba(0,0,0,0.1);
		flex: 1;
	}
	
	.player-input h3 {
		margin-bottom: 1rem;
		font-size: 1.5rem;
	}
	
	.vs-divider {
		font-size: 2rem;
		font-weight: bold;
		color: #ffeb3b;
		text-shadow: 2px 2px 4px rgba(0,0,0,0.5);
	}
	
	.kanji-input {
		font-size: 3rem;
		text-align: center;
		width: 100px;
		height: 100px;
		border: 3px solid #ffffff;
		border-radius: 15px;
		background: rgba(255, 255, 255, 0.9);
		font-family: serif;
		color: #333;
		font-weight: bold;
		transition: all 0.3s ease;
	}
	
	.kanji-input:focus {
		outline: none;
		box-shadow: 0 0 20px rgba(255, 255, 255, 0.5);
		transform: scale(1.05);
	}
	
	.kanji-input.player1:focus {
		border-color: #ff6b6b;
		box-shadow: 0 0 20px rgba(255, 107, 107, 0.5);
	}
	
	.kanji-input.player2:focus {
		border-color: #4ecdc4;
		box-shadow: 0 0 20px rgba(78, 205, 196, 0.5);
	}
	
	.kanji-stats {
		margin-top: 1rem;
		font-size: 0.9rem;
		text-align: left;
	}
	
	.kanji-stats p {
		margin: 0.5rem 0;
		padding: 0.25rem 0;
		border-bottom: 1px solid rgba(255, 255, 255, 0.2);
	}
	
	.start-button, .reset-button {
		background: linear-gradient(45deg, #e74c3c, #c0392b);
		color: white;
		border: none;
		padding: 1.5rem 3rem;
		font-size: 1.3rem;
		border-radius: 30px;
		cursor: pointer;
		transition: all 0.3s ease;
		margin: 1rem;
		font-weight: bold;
		text-transform: uppercase;
		letter-spacing: 1px;
	}
	
	.start-button:hover:not(:disabled), .reset-button:hover {
		transform: translateY(-3px);
		box-shadow: 0 10px 25px rgba(231, 76, 60, 0.4);
	}
	
	.start-button:disabled {
		background: #95a5a6;
		cursor: not-allowed;
		transform: none;
	}
	
	.canvas-container {
		margin: 2rem 0;
		border: 3px solid rgba(255, 255, 255, 0.3);
		border-radius: 15px;
		overflow: hidden;
		box-shadow: 0 10px 30px rgba(0,0,0,0.3);
		background: #1a1a2e;
	}
	
	canvas {
		display: block;
		width: 100%;
		height: auto;
	}
	
	.result {
		background: linear-gradient(45deg, #2ecc71, #27ae60);
		color: white;
		padding: 2rem;
		border-radius: 15px;
		margin: 2rem 0;
		box-shadow: 0 10px 30px rgba(0,0,0,0.2);
	}
	
	.result h2 {
		margin-bottom: 1rem;
		font-size: 2.5rem;
	}
	
	.game-info {
		background: rgba(52, 152, 219, 0.8);
		color: white;
		padding: 1.5rem;
		border-radius: 15px;
		margin-top: 1rem;
		backdrop-filter: blur(10px);
	}
	
	.game-info p {
		margin: 0.5rem 0;
		font-size: 1.1rem;
	}
	
	.instructions {
		background: rgba(255, 255, 255, 0.1);
		padding: 2rem;
		border-radius: 15px;
		margin-top: 2rem;
		backdrop-filter: blur(10px);
		border: 1px solid rgba(255, 255, 255, 0.2);
	}
	
	.instructions h3 {
		margin-bottom: 1rem;
		font-size: 1.5rem;
	}
	
	.instructions ul {
		text-align: left;
		max-width: 500px;
		margin: 0 auto;
	}
	
	.instructions li {
		margin: 0.5rem 0;
		font-size: 1.1rem;
	}
	
	@media (max-width: 768px) {
		.input-section {
			flex-direction: column;
		}
		
		.player-input {
			width: 100%;
			max-width: 400px;
		}
		
		.vs-divider {
			transform: rotate(90deg);
		}
		
		h1 {
			font-size: 2rem;
		}
		
		.kanji-input {
			font-size: 2.5rem;
			width: 80px;
			height: 80px;
		}
	}
</style>
