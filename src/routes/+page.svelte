<script>
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';
    import { text } from '@sveltejs/kit';

	import TestText from '../../static/TestText.txt?raw';
	
	let canvasContainer;
	let p5Instance;
	let isLoading = true;

	// テキストオプション
	const textOptions = [
		{
			name: "我輩は猫である（短）",
			text: "吾輩は猫である。名前はまだ無い。どこで生れたかとんと見当がつかぬ。何でも薄暗いじめじめした所でニャーニャー泣いていた事だけは記憶している。\\吾輩はここで初めて人間というものを見た。しかもあとで聞くとそれは書生という人間中で一番獰悪な種族であったそうだ。\\この書生というのは時々我々を捕えて煮て食うという話である。"
		},
		{
			name: "我輩は猫である（長）",
			text: `吾輩は猫である。名前はまだ無い。\\どこで生れたかとんと見当がつかぬ。何でも薄暗いじめじめした所でニャーニャー泣いていた事だけは記憶している。\\吾輩はここで始めて人間というものを見た。しかもあとで聞くとそれは書生という人間中で一番獰悪な種族であったそうだ。\\この書生というのは時々我々を捕えて煮て食うという話である。しかしその当時は何という考もなかったから別段恐しいとも思わなかった。\\ただ彼の掌に載せられてスーと持ち上げられた時何だかフワフワした感じがあったばかりである。掌の上で少し落ちついて書生の顔を見たのがいわゆる人間というものの見始であろう。\\この時妙なものだと思った感じが今でも残っている。第一毛をもって装飾されべきはずの顔がつるつるしてまるで薬缶だ。\\その後猫にもだいぶ逢ったがこんな片輪には一度も出会わした事がない。のみならず顔の真中があまりに突起している。\\そうしてその穴の中から時々ぷうぷうと煙を吹く。どうも咽せぽくて実に弱った。これが人間の飲む煙草というものである事はようやくこの頃知った。`
		},
		{
			name: "外部ファイル",
			text: TestText
		}
	];
	
	let currentTextIndex = 2; // デフォルトは完全版
	let drawText = textOptions[currentTextIndex].text;
	let showDebugPanel = false; // デバッグパネルの表示/非表示
	let showControlPanels = false; // コントロールパネルの表示/非表示（デフォルト非表示）
	
	// ドラッグ機能用の変数
	let isDragging = false;
	let isResizing = false;
	let dragOffset = { x: 0, y: 0 };
	let panelPosition = { x: 10, y: 10 }; // 初期位置（%）
	let panelSize = { width: 400, height: 150 }; // 初期サイズ（px）
	
	// p5.jsスケッチ関数
	const sketch = (p) => {
		// 変数定義
		let chars = [];
		let colorMode = 0;
		let scrollForce = { x: 0, y: 0 }; // スクロール力を格納
		let scrollWaves = []; // スクロール波紋効果の配列（重複対応）
		let isMousePressed = false; // マウスプレス状態を追跡
		let ripples = []; // 水紋効果用の配列
		let collisionEnabled = false; // 衝突判定のオン/オフ
		let lastFrameTime = 0; // 前フレームの時間
		let deltaTime = 16.67; // デルタタイム（ミリ秒、60FPS基準で初期化）
		let isWindowFocused = true; // ウィンドウのフォーカス状態
		let lastFocusTime = 0; // 最後にフォーカスを得た時間
		
		// 自動波紋生成用の変数
		let autoRippleTimer = 0; // 自動波紋のタイマー
		let autoRippleInterval = 80; // 自動波紋の間隔（フレーム数、約2秒）
		let autoRipplesEnabled = true; // 自動波紋のオン/オフ
		
		// パフォーマンス管理用変数
		let maxRipples = 10; // 同時に存在できる水紋の最大数
		let maxScrollWaves = 10; // 同時に存在できるスクロール波紋の最大数
		
		// マウス円のアニメーション用変数
		let currentCircleRadius = 20; // 現在の円の半径
		let targetCircleRadius = 20; // 目標の円の半径
		let currentCircleHue = 200; // 現在の円の色相
		let targetCircleHue = 200; // 目標の円の色相
		let currentCircleAlpha = 50; // 現在の円の透明度
		let targetCircleAlpha = 50; // 目標の円の透明度
		
		// setup関数 - 初期化
		p.setup = () => {
			// キャンバスを画面全体に作成
			p.createCanvas(p.windowWidth, p.windowHeight);
			p.colorMode(p.HSB, 360, 100, 100, 100);
			
			let xOffset = -5; // 初期位置のオフセット
			let yOffset = p.height / 4; // 垂直位置は中央に固定
			// 初期ボールを作成
			for (let i = 0; i < drawText.length; i++) {
				let c = drawText[i];
				if (i > 0) {
					// 前の文字と現在の文字のペアの幅
					let pair = drawText[i - 1] + drawText[i];
					let pairWidth = p.textWidth(pair);
					if (c === '\\') {
						xOffset = -5; // 改行時は初期位置に戻す
						yOffset += 24 * 1; // 次の行は少し下にずらす
						console.log(`New line at index ${i}, resetting xOffset to ${xOffset} and increasing yOffset to ${yOffset}`);
						continue;
					}

					// 前の文字の幅
					let prevWidth = p.textWidth(drawText[i - 1]);

					// 現在の文字の幅
					let currWidth = p.textWidth(drawText[i]);

					// カーニング差分（マイナスなら詰める）
					let kerning = pairWidth - (prevWidth + currWidth);
					console.log(`Pair: ${pair}, Kerning: ${kerning}`);
					// 改行直後はxOffsetを進めない
					if (drawText[i - 1] !== '\\') {
						xOffset += kerning;
						xOffset += p.textWidth(drawText[i]) + 12;
					}
				}
				if (c === '\\') {
					// エスケープ文字は無視
					continue;
				}
				chars.push({
					// x: p.random(p.width),
					// y: p.random(p.height),
					// vx: p.random(-3, 3),
					// vy: p.random(-3, 3),
					// size: p.random(20, 60),
					// hue: p.random(360),
					// alpha: 80
					text: c,
					originalText: c, // 元の文字を保持
					x: xOffset,
					y: yOffset,
					vx: 0,
					vy: 0,
					originalX: xOffset, // 元の位置を記録
					originalY: yOffset, // 元の位置を記録
					size: 16,
					hue: p.random(360),
					alpha: 80,
					rotation: 0, // 回転角度（ラジアン）
					angularVelocity: 0, // 角速度
					depth: 0, // 深度（水紋効果用）
					maxDepth: 10, // 最大深度
					isTitle: false // 通常の文字
				});
			}
			
			// タイトル文字「文字の波」を画面上部中央に配置
			const titleText = "文字の波";
			const titleSize = 48; // 大きめのサイズ
			p.textSize(titleSize); // タイトルサイズを設定して幅を測定
			const titleY = 80; // 画面上部からの距離
			
			let titleXOffset = (p.width - p.textWidth(titleText)) / 2; // 中央揃えの開始位置
			
			for (let i = 0; i < titleText.length; i++) {
				let char = titleText[i];
				let charWidth = p.textWidth(char);
				
				chars.push({
					text: char,
					originalText: char, // 元の文字を保持
					x: titleXOffset + charWidth / 2, // 文字の中心位置
					y: titleY,
					vx: 0,
					vy: 0,
					originalX: titleXOffset + charWidth / 2,
					originalY: titleY,
					size: titleSize,
					hue: 200, // 青系の色でタイトルを区別
					alpha: 90,
					rotation: 0,
					angularVelocity: 0,
					depth: 0,
					maxDepth: 15, // タイトル文字は少し深い効果
					isTitle: true // タイトル文字であることを示すフラグ
				});
				
				titleXOffset += charWidth + 4; // 次の文字の位置（少し間隔を空ける）
			}
		};
		
		// draw関数 - メインループ
		p.draw = () => {
			// デルタタイムの計算（p5.jsのdeltaTimeを使用）
			deltaTime = p.deltaTime; // p5.jsの組み込みdeltaTime（ミリ秒）
			
			// デルタタイムの上限を設定（バックグラウンド復帰時の異常値を防ぐ）
			const maxDeltaTime = 33.33; // 30FPS相当の最大デルタタイム
			if (deltaTime > maxDeltaTime) {
				deltaTime = maxDeltaTime;
			}
			
			// ウィンドウフォーカス復帰直後は更に制限を強化
			if (!isWindowFocused || (p.millis() - lastFocusTime) < 100) {
				deltaTime = Math.min(deltaTime, 16.67); // 60FPS相当に制限
			}
			
			// デルタタイムを60FPS基準で正規化（1.0 = 60FPS）
			let deltaTimeNormalized = deltaTime / 16.67;
			
			// 背景をクリア（淡いグラデーション効果）
			// p.background(220, 30, 15, 10);
			p.background(220, 30, 15, 100);
			// p.background(220, 80, 40, 100);
			
			// 自動波紋生成
			if (autoRipplesEnabled) {
				autoRippleTimer += deltaTimeNormalized;
				if (autoRippleTimer >= autoRippleInterval && ripples.length < maxRipples) {
					// ランダムな座標に波紋を生成（最大数制限あり）
					let randomX = p.random(100, p.width - 100); // 画面端から少し内側
					let randomY = p.random(100, p.height - 100);
					
					ripples.push({
						x: randomX,
						y: randomY,
						radius: 0,
						speed: p.random(6, 10), // 少し速度にランダム性を追加
						intensity: p.random(0.8, 1.2), // 強度にもランダム性を追加
						age: 0
					});
					
					autoRippleTimer = 0; // タイマーをリセット
				}
			}
			
			// 水紋効果の更新
			for (let i = ripples.length - 1; i >= 0; i--) {
				let ripple = ripples[i];
				ripple.radius += ripple.speed * deltaTimeNormalized; // デルタタイムを適用
				ripple.age += 1 * deltaTimeNormalized;
				
				// 水紋の範囲内の文字に深度を適用
				for (let char of chars) {
					let distance = p.dist(char.x, char.y, ripple.x, ripple.y);
					let rippleEdge = ripple.radius;
					let rippleWidth = 50; // 水紋の幅
					
					// 水紋の前縁付近の文字に深度を設定
					if (distance >= rippleEdge - rippleWidth && distance <= rippleEdge + rippleWidth) {
						let rippleIntensity = 1 - Math.abs(distance - rippleEdge) / rippleWidth;
						let depthValue = char.maxDepth * rippleIntensity * ripple.intensity;
						char.depth = Math.max(char.depth, depthValue);
						
						// 水紋による弱い放射状の力を追加
						if (distance > 0) {
							let forceDirection = {
								x: (char.x - ripple.x) / distance,
								y: (char.y - ripple.y) / distance
							};
							let forceStrength = rippleIntensity * 0.04 * deltaTimeNormalized; // デルタタイムを適用
							
							char.vx += forceDirection.x * forceStrength;
							char.vy += forceDirection.y * forceStrength;
							
							// 水紋による回転力も追加
							let rotationForce = rippleIntensity * 0.015 * deltaTimeNormalized; // デルタタイムを適用
							char.angularVelocity += (Math.random() - 0.5) * rotationForce;
						}
					}
				}
				
				// 水紋が画面外に出たら削除
				if (ripple.radius > p.width + p.height || ripple.age > 300) {
					ripples.splice(i, 1);
				}
			}
			
			// ボールを更新・描画
			for (let char of chars) {
				// depth（深度）の減衰
				if (char.depth > 0) {
					char.depth -= 0.5 * deltaTimeNormalized; // デルタタイムを適用
					if (char.depth < 0) char.depth = 0;
				}
				
				// スクロール波紋による力の適用（重複対応）
				for (let i = scrollWaves.length - 1; i >= 0; i--) {
					let scrollWave = scrollWaves[i];
					let currentTime = p.millis();
					let elapsedTime = currentTime - scrollWave.startTime;
					
					// 波紋がまだ持続時間内かチェック
					if (elapsedTime < scrollWave.duration) {
						// 各文字の垂直位置に基づいてオフセットを計算
						let charOffset;
						if (scrollWave.direction > 0) {
							// 上スクロール：上の文字ほど早く影響を受ける
							charOffset = char.originalY;
						} else {
							// 下スクロール：下の文字ほど早く影響を受ける
							charOffset = p.height - char.originalY;
						}
						
						// 波紋の到達時間を計算（文字の位置に基づく）
						let waveArrivalTime = charOffset * scrollWave.waveSpeed;
						let timeFromArrival = elapsedTime - waveArrivalTime;
						
						// 波紋が到達し、まだ効果範囲内の場合
						if (timeFromArrival > 0 && timeFromArrival < 300) { // 300ms間効果が持続
							// 時間経過に基づく減衰
							let timeIntensity = Math.exp(-timeFromArrival / 100); // 100msで減衰
							let force = scrollWave.strength * timeIntensity * deltaTimeNormalized;
							
							// 垂直方向の力を適用
							char.vy += scrollWave.direction * force;
							
							// わずかな水平方向のランダムな力も追加
							// char.vx += (Math.random() - 0.5) * force * 0.2;
							
							// // 回転力も追加
							// char.angularVelocity += (Math.random() - 0.5) * timeIntensity * 0.02 * deltaTimeNormalized;
						}
					} else {
						// 持続時間を過ぎた波紋を削除
						scrollWaves.splice(i, 1);
					}
				}
				
				// 従来のスクロール力を適用（常に適用）
				char.vx += scrollForce.x * deltaTimeNormalized;
				char.vy += scrollForce.y * deltaTimeNormalized;
				
				// 元の位置への復元力を計算（オーバーシュート防止）
				let restoreStrength = 0.01; // 復元力の強さ（一定値）
				let distanceToOriginal = p.dist(char.x, char.y, char.originalX, char.originalY);
				
				if (distanceToOriginal > 1) { // 元の位置から1ピクセル以上離れている場合
					// 方向ベクトルを正規化
					let directionX = (char.originalX - char.x) / distanceToOriginal;
					let directionY = (char.originalY - char.y) / distanceToOriginal;
					
					// 現在の速度と復元方向の内積を計算（目標に向かっているかチェック）
					let velocityTowardsTarget = char.vx * directionX + char.vy * directionY;
					
					// 距離に応じた減衰係数（近づくほど復元力を弱める）
					let distanceFactor = Math.min(1.0, distanceToOriginal / 50); // 50ピクセル以内で減衰開始
					
					// 速度制限：目標に向かう速度が大きすぎる場合は復元力を減らす
					let maxRestoreSpeed = 0.5; // 最大復元速度
					let speedDamping = 1.0;
					if (velocityTowardsTarget > maxRestoreSpeed) {
						speedDamping = 0.1; // 速度が大きい場合は復元力を大幅に減らす
					}
					
					let restoreX = directionX * restoreStrength * distanceFactor * speedDamping * deltaTimeNormalized;
					let restoreY = directionY * restoreStrength * distanceFactor * speedDamping * deltaTimeNormalized;
					
					char.vx += restoreX;
					char.vy += restoreY;
				}
				
				// 元の角度への復元力を計算（角度の場合）
				let angleRestoreStrength = 0.002; // 角度復元力の強さ
				let currentAngle = char.rotation % (2 * p.PI); // 現在の角度を0-2πに正規化
				
				// 最短経路で0度に戻るための角度差を計算
				let angleDifference = currentAngle;
				if (angleDifference > p.PI) {
					angleDifference -= 2 * p.PI; // 180度を超える場合は反対方向に回転
				}
				
				if (Math.abs(angleDifference) > 0.01) { // 角度差が0.01ラジアン以上の場合
					let angleRestore = -angleDifference * angleRestoreStrength * deltaTimeNormalized;
					char.angularVelocity += angleRestore;
				}
				
				// マウスからの斥力を計算
				if (p.mouseX > 0 && p.mouseY > 0) {
					let mouseDistance = p.dist(char.x, char.y, p.mouseX, p.mouseY);
					let repulsionRadius = isMousePressed ? 150 : 20; // 斥力の影響範囲
					
					if (mouseDistance < repulsionRadius) {
						// マウスから文字への方向ベクトルを計算
						let repulsionX = char.x - p.mouseX;
						let repulsionY = char.y - p.mouseY;
						
						// 正規化（距離で割る）
						if (mouseDistance > 0) {
							repulsionX /= mouseDistance;
							repulsionY /= mouseDistance;
						}
						
						// クリック状態に応じて斥力の強さを調整
						let baseStrength = isMousePressed ? 0.25 : 0.1; // クリック時は強く
						let repulsionStrength = (repulsionRadius - mouseDistance) / repulsionRadius * baseStrength * deltaTimeNormalized;
						
						// 斥力を速度に加算
						char.vx += repulsionX * repulsionStrength;
						char.vy += repulsionY * repulsionStrength;
						
						// 斥力による回転を追加（外積を使用）
						let torque = repulsionX * repulsionStrength * 0.5; // 水平方向の力で回転
						char.angularVelocity += torque;
					}
				}
				
				// 文字同士の衝突判定と処理（キーで制御可能）
				if (collisionEnabled) {
					// 復元力以外の力が加わっていない（静止状態に近い）文字は衝突判定をスキップ
					let speedThreshold = 0.03; // 速度の閾値
					let currentSpeed = p.sqrt(char.vx * char.vx + char.vy * char.vy);
					let distanceFromOriginal = p.dist(char.x, char.y, char.originalX, char.originalY);
					
					// 動いていない、または元の位置に近い文字は衝突判定をスキップ
					if (currentSpeed < speedThreshold && distanceFromOriginal < 20) {
						// この文字は静止状態なので衝突判定をスキップ
					} else {
						for (let other of chars) {
							if (other !== char) {
								// 相手の文字も同様にチェック
								let otherSpeed = p.sqrt(other.vx * other.vx + other.vy * other.vy);
								let otherDistanceFromOriginal = p.dist(other.x, other.y, other.originalX, other.originalY);
								
								// 両方の文字が静止状態なら衝突判定をスキップ
								if (currentSpeed < speedThreshold && distanceFromOriginal < 5 &&
									otherSpeed < speedThreshold && otherDistanceFromOriginal < 5) {
									continue;
								}
								
								let charDistance = p.dist(char.x, char.y, other.x, other.y) + 3; // 衝突判定の距離に少し余裕を持たせる
								let collisionRadius = (char.size + other.size) / 2; // 両方の文字のサイズを考慮
								
								if (charDistance < collisionRadius && charDistance > 0) {
									// 衝突時の位置調整（重なりを解消）
									let overlap = collisionRadius - charDistance;
									let separationX = (char.x - other.x) / charDistance * overlap * 0.5;
									let separationY = (char.y - other.y) / charDistance * overlap * 0.5;
									
									char.x += separationX;
									char.y += separationY;
									other.x -= separationX;
									other.y -= separationY;
									
									// 衝突後の速度計算（弾性衝突）
									let relativeVelocityX = char.vx - other.vx;
									let relativeVelocityY = char.vy - other.vy;
									let normalX = (char.x - other.x) / charDistance;
									let normalY = (char.y - other.y) / charDistance;
									
									// 法線方向の相対速度
									let relativeVelocityInNormal = relativeVelocityX * normalX + relativeVelocityY * normalY;
									
									// 衝突が分離方向なら処理しない
									if (relativeVelocityInNormal > 0) continue;
									
									// 反発係数（0.01で少し減衰）
									let restitution = 0.01;
									let impulse = -(1 + restitution) * relativeVelocityInNormal;
									
									// 速度を更新
									char.vx += impulse * normalX;
									char.vy += impulse * normalY;
									other.vx -= impulse * normalX;
									other.vy -= impulse * normalY;
									
									// 衝突による回転を追加
									let collisionTorque = impulse * 0.3; // 衝突の強さに比例した回転
									char.angularVelocity += collisionTorque;
									other.angularVelocity -= collisionTorque;
								}
							}
						}
					}
				}
				
				// 位置更新
				char.x += char.vx * deltaTimeNormalized;
				char.y += char.vy * deltaTimeNormalized;
				
				// 回転の更新
				char.rotation += char.angularVelocity * deltaTimeNormalized;
				
				// 角速度制限（最大角速度を設定）
				let maxAngularSpeed = 0.02; // 最大角速度（ラジアン/フレーム）
				if (Math.abs(char.angularVelocity) > maxAngularSpeed) {
					// 角速度を正規化して最大角速度に制限
					char.angularVelocity = Math.sign(char.angularVelocity) * maxAngularSpeed;
				}
				
				// 速度制限（最大速度を設定）
				let maxSpeed = 5.0; // 最大速度
				let currentSpeed = p.sqrt(char.vx * char.vx + char.vy * char.vy);
				
				if (currentSpeed > maxSpeed) {
					// 速度を正規化して最大速度に制限
					char.vx = (char.vx / currentSpeed) * maxSpeed;
					char.vy = (char.vy / currentSpeed) * maxSpeed;
				}
				
				// 境界での跳ね返り
				// if (char.x < char.size/2 || char.x > p.width - char.size/2) {
				// 	char.vx *= -0.9;
				// 	char.x = p.constrain(char.x, char.size/2, p.width - char.size/2);
				// }
				// if (char.y < char.size/2 || char.y > p.height - char.size/2) {
				// 	char.vy *= -0.9;
				// 	char.y = p.constrain(char.y, char.size/2, p.height - char.size/2);
				// }
				
				// 重力効果
				// char.vy += 0.1;
				
				// 摩擦
				let frictionRate = Math.pow(0.97, deltaTimeNormalized);
				char.vx *= frictionRate;
				char.vy *= frictionRate;
				
				// 角速度の減衰（回転の摩擦）
				let angularFrictionRate = Math.pow(0.90, deltaTimeNormalized);
				char.angularVelocity *= angularFrictionRate;
				
				// 回転角度の正規化（0-2πの範囲に保つ）
				char.rotation = char.rotation % (2 * p.PI);
				
				// 色の変化
				char.hue = (char.hue + 1 * deltaTimeNormalized) % 360;

				// カリング：画面外の文字は描画をスキップ
				let margin = 50; // 画面外でも少し余裕を持って描画
				let isVisible = char.x > -margin && 
							   char.x < p.width + margin && 
							   char.y > -margin && 
							   char.y < p.height + margin;
				
				if (!isVisible) {
					continue; // 画面外の文字は描画処理をスキップ
				}

				// depthに基づいてalphaを調整
				let alphaMultiplier = 1 - (char.depth / char.maxDepth) * 0.7; // depth が高いほど透明に
				let finalAlpha = char.alpha * alphaMultiplier;

				// タイトル文字の場合は元の文字を保持、通常の文字は水/波変換
				if (char.isTitle) {
					// タイトル文字は元の文字を保持
					char.text = char.originalText;
				} else {
					// 通常の文字は水/波変換
					if (char.depth > 7 && char.text == "水") {
						char.text = "波";
					} else if (char.depth < 8 && char.text == "波") {
						char.text = "水";
					}
				}

				p.fill(char.hue, 0, 100, finalAlpha);
				p.textFont('Arial');
				p.textAlign(p.CENTER, p.CENTER);
				p.textSize(char.size * 1.0);
				
				// 回転を適用して文字を描画
				p.push(); // 現在の変換状態を保存
				p.noStroke(); // 枠線なし
				p.translate(char.x, char.y); // 文字の位置に移動
				p.translate(0, -char.depth * 0.5); // 深度に応じて少し下に移動
				p.rotate(char.rotation); // 回転を適用
				p.text(char.text, 0, 0); // 原点（文字の中心）に描画
				p.pop(); // 変換状態を復元
				// p.strokeWeight(2);
				// p.stroke(char.hue, 0, 100, char.alpha);
				// p.noFill();
				// p.ellipse(char.x, char.y, char.size);
				// p.noStroke();
			}
			
			// マウス追従エフェクト（アニメーション付き）
			if (p.mouseX > 0 && p.mouseY > 0) {
				// 目標値を設定（実際の斥力範囲に合わせて）
				targetCircleRadius = isMousePressed ? 140 : 20;
				targetCircleHue = isMousePressed ? 200 : 220; // クリック時は赤、通常時は青
				targetCircleAlpha = isMousePressed ? 30 : 50;
			} else {
				// マウスが画面外の場合は円を消すように設定
				targetCircleRadius = 0;
				targetCircleAlpha = 0;
			}
			
			// アニメーション速度（0.0-1.0、1.0に近いほど速い）
			let animationSpeed = 0.12 * deltaTimeNormalized;
			
			// 現在値を目標値に向けてスムーズに変化
			currentCircleRadius += (targetCircleRadius - currentCircleRadius) * animationSpeed;
			currentCircleHue += (targetCircleHue - currentCircleHue) * animationSpeed;
			currentCircleAlpha += (targetCircleAlpha - currentCircleAlpha) * animationSpeed;
			
			// 円が見える状態の場合のみ描画
			if (currentCircleRadius > 1 && currentCircleAlpha > 1) {
				// メイン円の描画
				p.stroke(currentCircleHue, 60, 100, currentCircleAlpha);
				p.strokeWeight(1);
				p.noFill();
				
				// サイズに応じて波の強さを調整
				let waveIntensity = currentCircleRadius / 140; // 0.0-1.0の範囲
				
				if (isMousePressed && p.mouseX > 0 && p.mouseY > 0) {
					// クリック時：複数の脈動する円
					let waveSize = currentCircleRadius * 2 + p.sin(p.frameCount * 0.1) * 10;
					p.ellipse(p.mouseX, p.mouseY, waveSize);
				} else if (p.mouseX > 0 && p.mouseY > 0) {
					// 通常時：穏やかな波紋
					let waveSize = currentCircleRadius * 2 + p.sin(p.frameCount * 0.1) * 10;
					p.ellipse(p.mouseX, p.mouseY, waveSize);
				}
			}
			
			// FPS表示（Qキーで表示/非表示切り替え）
			if (showControlPanels) {
				p.fill(0, 0, 100);
				p.textAlign(p.LEFT, p.TOP);
				p.noStroke();
				p.textSize(16);
				p.text(`FPS: ${p.frameRate().toFixed(1)}`, 10, 20);
				p.text(`Chars: ${chars.length}`, 10, 40);
				p.text(`Frame: ${p.frameCount}`, 10, 60);
				p.text(`Collision: ${collisionEnabled ? 'ON' : 'OFF'}`, 10, 80); // 衝突判定の状態を表示
				p.text(`ΔTime: ${deltaTime.toFixed(1)}ms`, 10, 100); // デルタタイムを表示
				p.text(`Auto Ripples: ${autoRipplesEnabled ? 'ON' : 'OFF'}`, 10, 120); // 自動波紋の状態を表示
				p.text(`Ripples: ${ripples.length}/${maxRipples}`, 10, 140); // 水紋数を表示
				p.text(`Scroll Waves: ${scrollWaves.length}/${maxScrollWaves}`, 10, 160); // スクロール波紋数を表示
				
				// 画面内の文字数をカウントして表示
				let visibleCount = 0;
				let margin = 50;
				for (let char of chars) {
					if (char.x > -margin && char.x < p.width + margin && 
						char.y > -margin && char.y < p.height + margin) {
						visibleCount++;
					}
				}
				p.text(`Visible: ${visibleCount}`, 10, 180); // 画面内の文字数を表示
			}
			
			// スクロール力を徐々に減衰
			let scrollDecayRate = Math.pow(0.5, deltaTimeNormalized);
			scrollForce.x *= scrollDecayRate;
			scrollForce.y *= scrollDecayRate;
		};
		
		// マウスクリック時にボールを追加
		p.mousePressed = () => {
			isMousePressed = true; // マウスプレス状態を更新
			
			// 水紋効果: 新しい水紋を作成（数制限あり）
			if (p.mouseX > 0 && p.mouseX < p.width && p.mouseY > 0 && p.mouseY < p.height && ripples.length < maxRipples) {
				ripples.push({
					x: p.mouseX,
					y: p.mouseY,
					radius: 0, // 初期半径は0
					speed: 8, // 拡大速度
					intensity: 1.0, // 水紋の強度
					age: 0
				});
			}
			
			// if (p.mouseX > 0 && p.mouseX < p.width && p.mouseY > 0 && p.mouseY < p.height) {
			// 	const charList = ['A', 'a', 'あ'];
			// 	chars.push({
			// 		text: charList[Math.floor(p.random(charList.length))],
			// 		x: p.mouseX,
			// 		y: p.mouseY,
			// 		vx: p.random(-5, 5),
			// 		vy: p.random(-5, 5),
			// 		size: p.random(15, 40),
			// 		hue: p.random(360),
			// 		alpha: 80,
			// 		rotation: p.random(0, 2 * p.PI), // ランダムな初期回転
			// 		angularVelocity: p.random(-0.2, 0.2), // ランダムな初期角速度
			// 		depth: 0, // 新しい文字は深度0からスタート
			// 		maxDepth: 10
			// 	});
			// }
		};
		
		// マウスリリース時
		p.mouseReleased = () => {
			isMousePressed = false; // マウスプレス状態をリセット
		};
		
		// キーボード入力
		p.keyPressed = () => {
			if (p.key === 'c' || p.key === 'C') {
				// ボールをクリア
				chars = [];
				ripples = []; // 水紋効果もクリア
			} else if (p.key === 'r' || p.key === 'R') {
				chars = []; // クリック時に文字をクリア
				ripples = []; // 水紋効果もクリア
				
				let xOffset = 50; // 初期位置のオフセット
				let yOffset = p.height / 4; // 垂直位置は中央に固定
				// 初期ボールを作成
				for (let i = 0; i < drawText.length; i++) {
					let c = drawText[i];
					if (i > 0) {
						// 前の文字と現在の文字のペアの幅
						let pair = drawText[i - 1] + drawText[i];
						let pairWidth = p.textWidth(pair);
						if (c === '\\') {
							xOffset = 50; // 改行時は初期位置に戻す
							yOffset += 24 * 1; // 次の行は少し下にずらす
							console.log(`New line at index ${i}, resetting xOffset to ${xOffset} and increasing yOffset to ${yOffset}`);
							continue;
						}

						// 前の文字の幅
						let prevWidth = p.textWidth(drawText[i - 1]);

						// 現在の文字の幅
						let currWidth = p.textWidth(drawText[i]);

						// カーニング差分（マイナスなら詰める）
						let kerning = pairWidth - (prevWidth + currWidth);
						console.log(`Pair: ${pair}, Kerning: ${kerning}`);
						// 改行直後はxOffsetを進めない
						if (drawText[i - 1] !== '\\') {
							xOffset += kerning;
							xOffset += p.textWidth(drawText[i]) + 0;
						}
					}
					if (c === '\\') {
						// エスケープ文字は無視
						continue;
					}
					chars.push({
						// x: p.random(p.width),
						// y: p.random(p.height),
						// vx: p.random(-3, 3),
						// vy: p.random(-3, 3),
						// size: p.random(20, 60),
						// hue: p.random(360),
						// alpha: 80
						text: c,
						originalText: c, // 元の文字を保持
						x: xOffset,
						y: yOffset,
						vx: 0,
						vy: 0,
						originalX: xOffset, // 元の位置を記録
						originalY: yOffset, // 元の位置を記録
						size: 16,
						hue: p.random(360),
						alpha: 80,
						rotation: 0, // 回転角度（ラジアン）
						angularVelocity: 0, // 角速度
						depth: 0, // 深度（水紋効果用）
						maxDepth: 10, // 最大深度
						isTitle: false // 通常の文字
					});
				}
				
				// タイトル文字「文字の波」を画面上部中央に配置
				const titleText = "文字の波";
				const titleSize = 48; // 大きめのサイズ
				p.textSize(titleSize); // タイトルサイズを設定して幅を測定
				const titleY = 80; // 画面上部からの距離
				
				let titleXOffset = (p.width - p.textWidth(titleText)) / 2; // 中央揃えの開始位置
				
				for (let i = 0; i < titleText.length; i++) {
					let char = titleText[i];
					let charWidth = p.textWidth(char);
					
					chars.push({
						text: char,
						originalText: char, // 元の文字を保持
						x: titleXOffset + charWidth / 2, // 文字の中心位置
						y: titleY,
						vx: 0,
						vy: 0,
						originalX: titleXOffset + charWidth / 2,
						originalY: titleY,
						size: titleSize,
						hue: 200, // 青系の色でタイトルを区別
						alpha: 90,
						rotation: 0,
						angularVelocity: 0,
						depth: 0,
						maxDepth: 15, // タイトル文字は少し深い効果
						isTitle: true // タイトル文字であることを示すフラグ
					});
					
					titleXOffset += charWidth + 4; // 次の文字の位置（少し間隔を空ける）
				}
			} else if (p.key === ' ') {
				// スペースキーで一時停止/再生
				if (p.isLooping()) {
					p.noLoop();
				} else {
					p.loop();
				}
			} else if (p.key === 'd' || p.key === 'D') {
				// Dキーでデバッグパネルの表示/非表示
				showDebugPanel = !showDebugPanel;
			} else if (p.key === 'x' || p.key === 'X') {
				// Xキーで衝突判定のオン/オフ
				collisionEnabled = !collisionEnabled;
				console.log(`Collision detection: ${collisionEnabled ? 'ON' : 'OFF'}`);
			} else if (p.key === 'a' || p.key === 'A') {
				// Aキーで自動波紋のオン/オフ
				autoRipplesEnabled = !autoRipplesEnabled;
				console.log(`Auto ripples: ${autoRipplesEnabled ? 'ON' : 'OFF'}`);
				if (!autoRipplesEnabled) {
					autoRippleTimer = 0; // タイマーをリセット
				}
			} else if (p.key === 'q' || p.key === 'Q') {
				// Qキーでコントロールパネルの表示/非表示
				showControlPanels = !showControlPanels;
				console.log(`Control panels: ${showControlPanels ? 'VISIBLE' : 'HIDDEN'}`);
			}
		};
		
		// ウィンドウリサイズ時
		p.windowResized = () => {
			// キャンバスサイズを新しいウィンドウサイズに調整
			p.resizeCanvas(p.windowWidth, p.windowHeight);
		};
		
		// マウスホイールイベント
		p.mouseWheel = (event) => {
			// スクロール波紋の数を制限
			if (scrollWaves.length < maxScrollWaves) {
				// 新しいスクロール波紋を配列に追加（重複対応）
				scrollWaves.push({
					startTime: p.millis(),
					direction: -Math.sign(event.delta),
					strength: Math.abs(event.delta) * 0.01,
					waveSpeed: 0.8,
					duration: 1000
				});
			}
			
			// デフォルトのスクロール動作を防ぐ
			return false;
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
		
		// ウィンドウフォーカスイベントの監視
		const handleFocus = () => {
			isWindowFocused = true;
			lastFocusTime = performance.now();
			console.log('Window focused, reset delta time tracking');
		};
		
		const handleBlur = () => {
			isWindowFocused = false;
			console.log('Window blurred');
		};
		
		window.addEventListener('focus', handleFocus);
		window.addEventListener('blur', handleBlur);
		
		// クリーンアップ関数
		return () => {
			if (p5Instance) {
				p5Instance.remove();
			}
			window.removeEventListener('focus', handleFocus);
			window.removeEventListener('blur', handleBlur);
		};
	});
	
	// ドラッグ機能の実装
	function handleMouseDown(event) {
		if (event.target.classList.contains('resize-handle')) {
			isResizing = true;
			return;
		}
		isDragging = true;
		const rect = event.currentTarget.getBoundingClientRect();
		dragOffset.x = event.clientX - rect.left;
		dragOffset.y = event.clientY - rect.top;
	}
	
	function handleMouseMove(event) {
		if (isDragging) {
			panelPosition.x = ((event.clientX - dragOffset.x) / window.innerWidth) * 100;
			panelPosition.y = ((event.clientY - dragOffset.y) / window.innerHeight) * 100;
			
			// 画面外に出ないよう制限
			panelPosition.x = Math.max(0, Math.min(90, panelPosition.x));
			panelPosition.y = Math.max(0, Math.min(90, panelPosition.y));
		} else if (isResizing) {
			// リサイズ処理
			const newWidth = Math.max(200, event.clientX - (window.innerWidth * panelPosition.x / 100));
			const newHeight = Math.max(100, event.clientY - (window.innerHeight * panelPosition.y / 100));
			
			panelSize.width = Math.min(newWidth, window.innerWidth * 0.8);
			panelSize.height = Math.min(newHeight, window.innerHeight * 0.8);
		}
	}
	
	function handleMouseUp() {
		isDragging = false;
		isResizing = false;
	}

	// テキスト切り替え関数
	function switchText(index) {
		currentTextIndex = index;
		drawText = textOptions[currentTextIndex].text;
	}

	function resetChars() {
		let xOffset = 50; // 初期位置のオフセット
		let yOffset = p.height / 2; // 垂直位置は中央に固定
		// 初期ボールを作成
		for (let i = 0; i < drawText.length; i++) {
			let c = drawText[i];
			if (i > 0) {
				// 前の文字と現在の文字のペアの幅
				let pair = drawText[i - 1] + drawText[i];
				let pairWidth = p.textWidth(pair);
				if (c === '\\') {
					xOffset = 50; // 改行時は初期位置に戻す
					yOffset += 24; // 次の行は少し下にずらす
					console.log(`New line at index ${i}, resetting xOffset to ${xOffset} and increasing yOffset to ${yOffset}`);
					continue;
				}

				// 前の文字の幅
				let prevWidth = p.textWidth(drawText[i - 1]);

				// 現在の文字の幅
				let currWidth = p.textWidth(drawText[i]);

				// カーニング差分（マイナスなら詰める）
				let kerning = pairWidth - (prevWidth + currWidth);
				console.log(`Pair: ${pair}, Kerning: ${kerning}`);
				// 改行直後はxOffsetを進めない
				if (drawText[i - 1] !== '\\') {
					xOffset += kerning;
					xOffset += p.textWidth(drawText[i]) + 4;
				}
			}
			if (c === '\\') {
				// エスケープ文字は無視
				continue;
			}
			chars.push({
				// x: p.random(p.width),
				// y: p.random(p.height),
				// vx: p.random(-3, 3),
				// vy: p.random(-3, 3),
				// size: p.random(20, 60),
				// hue: p.random(360),
				// alpha: 80
				text: c,
				originalText: c, // 元の文字を保持
				x: xOffset,
				y: yOffset,
				vx: 0,
				vy: 0,
				originalX: xOffset, // 元の位置を記録
				originalY: yOffset, // 元の位置を記録
				size: 16,
				hue: p.random(360),
				alpha: 80,
				rotation: 0, // 回転角度（ラジアン）
				angularVelocity: 0, // 角速度
				depth: 0, // 深度（水紋効果用）
				maxDepth: 10, // 最大深度
				isTitle: false // 通常の文字
			});
		}
		
		// タイトル文字「文字の波」を画面上部中央に配置
		const titleText = "文字の波";
		const titleSize = 48; // 大きめのサイズ
		p.textSize(titleSize); // タイトルサイズを設定して幅を測定
		const titleY = 80; // 画面上部からの距離
		
		let titleXOffset = (p.width - p.textWidth(titleText)) / 2; // 中央揃えの開始位置
		
		for (let i = 0; i < titleText.length; i++) {
			let char = titleText[i];
			let charWidth = p.textWidth(char);
			
			chars.push({
				text: char,
				originalText: char, // 元の文字を保持
				x: titleXOffset + charWidth / 2, // 文字の中心位置
				y: titleY,
				vx: 0,
				vy: 0,
				originalX: titleXOffset + charWidth / 2,
				originalY: titleY,
				size: titleSize,
				hue: 200, // 青系の色でタイトルを区別
				alpha: 90,
				rotation: 0,
				angularVelocity: 0,
				depth: 0,
				maxDepth: 15, // タイトル文字は少し深い効果
				isTitle: true // タイトル文字であることを示すフラグ
			});
			
			titleXOffset += charWidth + 4; // 次の文字の位置（少し間隔を空ける）
		}
	}

</script>

<svelte:head>
	<title>文字の波</title>
	<meta name="description" content="Interactive p5.js canvas with bouncing balls" />
	<link rel="preload" href="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.7.0/p5.min.js" as="script" />
</svelte:head>

<div class="container">
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

	{#if showControlPanels}
		<div class="controls">
			<div class="control-section">
				<h3>📋 コントロール</h3>
				<ul>
					<li><strong>クリック:</strong> 水紋効果を作成</li>
					<li><strong>C キー:</strong> すべての文字をクリア</li>
					<li><strong>R キー:</strong> 文字をリセット</li>
					<li><strong>D キー:</strong> デバッグパネルの表示/非表示</li>
					<li><strong>X キー:</strong> 衝突判定のオン/オフ</li>
					<li><strong>A キー:</strong> 自動波紋のオン/オフ</li>
					<li><strong>Q キー:</strong> コントロールパネルの表示/非表示</li>
					<li><strong>スペース:</strong> アニメーションの一時停止/再生</li>
				</ul>
			</div>
			
			<div class="control-section">
				<h3>📝 テキスト選択</h3>
				<div class="text-selector">
					{#each textOptions as option, index}
						<button 
							class="text-option-btn"
							class:active={currentTextIndex === index}
							on:click={() => switchText(index)}
						>
							{option.name}
						</button>
					{/each}
				</div>
			</div>
		</div>
	{/if}
	
	{#if showDebugPanel}
		<div 
			class="debug-panel"
			style="left: {panelPosition.x}%; top: {panelPosition.y}%; width: {panelSize.width}px; height: {panelSize.height}px;"
			on:mousedown={handleMouseDown}
			role="dialog"
			aria-label="デバッグパネル"
			tabindex="-1"
		>
			<div class="debug-section">
				<div class="debug-content">
					<div class="text-display">
						<div class="text-content">{drawText}</div>
					</div>
				</div>
				<div class="resize-handle"></div>
			</div>
		</div>
	{/if}
</div>

<svelte:window on:mousemove={handleMouseMove} on:mouseup={handleMouseUp} />

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		font-family: 'Arial', sans-serif;
		overflow: hidden;
	}
	
	.container {
		position: fixed;
		top: 0;
		left: 0;
		width: 100vw;
		height: 100vh;
		overflow: hidden;
	}
	
	.canvas-wrapper {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
	}
	
	.canvas-container {
		width: 100%;
		height: 100%;
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
		background: rgba(0, 0, 0, 0.8);
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
		color: white;
		margin: 0;
		font-size: 1.1rem;
	}
	
	.controls {
		position: fixed;
		top: 20px;
		right: 20px;
		z-index: 10;
		max-width: 300px;
	}
	
	.control-section {
		background: rgba(0, 0, 0, 0.7);
		border-radius: 8px;
		padding: 1rem;
		backdrop-filter: blur(10px);
		border: 1px solid rgba(255, 255, 255, 0.2);
		color: white;
	}
	
	.control-section h3 {
		margin: 0 0 0.5rem 0;
		font-size: 1.1rem;
		color: #4ecdc4;
	}
	
	.control-section ul {
		list-style: none;
		padding: 0;
		margin: 0;
	}
	
	.control-section li {
		margin: 0.25rem 0;
		padding: 0.25rem;
		background: rgba(255, 255, 255, 0.1);
		border-radius: 4px;
		font-size: 0.9rem;
	}
	
	/* テキスト選択ボタンのスタイル */
	.text-selector {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		margin-bottom: 0.75rem;
	}
	
	.text-option-btn {
		background: rgba(255, 255, 255, 0.1);
		border: 1px solid rgba(255, 255, 255, 0.2);
		border-radius: 6px;
		padding: 0.5rem 0.75rem;
		color: white;
		font-size: 0.85rem;
		cursor: pointer;
		transition: all 0.2s ease;
		text-align: left;
	}
	
	.text-option-btn:hover {
		background: rgba(78, 205, 196, 0.2);
		border-color: rgba(78, 205, 196, 0.4);
		transform: translateY(-1px);
	}
	
	.text-option-btn.active {
		background: rgba(78, 205, 196, 0.3);
		border-color: rgba(78, 205, 196, 0.6);
		font-weight: bold;
	}
	
	/* p5.jsキャンバスのスタイル調整 */
	:global(canvas) {
		display: block;
	}
	
	/* デバッグパネルのスタイル */
	.debug-panel {
		position: fixed;
		z-index: 20;
		cursor: move;
		user-select: none;
		min-width: 200px;
		min-height: 100px;
	}
	
	.debug-section {
		background: rgba(20, 20, 30, 0.3);
		border-radius: 20px;
		/* backdrop-filter: blur(1px); */
		border: 1px solid rgba(255, 255, 255, 0.15);
		color: white;
		box-shadow: 
			0 8px 32px rgba(0, 0, 0, 0.3),
			0 2px 8px rgba(0, 0, 0, 0.2),
			inset 0 1px 0 rgba(255, 255, 255, 0.1);
		overflow: hidden;
		width: 100%;
		height: 100%;
		position: relative;
	}
	
	.debug-content {
		padding: 0.75rem;
		height: calc(100% - 1.5rem);
		overflow: auto;
	}
	
	.text-display {
		margin: 0;
	}
	
	.text-content {
		background: rgba(255, 255, 255, 0.02);
		border-radius: 12px;
		padding: 0.75rem;
		margin: 0;
		min-height: 1.5rem;
		word-wrap: break-word;
		/* font-family: 'Segoe UI', system-ui, sans-serif; */
		border: 1px solid rgba(255, 255, 255, 0.05);
		border-left: 3px solid rgba(78, 205, 196, 0.6);
		font-size: 1rem;
		line-height: 1.4;
		box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.05);
	}
	
	.resize-handle {
		position: absolute;
		bottom: 0;
		right: 0;
		width: 20px;
		height: 20px;
		cursor: nw-resize;
		background: linear-gradient(
			-45deg,
			transparent 0%,
			transparent 30%,
			rgba(255, 255, 255, 0.1) 30%,
			rgba(255, 255, 255, 0.1) 35%,
			transparent 35%,
			transparent 65%,
			rgba(255, 255, 255, 0.1) 65%,
			rgba(255, 255, 255, 0.1) 70%,
			transparent 70%
		);
		border-bottom-right-radius: 20px;
	}
	
	.resize-handle:hover {
		background: linear-gradient(
			-45deg,
			transparent 0%,
			transparent 30%,
			rgba(78, 205, 196, 0.3) 30%,
			rgba(78, 205, 196, 0.3) 35%,
			transparent 35%,
			transparent 65%,
			rgba(78, 205, 196, 0.3) 65%,
			rgba(78, 205, 196, 0.3) 70%,
			transparent 70%
		);
	}
</style>
