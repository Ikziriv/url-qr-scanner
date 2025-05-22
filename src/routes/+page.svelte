<script lang="ts">
	import QRCode from 'qrcode';
	// --- Configuration ---
	const QR_CODE_SIZE = 300; // pixels
	const LOGO_MAX_SIZE_PERCENT = 0.25; // Logo will take up to 25% of QR code width/height
	const LOGO_MARGIN = 5; // pixels, margin around the logo within its cleared space
	const INITIAL_QR_CONTENT = ''; // Start with empty content. Can be pre-filled e.g. 'https://tkdn.kemenperin.go.id/'
    const MAX_LOGO_SIZE_BYTES = 2 * 1024 * 1024; // 2MB limit for logo

	// --- State ---
	let qrContent: string = INITIAL_QR_CONTENT;
	let logoFile: File | null = null;
	let logoPreview: string | null = null;
	let canvasElement: HTMLCanvasElement | undefined; 
	let errorMessage: string = '';
	let qrCodeDataUrlForDownload: string = '';
	let currentGenerationId = 0; 
	let notificationMessage: string = '';
	let notificationType: 'success' | 'error' | '' = '';
	let notificationTimeoutId: number | null = null;

	function clearNotification() {
		notificationMessage = '';
		notificationType = '';
		if (notificationTimeoutId) {
			clearTimeout(notificationTimeoutId);
			notificationTimeoutId = null;
		}
	}

	function showNotification(message: string, type: 'success' | 'error', duration: number = 5000) {
		clearNotification(); // Clear any existing notification first
		notificationMessage = message;
		notificationType = type;
		if (duration > 0) {
			notificationTimeoutId = window.setTimeout(() => {
				clearNotification();
			}, duration);
		}
	}
	
	async function generateQRCode() {
		const localGenerationId = ++currentGenerationId;
		clearNotification(); 


		if (!canvasElement) {
			showNotification('Canvas Belum Tersedia.', 'error');
			return;
		}

		if (!qrContent || !qrContent.trim()) {
			showNotification('Silakan masukkan konten untuk QR code.', 'error');
			
			const ctx = canvasElement.getContext('2d');
			if (ctx) {
				ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);
			}
			qrCodeDataUrlForDownload = '';
			return;
		}
		errorMessage = ''; 

		try {
			const ctx = canvasElement.getContext('2d');
			if (!ctx) {
				showNotification('Tidak dapat memuat konteks canvas.', 'error');
				return;
			}

			canvasElement.width = QR_CODE_SIZE;
			canvasElement.height = QR_CODE_SIZE;

			await QRCode.toCanvas(canvasElement, qrContent, {
				width: QR_CODE_SIZE,
				errorCorrectionLevel: 'H', // 'H' (High) is best for logos
				margin: 2,
				color: {
					dark: '#000000',
					light: '#FFFFFF'
				}
			});

			if (localGenerationId !== currentGenerationId) return;

			if (logoFile && logoPreview) {
				const img = new Image();
				img.onload = () => {
					if (localGenerationId !== currentGenerationId) return;

					const logoActualMaxWidth = QR_CODE_SIZE * LOGO_MAX_SIZE_PERCENT;
					const logoActualMaxHeight = QR_CODE_SIZE * LOGO_MAX_SIZE_PERCENT;

					let logoWidth = img.width;
					let logoHeight = img.height;

					if (logoWidth > logoActualMaxWidth) {
						logoHeight = (logoActualMaxWidth / logoWidth) * logoHeight;
						logoWidth = logoActualMaxWidth;
					}
					if (logoHeight > logoActualMaxHeight) {
						logoWidth = (logoActualMaxHeight / logoHeight) * logoWidth;
						logoHeight = logoActualMaxHeight;
					}

					const logoX = (QR_CODE_SIZE - logoWidth) / 2;
					const logoY = (QR_CODE_SIZE - logoHeight) / 2;

					const clearX = logoX - LOGO_MARGIN;
					const clearY = logoY - LOGO_MARGIN;
					const clearWidth = logoWidth + 2 * LOGO_MARGIN;
					const clearHeight = logoHeight + 2 * LOGO_MARGIN;

					ctx.fillStyle = '#FFFFFF';
					ctx.fillRect(clearX, clearY, clearWidth, clearHeight);
					ctx.drawImage(img, logoX, logoY, logoWidth, logoHeight);

					qrCodeDataUrlForDownload = canvasElement!.toDataURL('image/png');
				};
				img.onerror = () => {
					if (localGenerationId !== currentGenerationId) return;
					showNotification('Gagal memuat gambar logo. QR code dibuat tanpa logo.', 'error');
					
					qrCodeDataUrlForDownload = canvasElement.toDataURL('image/png');
				};
				img.src = logoPreview; // Start loading the image
			} else {
				qrCodeDataUrlForDownload = canvasElement.toDataURL('image/png');
			}
		} catch (err) {
			if (localGenerationId !== currentGenerationId) return; // Stale error
			console.error('QR Code Generation Error:', err);
			showNotification(`Error generating QR Code: ${err instanceof Error ? err.message : String(err)}`, 'error');
			qrCodeDataUrlForDownload = '';

            if(canvasElement){
                const ctx = canvasElement.getContext('2d');
                if (ctx) {
                    ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);
                }
            }
		}
	}

	function handleLogoUpload(event: Event) {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];

		if (file) {
            const allowedTypes = ['image/png', 'image/jpeg', 'image/gif', 'image/svg+xml'];
            if (!allowedTypes.includes(file.type)) {
				showNotification('Jenis file logo tidak didukung (gunakan PNG, JPG, GIF, SVG).', 'error');
                logoFile = null;
                logoPreview = null;
                if (target) target.value = '';
                return;
            }
            if (file.size > MAX_LOGO_SIZE_BYTES) {
                errorMessage = `Ukuran file logo terlalu besar (maks ${MAX_LOGO_SIZE_BYTES / 1024 / 1024}MB).`;
				showNotification(errorMessage, 'error');
                
				logoFile = null;
                logoPreview = null;
                if (target) target.value = '';
                return;
            }
            errorMessage = ''; 
            clearNotification();

			logoFile = file;
			const reader = new FileReader();
			reader.onload = (e) => {
				logoPreview = e.target?.result as string;
			};
            reader.onerror = () => {
                errorMessage = 'Gagal membaca file logo.';
				showNotification(errorMessage, 'error');

                logoFile = null;
                logoPreview = null;
                if (target) target.value = '';
            }
			reader.readAsDataURL(file);
		} else {
			clearNotification();
			logoFile = null;
			logoPreview = null;
        }
	}

	function downloadQRCode() {
		if (!qrCodeDataUrlForDownload) {
			errorMessage = 'Tidak ada QR code untuk diunduh.';
			showNotification(errorMessage, 'error');
			return;
		}
		const link = document.createElement('a');
		link.download = 'qrcode-generated.png'; // Generic name
		link.href = qrCodeDataUrlForDownload;
		document.body.appendChild(link);
		link.click();
		document.body.removeChild(link);
		showNotification('QR Code berhasil diunduh!', 'success', 3000);
	}
	$: (qrContent, logoPreview), canvasElement && generateQRCode();
</script>

<svelte:head>
	<title>Aplikasi QR Generator</title>
	<meta name="description" content="Aplikasi Generate QR codes with optional logo, created by DATIN Team" />
</svelte:head>

<div class="flex justify-center items-center w-full h-auto md:h-screen bg-gray-100">
    <div class="flex flex-col justify-center items-center w-full h-auto md:max-w-6xl bg-white p-8 rounded-xl shaodw-xl relative">
        {#if notificationMessage}
        <div
            class="absolute bottom-0 left-0 right-0 h-auto rounded-b-xl transition-opacity duration-300 ease-in-out"
            class:bg-gradient-to-r={notificationType === 'error'}
            class:from-orange-500={notificationType === 'error'}
            class:to-red-500={notificationType === 'error'}
            class:bg-green-500={notificationType === 'success'}
        >
			<div class="flex justify-center items-center w-full h-auto py-1 px-4">
				<p class="text-zinc-50 font-bold text-xs uppercase">{notificationMessage}</p>
			</div>
		</div>
        {/if}

        <div class="flex flex-col md:flex-row justify-between items-start w-full mb-4">
            <div class="flex flex-col -space-y-1 w-auto h-auto items-start mb-8 md:mb-0">
                <h1 class="font-black uppercase text-2xl">Generator QR</h1>
                <p class="subtitle text-sm uppercase">
					<small>Created by</small><span class="text-xs font-bold ml-1"> DATIN Team</span>
				</p>
				<p class="subtitle text-sm uppercase">
				</p>
            </div>
            
            <div class="bg-zinc-200 rounded-xl w-24 h-24 p-4">
                {#if logoPreview}
                    <img src={logoPreview} alt="Logo Preview" />
                {/if}
            </div>
        </div>

        <div class="flex flex-col md:flex-row justify-between items-center md:items-end w-full h-auto pb-8">
            <div class="flex flex-col w-full md:w-auto h-auto">
				<div class="flex flex-col w-full h-auto items-start mb-4">
                    <label class="w-full h-auto p-2 border border-gray-300 text-zinc-700 text-xs uppercase font-bold" for="qrContentInput">
                        Upload Logo 
                    </label>
                    <input
                        type="file" 
                        id="logoUpload" 
                        accept="image/png, image/jpeg, image/svg+xml, image/gif" on:change={handleLogoUpload}
                        class="border border-gray-300 p-2 w-full md:w-96 mt-1"
                    />
                </div>

                <div class="flex flex-col w-full h-auto items-start mb-4">
                    <label class="w-full h-auto p-2 border border-gray-300 text-zinc-700 text-xs uppercase font-bold" for="qrContentInput">
                        URL Link 
                    </label>
                    <input
                        type="text"
                        id="qrContentInput"
                        bind:value={qrContent}
                        placeholder="https://example.com"
                        class="border border-gray-300 p-2 w-full md:w-96 mt-1"
                    />
                </div>
            </div>

            <div class="flex flex-col justify-center items-center text-center w-full md:w-auto h-auto">
                <canvas 
                    bind:this={canvasElement} 
                    width={QR_CODE_SIZE}
                    height={QR_CODE_SIZE}
                    class="bg-zinc-200 rounded-xl" 
                    style="border: 1px solid #eee;">
                </canvas>
                <div class="flex justify-center items-center w-full h-auto rounded shadow-sm py-1 mt-2">
                    <h2 class="text-xs text-gray-600 font-bold uppercase">Preview / QR Generated</h2>
                </div>
            </div>
        </div>

        <div class="flex flex-col w-full h-auto items-center">
            {#if qrCodeDataUrlForDownload}
                <button 
                    on:click={downloadQRCode} 
                    class="flex justify-between items-center w-full h-auto bg-blue-500 text-white font-bold py-2 px-4 rounded-lg shadow-md hover:bg-blue-600 transition duration-200 ease-in-out">
                    <svg  xmlns="http://www.w3.org/2000/svg"  width="24"  height="24"  viewBox="0 0 24 24"  fill="none"  stroke="currentColor"  stroke-width="2"  stroke-linecap="round"  stroke-linejoin="round"  class="icon icon-tabler icons-tabler-outline icon-tabler-download"><path stroke="none" d="M0 0h24v24H0z" fill="none"/><path d="M4 17v2a2 2 0 0 0 2 2h12a2 2 0 0 0 2 -2v-2" /><path d="M7 11l5 5l5 -5" /><path d="M12 4l0 12" /></svg>
                    Download QR
                </button>
            {:else if !notificationMessage}
                <!-- Show this prompt only if there's no content, no QR, and no error yet -->
                <p class="info-prompt">Masukan konten di atas untuk membuat QR code.</p>
            {/if}
        </div>
    </div>
</div>