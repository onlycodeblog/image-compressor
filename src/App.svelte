<script>
  import { onMount } from "svelte";
  import JSZip from "jszip";

  import mozjpegModule from "./codecs/mozjpeg/enc/mozjpeg_enc.js";
  import webpModule from "./codecs/webp/enc/webp_enc.js";
  import avifModule from "./codecs/avif/enc/avif_enc.js";
  import { decodeImageToImageData } from "./utils/image-decode.js";

  let fileInput;
  let quality = 75;
  let format = "mozjpeg";
  let results = [];
  let dropdownOpen = false;

  const toggleDropdown = () => {
    dropdownOpen = !dropdownOpen;
  };

  const selectFormat = (newFormat) => {
    format = newFormat;
    dropdownOpen = false;
  };

  // Close dropdown when clicking outside
  const handleClickOutside = (event) => {
    if (!event.target.closest(".custom-select-wrapper")) {
      dropdownOpen = false;
    }
  };

  onMount(() => {
    document.addEventListener("click", handleClickOutside);
    return () => {
      document.removeEventListener("click", handleClickOutside);
    };
  });

  const formatBytes = (bytes) => {
    if (bytes === 0) return "0 Bytes";
    const k = 1024;
    const sizes = ["Bytes", "KB", "MB", "GB"];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + " " + sizes[i];
  };

  const getMimeType = (format) => {
    return (
      {
        mozjpeg: "image/jpeg",
        webp: "image/webp",
        avif: "image/avif",
      }[format] || "image/jpeg"
    );
  };

  const handleFiles = async (files) => {
    if (files.length === 0) return;

    for (const file of files) {
      if (file.size > 10 * 1024 * 1024) {
        alert(`${file.name} is too large. Maximum size is 10MB.`);
        continue;
      }
      await processImage(file);
    }

    // Reset the file input so the same file can be selected again
    if (fileInput) {
      fileInput.value = null;
    }
  };

  const processImage = async (file) => {
    const imageData = await decodeImageToImageData(file);

    const module =
      format === "mozjpeg"
        ? await mozjpegModule()
        : format === "webp"
          ? await webpModule()
          : await avifModule();

    const encodeOptions =
      format === "mozjpeg"
        ? {
            // MozJPEG options
            quality,
            baseline: true,
            progressive: false,
            optimize_coding: true,
            color_space: 3,
            chroma_subsample: 1,
            auto_subsample: true,
            arithmetic: false,
            smoothing: 0,
            quant_table: 3,
            trellis_multipass: false,
            trellis_opt_zero: false,
            trellis_opt_table: false,
            trellis_loops: 1,
            separate_chroma_quality: false,
            chroma_quality: 100,
          }
        : format === "webp"
          ? {
              quality,
              // WebP options
              lossless: 0, // 0 = lossy, 1 = lossless
              method: 4, // Compression method (0-6, higher = better but slower)
              image_hint: 0, // WebPImageHint: 0=default, 1=picture, 2=photo, 3=graph
              target_size: 0, // Target size in bytes (0 = disabled)
              target_PSNR: 0, // Target PSNR (0 = disabled)
              segments: 4, // Number of segments (1-4)
              sns_strength: 50, // Spatial noise shaping (0-100)
              filter_strength: 60, // Filter strength (0-100)
              filter_sharpness: 0, // Filter sharpness (0-7)
              filter_type: 1, // Filtering type (0=simple, 1=strong)
              autofilter: 0, // Auto-adjust filter (0-1)
              alpha_compression: 1, // Alpha compression (0-1)
              alpha_filtering: 1, // Alpha filtering (0-2)
              alpha_quality: 100, // Alpha quality (0-100)
              pass: 1, // Number of passes (1-10)
              show_compressed: 0, // Show compressed picture
              preprocessing: 0, // Preprocessing filter
              partitions: 0, // Number of partitions (0-3)
              partition_limit: 0, // Quality degradation allowed (0-100)
              emulate_jpeg_size: 0, // Emulate JPEG size
              thread_level: 0, // Multi-threading level
              low_memory: 0, // Reduce memory usage
              near_lossless: 100, // Near lossless quality (0-100)
              exact: 0, // Preserve RGB values under lossless
              use_delta_palette: 0, // Use delta palette
              use_sharp_yuv: 0, // Use sharp YUV
            }
          : {
              // AVIF options
              quality,
              qualityAlpha: -1, // Alpha quality (-1 = same as quality)
              cqLevel: -1, // Constant quality level (-1 = auto)
              cqLevelAlpha: -1, // Alpha CQ level (-1 = same as cqLevel)
              denoiseLevel: -1, // Denoise level (-1 = auto)
              tileColsLog2: 0, // Tile columns (log2)
              tileRowsLog2: 0, // Tile rows (log2)
              speed: 6, // Encoding speed (0-10, higher = faster)
              subsample: 1, // Chroma subsampling (0=444, 1=422, 2=420)
              enableSharpYUV: false, // Enable sharp YUV conversion (required field!)
              chromaDeltaQ: false, // Enable chroma delta quantization
              sharpness: 0, // Sharpness (0-7)
              tune: 0, // Tune (0=auto, 1=psnr, 2=ssim)
            };

    const encodedBinary = await module.encode(
      imageData.data,
      imageData.width,
      imageData.height,
      encodeOptions
    );

    if (!encodedBinary || encodedBinary.length < 100) {
      alert("Encoding failed: Invalid or empty image buffer");
      return;
    }

    const blob = new Blob([encodedBinary], { type: getMimeType(format) });

    const savings = ((file.size - blob.size) / file.size) * 100;
    const ext = format === "mozjpeg" ? "jpg" : format;
    const downloadUrl = URL.createObjectURL(blob);

    results = [
      ...results,
      {
        name: file.name,
        originalSize: file.size,
        compressedSize: blob.size,
        savings,
        previewUrl: URL.createObjectURL(file),
        downloadUrl,
        extension: ext,
        blob: blob,
      },
    ];
  };

  const downloadFile = (url, name) => {
    const a = document.createElement("a");
    a.href = url;
    a.download = name;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  };

  const downloadAllAsZip = async () => {
    if (results.length === 0) return;

    const zip = new JSZip();
    const usedFilenames = new Set();

    // Helper function to generate unique filename
    const getUniqueFilename = (originalFilename) => {
      let filename = originalFilename;
      let counter = 0;

      while (usedFilenames.has(filename)) {
        counter++;
        const baseName = originalFilename.replace(/\.[^/.]+$/, "");
        const extension = originalFilename.match(/\.[^/.]+$/)?.[0] || "";
        filename = `${baseName}(${counter})${extension}`;
      }

      usedFilenames.add(filename);
      return filename;
    };

    // Add each compressed image to the zip
    results.forEach((item) => {
      const originalFilename = item.name.replace(
        /\.[^/.]+$/,
        `.${item.extension}`
      );
      const uniqueFilename = getUniqueFilename(originalFilename);
      zip.file(uniqueFilename, item.blob);
    });

    // Generate the zip file
    try {
      const zipBlob = await zip.generateAsync({
        type: "blob",
        compression: "DEFLATE",
        compressionOptions: { level: 6 },
      });

      // Create download link
      const zipUrl = URL.createObjectURL(zipBlob);
      const currentDate = new Date().toISOString().split("T")[0];
      const zipFilename = `compressed-images-${currentDate}.zip`;

      downloadFile(zipUrl, zipFilename);
    } catch (error) {
      alert("Failed to create ZIP file: " + error.message);
    }
  };
</script>

<div class="container">
  <header>
    <h1>Image Squash</h1>
    <p>Compress images privately in your browser</p>
  </header>

  <main>
    <div
      class="upload-area"
      role="button"
      tabindex="0"
      on:click={() => fileInput.click()}
      on:keydown={(e) => e.key === "Enter" && fileInput.click()}
      on:dragover|preventDefault
      on:dragleave|preventDefault
      on:drop|preventDefault={(e) => handleFiles([...e.dataTransfer.files])}
    >
      <div class="upload-content">
        <svg
          class="upload-icon"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
        >
          <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
          <polyline points="7,10 12,15 17,10" />
          <line x1="12" y1="15" x2="12" y2="3" />
        </svg>
        <h3>Drop images here or click to upload</h3>
        <p>Supports JPEG, PNG, WebP, AVIF • Max 10MB per file</p>
      </div>
      <input
        type="file"
        bind:this={fileInput}
        accept="image/*"
        multiple
        hidden
        on:change={(e) => handleFiles([...e.target.files])}
      />
    </div>

    <div class="controls">
      <div class="quality-control">
        <label for="quality">Quality: {quality}%</label>
        <input
          type="range"
          id="quality"
          min="1"
          max="100"
          bind:value={quality}
        />
      </div>
      <div class="format-control">
        <label for="format">Output Format</label>

        <div class="custom-select-wrapper" class:open={dropdownOpen}>
          <!-- svelte-ignore a11y-click-events-have-key-events -->
          <div class="custom-select" on:click={toggleDropdown}>
            <span class="select-text">
              {format === "mozjpeg"
                ? "JPEG (MozJPEG)"
                : format === "webp"
                  ? "WebP"
                  : "AVIF"}
            </span>

            <div class="select-arrow" class:rotated={dropdownOpen}>
              <svg
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
              >
                <polyline points="6,9 12,15 18,9"></polyline>
              </svg>
            </div>
          </div>

          <div class="dropdown-options">
            <!-- svelte-ignore a11y-click-events-have-key-events -->
            <div
              class="dropdown-option"
              class:selected={format === "mozjpeg"}
              on:click={() => selectFormat("mozjpeg")}
            >
              <span>JPEG (MozJPEG)</span>
              <span class="format-description">Best compression for photos</span
              >
            </div>
            <!-- svelte-ignore a11y-click-events-have-key-events -->
            <div
              class="dropdown-option"
              class:selected={format === "webp"}
              on:click={() => selectFormat("webp")}
            >
              <span>WebP</span>
              <span class="format-description"
                >Modern format, good compression</span
              >
            </div>
            <!-- svelte-ignore a11y-click-events-have-key-events -->
            <div
              class="dropdown-option"
              class:selected={format === "avif"}
              on:click={() => selectFormat("avif")}
            >
              <span>AVIF</span>
              <span class="format-description"
                >Latest format, best compression</span
              >
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Add Download All button when there are multiple results -->
    {#if results.length > 1}
      <div class="download-all-section">
        <button class="download-all-btn" on:click={downloadAllAsZip}>
          <svg
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
          >
            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
            <polyline points="7,10 12,15 17,10" />
            <line x1="12" y1="15" x2="12" y2="3" />
          </svg>
          Download All as ZIP ({results.length} images)
        </button>
      </div>
    {/if}

    <div class="results">
      {#each results as item}
        <div class="result-item">
          <img class="result-preview" src={item.previewUrl} alt={item.name} />
          <div class="result-info">
            <h4>{item.name}</h4>
            <p class="compressed-name">
              {item.name.replace(/\.[^/.]+$/, `.${item.extension}`)}
            </p>
            <div class="size-info">
              <span>Original: {formatBytes(item.originalSize)}</span>
              <span>Compressed: {formatBytes(item.compressedSize)}</span>
              <span class="savings"
                >{item.savings >= 0 ? "-" : "-"}{Math.abs(item.savings).toFixed(
                  1
                )}%</span
              >
            </div>
          </div>
          <button
            class="download-btn"
            on:click={() =>
              downloadFile(
                item.downloadUrl,
                item.name.replace(/\.[^/.]+$/, `.${item.extension}`)
              )}
          >
            Download
          </button>
        </div>
      {/each}
    </div>
  </main>
</div>
