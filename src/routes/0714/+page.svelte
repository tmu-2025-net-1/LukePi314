<script>
	import { onMount } from 'svelte';
	
	let container;
	let mouseX = 0;
	let mouseY = 0;
	let letters = [];
	let isMouseDown = false;
	let circleElement;
	
	// パフォーマンス最適化のための変数
	let animationId = null;
	let isAnimating = false;
	let lastAnimationTime = 0;
	let mouseMovementQueue = [];
	let lastMouseUpdate = 0;
	let fps = 0;
	let frameCount = 0;
	let lastFPSUpdate = 0;
	let activeLetters = 0;
	let showDebugInfo = false;
	let lettersNeedingUpdate = new Set();
	let cachedPositions = new Map();
	let lastPositionUpdate = 0;
	let frameSkipCount = 0;
	
	const title = "ようこそSvelteKitへ！このインタラクティブデモでは、マウスの動きに合わせて文字が楽しく反応します。斥力エフェクトを体験してみてください。SvelteKitの可能性を遊びながら感じてみましょう！";
    const subtitle = "Move your mouse near the text above to see the letters scatter away, then gently return to their original positions. This is a fun way to explore interactive UI possibilities with SvelteKit.";
	
	onMount(() => {
		// 文字を個別の要素として初期化
		initializeLetters();
		
		// マウスの動きを追跡（デバウンス付き）
		window.addEventListener('mousemove', throttledMouseMove);
		window.addEventListener('mousedown', handleMouseDown);
		window.addEventListener('mouseup', handleMouseUp);
		
		// アニメーションループを開始
		startAnimationLoop();
		
		return () => {
			window.removeEventListener('mousemove', throttledMouseMove);
			window.removeEventListener('mousedown', handleMouseDown);
			window.removeEventListener('mouseup', handleMouseUp);
			if (animationId) {
				cancelAnimationFrame(animationId);
			}
		};
	});

	// スロットルされたマウス移動ハンドラー
	function throttledMouseMove(event) {
		const now = Date.now();
		if (now - lastMouseUpdate > 16) { // 60FPS制限に変更
			mouseX = event.clientX;
			mouseY = event.clientY;
			lastMouseUpdate = now;
			
			// アニメーションフレームをリクエスト
			if (!isAnimating) {
				isAnimating = true;
				animationId = requestAnimationFrame(updateLetters);
			}
		}
	}

	// アニメーションループの開始
	function startAnimationLoop() {
		function loop() {
			// FPSが低い場合はフレームスキップ
			if (fps < 30 && frameSkipCount % 2 === 0) {
				frameSkipCount++;
				animationId = requestAnimationFrame(loop);
				return;
			}
			frameSkipCount++;
			
			updateLetters();
			animationId = requestAnimationFrame(loop);
		}
		loop();
	}

	// 文字の状態を更新する最適化された関数
	function updateLetters() {
		if (!letters.length) return;
		
		// FPS計算
		const now = performance.now();
		frameCount++;
		if (now - lastFPSUpdate > 1000) {
			fps = Math.round(frameCount * 1000 / (now - lastFPSUpdate));
			frameCount = 0;
			lastFPSUpdate = now;
		}
		
		// 位置キャッシュを更新（500msごと、またはFPSが低い場合は1秒ごと）
		const cacheUpdateInterval = fps < 30 ? 1000 : 500;
		if (now - lastPositionUpdate > cacheUpdateInterval) {
			updatePositionCache();
			lastPositionUpdate = now;
		}
		
		let needsUpdate = false;
		let currentActiveLetters = 0;
		const maxDistance = isMouseDown ? 300 : 150;
		
		// 近い文字のみを効率的に処理
		const nearbyLetters = findNearbyLetters(maxDistance);
		
		// 以前アクティブだった文字をリセット
		lettersNeedingUpdate.forEach(index => {
			const letter = letters[index];
			if (!nearbyLetters.has(index)) {
				resetLetterToOriginal(letter);
			}
		});
		
		// 新しくアクティブな文字を処理
		nearbyLetters.forEach(index => {
			const letter = letters[index];
			const position = cachedPositions.get(index);
			if (position) {
				const dx = position.x - mouseX;
				const dy = position.y - mouseY;
				const distance = Math.sqrt(dx * dx + dy * dy);
				
				if (distance < maxDistance && distance > 0) {
					currentActiveLetters++;
					needsUpdate = true;
					applyLetterEffect(letter, dx, dy, distance, maxDistance);
				}
			}
		});
		
		lettersNeedingUpdate = nearbyLetters;
		activeLetters = currentActiveLetters;
		
		// 円の位置を更新
		updateCirclePosition();
		
		isAnimating = needsUpdate;
	}
	
	// 位置キャッシュを更新
	function updatePositionCache() {
		letters.forEach((letter, index) => {
			const rect = letter.element.getBoundingClientRect();
			cachedPositions.set(index, {
				x: rect.left + rect.width / 2,
				y: rect.top + rect.height / 2
			});
		});
	}
	
	// 近くの文字を効率的に見つける
	function findNearbyLetters(maxDistance) {
		const nearby = new Set();
		const searchDistance = maxDistance + 50; // 少し余裕を持たせる
		
		for (let i = 0; i < letters.length; i++) {
			const position = cachedPositions.get(i);
			if (position) {
				const dx = position.x - mouseX;
				const dy = position.y - mouseY;
				const distance = Math.sqrt(dx * dx + dy * dy);
				
				if (distance < searchDistance) {
					nearby.add(i);
				}
			}
		}
		
		return nearby;
	}
	
	// 文字を元の状態にリセット
	function resetLetterToOriginal(letter) {
		// 色の復元処理をコメントアウト（パフォーマンス向上のため）
		/*
		if (letter.element.classList.contains('title-letter')) {
			letter.element.style.color = '#ffffff';
			letter.element.style.textShadow = '2px 2px 4px rgba(0, 0, 0, 0.3)';
		} else {
			letter.element.style.color = '#e0e0e0';
			letter.element.style.textShadow = '1px 1px 2px rgba(0, 0, 0, 0.2)';
		}
		*/
		
		// transformのみリセット（軽量化）
		letter.element.style.transition = 'transform 6.0s ease-out';
		letter.element.style.transform = 'translate(0px, 0px)';
		letter.inRange = false;
	}

	// 個別の文字にエフェクトを適用する関数
	function applyLetterEffect(letter, dx, dy, distance, maxDistance) {
		if (distance < maxDistance && distance > 0) {
			// 斥力の強さ（距離に反比例）
			const force = (maxDistance - distance) / maxDistance;
			// クリック時は斥力も強くする
			const pushStrength = force * (isMouseDown ? 80 : 50); // 最大移動距離
			
			// 正規化された方向ベクトル（斥力ベクトル）
			const normalizedX = dx / distance;
			const normalizedY = dy / distance;
			
			// 文字を押し出す
			const offsetX = normalizedX * pushStrength;
			const offsetY = normalizedY * pushStrength;
			
			// 現在の変形を取得（キャッシュされた値を使用）
			let currentX = letter.cachedX || 0;
			let currentY = letter.cachedY || 0;
			
			// 現在地からoffsetへのベクトル
			const moveVectorX = offsetX - currentX;
			const moveVectorY = offsetY - currentY;
			
			// 斥力ベクトルとの内積を計算
			const dotProduct = moveVectorX * normalizedX + moveVectorY * normalizedY;
			
			// 内積が正の時は素早く、非正の時はゆっくり
			const transitionSpeed = dotProduct > 0 ? '0.2s' : '6.0s';
			
			// 色変更処理をコメントアウト（パフォーマンス向上のため）
			/*
			// 内積に基づいて色を変更
			if (dotProduct > 0) {
				// 内積が正の場合：緑色（素早く移動）
				letter.element.style.color = '#4ecdc4';
				letter.element.style.textShadow = '0 0 8px rgba(78, 205, 196, 0.6)';
			} else {
				// 内積が非正の場合：赤色（ゆっくり移動）
				letter.element.style.color = '#ff6b6b';
				letter.element.style.textShadow = '0 0 8px rgba(255, 107, 107, 0.6)';
			}
			*/
			
			// transformのみ適用（軽量化）
			letter.element.style.transition = `transform ${transitionSpeed} ease-out`;
			letter.element.style.transform = `translate(${offsetX}px, ${offsetY}px)`;
			
			// 位置をキャッシュ
			letter.cachedX = offsetX;
			letter.cachedY = offsetY;
			letter.inRange = true;
		} else {
			resetLetterToOriginal(letter);
		}
	}

	function lerp(a, b, t) {
		return a + (b - a) * t;
	}
	
	function initializeLetters() {
		const titleElement = container.querySelector('.title');
		const subtitleElement = container.querySelector('.subtitle');
		
		// タイトルの文字を処理
		createLetterElements(titleElement, title, 'title-letter');
		
		// サブタイトルの文字を処理
		createLetterElements(subtitleElement, subtitle, 'subtitle-letter');
	}
	
	function createLetterElements(element, text, className) {
		element.innerHTML = '';
		
		for (let i = 0; i < text.length; i++) {
			const char = text[i];
			const span = document.createElement('span');
			span.textContent = char === ' ' ? '\u00A0' : char; // スペースを非改行スペースに
			span.className = className;
			span.style.display = 'inline-block';
			span.style.transition = 'none'; // 初期状態ではtransitionなし
			element.appendChild(span);
			
			letters.push({
				element: span,
				originalX: 0,
				originalY: 0,
				currentX: 0,
				currentY: 0,
				inRange: false, // 影響範囲内にあるかどうかのフラグ
				cachedX: 0, // 変形のキャッシュ
				cachedY: 0
			});
		}
		
		// 初期位置を記録
		setTimeout(() => {
			letters.forEach((letter, index) => {
				const rect = letter.element.getBoundingClientRect();
				letter.originalX = rect.left + rect.width / 2;
				letter.originalY = rect.top + rect.height / 2;
				letter.currentX = letter.originalX;
				letter.currentY = letter.originalY;
				
				// 位置キャッシュも初期化
				cachedPositions.set(index, {
					x: letter.originalX,
					y: letter.originalY
				});
			});
		}, 100);
	}
	
	function handleMouseDown(event) {
		isMouseDown = true;
		mouseX = event.clientX;
		mouseY = event.clientY;
		updateCirclePosition();
		
		// 即座にアニメーションを開始
		if (!isAnimating) {
			isAnimating = true;
			animationId = requestAnimationFrame(updateLetters);
		}
	}
	
	function handleMouseUp(event) {
		isMouseDown = false;
		mouseX = event.clientX;
		mouseY = event.clientY;
		updateCirclePosition();
		
		// 即座にアニメーションを開始
		if (!isAnimating) {
			isAnimating = true;
			animationId = requestAnimationFrame(updateLetters);
		}
	}

	function updateCirclePosition() {
		if (circleElement) {
			const radius = isMouseDown ? 300 : 150;
			circleElement.style.left = `${mouseX - radius}px`;
			circleElement.style.top = `${mouseY - radius}px`;
			circleElement.style.width = `${radius * 2}px`;
			circleElement.style.height = `${radius * 2}px`;
		}
	}
	
	// デバッグ情報の表示切り替え
	function toggleDebugInfo() {
		showDebugInfo = !showDebugInfo;
	}
</script>

<div bind:this={container} class="container">
	<!-- 影響範囲を示す円 -->
	<div bind:this={circleElement} class="influence-circle"></div>
	
	<!-- デバッグ情報パネル -->
	{#if showDebugInfo}
		<div class="debug-panel">
			<h3>パフォーマンス情報</h3>
			<div class="debug-info">
				<div class="fps-indicator" class:low-fps={fps < 30}>FPS: {fps}</div>
				<div>アクティブ文字数: {activeLetters} / {letters.length}</div>
				<div>処理対象文字数: {lettersNeedingUpdate.size}</div>
				<div>マウス位置: ({mouseX}, {mouseY})</div>
				<div>クリック状態: {isMouseDown ? 'ON' : 'OFF'}</div>
				<div>アニメーション中: {isAnimating ? 'YES' : 'NO'}</div>
				<div>フレームスキップ: {frameSkipCount}</div>
			</div>
		</div>
	{/if}
	
	<!-- デバッグ情報切り替えボタン -->
	<button class="debug-toggle" on:click={toggleDebugInfo}>
		{showDebugInfo ? '📊 情報を隠す' : '📊 情報を表示'}
	</button>
	
	<h1 class="title" aria-label="Interactive title text"></h1>
	<p class="subtitle" aria-label="Interactive subtitle text"></p>
	<div class="instructions">
		<p>マウスを文字の近くに移動して、斥力エフェクトを体験してください</p>
		<p><strong>クリックしながら移動すると、より強力な斥力効果を体験できます！</strong></p>
		<div class="performance-info">
			<h3>パフォーマンス最適化機能</h3>
			<ul>
				<li>🚀 requestAnimationFrameによる滑らかな描画</li>
				<li>⚡ 変更が必要な文字のみを更新</li>
				<li>🎯 スロットリング機能でCPU使用率を削減</li>
				<li>📊 リアルタイム状態管理で効率的な処理</li>
				<li>🎨 色変更処理を無効化（パフォーマンス重視）</li>
			</ul>
		</div>
		<div class="color-legend" style="opacity: 0.5;">
			<div class="legend-item">
				<span style="color: #999; font-size: 0.8rem;">※ 色分け機能は現在無効化されています（パフォーマンス向上のため）</span>
			</div>
			<!--
			<div class="legend-item">
				<span class="color-sample fast"></span>
				<span>緑色：素早く移動（内積 > 0）</span>
			</div>
			<div class="legend-item">
				<span class="color-sample slow"></span>
				<span>赤色：ゆっくり移動（内積 ≤ 0）</span>
			</div>
			-->
		</div>
		<p>Visit <a href="https://svelte.dev/docs/kit">svelte.dev/docs/kit</a> to read the documentation</p>
	</div>
</div>

<style>
	.container {
		min-height: 100vh;
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		padding: 2rem;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
		font-family: 'Arial', sans-serif;
	}
	
	.title {
		font-size: 3rem;
		font-weight: bold;
		margin-bottom: 1rem;
		text-align: center;
		cursor: default;
		user-select: none;
	}
	
	.subtitle {
		font-size: 1.5rem;
		margin-bottom: 2rem;
		text-align: center;
		cursor: default;
		user-select: none;
		color: #e0e0e0;
	}
	
	:global(.title-letter) {
		position: relative;
		color: #ffffff;
		text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
		will-change: transform, color, text-shadow;
		transform: translateZ(0); /* ハードウェアアクセラレーション */
	}
	
	:global(.subtitle-letter) {
		position: relative;
		color: #e0e0e0;
		text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.2);
		will-change: transform, color, text-shadow;
		transform: translateZ(0); /* ハードウェアアクセラレーション */
	}
	
	.instructions {
		text-align: center;
		max-width: 600px;
		line-height: 1.6;
	}
	
	.instructions p {
		margin-bottom: 1rem;
		opacity: 0.9;
	}
	
	.instructions a {
		color: #ffeb3b;
		text-decoration: underline;
		transition: color 0.3s ease;
	}
	
	.instructions a:hover {
		color: #fff59d;
	}
	
	.performance-info {
		margin: 1.5rem 0;
		padding: 1.5rem;
		background: rgba(0, 0, 0, 0.2);
		border-radius: 12px;
		border: 2px solid rgba(255, 255, 255, 0.1);
		backdrop-filter: blur(10px);
	}
	
	.performance-info h3 {
		margin: 0 0 1rem 0;
		color: #ffeb3b;
		font-size: 1.1rem;
		text-align: center;
	}
	
	.performance-info ul {
		list-style: none;
		padding: 0;
		margin: 0;
	}
	
	.performance-info li {
		margin: 0.5rem 0;
		padding: 0.5rem;
		background: rgba(255, 255, 255, 0.05);
		border-radius: 6px;
		font-size: 0.9rem;
		border-left: 3px solid rgba(255, 235, 59, 0.4);
	}
	
	.color-legend {
		margin: 1.5rem 0;
		padding: 1rem;
		background: rgba(255, 255, 255, 0.1);
		border-radius: 8px;
		backdrop-filter: blur(10px);
	}
	
	.legend-item {
		display: flex;
		align-items: center;
		justify-content: center;
		margin: 0.5rem 0;
		font-size: 0.9rem;
	}
	
	.color-sample {
		display: inline-block;
		width: 20px;
		height: 20px;
		border-radius: 50%;
		margin-right: 0.5rem;
		border: 2px solid rgba(255, 255, 255, 0.3);
	}
	
	.color-sample.fast {
		background-color: #4ecdc4;
		box-shadow: 0 0 8px rgba(78, 205, 196, 0.6);
	}
	
	.color-sample.slow {
		background-color: #ff6b6b;
		box-shadow: 0 0 8px rgba(255, 107, 107, 0.6);
	}
	
	.influence-circle {
		position: fixed;
		border: 2px solid rgba(255, 255, 255, 0.3);
		border-radius: 50%;
		pointer-events: none;
		z-index: 1000;
		transition: all 0.1s ease-out;
		background: radial-gradient(
			circle,
			rgba(255, 255, 255, 0.05) 0%,
			rgba(255, 255, 255, 0.02) 50%,
			transparent 100%
		);
		box-shadow: 
			0 0 20px rgba(255, 255, 255, 0.1),
			inset 0 0 20px rgba(255, 255, 255, 0.05);
	}
	
	.debug-panel {
		position: fixed;
		top: 20px;
		right: 20px;
		background: rgba(0, 0, 0, 0.8);
		backdrop-filter: blur(10px);
		border: 2px solid rgba(255, 255, 255, 0.2);
		border-radius: 12px;
		padding: 1rem;
		z-index: 1001;
		min-width: 250px;
		font-family: monospace;
	}
	
	.debug-panel h3 {
		margin: 0 0 0.5rem 0;
		color: #ffeb3b;
		font-size: 1rem;
		text-align: center;
	}
	
	.debug-info {
		display: flex;
		flex-direction: column;
		gap: 0.3rem;
	}
	
	.debug-info div {
		font-size: 0.8rem;
		color: #e0e0e0;
		padding: 0.2rem 0;
		border-bottom: 1px solid rgba(255, 255, 255, 0.1);
	}
	
	.debug-info div:last-child {
		border-bottom: none;
	}
	
	.fps-indicator {
		font-weight: bold;
	}
	
	.fps-indicator.low-fps {
		color: #ff6b6b !important;
		background: rgba(255, 107, 107, 0.1) !important;
	}
	
	.debug-toggle {
		position: fixed;
		top: 20px;
		left: 20px;
		background: rgba(0, 0, 0, 0.7);
		color: white;
		border: 2px solid rgba(255, 255, 255, 0.2);
		border-radius: 8px;
		padding: 0.5rem 1rem;
		font-size: 0.9rem;
		cursor: pointer;
		transition: all 0.3s ease;
		z-index: 1001;
		backdrop-filter: blur(5px);
	}
	
	.debug-toggle:hover {
		background: rgba(255, 255, 255, 0.1);
		border-color: rgba(255, 255, 255, 0.4);
		transform: scale(1.05);
	}
	
	@media (max-width: 768px) {
		.title {
			font-size: 2rem;
		}
		
		.subtitle {
			font-size: 1.2rem;
		}
		
		.container {
			padding: 1rem;
		}
		
		.debug-panel {
			top: 10px;
			right: 10px;
			min-width: 200px;
			font-size: 0.8rem;
		}
		
		.debug-toggle {
			top: 10px;
			left: 10px;
			padding: 0.3rem 0.8rem;
			font-size: 0.8rem;
		}
	}
</style>
