<script>
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';
	
	let canvasContainer;
	let p5Instance;
	let isLoading = true;
	
	// p5.jsスケッチ関数
	const sketch = (p) => {
		// 変数定義
		let balls = [];
		let colorMode = 0;
		
		// setup関数 - 初期化
		p.setup = () => {
			// キャンバスを作成
			p.createCanvas(800, 600);
			p.colorMode(p.HSB, 360, 100, 100, 100);
			
			// 初期ボールを作成
			for (let i = 0; i < 5; i++) {
				balls.push({
					x: p.random(p.width),
					y: p.random(p.height),
					vx: p.random(-3, 3),
					vy: p.random(-3, 3),
					size: p.random(20, 60),
					hue: p.random(360),
					alpha: 80
				});
			}
		};
		
		// draw関数 - メインループ
		p.draw = () => {
			// 背景をクリア（淡いグラデーション効果）
			// p.background(220, 30, 15, 10);
			p.background(220, 30, 15, 100);
			
			// ボールを更新・描画
			for (let ball of balls) {
				// 位置更新
				ball.x += ball.vx;
				ball.y += ball.vy;
				
				// 境界での跳ね返り
				if (ball.x < ball.size/2 || ball.x > p.width - ball.size/2) {
					ball.vx *= -0.9;
					ball.x = p.constrain(ball.x, ball.size/2, p.width - ball.size/2);
				}
				if (ball.y < ball.size/2 || ball.y > p.height - ball.size/2) {
					ball.vy *= -0.9;
					ball.y = p.constrain(ball.y, ball.size/2, p.height - ball.size/2);
				}
				
				// 重力効果
				ball.vy += 0.1;
				
				// 摩擦
				ball.vx *= 0.99;
				ball.vy *= 0.99;
				
				// 色の変化
				ball.hue = (ball.hue + 1) % 360;
				
				// ボールを描画
				p.fill(ball.hue, 70, 90, ball.alpha);
				p.stroke(ball.hue, 80, 60);
				p.strokeWeight(2);
				p.ellipse(ball.x, ball.y, ball.size);
				
				// 中心に小さな点
				p.fill(ball.hue, 90, 100, 90);
				p.noStroke();
				p.ellipse(ball.x, ball.y, ball.size * 0.3);
			}
			
			// マウス追従エフェクト
			if (p.mouseX > 0 && p.mouseY > 0) {
				p.stroke(0, 0, 100, 50);
				p.strokeWeight(1);
				p.noFill();
				p.ellipse(p.mouseX, p.mouseY, 50 + p.sin(p.frameCount * 0.1) * 20);
			}
			
			// FPS表示
			p.fill(0, 0, 100);
			p.noStroke();
			p.textSize(16);
			p.text(`FPS: ${p.frameRate().toFixed(1)}`, 10, 20);
			p.text(`Balls: ${balls.length}`, 10, 40);
			p.text(`Frame: ${p.frameCount}`, 10, 60);
		};
		
		// マウスクリック時にボールを追加
		p.mousePressed = () => {
			if (p.mouseX > 0 && p.mouseX < p.width && p.mouseY > 0 && p.mouseY < p.height) {
				balls.push({
					x: p.mouseX,
					y: p.mouseY,
					vx: p.random(-5, 5),
					vy: p.random(-8, -2),
					size: p.random(15, 40),
					hue: p.random(360),
					alpha: 80
				});
			}
		};
		
		// キーボード入力
		p.keyPressed = () => {
			if (p.key === 'c' || p.key === 'C') {
				// ボールをクリア
				balls = [];
			} else if (p.key === 'r' || p.key === 'R') {
				// ボールをリセット
				balls = [];
				for (let i = 0; i < 5; i++) {
					balls.push({
						x: p.random(p.width),
						y: p.random(p.height),
						vx: p.random(-3, 3),
						vy: p.random(-3, 3),
						size: p.random(20, 60),
						hue: p.random(360),
						alpha: 80
					});
				}
			} else if (p.key === ' ') {
				// スペースキーで一時停止/再生
				if (p.isLooping()) {
					p.noLoop();
				} else {
					p.loop();
				}
			}
		};
		
		// ウィンドウリサイズ時
		p.windowResized = () => {
			// キャンバスサイズを調整（オプション）
			// p.resizeCanvas(p.windowWidth, p.windowHeight);
		};
	};
	
	onMount(async () => {
		// ブラウザ環境でのみ実行
		if (!browser) return;
		
		// p5.jsをCDNから動的にロード
		const script = document.createElement('script');
		script.src = 'https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.7.0/p5.min.js';
		script.onload = () => {
			console.log('p5.js loaded, canvasContainer:', canvasContainer);
			if (canvasContainer) {
				p5Instance = new window.p5(sketch, canvasContainer);
				isLoading = false;
				console.log('p5 instance created successfully');
			} else {
				console.error('canvasContainer is not available');
			}
		};
		document.head.appendChild(script);
		
		// クリーンアップ関数
		return () => {
			if (p5Instance) {
				p5Instance.remove();
			}
		};
	});
</script>

<svelte:head>
	<title>p5.js Canvas Demo</title>
	<meta name="description" content="Interactive p5.js canvas with bouncing balls" />
	<link rel="preload" href="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.7.0/p5.min.js" as="script" />
</svelte:head>

<div class="container">
	<header>
		<h1>p5.js Interactive Canvas</h1>
	</header>
	
	<div class="canvas-wrapper">
		<div bind:this={canvasContainer} class="canvas-container">
			{#if isLoading}
				<div class="loading">
					<div class="loading-spinner"></div>
					<p>p5.js を読み込み中...</p>
				</div>
			{/if}
		</div>
	</div>
	
	<div class="controls">
		<div class="control-section">
			<h3>📋 コントロール</h3>
			<ul>
				<li><strong>クリック:</strong> 新しいボールを追加</li>
				<li><strong>C キー:</strong> すべてのボールをクリア</li>
				<li><strong>R キー:</strong> ボールをリセット</li>
				<li><strong>スペース:</strong> アニメーションの一時停止/再生</li>
			</ul>
		</div>
	</div>
</div>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		font-family: 'Arial', sans-serif;
	}
	
	.container {
		min-height: 100vh;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
		padding: 2rem;
		display: flex;
		flex-direction: column;
		align-items: center;
	}
	
	header {
		text-align: center;
		margin-bottom: 2rem;
	}
	
	header h1 {
		font-size: 2.5rem;
		margin-bottom: 0.5rem;
		text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
	}
	
	header p {
		font-size: 1.2rem;
		opacity: 0.9;
		margin: 0;
	}
	
	.canvas-wrapper {
		background: rgba(255, 255, 255, 0.1);
		border-radius: 12px;
		padding: 1rem;
		margin-bottom: 2rem;
		backdrop-filter: blur(10px);
		border: 2px solid rgba(255, 255, 255, 0.2);
		box-shadow: 0 8px 16px rgba(0, 0, 0, 0.3);
	}
	
	.canvas-container {
		border-radius: 8px;
		overflow: hidden;
		box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
		width: 800px;
		height: 600px;
		position: relative;
	}
	
	.loading {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		background: rgba(255, 255, 255, 0.05);
		border-radius: 8px;
		border: 2px dashed rgba(255, 255, 255, 0.3);
		z-index: 1;
	}
	
	.loading-spinner {
		width: 50px;
		height: 50px;
		border: 4px solid rgba(255, 255, 255, 0.3);
		border-top: 4px solid #4ecdc4;
		border-radius: 50%;
		animation: spin 1s linear infinite;
		margin-bottom: 1rem;
	}
	
	@keyframes spin {
		0% { transform: rotate(0deg); }
		100% { transform: rotate(360deg); }
	}
	
	.loading p {
		color: rgba(255, 255, 255, 0.8);
		margin: 0;
		font-size: 1.1rem;
	}
	
	.controls {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
		gap: 2rem;
		max-width: 800px;
		width: 100%;
		margin-bottom: 2rem;
	}
	
	.control-section {
		background: rgba(255, 255, 255, 0.1);
		border-radius: 8px;
		padding: 1.5rem;
		backdrop-filter: blur(10px);
		border: 1px solid rgba(255, 255, 255, 0.2);
	}
	
	.control-section h3 {
		margin: 0 0 1rem 0;
		font-size: 1.3rem;
		color: #ffeb3b;
	}
	
	.control-section ul {
		list-style: none;
		padding: 0;
		margin: 0;
	}
	
	.control-section li {
		margin: 0.5rem 0;
		padding: 0.5rem;
		background: rgba(255, 255, 255, 0.05);
		border-radius: 4px;
		border-left: 3px solid rgba(255, 235, 59, 0.4);
	}
	
	.info {
		max-width: 800px;
		width: 100%;
		background: rgba(255, 255, 255, 0.1);
		border-radius: 8px;
		padding: 1.5rem;
		backdrop-filter: blur(10px);
		border: 1px solid rgba(255, 255, 255, 0.2);
	}
	
	.info h3 {
		margin: 0 0 1rem 0;
		font-size: 1.3rem;
		color: #ffeb3b;
		text-align: center;
	}
	
	.tech-grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
		gap: 1rem;
	}
	
	.tech-item {
		background: rgba(255, 255, 255, 0.05);
		padding: 1rem;
		border-radius: 6px;
		text-align: center;
		border: 1px solid rgba(255, 255, 255, 0.1);
	}
	
	.tech-item strong {
		color: #4ecdc4;
		display: block;
		margin-bottom: 0.5rem;
	}
	
	@media (max-width: 768px) {
		.container {
			padding: 1rem;
		}
		
		header h1 {
			font-size: 2rem;
		}
		
		.canvas-wrapper {
			padding: 0.5rem;
		}
		
		.controls {
			grid-template-columns: 1fr;
			gap: 1rem;
		}
		
		.control-section,
		.info {
			padding: 1rem;
		}
	}
	
	/* p5.jsキャンバスのスタイル調整 */
	:global(main) {
		border-radius: 8px;
	}
</style>
