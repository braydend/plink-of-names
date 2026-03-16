<script lang="ts">
	import Matter from 'matter-js';
	import { onMount } from 'svelte';

	let {
		names = [],
		onresult,
		pegDensity = 2,
		boardScale = 1,
		chaos = 0
	}: {
		names: string[];
		onresult?: (name: string) => void;
		pegDensity?: number;
		boardScale?: number;
		chaos?: number;
	} = $props();

	let containerEl = $state<HTMLDivElement>();
	let canvasEl = $state<HTMLCanvasElement>();

	// Board configuration
	const PEG_RADIUS = 6;
	const BALL_RADIUS = 10;
	const MIN_BUCKET_HEIGHT = 60;
	const MAX_BUCKET_HEIGHT = 140;
	const TOP_PADDING = 60;
	const SIDE_PADDING = 40;
	const WALL_THICKNESS = 10;
	const FLOOR_THICKNESS = 40;
	const DIVIDER_WIDTH = 4;
	const MIN_PEG_SPACING_Y = 35;
	const MAX_PEG_SPACING_Y = 55;

	// Colours
	const PEG_COLOURS = ['#f472b6', '#a78bfa', '#60a5fa', '#34d399', '#fbbf24', '#fb923c'];
	const BUCKET_COLOURS = [
		'#ec4899',
		'#8b5cf6',
		'#3b82f6',
		'#10b981',
		'#f59e0b',
		'#f97316',
		'#ef4444',
		'#06b6d4'
	];

	let engine: Matter.Engine | null = null;
	let runner: Matter.Runner | null = null;
	let animFrameId: number;
	let worldWidth = $state(0); // physics world dimensions
	let worldHeight = $state(0);
	let viewWidth = $state(0); // canvas/viewport dimensions
	let viewHeight = $state(0);
	let pegRows = $state(0);
	let pegSpacingY = $state(45);
	let bucketHeight = $state(MIN_BUCKET_HEIGHT);
	let rotateLabels = $state(false);
	let dropping = $state(false);
	let ballBody: Matter.Body | null = null;
	let settleCheckInterval: ReturnType<typeof setInterval> | null = null;

	// Store references for rendering
	let pegs: { x: number; y: number; colour: string }[] = [];
	let buckets: { x: number; width: number; name: string; colour: string }[] = [];
	let dividerBodies: Matter.Body[] = [];

	// Result / celebration state
	let winningBucketIndex: number | null = null;
	let winningName: string | null = null;

	// Camera follow state (tracks ball during drop)
	const FOLLOW_ZOOM = 1.8;
	let cameraX = 0;
	let cameraY = 0;
	let cameraZoom = 1;
	let followingBall = false;

	// Result zoom state (zooms into winning bucket)
	let zoomLevel = 1;
	let zoomTargetX = 0;
	let zoomTargetY = 0;
	let zoomProgress = 0;
	let isZooming = $state(false);
	const ZOOM_SCALE = 2.5;
	const ZOOM_DURATION = 800;
	let zoomStartTime = 0;

	// Confetti state
	interface ConfettiParticle {
		x: number;
		y: number;
		vx: number;
		vy: number;
		colour: string;
		size: number;
		rotation: number;
		rotationSpeed: number;
		life: number;
		maxLife: number;
	}
	let confetti: ConfettiParticle[] = [];

	// Winner text animation
	let showWinnerOverlay = false;
	let winnerTextOpacity = 0;
	let winnerTextScale = 0;

	// Derived: can drop?
	let canDrop = $derived(names.length >= 2 && !dropping);

	function shuffleArray<T>(array: T[]): T[] {
		const shuffled = [...array];
		for (let i = shuffled.length - 1; i > 0; i--) {
			const j = Math.floor(Math.random() * (i + 1));
			[shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
		}
		return shuffled;
	}

	function calculateBoard(nameCount: number) {
		if (!containerEl) return;

		const containerWidth = containerEl.clientWidth;
		const containerHeight = containerEl.clientHeight;

		// View = canvas size, always matches viewport
		viewWidth = Math.max(containerWidth, 300);
		viewHeight = Math.max(containerHeight, 400);

		// World = physics board, can be taller than viewport via boardScale
		worldWidth = viewWidth;
		worldHeight = Math.max(viewHeight * boardScale, 400);

		// Determine if labels need rotation (when buckets are narrow)
		const testBucketWidth = (worldWidth - SIDE_PADDING * 2) / nameCount;
		rotateLabels = testBucketWidth < 70;

		// Dynamic bucket height — taller when labels are rotated for legibility
		bucketHeight = rotateLabels
			? Math.min(MAX_BUCKET_HEIGHT, Math.max(MIN_BUCKET_HEIGHT, 12 * 8 + 20))
			: MIN_BUCKET_HEIGHT;

		// Available height for pegs
		const availableHeight = worldHeight - TOP_PADDING - bucketHeight - FLOOR_THICKNESS - 10;

		// Calculate how many rows we need to fill the space at ideal spacing
		const idealRows = Math.floor(availableHeight / MAX_PEG_SPACING_Y);
		pegRows = Math.max(6, idealRows);

		// Calculate peg spacing to fill the available height evenly
		pegSpacingY = availableHeight / pegRows;
	}

	function buildBoard(nameCount: number) {
		if (!canvasEl || !containerEl) return;

		cleanup();

		calculateBoard(nameCount);

		// Canvas always matches viewport; camera pans over the larger world
		canvasEl.width = viewWidth;
		canvasEl.height = viewHeight;

		engine = Matter.Engine.create({
			positionIterations: 10,
			velocityIterations: 10
		});
		engine.gravity.y = 1;

		const bodies: Matter.Body[] = [];
		pegs = [];
		buckets = [];
		dividerBodies = [];

		const bucketCount = nameCount;
		const bucketAreaWidth = worldWidth - SIDE_PADDING * 2;
		const bucketWidth = bucketAreaWidth / bucketCount;

		// Create pegs in alternating offset rows
		// Peg columns based on density and available width, independent of name count
		const idealPegSpacing = 120 - pegDensity * 10; // density 1 = 110px, density 10 = 20px
		const pegCols = Math.max(3, Math.round(bucketAreaWidth / idealPegSpacing));
		const pegSpacingX = bucketAreaWidth / pegCols;

		// Chaos: removal chance scales from 0% (chaos=0) to ~70% (chaos=5)
		const removalChance = chaos * 0.14;
		// Protect the last few rows near buckets so the ball funnels correctly
		const protectedRows = 3;

		for (let row = 0; row < pegRows; row++) {
			const isOffsetRow = row % 2 === 1;
			const pegsInRow = isOffsetRow ? pegCols : pegCols + 1;
			const isProtectedRow = row >= pegRows - protectedRows;
			// Also protect edge pegs (first and last in each row) to keep the ball in bounds
			const isEdge = (col: number) => col === 0 || col === pegsInRow - 1;

			for (let col = 0; col < pegsInRow; col++) {
				// Skip peg randomly based on chaos, unless it's protected
				if (!isProtectedRow && !isEdge(col) && Math.random() < removalChance) {
					continue;
				}

				const x = isOffsetRow
					? SIDE_PADDING + pegSpacingX / 2 + col * pegSpacingX
					: SIDE_PADDING + col * pegSpacingX;

				const y = TOP_PADDING + row * pegSpacingY;

				const peg = Matter.Bodies.circle(x, y, PEG_RADIUS, {
					isStatic: true,
					restitution: 0.5,
					friction: 0.05
				});
				bodies.push(peg);

				pegs.push({
					x,
					y,
					colour: PEG_COLOURS[(row + col) % PEG_COLOURS.length]
				});
			}
		}

		// Bucket area
		const bucketY = TOP_PADDING + pegRows * pegSpacingY;

		// Bucket dividers — extend above bucket area to catch the ball
		const dividerHeight = bucketHeight + 20;
		for (let i = 0; i <= bucketCount; i++) {
			const x = SIDE_PADDING + i * bucketWidth;

			const divider = Matter.Bodies.rectangle(
				x,
				bucketY + bucketHeight / 2 - 10,
				DIVIDER_WIDTH + 2,
				dividerHeight,
				{
					isStatic: true,
					restitution: 0.3
				}
			);
			bodies.push(divider);
			dividerBodies.push(divider);
		}

		// Floor — thick to prevent tunneling
		const floor = Matter.Bodies.rectangle(
			worldWidth / 2,
			bucketY + bucketHeight + FLOOR_THICKNESS / 2,
			worldWidth,
			FLOOR_THICKNESS,
			{ isStatic: true }
		);
		bodies.push(floor);

		// Side walls — positioned so inner edge aligns with edge pegs
		// This eliminates the gap where the ball could get wedged
		const wallX = SIDE_PADDING - PEG_RADIUS - BALL_RADIUS;
		const leftWall = Matter.Bodies.rectangle(
			wallX - WALL_THICKNESS / 2,
			worldHeight / 2,
			WALL_THICKNESS,
			worldHeight,
			{ isStatic: true }
		);
		const rightWall = Matter.Bodies.rectangle(
			worldWidth - wallX + WALL_THICKNESS / 2,
			worldHeight / 2,
			WALL_THICKNESS,
			worldHeight,
			{ isStatic: true }
		);
		bodies.push(leftWall, rightWall);

		// Shuffle names randomly for bucket assignment
		const shuffledNames = shuffleArray(names);

		// Store bucket info with shuffled names
		for (let i = 0; i < bucketCount; i++) {
			buckets.push({
				x: SIDE_PADDING + i * bucketWidth,
				width: bucketWidth,
				name: shuffledNames[i],
				colour: BUCKET_COLOURS[i % BUCKET_COLOURS.length]
			});
		}

		Matter.Composite.add(engine.world, bodies);

		runner = Matter.Runner.create();
		Matter.Runner.run(runner, engine);

		renderLoop();
	}

	export function randomise() {
		if (names.length >= 2) {
			buildBoard(names.length);
		}
	}

	export function dropBall() {
		if (!canDrop || !engine) return;

		dropping = true;
		winningBucketIndex = null;
		winningName = null;
		showWinnerOverlay = false;
		winnerTextOpacity = 0;
		winnerTextScale = 0;
		confetti = [];
		confettiSpawned = false;
		zoomProgress = 0;
		isZooming = false;
		zoomLevel = 1;

		// Scroll to top and lock — camera handles tracking from here
		if (containerEl) {
			containerEl.scrollTop = 0;
		}

		// Enable ball-follow camera
		followingBall = true;
		cameraX = worldWidth / 2;
		cameraY = 10;
		cameraZoom = 1;

		// Add slight random horizontal offset so repeated drops vary
		const offsetX = (Math.random() - 0.5) * 20;

		ballBody = Matter.Bodies.circle(worldWidth / 2 + offsetX, 10, BALL_RADIUS, {
			restitution: 0.4,
			friction: 0.05,
			density: 0.002,
			frictionAir: 0.001
		});

		Matter.Composite.add(engine.world, ballBody);

		startSettleCheck();
	}

	function startSettleCheck() {
		let stillFrames = 0;
		let inBucketFrames = 0;
		const SETTLE_THRESHOLD = 0.3;
		const FRAMES_TO_SETTLE = 30;
		const MAX_BUCKET_FRAMES = 180; // 3s fallback — if in bucket area this long, force settle

		settleCheckInterval = setInterval(() => {
			if (!ballBody) return;

			const speed = ballBody.speed;
			const bucketY = TOP_PADDING + pegRows * pegSpacingY;
			const ballY = ballBody.position.y;

			// Ball fell off screen — force settle using last known position
			if (ballY > worldHeight + 100) {
				onBallSettled();
				return;
			}

			// Not in bucket area yet
			if (ballY < bucketY) {
				stillFrames = 0;
				inBucketFrames = 0;
				return;
			}

			// In bucket area — track time spent here as fallback
			inBucketFrames++;

			if (speed < SETTLE_THRESHOLD) {
				stillFrames++;
			} else {
				stillFrames = 0;
			}

			if (stillFrames >= FRAMES_TO_SETTLE || inBucketFrames >= MAX_BUCKET_FRAMES) {
				onBallSettled();
			}
		}, 1000 / 60);
	}

	function onBallSettled() {
		if (settleCheckInterval) {
			clearInterval(settleCheckInterval);
			settleCheckInterval = null;
		}

		if (!ballBody) return;

		// Determine which bucket the ball landed in
		const ballX = ballBody.position.x;
		let foundIndex = -1;

		for (let i = 0; i < buckets.length; i++) {
			const bucket = buckets[i];
			if (ballX >= bucket.x && ballX < bucket.x + bucket.width) {
				foundIndex = i;
				break;
			}
		}

		// Fallback: find closest bucket
		if (foundIndex === -1 && buckets.length > 0) {
			let closestDist = Infinity;
			for (let i = 0; i < buckets.length; i++) {
				const center = buckets[i].x + buckets[i].width / 2;
				const dist = Math.abs(ballX - center);
				if (dist < closestDist) {
					closestDist = dist;
					foundIndex = i;
				}
			}
		}

		if (foundIndex >= 0) {
			winningBucketIndex = foundIndex;
			winningName = buckets[foundIndex].name;

			// Stop following, start result zoom
			followingBall = false;

			// Set zoom target to winning bucket center
			const bucket = buckets[foundIndex];
			zoomTargetX = bucket.x + bucket.width / 2;
			zoomTargetY = TOP_PADDING + pegRows * pegSpacingY + bucketHeight / 2;

			// Start zoom animation
			isZooming = true;
			zoomStartTime = performance.now();
		}

		dropping = false;
	}

	function easeOutCubic(t: number): number {
		return 1 - Math.pow(1 - t, 3);
	}

	function easeOutElastic(t: number): number {
		if (t === 0 || t === 1) return t;
		return Math.pow(2, -10 * t) * Math.sin((t * 10 - 0.75) * ((2 * Math.PI) / 3)) + 1;
	}

	function spawnConfetti(centerX: number, centerY: number) {
		const colours = [
			'#ec4899',
			'#8b5cf6',
			'#3b82f6',
			'#10b981',
			'#f59e0b',
			'#ef4444',
			'#fbbf24',
			'#06b6d4',
			'#f472b6',
			'#a78bfa'
		];
		for (let i = 0; i < 80; i++) {
			const angle = Math.random() * Math.PI * 2;
			const speed = 2 + Math.random() * 6;
			confetti.push({
				x: centerX,
				y: centerY,
				vx: Math.cos(angle) * speed,
				vy: Math.sin(angle) * speed - 3,
				colour: colours[Math.floor(Math.random() * colours.length)],
				size: 3 + Math.random() * 5,
				rotation: Math.random() * Math.PI * 2,
				rotationSpeed: (Math.random() - 0.5) * 0.3,
				life: 0,
				maxLife: 60 + Math.random() * 60
			});
		}
	}

	function updateConfetti() {
		for (let i = confetti.length - 1; i >= 0; i--) {
			const p = confetti[i];
			p.x += p.vx;
			p.y += p.vy;
			p.vy += 0.1;
			p.vx *= 0.99;
			p.rotation += p.rotationSpeed;
			p.life++;

			if (p.life >= p.maxLife) {
				confetti.splice(i, 1);
			}
		}
	}

	function drawConfetti(ctx: CanvasRenderingContext2D) {
		for (const p of confetti) {
			const alpha = 1 - p.life / p.maxLife;
			ctx.save();
			ctx.translate(p.x, p.y);
			ctx.rotate(p.rotation);
			ctx.globalAlpha = alpha;
			ctx.fillStyle = p.colour;
			ctx.fillRect(-p.size / 2, -p.size / 4, p.size, p.size / 2);
			ctx.restore();
		}
		ctx.globalAlpha = 1;
	}

	let confettiSpawned = false;

	export function resetBall() {
		if (ballBody && engine) {
			Matter.Composite.remove(engine.world, ballBody);
			ballBody = null;
		}
		if (settleCheckInterval) {
			clearInterval(settleCheckInterval);
			settleCheckInterval = null;
		}
		dropping = false;
		followingBall = false;
		cameraZoom = 1;
		winningBucketIndex = null;
		winningName = null;
		showWinnerOverlay = false;
		winnerTextOpacity = 0;
		winnerTextScale = 0;
		confetti = [];
		confettiSpawned = false;
		zoomProgress = 0;
		isZooming = false;
		zoomLevel = 1;
	}

	function renderLoop() {
		if (!canvasEl || !engine) return;

		const ctx = canvasEl.getContext('2d');
		if (!ctx) return;

		const now = performance.now();

		// Update ball-follow camera
		if (followingBall && ballBody) {
			// Adaptive lerp — faster when ball is further from camera
			const dx = ballBody.position.x - cameraX;
			const dy = ballBody.position.y - cameraY;
			const dist = Math.sqrt(dx * dx + dy * dy);
			// Base lerp + extra speed proportional to distance, so camera never falls behind
			const lerp = Math.min(0.5, 0.12 + dist * 0.002);
			cameraX += dx * lerp;
			cameraY += dy * lerp;
			// Zoom in when world is taller than the visible viewport
			const needsFollow = worldHeight > viewHeight;
			const targetZoom = needsFollow ? FOLLOW_ZOOM : 1;
			cameraZoom += (targetZoom - cameraZoom) * 0.03;
		} else if (!isZooming) {
			// Ease camera back to neutral when not following or zooming
			cameraZoom += (1 - cameraZoom) * 0.05;
		}

		// Update result zoom
		if (isZooming) {
			const elapsed = now - zoomStartTime;
			zoomProgress = Math.min(elapsed / ZOOM_DURATION, 1);
			const easedProgress = easeOutCubic(zoomProgress);
			zoomLevel = 1 + (ZOOM_SCALE - 1) * easedProgress;

			// Spawn confetti and show overlay once zoom completes
			if (zoomProgress >= 1 && !confettiSpawned) {
				confettiSpawned = true;
				spawnConfetti(zoomTargetX, zoomTargetY);
				showWinnerOverlay = true;

				// Notify parent after zoom finishes
				if (winningName && onresult) {
					onresult(winningName);
				}
			}
		}

		// Animate winner text
		if (showWinnerOverlay) {
			winnerTextOpacity = Math.min(winnerTextOpacity + 0.04, 1);
			winnerTextScale = Math.min(winnerTextScale + 0.05, 1);
		}

		updateConfetti();

		ctx.clearRect(0, 0, viewWidth, viewHeight);

		// Apply camera transform (follow or result zoom)
		ctx.save();
		if (isZooming && zoomLevel > 1) {
			// Result zoom — zoom into winning bucket, relative to viewport
			const resultCx = Math.max(viewWidth / 2 / zoomLevel, Math.min(worldWidth - viewWidth / 2 / zoomLevel, zoomTargetX));
			const resultCy = Math.max(viewHeight / 2 / zoomLevel, Math.min(worldHeight - viewHeight / 2 / zoomLevel, zoomTargetY));

			ctx.translate(viewWidth / 2, viewHeight / 2);
			ctx.scale(zoomLevel, zoomLevel);
			ctx.translate(-resultCx, -resultCy);
		} else if (cameraZoom > 1.01) {
			// Ball-follow camera — zoom centred on ball, clamped to world bounds
			const visW = viewWidth / cameraZoom;
			const visH = viewHeight / cameraZoom;

			const cx = Math.max(visW / 2, Math.min(worldWidth - visW / 2, cameraX));
			const cy = Math.max(visH / 2, Math.min(worldHeight - visH / 2, cameraY));

			ctx.translate(viewWidth / 2, viewHeight / 2);
			ctx.scale(cameraZoom, cameraZoom);
			ctx.translate(-cx, -cy);
		} else if (worldHeight > viewHeight) {
			// Not dropping/zooming but world is taller — scale to fit so user sees everything
			const fitScale = viewHeight / worldHeight;
			const fitOffsetX = (viewWidth - worldWidth * fitScale) / 2;
			ctx.translate(fitOffsetX, 0);
			ctx.scale(fitScale, fitScale);
		}

		// Draw buckets
		const bucketY = TOP_PADDING + pegRows * pegSpacingY;
		for (let i = 0; i < buckets.length; i++) {
			const bucket = buckets[i];
			const isWinner = i === winningBucketIndex;

			if (isWinner && zoomProgress > 0) {
				const pulse = 0.6 + 0.4 * Math.sin(now / 200);
				ctx.shadowColor = bucket.colour;
				ctx.shadowBlur = 20 * pulse;
				ctx.fillStyle = bucket.colour + '60';
				ctx.fillRect(bucket.x, bucketY, bucket.width, bucketHeight);
				ctx.shadowBlur = 0;

				ctx.strokeStyle = bucket.colour;
				ctx.lineWidth = 3;
				ctx.strokeRect(bucket.x + 1.5, bucketY + 1.5, bucket.width - 3, bucketHeight - 3);
			} else {
				ctx.fillStyle = bucket.colour + '20';
				ctx.fillRect(bucket.x, bucketY, bucket.width, bucketHeight);
			}

			// Bucket label
			const isWinnerLabel = isWinner && zoomProgress > 0;
			ctx.fillStyle = isWinnerLabel ? '#ffffff' : bucket.colour;
			const fontSize = isWinnerLabel ? 14 : 11;
			ctx.font = `bold ${fontSize}px system-ui, sans-serif`;
			ctx.textAlign = 'center';
			ctx.textBaseline = 'middle';

			const bucketCenterX = bucket.x + bucket.width / 2;
			const bucketCenterY = bucketY + bucketHeight / 2;

			if (rotateLabels) {
				// Rotate text vertically for narrow buckets
				ctx.save();
				ctx.translate(bucketCenterX, bucketCenterY);
				ctx.rotate(-Math.PI / 2);
				// Truncate based on bucket height (since text runs vertically)
				const maxChars = Math.floor((bucketHeight - 10) / 7);
				const label =
					bucket.name.length > maxChars
						? bucket.name.slice(0, maxChars - 1) + '…'
						: bucket.name;
				ctx.fillText(label, 0, 0);
				ctx.restore();
			} else {
				const maxChars = Math.floor(bucket.width / 8);
				const label =
					bucket.name.length > maxChars
						? bucket.name.slice(0, maxChars - 1) + '…'
						: bucket.name;
				ctx.fillText(label, bucketCenterX, bucketCenterY);
			}
		}

		// Draw bucket dividers
		ctx.fillStyle = 'rgba(255, 255, 255, 0.3)';
		for (const divider of dividerBodies) {
			ctx.fillRect(
				divider.position.x - DIVIDER_WIDTH / 2,
				bucketY,
				DIVIDER_WIDTH,
				bucketHeight
			);
		}

		// Draw pegs
		for (const peg of pegs) {
			ctx.beginPath();
			ctx.arc(peg.x, peg.y, PEG_RADIUS, 0, Math.PI * 2);
			ctx.fillStyle = peg.colour;
			ctx.fill();

			ctx.beginPath();
			ctx.arc(peg.x, peg.y, PEG_RADIUS + 2, 0, Math.PI * 2);
			ctx.strokeStyle = peg.colour + '40';
			ctx.lineWidth = 2;
			ctx.stroke();
		}

		// Draw balls
		const allBodies = Matter.Composite.allBodies(engine!.world);
		for (const body of allBodies) {
			if (body.isStatic) continue;

			const { x, y } = body.position;
			const radius = (body as any).circleRadius || BALL_RADIUS;

			const gradient = ctx.createRadialGradient(x, y, 0, x, y, radius + 4);
			gradient.addColorStop(0, '#ffffff');
			gradient.addColorStop(0.6, '#fbbf24');
			gradient.addColorStop(1, '#f59e0b00');

			ctx.beginPath();
			ctx.arc(x, y, radius + 4, 0, Math.PI * 2);
			ctx.fillStyle = gradient;
			ctx.fill();

			ctx.beginPath();
			ctx.arc(x, y, radius, 0, Math.PI * 2);
			ctx.fillStyle = '#fbbf24';
			ctx.fill();
			ctx.strokeStyle = '#f59e0b';
			ctx.lineWidth = 2;
			ctx.stroke();
		}

		// Draw confetti (in world space so it zooms too)
		drawConfetti(ctx);

		ctx.restore(); // end zoom transform

		// Draw winner overlay text on top of everything (not zoomed)
		if (showWinnerOverlay && winningName) {
			const overlayScale = easeOutElastic(winnerTextScale);

			ctx.save();
			ctx.globalAlpha = winnerTextOpacity;
			ctx.translate(viewWidth / 2, viewHeight * 0.3);
			ctx.scale(overlayScale, overlayScale);

			ctx.font = 'bold 32px system-ui, sans-serif';
			ctx.fillStyle = 'rgba(0, 0, 0, 0.6)';
			const textWidth = ctx.measureText(winningName).width;
			const padding = 30;
			ctx.beginPath();
			const rectW = Math.max(textWidth + padding * 2, 200);
			ctx.roundRect(-rectW / 2, -35, rectW, 70, 16);
			ctx.fill();

			ctx.fillStyle = '#ffffff';
			ctx.textAlign = 'center';
			ctx.textBaseline = 'middle';
			ctx.fillText(winningName, 0, 0);

			ctx.restore();
		}

		animFrameId = requestAnimationFrame(renderLoop);
	}

	function cleanup() {
		if (animFrameId) cancelAnimationFrame(animFrameId);
		if (settleCheckInterval) {
			clearInterval(settleCheckInterval);
			settleCheckInterval = null;
		}
		if (runner) Matter.Runner.stop(runner);
		if (engine) {
			Matter.Composite.clear(engine.world, false);
			Matter.Engine.clear(engine);
		}
		engine = null;
		runner = null;
		ballBody = null;
		dropping = false;
		followingBall = false;
		cameraZoom = 1;
		winningBucketIndex = null;
		winningName = null;
		showWinnerOverlay = false;
		confetti = [];
		confettiSpawned = false;
		zoomProgress = 0;
		isZooming = false;
		zoomLevel = 1;
	}

	let resizeObserver: ResizeObserver | null = null;
	let containerReady = $state(false);

	// Watch container size changes
	$effect(() => {
		if (!containerEl) return;

		resizeObserver = new ResizeObserver(() => {
			containerReady = true;
			// Don't rebuild during a drop or result zoom — it would destroy the ball
			if (names.length >= 2 && !dropping && !isZooming) {
				buildBoard(names.length);
			}
		});
		resizeObserver.observe(containerEl);

		return () => {
			resizeObserver?.disconnect();
			resizeObserver = null;
		};
	});

	// Rebuild board when names change (once container is ready, not during a drop)
	$effect(() => {
		if (!containerReady || dropping || isZooming) return;
		if (names.length >= 2) {
			const _count = names.length;
			buildBoard(_count);
		} else {
			cleanup();
		}
	});

	onMount(() => {
		return () => cleanup();
	});
</script>

<div class="h-full w-full overflow-hidden" bind:this={containerEl}>
	{#if names.length < 2}
		<div class="flex h-full items-center justify-center text-center">
			<div>
				<p class="text-lg text-white/50">Add at least 2 names to get started</p>
				<p class="mt-2 text-sm text-white/30">The Plinko board will appear here</p>
			</div>
		</div>
	{:else}
		<canvas
			bind:this={canvasEl}
			width={viewWidth}
			height={viewHeight}
			style="width: {viewWidth}px; height: {viewHeight}px;"
		></canvas>
	{/if}
</div>
