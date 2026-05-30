<script setup>
import { ref, watch, nextTick } from 'vue';

// State management
const photoImage = ref(null);
const photoName = ref('');
const overlayImage = ref(null);
const overlayName = ref('');
const position = ref('bottom-left');
const scale = ref(35); // 10 to 100
const padding = ref(4);  // 0 to 15
const opacity = ref(100); // 10 to 100

const canvasRef = ref(null);
const photoResText = ref('');
const isPhotoDragover = ref(false);
const isOverlayDragover = ref(false);

// Watch for state changes to redraw the composition
watch([photoImage, overlayImage, position, scale, padding, opacity], () => {
    nextTick(() => {
        drawComposition();
    });
});

// File Upload Handlers (Handled natively by z-indexed file inputs)

const onPhotoChange = (e) => {
    if (e.target.files.length) {
        handlePhotoFile(e.target.files[0]);
    }
};

const onOverlayChange = (e) => {
    if (e.target.files.length) {
        handleOverlayFile(e.target.files[0]);
    }
};

const onPhotoDrop = (e) => {
    isPhotoDragover.value = false;
    if (e.dataTransfer.files.length) {
        handlePhotoFile(e.dataTransfer.files[0]);
    }
};

const onOverlayDrop = (e) => {
    isOverlayDragover.value = false;
    if (e.dataTransfer.files.length) {
        handleOverlayFile(e.dataTransfer.files[0]);
    }
};

// Reading files as Images
const handlePhotoFile = (file) => {
    if (!file.type.startsWith('image/')) {
        alert('Пожалуйста, выберите файл изображения (JPG, PNG, WebP и др.)');
        return;
    }
    photoName.value = file.name;
    
    const reader = new FileReader();
    reader.onload = (e) => {
        const img = new Image();
        img.onload = () => {
            photoImage.value = img;
            photoResText.value = `Разрешение: ${img.naturalWidth} x ${img.naturalHeight}`;
        };
        img.src = e.target.result;
    };
    reader.readAsDataURL(file);
};

const handleOverlayFile = (file) => {
    if (file.type !== 'image/png') {
        alert('Для корректного наложения оверлея требуется прозрачный файл формата PNG.');
        return;
    }
    overlayName.value = file.name;
    
    const reader = new FileReader();
    reader.onload = (e) => {
        const img = new Image();
        img.onload = () => {
            overlayImage.value = img;
        };
        img.src = e.target.result;
    };
    reader.readAsDataURL(file);
};

// Canvas drawing engine (full resolution logic)
const drawComposition = () => {
    const canvas = canvasRef.value;
    if (!canvas || !photoImage.value || !overlayImage.value) return;

    const ctx = canvas.getContext('2d');
    const photo = photoImage.value;
    const overlay = overlayImage.value;

    const w = photo.naturalWidth;
    const h = photo.naturalHeight;

    canvas.width = w;
    canvas.height = h;

    ctx.clearRect(0, 0, w, h);

    // 1. Draw photo background
    ctx.drawImage(photo, 0, 0, w, h);

    // 2. Draw overlay with custom settings
    const overlayAspect = overlay.naturalWidth / overlay.naturalHeight;
    const targetW = w * (scale.value / 100);
    const targetH = targetW / overlayAspect;

    const minDim = Math.min(w, h);
    const pad = minDim * (padding.value / 100);

    let x = 0;
    let y = 0;

    switch (position.value) {
        case 'top-left':
            x = pad;
            y = pad;
            break;
        case 'top-right':
            x = w - targetW - pad;
            y = pad;
            break;
        case 'center':
            x = (w - targetW) / 2;
            y = (h - targetH) / 2;
            break;
        case 'bottom-left':
            x = pad;
            y = h - targetH - pad;
            break;
        case 'bottom-right':
            x = w - targetW - pad;
            y = h - targetH - pad;
            break;
    }

    ctx.save();
    ctx.globalAlpha = opacity.value / 100;
    ctx.drawImage(overlay, x, y, targetW, targetH);
    ctx.restore();
};

// High-res JPEG Export
const downloadImage = () => {
    const canvas = canvasRef.value;
    if (!canvas || !photoImage.value || !overlayImage.value) return;

    let baseName = 'strava_overlay';
    if (photoName.value) {
        const lastDot = photoName.value.lastIndexOf('.');
        if (lastDot !== -1) {
            baseName = photoName.value.substring(0, lastDot) + '_strava';
        } else {
            baseName = photoName.value + '_strava';
        }
    }

    canvas.toBlob((blob) => {
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `${baseName}.jpg`;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
    }, 'image/jpeg', 0.95);
};
</script>

<template>
    <div class="background-glow"></div>
    
    <div class="app-container">
        <header class="app-header">
            <div class="logo">
                <span class="logo-accent">Strava</span> Overlay
            </div>
            <p class="subtitle">Быстрое наложение спортивного оверлея на ваши фотографии без потери качества (Vue 3)</p>
        </header>

        <main class="app-main">
            <!-- Left Panel: Uploaders & Controls -->
            <section class="control-panel">
                <div class="card drag-drop-section">
                    <h2>1. Загрузка файлов</h2>
                    
                    <!-- Photo Dropzone -->
                    <div 
                        class="dropzone" 
                        :class="{ dragover: isPhotoDragover }"
                        @dragenter.prevent="isPhotoDragover = true"
                        @dragover.prevent="isPhotoDragover = true"
                        @dragleave.prevent="isPhotoDragover = false"
                        @drop.prevent="onPhotoDrop"
                    >
                        <input 
                            type="file" 
                            id="photo-input-vue" 
                            accept="image/*" 
                            class="file-input" 
                            @change="onPhotoChange"
                        >
                        <div class="dropzone-content">
                            <div class="icon-wrapper photo-icon">
                                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect><circle cx="8.5" cy="8.5" r="1.5"></circle><polyline points="21 15 16 10 5 21"></polyline></svg>
                            </div>
                            <div class="text-content">
                                <h3>Основное фото</h3>
                                <p>Перетащите сюда или нажмите для выбора</p>
                                <span class="file-name-display">{{ photoName || 'Файл не выбран' }}</span>
                            </div>
                        </div>
                    </div>

                    <!-- Overlay Dropzone -->
                    <div 
                        class="dropzone" 
                        :class="{ dragover: isOverlayDragover }"
                        @dragenter.prevent="isOverlayDragover = true"
                        @dragover.prevent="isOverlayDragover = true"
                        @dragleave.prevent="isOverlayDragover = false"
                        @drop.prevent="onOverlayDrop"
                    >
                        <input 
                            type="file" 
                            id="overlay-input-vue" 
                            accept="image/png" 
                            class="file-input" 
                            @change="onOverlayChange"
                        >
                        <div class="dropzone-content">
                            <div class="icon-wrapper overlay-icon">
                                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 2L2 7l10 5 10-5-10-5zM2 17l10 5 10-5M2 12l10 5 10-5"></path></svg>
                            </div>
                            <div class="text-content">
                                <h3>Strava Оверлей (PNG)</h3>
                                <p>Прозрачный PNG скриншот</p>
                                <span class="file-name-display">{{ overlayName || 'Файл не выбран' }}</span>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- Customizer card, starts disabled until photo is loaded -->
                <div class="card controls-section" :class="{ disabled: !photoImage }">
                    <h2>2. Настройка оверлея</h2>
                    
                    <!-- Position Picker -->
                    <div class="control-group">
                        <label>Положение оверлея</label>
                        <div class="position-grid">
                            <button 
                                type="button" 
                                class="pos-btn" 
                                :class="{ active: position === 'top-left' }" 
                                @click="position = 'top-left'"
                                title="Сверху слева"
                            >↖</button>
                            <button 
                                type="button" 
                                class="pos-btn" 
                                :class="{ active: position === 'top-right' }" 
                                @click="position = 'top-right'"
                                title="Сверху справа"
                            >↗</button>
                            <button 
                                type="button" 
                                class="pos-btn" 
                                :class="{ active: position === 'center' }" 
                                @click="position = 'center'"
                                title="По центру"
                            >⊙</button>
                            <button 
                                type="button" 
                                class="pos-btn" 
                                :class="{ active: position === 'bottom-left' }" 
                                @click="position = 'bottom-left'"
                                title="Снизи слева"
                            >↙</button>
                            <button 
                                type="button" 
                                class="pos-btn" 
                                :class="{ active: position === 'bottom-right' }" 
                                @click="position = 'bottom-right'"
                                title="Снизу справа"
                            >↘</button>
                        </div>
                    </div>

                    <!-- Scale Slider -->
                    <div class="control-group">
                        <div class="slider-header">
                            <label for="scale-range-vue">Размер оверлея</label>
                            <span class="value-display">{{ scale }}%</span>
                        </div>
                        <input type="range" id="scale-range-vue" min="10" max="100" v-model.number="scale" class="slider">
                    </div>

                    <!-- Padding Slider -->
                    <div class="control-group">
                        <div class="slider-header">
                            <label for="padding-range-vue">Отступ от краев</label>
                            <span class="value-display">{{ padding }}%</span>
                        </div>
                        <input type="range" id="padding-range-vue" min="0" max="15" v-model.number="padding" class="slider">
                    </div>

                    <!-- Opacity Slider -->
                    <div class="control-group">
                        <div class="slider-header">
                            <label for="opacity-range-vue">Непрозрачность</label>
                            <span class="value-display">{{ opacity }}%</span>
                        </div>
                        <input type="range" id="opacity-range-vue" min="10" max="100" v-model.number="opacity" class="slider">
                    </div>
                </div>
            </section>

            <!-- Right Panel: Live Preview & Action -->
            <section class="preview-panel">
                <div class="card preview-card">
                    <h2>3. Предпросмотр</h2>
                    <div class="preview-container">
                        <div class="preview-placeholder" v-if="!photoImage || !overlayImage">
                            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1" class="placeholder-icon"><rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect><circle cx="8.5" cy="8.5" r="1.5"></circle><polyline points="21 15 16 10 5 21"></polyline></svg>
                            <p>Загрузите фото и оверлей, чтобы увидеть результат</p>
                        </div>
                        <canvas ref="canvasRef" :class="{ hidden: !photoImage || !overlayImage }"></canvas>
                    </div>

                    <div class="action-footer">
                        <div class="info-badges" v-if="photoImage">
                            <span class="badge">{{ photoResText }}</span>
                        </div>
                        <button 
                            @click="downloadImage" 
                            class="download-button" 
                            :disabled="!photoImage || !overlayImage"
                        >
                            <svg class="btn-icon" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path><polyline points="7 10 12 15 17 10"></polyline><line x1="12" y1="15" x2="12" y2="3"></line></svg>
                            Скачать в оригинальном качестве
                        </button>
                    </div>
                </div>
            </section>
        </main>

        <footer class="app-footer">
            <p>100% Конфиденциально: Все вычисления происходят прямо в вашем браузере. Ваши фото не загружаются на сервер.</p>
        </footer>
    </div>
</template>

<style>
:root {
    --bg-primary: #0a0b0d;
    --bg-secondary: #121418;
    --bg-card: rgba(22, 24, 28, 0.7);
    --border-color: rgba(255, 255, 255, 0.08);
    --border-hover: rgba(252, 97, 0, 0.4);
    
    --brand-color: #fc6100;
    --brand-hover: #ff7525;
    --brand-glow: rgba(252, 97, 0, 0.25);
    
    --text-primary: #f3f4f6;
    --text-secondary: #9ca3af;
    --text-muted: #6b7280;
    
    --radius-sm: 8px;
    --radius-md: 14px;
    --radius-lg: 20px;
    
    --transition-smooth: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

body {
    background-color: var(--bg-primary);
    color: var(--text-primary);
    font-family: 'Plus Jakarta Sans', 'Outfit', -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
    min-height: 100vh;
    overflow-x: hidden;
    position: relative;
    line-height: 1.5;
}

/* Background aesthetic glows */
.background-glow {
    position: absolute;
    width: 600px;
    height: 600px;
    top: -200px;
    left: -100px;
    background: radial-gradient(circle, rgba(252, 97, 0, 0.08) 0%, rgba(10, 11, 13, 0) 70%);
    z-index: -1;
    pointer-events: none;
}

.app-container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 2.5rem 1.5rem;
    display: flex;
    flex-direction: column;
    min-height: 100vh;
}

/* Header */
.app-header {
    margin-bottom: 2.5rem;
    text-align: center;
}

.logo {
    font-family: 'Outfit', sans-serif;
    font-size: 2.5rem;
    font-weight: 700;
    letter-spacing: -0.03em;
    color: var(--text-primary);
    margin-bottom: 0.5rem;
}

.logo-accent {
    color: var(--brand-color);
    position: relative;
}

.subtitle {
    color: var(--text-secondary);
    font-size: 1.05rem;
    max-width: 600px;
    margin: 0 auto;
    font-weight: 400;
}

/* Main Grid Layout */
.app-main {
    display: grid;
    grid-template-columns: 460px 1fr;
    gap: 2rem;
    align-items: start;
    flex-grow: 1;
}

/* Cards style */
.card {
    background: var(--bg-card);
    backdrop-filter: blur(16px);
    -webkit-backdrop-filter: blur(16px);
    border: 1px solid var(--border-color);
    border-radius: var(--radius-lg);
    padding: 2rem;
    box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.3);
    margin-bottom: 2rem;
    transition: var(--transition-smooth);
}

.card:hover {
    border-color: rgba(255, 255, 255, 0.12);
}

.card h2 {
    font-size: 1.25rem;
    font-weight: 600;
    margin-bottom: 1.5rem;
    color: var(--text-primary);
    display: flex;
    align-items: center;
    border-bottom: 1px solid var(--border-color);
    padding-bottom: 0.75rem;
}

/* Dropzone styling */
.dropzone {
    position: relative;
    border: 2px dashed var(--border-color);
    border-radius: var(--radius-md);
    padding: 1.5rem;
    text-align: center;
    cursor: pointer;
    background: rgba(255, 255, 255, 0.01);
    transition: var(--transition-smooth);
    margin-bottom: 1.25rem;
}

.dropzone:last-child {
    margin-bottom: 0;
}

.dropzone:hover, .dropzone.dragover {
    border-color: var(--brand-color);
    background: rgba(252, 97, 0, 0.03);
    box-shadow: 0 0 15px var(--brand-glow);
}

.file-input {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    opacity: 0;
    cursor: pointer;
    z-index: 10;
}

.dropzone-content {
    display: flex;
    align-items: center;
    gap: 1.25rem;
    text-align: left;
}

.icon-wrapper {
    width: 48px;
    height: 48px;
    border-radius: var(--radius-sm);
    background: rgba(255, 255, 255, 0.05);
    display: flex;
    align-items: center;
    justify-content: center;
    color: var(--text-secondary);
    transition: var(--transition-smooth);
    flex-shrink: 0;
}

.dropzone:hover .icon-wrapper {
    background: var(--brand-color);
    color: white;
    transform: scale(1.05);
}

.text-content h3 {
    font-size: 1rem;
    font-weight: 600;
    margin-bottom: 0.25rem;
}

.text-content p {
    font-size: 0.85rem;
    color: var(--text-secondary);
}

.file-name-display {
    display: block;
    font-size: 0.75rem;
    color: var(--text-muted);
    margin-top: 0.25rem;
    word-break: break-all;
}

/* Control Panel Sliders & Positions */
.controls-section {
    transition: var(--transition-smooth);
}

.controls-section.disabled {
    opacity: 0.35;
    pointer-events: none;
    user-select: none;
}

.control-group {
    margin-bottom: 1.5rem;
}

.control-group:last-child {
    margin-bottom: 0;
}

.control-group > label {
    display: block;
    font-size: 0.9rem;
    font-weight: 500;
    color: var(--text-secondary);
    margin-bottom: 0.75rem;
}

.slider-header {
    display: flex;
    justify-content: space-between;
    margin-bottom: 0.5rem;
}

.slider-header label {
    font-size: 0.9rem;
    font-weight: 500;
    color: var(--text-secondary);
}

.value-display {
    font-size: 0.85rem;
    font-weight: 600;
    color: var(--brand-color);
    background: rgba(252, 97, 0, 0.1);
    padding: 0.1rem 0.5rem;
    border-radius: 4px;
}

/* 3x3 Position Grid Selector */
.position-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    grid-template-rows: repeat(3, 1fr);
    gap: 8px;
    width: 140px;
    height: 140px;
    margin: 0 auto;
    background: rgba(255, 255, 255, 0.02);
    border: 1px solid var(--border-color);
    border-radius: var(--radius-md);
    padding: 8px;
}

.pos-btn {
    background: rgba(255, 255, 255, 0.04);
    border: 1px solid var(--border-color);
    border-radius: 8px;
    color: var(--text-secondary);
    cursor: pointer;
    font-size: 1.1rem;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: var(--transition-smooth);
}

.pos-btn:hover {
    background: rgba(255, 255, 255, 0.08);
    color: var(--text-primary);
    border-color: rgba(255, 255, 255, 0.2);
}

.pos-btn.active {
    background: var(--brand-color);
    color: white;
    border-color: var(--brand-color);
    box-shadow: 0 0 10px var(--brand-glow);
}

/* Explicitly positioning layout elements inside the 3x3 grid */
.pos-btn[data-pos="top-left"] { grid-column: 1; grid-row: 1; }
.pos-btn[data-pos="top-right"] { grid-column: 3; grid-row: 1; }
.pos-btn[data-pos="center"] { grid-column: 2; grid-row: 2; }
.pos-btn[data-pos="bottom-left"] { grid-column: 1; grid-row: 3; }
.pos-btn[data-pos="bottom-right"] { grid-column: 3; grid-row: 3; }

/* Hidden Grid spaces that won't show anything */
.position-grid::before { content: ''; grid-column: 2; grid-row: 1; }
.position-grid::after { content: ''; grid-column: 1; grid-row: 2; }

/* Custom Sliders */
.slider {
    -webkit-appearance: none;
    width: 100%;
    height: 6px;
    border-radius: 3px;
    background: rgba(255, 255, 255, 0.1);
    outline: none;
    transition: var(--transition-smooth);
}

.slider::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: var(--brand-color);
    cursor: pointer;
    box-shadow: 0 0 8px var(--brand-glow);
    transition: transform 0.1s;
}

.slider::-webkit-slider-thumb:hover {
    transform: scale(1.2);
    background: var(--brand-hover);
}

.slider::-moz-range-thumb {
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: var(--brand-color);
    cursor: pointer;
    border: none;
    box-shadow: 0 0 8px var(--brand-glow);
    transition: transform 0.1s;
}

.slider::-moz-range-thumb:hover {
    transform: scale(1.2);
    background: var(--brand-hover);
}

/* Preview Area */
.preview-card {
    height: 100%;
    display: flex;
    flex-direction: column;
    margin-bottom: 0;
}

.preview-container {
    flex-grow: 1;
    min-height: 400px;
    background: rgba(0, 0, 0, 0.2);
    border: 1px solid var(--border-color);
    border-radius: var(--radius-md);
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    overflow: hidden;
    margin-bottom: 1.5rem;
}

.preview-placeholder {
    display: flex;
    flex-direction: column;
    align-items: center;
    color: var(--text-muted);
    text-align: center;
    padding: 2rem;
}

.placeholder-icon {
    width: 64px;
    height: 64px;
    stroke-width: 1;
    margin-bottom: 1rem;
    color: rgba(255, 255, 255, 0.15);
}

.preview-placeholder p {
    font-size: 0.95rem;
    max-width: 250px;
}

canvas {
    max-width: 100%;
    max-height: 520px;
    border-radius: 8px;
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.5);
    background: #000;
    object-fit: contain;
}

.hidden {
    display: none !important;
}

/* Action Footer & Buttons */
.action-footer {
    display: flex;
    flex-direction: column;
    gap: 1rem;
}

.info-badges {
    display: flex;
    gap: 0.5rem;
    flex-wrap: wrap;
}

.badge {
    font-size: 0.75rem;
    background: rgba(255, 255, 255, 0.05);
    border: 1px solid var(--border-color);
    padding: 0.25rem 0.75rem;
    border-radius: 100px;
    color: var(--text-secondary);
}

.download-button {
    background: var(--brand-color);
    color: white;
    border: none;
    border-radius: var(--radius-md);
    padding: 1rem 1.5rem;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.75rem;
    transition: var(--transition-smooth);
    box-shadow: 0 4px 20px rgba(252, 97, 0, 0.2);
}

.download-button:hover:not(:disabled) {
    background: var(--brand-hover);
    box-shadow: 0 6px 24px rgba(252, 97, 0, 0.4);
    transform: translateY(-2px);
}

.download-button:active:not(:disabled) {
    transform: translateY(0);
}

.download-button:disabled {
    background: #252830;
    color: var(--text-muted);
    cursor: not-allowed;
    box-shadow: none;
}

.btn-icon {
    width: 20px;
    height: 20px;
}

/* App Footer */
.app-footer {
    margin-top: 3rem;
    text-align: center;
    color: var(--text-muted);
    font-size: 0.85rem;
    border-top: 1px solid var(--border-color);
    padding-top: 1.5rem;
}

/* Responsiveness */
@media (max-width: 1024px) {
    .app-main {
        grid-template-columns: 1fr;
    }
    
    .preview-container {
        min-height: 350px;
    }
}

@media (max-width: 640px) {
    .app-container {
        padding: 1.5rem 1rem;
    }
    
    .logo {
        font-size: 2rem;
    }
    
    .card {
        padding: 1.25rem;
    }
    
    .dropzone-content {
        flex-direction: column;
        text-align: center;
        gap: 0.75rem;
    }
    
    .dropzone-content .text-content {
        text-align: center;
    }
}
</style>
