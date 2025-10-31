<script>
	import { onMount } from 'svelte';

	// State Management
	let currentSvg = null;
	let previewActive = false;
	let isDragOver = false;
	let svgCode = '';
	let canvasElement;
	let fileName = '';
	let isProcessing = false;
	let processingMessage = '';

	// Customization Options
	let backgroundColor = 'transparent';
	let padding = 0;

	// Available sizes with selection state
	let sizes = [
		{ size: 16, name: 'favicon-16x16.png', selected: true, category: 'Browser' },
		{ size: 32, name: 'favicon-32x32.png', selected: true, category: 'Browser' },
		{ size: 48, name: 'favicon-48x48.png', selected: true, category: 'Browser' },
		{ size: 180, name: 'apple-touch-icon.png', selected: true, category: 'iOS' },
		{ size: 192, name: 'android-chrome-192x192.png', selected: true, category: 'Android' },
		{ size: 512, name: 'android-chrome-512x512.png', selected: true, category: 'Android' }
	];

	let sizePreviews = [];
	let toasts = [];
	let htmlSnippet = '';
	let manifestJson = '';

	// Toast Notification System
	function showToast(message, type = 'info', duration = 3000) {
		const id = Date.now();
		toasts = [...toasts, { id, message, type }];

		if (duration > 0) {
			setTimeout(() => {
				toasts = toasts.filter((t) => t.id !== id);
			}, duration);
		}
	}

	function removeToast(id) {
		toasts = toasts.filter((t) => t.id !== id);
	}

	// Keyboard Shortcuts
	onMount(() => {
		const handleKeyboard = (e) => {
			// Ctrl/Cmd + K to focus file input
			if ((e.ctrlKey || e.metaKey) && e.key === 'k') {
				e.preventDefault();
				document.querySelector('input[type="file"]')?.click();
			}
			// Ctrl/Cmd + R to reset
			if ((e.ctrlKey || e.metaKey) && e.key === 'r' && currentSvg) {
				e.preventDefault();
				resetAll();
			}
			// Escape to clear
			if (e.key === 'Escape' && currentSvg) {
				resetAll();
			}
		};

		window.addEventListener('keydown', handleKeyboard);
		return () => window.removeEventListener('keydown', handleKeyboard);
	});

	// Drag and Drop Handlers
	function handleDragOver(e) {
		e.preventDefault();
		isDragOver = true;
	}

	function handleDragLeave() {
		isDragOver = false;
	}

	function handleDrop(e) {
		e.preventDefault();
		isDragOver = false;

		const files = e.dataTransfer.files;
		if (files.length > 0) {
			handleFile(files[0]);
		}
	}

	// Keyboard support for upload area
	function handleUploadKeyDown(e) {
		if (e.key === 'Enter' || e.key === ' ') {
			e.preventDefault();
			document.querySelector('input[type="file"]')?.click();
		}
	}

	// File Handling
	function handleFileInput(e) {
		if (e.target.files.length > 0) {
			handleFile(e.target.files[0]);
		}
	}

	function handleFile(file) {
		if (!file.type.includes('svg')) {
			showToast('Please upload an SVG file', 'error');
			return;
		}

		fileName = file.name;
		const reader = new FileReader();

		reader.onload = (e) => {
			try {
				loadSvg(e.target.result);
			} catch (error) {
				showToast('Failed to read file: ' + error.message, 'error');
			}
		};

		reader.onerror = () => {
			showToast('Failed to read file', 'error');
		};

		reader.readAsText(file);
	}

	function loadFromTextarea() {
		const code = svgCode.trim();
		if (!code) {
			showToast('Please paste SVG code first', 'error');
			return;
		}
		fileName = 'pasted-svg.svg';
		loadSvg(code);
	}

	// SVG Validation and Loading
	function validateSvg(svgContent) {
		const parser = new DOMParser();
		const doc = parser.parseFromString(svgContent, 'image/svg+xml');
		const parserError = doc.querySelector('parsererror');

		if (parserError) {
			throw new Error('Invalid SVG: XML parsing error');
		}

		const svgElement = doc.querySelector('svg');
		if (!svgElement) {
			throw new Error('Invalid SVG: No <svg> element found');
		}

		return true;
	}

	async function loadSvg(svgContent) {
		try {
			isProcessing = true;
			processingMessage = 'Validating SVG...';

			validateSvg(svgContent);
			currentSvg = svgContent;
			previewActive = true;

			processingMessage = 'Generating previews...';
			await generateSizePreviews();
			generateHtmlSnippet();
			generateManifest();

			showToast('SVG loaded successfully!', 'success');
		} catch (error) {
			showToast('Error loading SVG: ' + error.message, 'error');
			console.error(error);
		} finally {
			isProcessing = false;
			processingMessage = '';
		}
	}

	// Generate Size Previews
	async function generateSizePreviews() {
		sizePreviews = [];
		const selectedSizes = sizes.filter((s) => s.selected);

		for (let i = 0; i < selectedSizes.length; i++) {
			const item = selectedSizes[i];
			processingMessage = `Generating preview ${i + 1} of ${selectedSizes.length}...`;

			try {
				const img = await svgToPng(currentSvg, item.size, item.size);
				sizePreviews.push({
					...item,
					imgSrc: img.src
				});
			} catch (error) {
				showToast(`Failed to generate ${item.size}x${item.size} preview`, 'error');
				console.error(error);
			}
		}
	}

	// SVG to PNG Conversion with Background and Padding
	async function svgToPng(svgContent, width, height) {
		return new Promise((resolve, reject) => {
			const img = new Image();
			const blob = new Blob([svgContent], { type: 'image/svg+xml' });
			const url = URL.createObjectURL(blob);

			img.onload = () => {
				try {
					canvasElement.width = width;
					canvasElement.height = height;
					const ctx = canvasElement.getContext('2d');

					// Apply background
					if (backgroundColor !== 'transparent') {
						ctx.fillStyle = backgroundColor;
						ctx.fillRect(0, 0, width, height);
					} else {
						ctx.clearRect(0, 0, width, height);
					}

					// Apply padding
					const paddedSize = width - padding * 2;
					const x = padding;
					const y = padding;

					ctx.drawImage(img, x, y, paddedSize, paddedSize);

					const resultImg = new Image();
					resultImg.src = canvasElement.toDataURL('image/png');

					URL.revokeObjectURL(url);
					resolve(resultImg);
				} catch (error) {
					URL.revokeObjectURL(url);
					reject(error);
				}
			};

			img.onerror = () => {
				URL.revokeObjectURL(url);
				reject(new Error('Failed to load SVG image'));
			};

			img.src = url;
		});
	}

	// Download Individual PNG
	async function downloadPNG(size, filename) {
		try {
			isProcessing = true;
			processingMessage = `Generating ${filename}...`;

			canvasElement.width = size;
			canvasElement.height = size;
			const ctx = canvasElement.getContext('2d');

			const blob = new Blob([currentSvg], { type: 'image/svg+xml' });
			const url = URL.createObjectURL(blob);

			const img = new Image();
			img.onload = () => {
				// Apply background
				if (backgroundColor !== 'transparent') {
					ctx.fillStyle = backgroundColor;
					ctx.fillRect(0, 0, size, size);
				} else {
					ctx.clearRect(0, 0, size, size);
				}

				// Apply padding
				const paddedSize = size - padding * 2;
				const x = padding;
				const y = padding;

				ctx.drawImage(img, x, y, paddedSize, paddedSize);

				canvasElement.toBlob((blob) => {
					const a = document.createElement('a');
					a.href = URL.createObjectURL(blob);
					a.download = filename;
					document.body.appendChild(a);
					a.click();
					document.body.removeChild(a);
					URL.revokeObjectURL(url);

					showToast(`Downloaded ${filename}`, 'success', 2000);
					isProcessing = false;
					processingMessage = '';
				}, 'image/png');
			};

			img.onerror = () => {
				URL.revokeObjectURL(url);
				showToast(`Failed to generate ${filename}`, 'error');
				isProcessing = false;
				processingMessage = '';
			};

			img.src = url;
		} catch (error) {
			showToast(`Error: ${error.message}`, 'error');
			isProcessing = false;
			processingMessage = '';
		}
	}

	// Download All PNGs
	async function downloadAllPNG() {
		const selectedSizes = sizes.filter((s) => s.selected);

		if (selectedSizes.length === 0) {
			showToast('Please select at least one size', 'error');
			return;
		}

		try {
			isProcessing = true;

			for (let i = 0; i < selectedSizes.length; i++) {
				const item = selectedSizes[i];
				processingMessage = `Downloading ${i + 1} of ${selectedSizes.length}...`;
				await downloadPNG(item.size, item.name);
				await new Promise((resolve) => setTimeout(resolve, 300));
			}

			showToast(`Successfully downloaded ${selectedSizes.length} files!`, 'success');
		} catch (error) {
			showToast('Error during bulk download: ' + error.message, 'error');
		} finally {
			isProcessing = false;
			processingMessage = '';
		}
	}

	// Download as ZIP
	async function downloadAsZip() {
		const selectedSizes = sizes.filter((s) => s.selected);

		if (selectedSizes.length === 0) {
			showToast('Please select at least one size', 'error');
			return;
		}

		try {
			isProcessing = true;
			processingMessage = 'Creating ZIP file...';

			// Note: This is a simplified version. In production, you'd use JSZip library
			showToast('ZIP download requires additional library (JSZip)', 'error', 5000);
		} catch (error) {
			showToast('Error creating ZIP: ' + error.message, 'error');
		} finally {
			isProcessing = false;
			processingMessage = '';
		}
	}

	// Generate Favicon ICO
	async function generateFaviconIco() {
		try {
			isProcessing = true;
			processingMessage = 'Generating favicon.ico...';

			const icoSizes = [16, 32, 48];
			const images = [];

			for (const size of icoSizes) {
				canvasElement.width = size;
				canvasElement.height = size;
				const ctx = canvasElement.getContext('2d');

				const blob = new Blob([currentSvg], { type: 'image/svg+xml' });
				const url = URL.createObjectURL(blob);

				const imageData = await new Promise((resolve, reject) => {
					const img = new Image();
					img.onload = () => {
						if (backgroundColor !== 'transparent') {
							ctx.fillStyle = backgroundColor;
							ctx.fillRect(0, 0, size, size);
						} else {
							ctx.clearRect(0, 0, size, size);
						}

						const paddedSize = size - padding * 2;
						const x = padding;
						const y = padding;

						ctx.drawImage(img, x, y, paddedSize, paddedSize);
						resolve(ctx.getImageData(0, 0, size, size));
					};
					img.onerror = () => reject(new Error('Failed to load image'));
					img.src = url;
				});

				images.push({ size, imageData });
			}

			const icoData = createIcoFile(images);
			const blob = new Blob([icoData], { type: 'image/x-icon' });

			const a = document.createElement('a');
			a.href = URL.createObjectURL(blob);
			a.download = 'favicon.ico';
			document.body.appendChild(a);
			a.click();
			document.body.removeChild(a);

			showToast('favicon.ico generated successfully!', 'success');
		} catch (error) {
			showToast('Error generating ICO: ' + error.message, 'error');
		} finally {
			isProcessing = false;
			processingMessage = '';
		}
	}

	// ICO File Creation Helper
	function createIcoFile(images) {
		const numImages = images.length;
		let dataOffset = 6 + numImages * 16;
		const imageBuffers = [];

		for (const img of images) {
			const bmpData = createBMPData(img.imageData);
			imageBuffers.push(bmpData);
		}

		let totalSize = dataOffset;
		for (const buf of imageBuffers) {
			totalSize += buf.length;
		}

		const output = new Uint8Array(totalSize);
		const view = new DataView(output.buffer);

		view.setUint16(0, 0, true);
		view.setUint16(2, 1, true);
		view.setUint16(4, numImages, true);

		let offset = dataOffset;
		for (let i = 0; i < numImages; i++) {
			const img = images[i];
			const buf = imageBuffers[i];
			const entryOffset = 6 + i * 16;

			output[entryOffset] = img.size === 256 ? 0 : img.size;
			output[entryOffset + 1] = img.size === 256 ? 0 : img.size;
			output[entryOffset + 2] = 0;
			output[entryOffset + 3] = 0;
			view.setUint16(entryOffset + 4, 1, true);
			view.setUint16(entryOffset + 6, 32, true);
			view.setUint32(entryOffset + 8, buf.length, true);
			view.setUint32(entryOffset + 12, offset, true);

			offset += buf.length;
		}

		offset = dataOffset;
		for (const buf of imageBuffers) {
			output.set(buf, offset);
			offset += buf.length;
		}

		return output;
	}

	function createBMPData(imageData) {
		const width = imageData.width;
		const height = imageData.height;
		const data = imageData.data;

		const headerSize = 40;
		const rowSize = Math.floor((width * 32 + 31) / 32) * 4;
		const imageSize = rowSize * height;
		const andMaskRowSize = Math.floor((width + 31) / 32) * 4;
		const andMaskSize = andMaskRowSize * height;
		const totalSize = headerSize + imageSize + andMaskSize;

		const buffer = new Uint8Array(totalSize);
		const view = new DataView(buffer.buffer);

		view.setUint32(0, headerSize, true);
		view.setInt32(4, width, true);
		view.setInt32(8, height * 2, true);
		view.setUint16(12, 1, true);
		view.setUint16(14, 32, true);
		view.setUint32(16, 0, true);
		view.setUint32(20, imageSize, true);
		view.setInt32(24, 0, true);
		view.setInt32(28, 0, true);
		view.setUint32(32, 0, true);
		view.setUint32(36, 0, true);

		let offset = headerSize;
		for (let y = height - 1; y >= 0; y--) {
			for (let x = 0; x < width; x++) {
				const i = (y * width + x) * 4;
				buffer[offset++] = data[i + 2];
				buffer[offset++] = data[i + 1];
				buffer[offset++] = data[i];
				buffer[offset++] = data[i + 3];
			}
			while (offset % 4 !== 0) {
				buffer[offset++] = 0;
			}
		}

		for (let i = 0; i < andMaskSize; i++) {
			buffer[offset++] = 0;
		}

		return buffer;
	}

	// Generate HTML Snippet
	function generateHtmlSnippet() {
		htmlSnippet = `<!-- Favicon Links -->
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
<link rel="icon" type="image/png" sizes="192x192" href="/android-chrome-192x192.png">
<link rel="icon" type="image/png" sizes="512x512" href="/android-chrome-512x512.png">
<link rel="manifest" href="/site.webmanifest">`;
	}

	// Generate Web Manifest
	function generateManifest() {
		const manifest = {
			name: 'Your App Name',
			short_name: 'App',
			icons: [
				{
					src: '/android-chrome-192x192.png',
					sizes: '192x192',
					type: 'image/png'
				},
				{
					src: '/android-chrome-512x512.png',
					sizes: '512x512',
					type: 'image/png'
				}
			],
			theme_color: '#ffffff',
			background_color: '#ffffff',
			display: 'standalone'
		};
		manifestJson = JSON.stringify(manifest, null, 2);
	}

	// Copy to Clipboard
	async function copyToClipboard(text, label) {
		try {
			await navigator.clipboard.writeText(text);
			showToast(`${label} copied to clipboard!`, 'success', 2000);
		} catch (error) {
			showToast('Failed to copy to clipboard', 'error');
		}
	}

	// Download as Text File
	function downloadTextFile(content, filename) {
		const blob = new Blob([content], { type: 'text/plain' });
		const a = document.createElement('a');
		a.href = URL.createObjectURL(blob);
		a.download = filename;
		document.body.appendChild(a);
		a.click();
		document.body.removeChild(a);
		showToast(`${filename} downloaded!`, 'success', 2000);
	}

	// Reset All
	function resetAll() {
		currentSvg = null;
		previewActive = false;
		svgCode = '';
		fileName = '';
		sizePreviews = [];
		htmlSnippet = '';
		manifestJson = '';
		backgroundColor = 'transparent';
		padding = 0;
		sizes = sizes.map((s) => ({ ...s, selected: true }));
		showToast('Reset complete', 'info', 2000);
	}

	// Regenerate previews when options change
	async function regeneratePreviews() {
		if (!currentSvg) return;

		try {
			isProcessing = true;
			processingMessage = 'Regenerating previews...';
			await generateSizePreviews();
			showToast('Previews updated!', 'success', 2000);
		} catch (error) {
			showToast('Failed to regenerate previews', 'error');
		} finally {
			isProcessing = false;
			processingMessage = '';
		}
	}

	// Toggle all sizes
	function toggleAllSizes() {
		const allSelected = sizes.every((s) => s.selected);
		sizes = sizes.map((s) => ({ ...s, selected: !allSelected }));
	}
</script>

<div class="page-wrapper">
	<!-- Toast Notifications -->
	<div class="toast-container" aria-live="polite" aria-atomic="true">
		{#each toasts as toast (toast.id)}
			<div class="toast toast-{toast.type}" role="alert">
				<span class="toast-icon">
					{#if toast.type === 'success'}✓{/if}
					{#if toast.type === 'error'}✕{/if}
					{#if toast.type === 'info'}ℹ{/if}
				</span>
				<span class="toast-message">{toast.message}</span>
				<button
					class="toast-close"
					on:click={() => removeToast(toast.id)}
					aria-label="Close notification">×</button
				>
			</div>
		{/each}
	</div>

	<!-- Loading Overlay -->
	{#if isProcessing}
		<div class="loading-overlay">
			<div class="loading-spinner"></div>
			<p>{processingMessage}</p>
		</div>
	{/if}

	<div class="container">
		<!-- Header Section -->
		<header class="header">
			<h1>🎨 SVG to Favicon Generator</h1>
			<p>Upload your SVG and generate all favicon sizes instantly</p>
			<div class="keyboard-hints">
				<kbd>Ctrl+K</kbd> Upload • <kbd>Ctrl+R</kbd> Reset • <kbd>Esc</kbd> Clear
			</div>
		</header>

		<!-- Upload Section -->
		<section class="upload-section">
			<div
				class="upload-area"
				class:dragover={isDragOver}
				on:dragover={handleDragOver}
				on:dragleave={handleDragLeave}
				on:drop={handleDrop}
				on:keydown={handleUploadKeyDown}
				role="button"
				tabindex="0"
				aria-label="Upload SVG file"
			>
				<div class="upload-icon">📁</div>
				<h3>Drop your SVG file here</h3>
				<p>or click to browse</p>
				{#if fileName}
					<p class="file-name">Current: {fileName}</p>
				{/if}
				<label class="btn">
					Choose File
					<input
						type="file"
						accept=".svg,image/svg+xml"
						on:change={handleFileInput}
						aria-label="Choose SVG file"
						hidden
					/>
				</label>
			</div>

			<div class="divider">
				<span>OR</span>
			</div>

			<div class="textarea-section">
				<label for="svgCode">Paste SVG Code:</label>
				<textarea
					id="svgCode"
					bind:value={svgCode}
					placeholder="Paste your SVG code here..."
					aria-label="SVG code input"
				></textarea>
				<button class="btn" on:click={loadFromTextarea}> Load SVG </button>
			</div>

			{#if currentSvg}
				<div class="reset-section">
					<button class="btn btn-secondary" on:click={resetAll}> 🔄 Reset All </button>
				</div>
			{/if}
		</section>

		<!-- Customization Options -->
		{#if previewActive}
			<section class="options-section">
				<h2>⚙️ Customization Options</h2>

				<div class="options-grid">
					<div class="option-group">
						<label for="backgroundColor">Background Color:</label>
						<div class="color-options">
							<button
								class="color-btn"
								class:active={backgroundColor === 'transparent'}
								on:click={() => {
									backgroundColor = 'transparent';
									regeneratePreviews();
								}}
								aria-label="Transparent background"
							>
								<div class="color-preview transparent"></div>
								Transparent
							</button>
							<button
								class="color-btn"
								class:active={backgroundColor === '#ffffff'}
								on:click={() => {
									backgroundColor = '#ffffff';
									regeneratePreviews();
								}}
								aria-label="White background"
							>
								<div class="color-preview" style="background: #ffffff;"></div>
								White
							</button>
							<button
								class="color-btn"
								class:active={backgroundColor === '#000000'}
								on:click={() => {
									backgroundColor = '#000000';
									regeneratePreviews();
								}}
								aria-label="Black background"
							>
								<div class="color-preview" style="background: #000000;"></div>
								Black
							</button>
							<label
								class="color-btn"
								class:active={!['transparent', '#ffffff', '#000000'].includes(backgroundColor)}
							>
								<div
									class="color-preview"
									style="background: {backgroundColor === 'transparent'
										? '#ccc'
										: backgroundColor};"
								></div>
								Custom
								<input
									type="color"
									bind:value={backgroundColor}
									on:change={regeneratePreviews}
									hidden
								/>
							</label>
						</div>
					</div>

					<div class="option-group">
						<label for="padding">
							Padding: {padding}px
						</label>
						<input
							type="range"
							id="padding"
							bind:value={padding}
							on:change={regeneratePreviews}
							min="0"
							max="50"
							step="1"
							aria-label="Icon padding"
						/>
					</div>
				</div>
			</section>

			<!-- Size Selection -->
			<section class="size-selection-section">
				<div class="section-header">
					<h2>📐 Select Sizes to Generate</h2>
					<button class="btn btn-text" on:click={toggleAllSizes}>
						{sizes.every((s) => s.selected) ? 'Deselect All' : 'Select All'}
					</button>
				</div>

				<div class="size-checkboxes">
					{#each sizes as size}
						<label class="checkbox-label">
							<input type="checkbox" bind:checked={size.selected} on:change={regeneratePreviews} />
							<span class="checkbox-text">
								<strong>{size.size}×{size.size}px</strong>
								<small>{size.category}</small>
							</span>
						</label>
					{/each}
				</div>
			</section>

			<!-- Preview Section -->
			<section class="preview-section">
				<h2>👀 Preview & Generate</h2>

				<div
					class="preview-container"
					style="background: {backgroundColor === 'transparent'
						? 'url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAMUlEQVQ4T2NkYGAQYcAP3uCTZhw1gGGYhAGBZIA/nYDCgBDAm9BGDWAAJyRCgLaBCAAgXwixzAS0pgAAAABJRU5ErkJggg==)'
						: backgroundColor}"
				>
					<div class="svg-preview">
						{@html currentSvg}
					</div>
				</div>

				<div class="bulk-actions">
					<button class="btn btn-primary" on:click={downloadAllPNG} disabled={isProcessing}>
						📦 Download All PNG Files
					</button>
					<button class="btn btn-primary" on:click={generateFaviconIco} disabled={isProcessing}>
						🔷 Generate favicon.ico
					</button>
				</div>

				<h3>Individual Sizes</h3>

				<div class="sizes-grid">
					{#each sizePreviews as preview}
						<div class="size-card">
							<h4>{preview.size}×{preview.size}px</h4>
							<div
								class="size-preview"
								style="background: {backgroundColor === 'transparent'
									? 'url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAMUlEQVQ4T2NkYGAQYcAP3uCTZhw1gGGYhAGBZIA/nYDCgBDAm9BGDWAAJyRCgLaBCAAgXwixzAS0pgAAAABJRU5ErkJggg==)'
									: backgroundColor}"
							>
								<img
									src={preview.imgSrc}
									alt="{preview.size}x{preview.size} preview"
									width={preview.size}
									height={preview.size}
								/>
							</div>
							<button
								on:click={() => downloadPNG(preview.size, preview.name)}
								disabled={isProcessing}
							>
								Download {preview.name}
							</button>
						</div>
					{/each}
				</div>
			</section>

			<!-- HTML Snippet Section -->
			<section class="code-section">
				<h2>📝 HTML Code Snippet</h2>
				<p class="section-description">Add these tags to your HTML &lt;head&gt;</p>

				<div class="code-block">
					<pre><code>{htmlSnippet}</code></pre>
					<div class="code-actions">
						<button
							class="btn btn-secondary btn-sm"
							on:click={() => copyToClipboard(htmlSnippet, 'HTML snippet')}
						>
							📋 Copy
						</button>
						<button
							class="btn btn-secondary btn-sm"
							on:click={() => downloadTextFile(htmlSnippet, 'favicon-links.html')}
						>
							⬇️ Download
						</button>
					</div>
				</div>
			</section>

			<!-- Web Manifest Section -->
			<section class="code-section">
				<h2>📱 Web App Manifest</h2>
				<p class="section-description">Save as site.webmanifest in your root directory</p>

				<div class="code-block">
					<pre><code>{manifestJson}</code></pre>
					<div class="code-actions">
						<button
							class="btn btn-secondary btn-sm"
							on:click={() => copyToClipboard(manifestJson, 'Manifest')}
						>
							📋 Copy
						</button>
						<button
							class="btn btn-secondary btn-sm"
							on:click={() => downloadTextFile(manifestJson, 'site.webmanifest')}
						>
							⬇️ Download
						</button>
					</div>
				</div>
			</section>
		{/if}

		<!-- Info Section -->
		<section class="info-box">
			<h3>📋 Generated Files</h3>
			<ul>
				<li><strong>favicon-16x16.png</strong> - Tiny browser tab icon</li>
				<li><strong>favicon-32x32.png</strong> - Standard browser tab icon</li>
				<li><strong>favicon-48x48.png</strong> - High-DPI browser tab icon</li>
				<li><strong>apple-touch-icon.png (180x180)</strong> - iOS home screen icon</li>
				<li><strong>android-chrome-192x192.png</strong> - Android home screen icon</li>
				<li><strong>android-chrome-512x512.png</strong> - High-res Android icon & PWA splash</li>
				<li><strong>favicon.ico</strong> - Multi-resolution ICO file (16, 32, 48)</li>
			</ul>
		</section>
	</div>
</div>

<canvas bind:this={canvasElement} style="display: none;" aria-hidden="true"></canvas>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		font-family:
			-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
	}

	.page-wrapper {
		min-height: 100vh;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		padding: 3rem 1.5rem;
		position: relative;
	}

	.container {
		max-width: 1200px;
		margin: 0 auto;
		display: flex;
		flex-direction: column;
		gap: 2rem;
	}

	/* Toast Notifications */
	.toast-container {
		position: fixed;
		top: 1.5rem;
		right: 1.5rem;
		z-index: 1000;
		display: flex;
		flex-direction: column;
		gap: 0.75rem;
		max-width: 400px;
	}

	.toast {
		background: white;
		padding: 1rem 1.25rem;
		border-radius: 0.5rem;
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
		display: flex;
		align-items: center;
		gap: 0.75rem;
		animation: slideIn 0.3s ease;
		border-left: 4px solid;
	}

	.toast-success {
		border-color: #10b981;
	}

	.toast-error {
		border-color: #ef4444;
	}

	.toast-info {
		border-color: #3b82f6;
	}

	.toast-icon {
		font-size: 1.25rem;
		font-weight: bold;
	}

	.toast-success .toast-icon {
		color: #10b981;
	}

	.toast-error .toast-icon {
		color: #ef4444;
	}

	.toast-info .toast-icon {
		color: #3b82f6;
	}

	.toast-message {
		flex: 1;
		color: #333;
	}

	.toast-close {
		background: none;
		border: none;
		font-size: 1.5rem;
		color: #999;
		cursor: pointer;
		padding: 0;
		width: 24px;
		height: 24px;
		display: flex;
		align-items: center;
		justify-content: center;
		transition: color 0.2s;
	}

	.toast-close:hover {
		color: #333;
	}

	@keyframes slideIn {
		from {
			transform: translateX(100%);
			opacity: 0;
		}
		to {
			transform: translateX(0);
			opacity: 1;
		}
	}

	/* Loading Overlay */
	.loading-overlay {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(0, 0, 0, 0.7);
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		z-index: 999;
		gap: 1.5rem;
	}

	.loading-overlay p {
		color: white;
		font-size: 1.1rem;
		font-weight: 500;
	}

	.loading-spinner {
		width: 60px;
		height: 60px;
		border: 4px solid rgba(255, 255, 255, 0.3);
		border-top-color: white;
		border-radius: 50%;
		animation: spin 0.8s linear infinite;
	}

	@keyframes spin {
		to {
			transform: rotate(360deg);
		}
	}

	/* Header */
	.header {
		text-align: center;
		color: white;
		margin-bottom: 1rem;
	}

	.header h1 {
		font-size: clamp(2rem, 5vw, 2.5rem);
		margin: 0 0 0.75rem 0;
		text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.2);
		font-weight: 700;
	}

	.header p {
		font-size: clamp(1rem, 2.5vw, 1.1rem);
		margin: 0 0 1rem 0;
		opacity: 0.95;
	}

	.keyboard-hints {
		font-size: 0.875rem;
		opacity: 0.9;
	}

	.keyboard-hints kbd {
		background: rgba(255, 255, 255, 0.2);
		padding: 0.25rem 0.5rem;
		border-radius: 0.25rem;
		font-family: monospace;
		font-size: 0.85em;
	}

	/* Upload Section */
	.upload-section {
		background: white;
		border-radius: 1rem;
		padding: 2.5rem;
		box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
	}

	.upload-area {
		border: 3px dashed #667eea;
		border-radius: 0.75rem;
		padding: 3rem 2rem;
		text-align: center;
		background: #f8f9ff;
		cursor: pointer;
		transition: all 0.3s ease;
		outline: none;
	}

	.upload-area:hover,
	.upload-area:focus,
	.upload-area.dragover {
		border-color: #764ba2;
		background: #f0f2ff;
		transform: translateY(-2px);
	}

	.upload-icon {
		font-size: 3rem;
		margin-bottom: 1rem;
	}

	.upload-area h3 {
		color: #333;
		margin: 0 0 0.5rem 0;
		font-size: 1.25rem;
	}

	.upload-area p {
		color: #666;
		margin: 0 0 1.5rem 0;
	}

	.file-name {
		font-weight: 600;
		color: #667eea !important;
		margin-top: 0.5rem !important;
	}

	.divider {
		text-align: center;
		margin: 2rem 0;
		color: #999;
		position: relative;
	}

	.divider::before,
	.divider::after {
		content: '';
		position: absolute;
		top: 50%;
		width: 40%;
		height: 1px;
		background: #e0e0e0;
	}

	.divider::before {
		left: 0;
	}

	.divider::after {
		right: 0;
	}

	.divider span {
		background: white;
		padding: 0 1rem;
		position: relative;
		z-index: 1;
		font-weight: 600;
	}

	.textarea-section {
		margin-top: 2rem;
	}

	.textarea-section label {
		display: block;
		margin-bottom: 0.75rem;
		color: #333;
		font-weight: 600;
		font-size: 1rem;
	}

	.textarea-section textarea {
		width: 100%;
		min-height: 150px;
		padding: 1rem;
		border: 2px solid #e0e0e0;
		border-radius: 0.5rem;
		font-family: 'Courier New', monospace;
		font-size: 0.875rem;
		resize: vertical;
		transition: border-color 0.2s;
	}

	.textarea-section textarea:focus {
		outline: none;
		border-color: #667eea;
	}

	.textarea-section button {
		margin-top: 1rem;
	}

	.reset-section {
		margin-top: 2rem;
		padding-top: 2rem;
		border-top: 1px solid #e0e0e0;
		text-align: center;
	}

	/* Buttons */
	.btn {
		display: inline-block;
		padding: 0.875rem 2rem;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
		border: none;
		border-radius: 0.5rem;
		font-size: 1rem;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s ease;
		box-shadow: 0 2px 8px rgba(102, 126, 234, 0.3);
	}

	.btn:hover:not(:disabled) {
		transform: translateY(-2px);
		box-shadow: 0 5px 15px rgba(102, 126, 234, 0.4);
	}

	.btn:active:not(:disabled) {
		transform: translateY(0);
	}

	.btn:disabled {
		opacity: 0.6;
		cursor: not-allowed;
	}

	.btn-primary {
		padding: 1rem 2.5rem;
		font-size: 1.05rem;
	}

	.btn-secondary {
		background: #6b7280;
		box-shadow: 0 2px 8px rgba(107, 114, 128, 0.3);
	}

	.btn-secondary:hover:not(:disabled) {
		box-shadow: 0 5px 15px rgba(107, 114, 128, 0.4);
	}

	.btn-text {
		background: transparent;
		color: #667eea;
		box-shadow: none;
		padding: 0.5rem 1rem;
	}

	.btn-text:hover {
		background: rgba(102, 126, 234, 0.1);
		transform: none;
	}

	.btn-sm {
		padding: 0.5rem 1rem;
		font-size: 0.875rem;
	}

	/* Options Section */
	.options-section {
		background: white;
		border-radius: 1rem;
		padding: 2.5rem;
		box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
	}

	.options-section h2 {
		color: #333;
		margin: 0 0 1.5rem 0;
		font-size: 1.5rem;
	}

	.options-grid {
		display: grid;
		gap: 2rem;
	}

	.option-group label {
		display: block;
		margin-bottom: 1rem;
		color: #333;
		font-weight: 600;
	}

	.color-options {
		display: flex;
		gap: 0.75rem;
		flex-wrap: wrap;
	}

	.color-btn {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 0.5rem;
		padding: 0.75rem 1rem;
		border: 2px solid #e0e0e0;
		border-radius: 0.5rem;
		background: white;
		cursor: pointer;
		transition: all 0.2s;
		font-size: 0.875rem;
		color: #666;
	}

	.color-btn:hover {
		border-color: #667eea;
		transform: translateY(-2px);
	}

	.color-btn.active {
		border-color: #667eea;
		background: #f8f9ff;
		color: #667eea;
		font-weight: 600;
	}

	.color-preview {
		width: 40px;
		height: 40px;
		border-radius: 0.375rem;
		border: 2px solid #e0e0e0;
	}

	.color-preview.transparent {
		background:
			linear-gradient(45deg, #ccc 25%, transparent 25%),
			linear-gradient(-45deg, #ccc 25%, transparent 25%),
			linear-gradient(45deg, transparent 75%, #ccc 75%),
			linear-gradient(-45deg, transparent 75%, #ccc 75%);
		background-size: 10px 10px;
		background-position:
			0 0,
			0 5px,
			5px -5px,
			-5px 0px;
	}

	input[type='range'] {
		width: 100%;
		height: 6px;
		border-radius: 3px;
		background: #e0e0e0;
		outline: none;
		-webkit-appearance: none;
	}

	input[type='range']::-webkit-slider-thumb {
		-webkit-appearance: none;
		appearance: none;
		width: 20px;
		height: 20px;
		border-radius: 50%;
		background: #667eea;
		cursor: pointer;
		transition: all 0.2s;
	}

	input[type='range']::-webkit-slider-thumb:hover {
		background: #764ba2;
		transform: scale(1.1);
	}

	input[type='range']::-moz-range-thumb {
		width: 20px;
		height: 20px;
		border-radius: 50%;
		background: #667eea;
		cursor: pointer;
		border: none;
		transition: all 0.2s;
	}

	input[type='range']::-moz-range-thumb:hover {
		background: #764ba2;
		transform: scale(1.1);
	}

	/* Size Selection */
	.size-selection-section {
		background: white;
		border-radius: 1rem;
		padding: 2.5rem;
		box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
	}

	.section-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 1.5rem;
		flex-wrap: wrap;
		gap: 1rem;
	}

	.section-header h2 {
		margin: 0;
		color: #333;
		font-size: 1.5rem;
	}

	.size-checkboxes {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
		gap: 1rem;
	}

	.checkbox-label {
		display: flex;
		align-items: center;
		gap: 0.75rem;
		padding: 1rem;
		background: #f8f9ff;
		border-radius: 0.5rem;
		cursor: pointer;
		transition: all 0.2s;
	}

	.checkbox-label:hover {
		background: #f0f2ff;
	}

	.checkbox-label input[type='checkbox'] {
		width: 20px;
		height: 20px;
		cursor: pointer;
	}

	.checkbox-text {
		display: flex;
		flex-direction: column;
		gap: 0.25rem;
	}

	.checkbox-text strong {
		color: #333;
	}

	.checkbox-text small {
		color: #666;
		font-size: 0.875rem;
	}

	/* Preview Section */
	.preview-section {
		background: white;
		border-radius: 1rem;
		padding: 2.5rem;
		box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
		display: flex;
		flex-direction: column;
		gap: 2rem;
	}

	.preview-section h2,
	.preview-section h3 {
		color: #333;
		margin: 0;
		font-size: 1.5rem;
	}

	.preview-section h3 {
		font-size: 1.25rem;
		margin-top: 1rem;
	}

	.preview-container {
		display: flex;
		justify-content: center;
		align-items: center;
		min-height: 200px;
		border-radius: 0.75rem;
		padding: 2rem;
		border: 2px solid #e0e0e0;
	}

	.svg-preview :global(svg) {
		max-width: 300px;
		max-height: 300px;
	}

	/* Bulk Actions */
	.bulk-actions {
		background: #f8f9ff;
		border-radius: 0.75rem;
		padding: 2rem;
		display: flex;
		gap: 1rem;
		flex-wrap: wrap;
		justify-content: center;
	}

	/* Sizes Grid */
	.sizes-grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
		gap: 1.5rem;
	}

	.size-card {
		background: #f8f9ff;
		border-radius: 0.75rem;
		padding: 1.5rem;
		text-align: center;
		transition:
			transform 0.2s,
			box-shadow 0.2s;
	}

	.size-card:hover {
		transform: translateY(-4px);
		box-shadow: 0 8px 20px rgba(0, 0, 0, 0.1);
	}

	.size-card h4 {
		color: #333;
		margin: 0 0 1rem 0;
		font-size: 1rem;
	}

	.size-preview {
		border-radius: 0.5rem;
		padding: 1.25rem;
		margin-bottom: 1rem;
		display: flex;
		justify-content: center;
		align-items: center;
		min-height: 100px;
		border: 2px solid #e0e0e0;
	}

	.size-preview img {
		display: block;
	}

	.size-card button {
		width: 100%;
		padding: 0.75rem 1rem;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
		border: none;
		border-radius: 0.5rem;
		font-size: 0.875rem;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s ease;
	}

	.size-card button:hover:not(:disabled) {
		transform: translateY(-2px);
		box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
	}

	.size-card button:disabled {
		opacity: 0.6;
		cursor: not-allowed;
	}

	/* Code Section */
	.code-section {
		background: white;
		border-radius: 1rem;
		padding: 2.5rem;
		box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
	}

	.code-section h2 {
		color: #333;
		margin: 0 0 0.5rem 0;
		font-size: 1.5rem;
	}

	.section-description {
		color: #666;
		margin: 0 0 1.5rem 0;
	}

	.code-block {
		position: relative;
		background: #1e1e1e;
		border-radius: 0.5rem;
		overflow: hidden;
	}

	.code-block pre {
		margin: 0;
		padding: 1.5rem;
		overflow-x: auto;
	}

	.code-block code {
		color: #d4d4d4;
		font-family: 'Courier New', monospace;
		font-size: 0.875rem;
		line-height: 1.6;
	}

	.code-actions {
		display: flex;
		gap: 0.5rem;
		padding: 1rem 1.5rem;
		background: #2d2d2d;
		border-top: 1px solid #444;
	}

	/* Info Box */
	.info-box {
		background: white;
		border-radius: 1rem;
		padding: 2.5rem;
		box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
	}

	.info-box h3 {
		color: #333;
		margin: 0 0 1.5rem 0;
		font-size: 1.25rem;
	}

	.info-box ul {
		list-style: none;
		padding: 0;
		margin: 0;
	}

	.info-box li {
		padding: 1rem 0;
		color: #666;
		border-bottom: 1px solid #f0f0f0;
		line-height: 1.6;
	}

	.info-box li:last-child {
		border-bottom: none;
		padding-bottom: 0;
	}

	.info-box li strong {
		color: #333;
		font-weight: 600;
	}

	/* Responsive */
	@media (max-width: 768px) {
		.page-wrapper {
			padding: 2rem 1rem;
		}

		.upload-section,
		.preview-section,
		.options-section,
		.size-selection-section,
		.code-section,
		.info-box {
			padding: 1.5rem;
		}

		.upload-area {
			padding: 2rem 1rem;
		}

		.sizes-grid {
			grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
			gap: 1rem;
		}

		.bulk-actions {
			padding: 1.5rem;
		}

		.btn-primary {
			width: 100%;
		}

		.toast-container {
			left: 1rem;
			right: 1rem;
			top: 1rem;
		}

		.size-checkboxes {
			grid-template-columns: 1fr;
		}

		.color-options {
			justify-content: center;
		}
	}
</style>
