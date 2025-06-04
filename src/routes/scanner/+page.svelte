<script lang="ts">
    import { onMount, onDestroy } from 'svelte';
    import jsQR from 'jsqr';

    let videoElement: HTMLVideoElement | null = null;
    let canvasElement: HTMLCanvasElement | null = null;
    let canvasContext: CanvasRenderingContext2D | null = null;
    let scanning = false;
    let scannedData: string | null = null;
    let scanError: string | null = null;
    let stream: MediaStream | null = null;

    async function startScan() {
        scannedData = null;
        scanError = null;
        if (!videoElement || !canvasElement) {
            scanError = 'Video or canvas element not ready.';
            return;
        }

        canvasContext = canvasElement.getContext('2d');
        if (!canvasContext) {
            scanError = 'Could not get canvas context.';
            return;
        }

        try {
            stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
            videoElement.srcObject = stream;
            videoElement.setAttribute('playsinline', 'true'); // Required for iOS
            await videoElement.play();
            scanning = true;
            requestAnimationFrame(tick);
        } catch (err) {
            console.error('Error accessing camera:', err);
            scanError = 'Could not access camera. Please ensure permissions are granted.';
            if (err instanceof Error && err.name === "NotAllowedError") {
                scanError = 'Camera access denied. Please allow camera access in your browser settings.';
            } else if (err instanceof Error && err.name === "NotFoundError") {
                scanError = 'No camera found. Please ensure a camera is connected and enabled.';
            }
            scanning = false;
        }
    }

    function stopScan() {
        scanning = false;
        if (videoElement) {
            videoElement.pause();
        }
        if (stream) {
            stream.getTracks().forEach(track => track.stop());
            stream = null;
        }
        if (videoElement && videoElement.srcObject) {
            videoElement.srcObject = null;
        }
    }

    function tick() {
        if (!scanning || !videoElement || !canvasElement || !canvasContext) {
            return;
        }

        if (videoElement.readyState === videoElement.HAVE_ENOUGH_DATA) {
            canvasElement.height = videoElement.videoHeight;
            canvasElement.width = videoElement.videoWidth;
            canvasContext.drawImage(videoElement, 0, 0, canvasElement.width, canvasElement.height);
            
            const imageData = canvasContext.getImageData(0, 0, canvasElement.width, canvasElement.height);
            const code = jsQR(imageData.data, imageData.width, imageData.height, {
                inversionAttempts: 'dontInvert',
            });

            if (code) {
                scannedData = code.data;
                // You could automatically stop scanning or navigate, or show the data
                // For now, we'll just display it and let the user stop manually
                // stopScan(); 
                // alert(`Scanned QR Code: ${code.data}`);
            }
        }
        if (scanning) {
            requestAnimationFrame(tick);
        }
    }

    onMount(() => {
        // Automatically try to start scanning when the component mounts
        // You might want to trigger this with a button press instead
        // startScan();
    });

    onDestroy(() => {
        stopScan();
    });

    function handleScannedData() {
        if (scannedData) {
            // Example: Try to open as URL if it looks like one
            try {
                const url = new URL(scannedData);
                window.open(url.toString(), '_blank');
            } catch (e) {
                // Not a valid URL, just display it or handle differently
                alert(`Scanned content: ${scannedData}`);
            }
        }
    }
</script>

<svelte:head>
    <title>Aplikasi QR Code Scanner</title>
    <meta name="description" content="Scan QR codes using your camera." />
</svelte:head>

<div class="flex flex-col items-center justify-center min-h-screen bg-gray-100 p-4">
    <div class="bg-white p-6 md:p-8 rounded-xl shadow-xl border w-full max-w-lg">
        <h1 class="text-md font-normal tracking-widest text-center mb-6 uppercase">QR Code Scanner</h1>

        <div class="relative w-full aspect-square bg-gray-200 rounded overflow-hidden mb-4">
            <video bind:this={videoElement} class="w-full h-full object-cover" playsinline></video>
            <!-- Canvas is used for processing, can be hidden if not needed for display -->
            <canvas bind:this={canvasElement} class="hidden"></canvas>
        </div>

        {#if scanError}
            <p class="text-red-500 text-center mb-4">{scanError}</p>
        {/if}

        {#if scannedData}
            <div class="mt-4 p-4 bg-green-100 border border-green-400 text-green-700 rounded">
                <h2 class="font-semibold">Scanned Data:</h2>
                <p class="break-all">{scannedData}</p>
                <button 
                    on:click={handleScannedData}
                    class="mt-2 bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-3 rounded text-sm"
                >
                    Open/Handle Data
                </button>
                <button 
                    on:click={() => { scannedData = null; scanError = null; startScan(); }}
                    class="mt-2 ml-2 bg-gray-500 hover:bg-gray-600 text-white font-bold py-2 px-3 rounded text-sm"
                >
                    Scan Again
                </button>
            </div>
        {/if}
        
        <div class="flex flex-row justify-center items-center w-full h-auto mt-6">
            {#if !scanning && !scannedData}
                <button 
                    on:click={startScan} 
                    class="w-full bg-zinc-900 hover:bg-zinc-950 text-white font-bold py-3 px-4 rounded-lg transition duration-150 ease-in-out"
                >
                    Mulai
                </button>
            {/if}

            {#if scanning}
                <button 
                    on:click={stopScan} 
                    class="w-full bg-red-500 hover:bg-red-600 text-white font-bold py-3 px-4 rounded-lg transition duration-150 ease-in-out"
                >
                    Hentikan
                </button>
            {/if}
            
            <div class="w-full text-center">
                <a href="/" class="text-zinc-500 hover:text-zinc-700">&larr; Kembali</a>
            </div>
        </div>
    </div>
</div>

<style>
    /* Add any specific styles here if needed */
</style>
