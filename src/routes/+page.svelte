<script lang="ts">
    import { onMount } from 'svelte';
    
    // Svelte 5 runes for reactive state
    let videoUrl = $state('');
    let videoId = $state('');
    let startTime = $state(0);
    let endTime = $state(30);
    let delayTime = $state(1);
    let playbackSpeed = $state(1.0);
    let isLooping = $state(false);
    let currentTime = $state(0);
    let isPlayerReady = $state(false);
    let showToast = $state(false);
    let toastMessage = $state('');
    
    let player: any = $state(null);
    let loopTimeout: number | null = $state(null);
    let timeUpdateInterval: number | null = $state(null);
    
    // YouTube API setup
    let YT: any = $state(null);
    let apiReady = $state(false);
    
    // Derived state using $derived rune
    let formattedCurrentTime = $derived(formatTime(currentTime));
    let loopDuration = $derived(endTime - startTime);
    let actualLoopDuration = $derived(loopDuration / playbackSpeed);
    let loopInfo = $derived(`Loop: ${formatTime(startTime)} - ${formatTime(endTime)} (${delayTime}s delay)`);
    let speedInfo = $derived(`Speed: ${playbackSpeed}x`);
    
    onMount(() => {
      loadYouTubeAPI();
      loadFromURL();
      
      return () => {
        if (loopTimeout) clearTimeout(loopTimeout);
        if (timeUpdateInterval) clearInterval(timeUpdateInterval);
      };
    });
    
    function loadFromURL() {
      const urlParams = new URLSearchParams(window.location.search);
      
      if (urlParams.get('url')) {
        videoUrl = urlParams.get('url') || '';
      }
      if (urlParams.get('start')) {
        startTime = parseFloat(urlParams.get('start') || '0');
      }
      if (urlParams.get('end')) {
        endTime = parseFloat(urlParams.get('end') || '30');
      }
      if (urlParams.get('delay')) {
        delayTime = parseFloat(urlParams.get('delay') || '1');
      }
      if (urlParams.get('speed')) {
        playbackSpeed = parseFloat(urlParams.get('speed') || '1');
      }
      
      // Auto-load video if URL is provided
      if (videoUrl) {
        setTimeout(() => loadVideo(), 100);
      }
    }
    
    function shareLoop() {
      const url = new URL(window.location.href);
      url.searchParams.set('url', videoUrl);
      url.searchParams.set('start', startTime.toString());
      url.searchParams.set('end', endTime.toString());
      url.searchParams.set('delay', delayTime.toString());
      url.searchParams.set('speed', playbackSpeed.toString());
      
      navigator.clipboard.writeText(url.toString()).then(() => {
        showToastMessage('Loop configuration copied to clipboard!');
      }).catch(() => {
        // Fallback for older browsers
        const textArea = document.createElement('textarea');
        textArea.value = url.toString();
        document.body.appendChild(textArea);
        textArea.select();
        document.execCommand('copy');
        document.body.removeChild(textArea);
        showToastMessage('Loop configuration copied to clipboard!');
      });
    }
    
    function showToastMessage(message: string) {
      toastMessage = message;
      showToast = true;
      setTimeout(() => {
        showToast = false;
      }, 3000);
    }
    
    function loadYouTubeAPI() {
      if ((window as any).YT) {
        YT = (window as any).YT;
        apiReady = true;
        return;
      }
      
      const tag = document.createElement('script');
      tag.src = 'https://www.youtube.com/iframe_api';
      
      (window as any).onYouTubeIframeAPIReady = () => {
        YT = (window as any).YT;
        apiReady = true;
      };
      
      document.head.appendChild(tag);
    }
    
    function extractVideoId(url: string) {
      const regex = /(?:youtube\.com\/watch\?v=|youtu\.be\/|youtube\.com\/embed\/)([^&\n?#]+)/;
      const match = url.match(regex);
      return match ? match[1] : null;
    }
    
    function loadVideo() {
      const id = extractVideoId(videoUrl);
      if (!id) {
        alert('Please enter a valid YouTube URL');
        return;
      }
      
      if (!apiReady) {
        alert('YouTube API not ready yet, please wait a moment');
        return;
      }
      
      videoId = id;
      
      if (player) {
        player.destroy();
      }
      
      player = new YT.Player('youtube-player', {
        height: '315',
        width: '100%',
        videoId: id,
        playerVars: {
          'playsinline': 1,
          'controls': 1,
          'rel': 0,
          'modestbranding': 1
        },
        events: {
          'onReady': onPlayerReady,
          'onStateChange': onPlayerStateChange
        }
      });
    }
    
    function onPlayerReady(event: any) {
      isPlayerReady = true;
      startTimeTracking();
      // Set initial playback speed
      if (player && player.setPlaybackRate) {
        player.setPlaybackRate(playbackSpeed);
      }
    }
    
    function onPlayerStateChange(event: any) {
      // Handle state changes if needed
    }
    
    function startTimeTracking() {
      if (timeUpdateInterval) clearInterval(timeUpdateInterval);
      
      timeUpdateInterval = setInterval(() => {
        if (player && player.getCurrentTime && isPlayerReady) {
          currentTime = player.getCurrentTime();
        }
      }, 100);
    }
    
    function setCurrentAsStart() {
      if (player && isPlayerReady) {
        startTime = Math.floor(player.getCurrentTime());
      }
    }
    
    function setCurrentAsEnd() {
      if (player && isPlayerReady) {
        endTime = Math.floor(player.getCurrentTime());
      }
    }
    
    function jumpToStart() {
      if (player && isPlayerReady) {
        player.seekTo(startTime);
      }
    }
    
    function jumpToEnd() {
      if (player && isPlayerReady) {
        player.seekTo(endTime);
      }
    }
    
    function setPlaybackSpeed(speed: number) {
      playbackSpeed = speed;
      if (player && player.setPlaybackRate && isPlayerReady) {
        player.setPlaybackRate(speed);
      }
    }
    
    function toggleLoop() {
      if (!player || !isPlayerReady) return;
      
      if (isLooping) {
        stopLoop();
      } else {
        startLoop();
      }
    }
    
    function startLoop() {
      isLooping = true;
      executeLoop();
    }
    
    function stopLoop() {
      isLooping = false;
      if (loopTimeout) {
        clearTimeout(loopTimeout);
        loopTimeout = null;
      }
      if (player) {
        player.pauseVideo();
      }
    }
    
    function executeLoop() {
      if (!isLooping || !player) return;
      
      // Seek to start and play
      player.seekTo(startTime);
      player.playVideo();
      
      // Set timeout for when to pause and restart
      const loopDurationMs = actualLoopDuration * 1000;
      const delayMs = delayTime * 1000;
      
      loopTimeout = setTimeout(() => {
        if (!isLooping) return;
        
        // Pause the video
        player.pauseVideo();
        
        // Wait for delay, then restart loop
        loopTimeout = setTimeout(() => {
          if (isLooping) {
            executeLoop(); // Recursive call
          }
        }, delayMs);
      }, loopDurationMs);
    }
    
    function formatTime(seconds: number) {
      const mins = Math.floor(seconds / 60);
      const secs = Math.floor(seconds % 60);
      return `${mins}:${secs.toString().padStart(2, '0')}`;
    }
    
    // Effect to handle cleanup when component unmounts
    $effect(() => {
      return () => {
        if (loopTimeout) clearTimeout(loopTimeout);
        if (timeUpdateInterval) clearInterval(timeUpdateInterval);
        if (player) player.destroy();
      };
    });
  </script>
  
  <div class="container">
    <h1>🎷 Transcription Practice Looper</h1>
    
    {#if showToast}
      <div class="toast">
        {toastMessage}
      </div>
    {/if}
    
    <div class="url-section">
      <input 
        bind:value={videoUrl}
        type="text" 
        class="url-input" 
        placeholder="Paste YouTube URL here..."
      />
      <button class="button button-primary" onclick={loadVideo}>
        Load Video
      </button>
      <button class="button button-secondary" onclick={shareLoop}>
        Share Loop
      </button>
    </div>
    
    <div class="video-container">
      <div id="youtube-player"></div>
    </div>
    
    <div class="controls">
      <div class="control-group">
        <h3>Loop Points (in seconds)</h3>
        <div class="time-inputs">
          <input 
            bind:value={startTime}
            type="number" 
            class="time-input" 
            placeholder="Start" 
            min="0"
          />
          <span>to</span>
          <input 
            bind:value={endTime}
            type="number" 
            class="time-input" 
            placeholder="End" 
            min="0"
          />
        </div>
        <div class="quick-buttons">
          <button class="button button-secondary small-button" onclick={setCurrentAsStart}>
            Set Start
          </button>
          <button class="button button-secondary small-button" onclick={setCurrentAsEnd}>
            Set End
          </button>
        </div>
      </div>
      
      <div class="control-group">
        <h3>Playback Speed</h3>
        <div class="speed-controls">
          <div class="speed-slider-container">
            <input 
              bind:value={playbackSpeed}
              type="range" 
              min="0.25" 
              max="2" 
              step="0.25" 
              class="speed-slider"
              oninput={(e) => setPlaybackSpeed(parseFloat((e.target as HTMLInputElement).value))}
            />
            <div class="speed-value">{playbackSpeed}x</div>
          </div>
          <div class="speed-presets">
            <button class="button button-secondary small-button" onclick={() => setPlaybackSpeed(0.25)}>
              0.25x
            </button>
            <button class="button button-secondary small-button" onclick={() => setPlaybackSpeed(0.5)}>
              0.5x
            </button>
            <button class="button button-secondary small-button" onclick={() => setPlaybackSpeed(0.75)}>
              0.75x
            </button>
            <button class="button button-secondary small-button" onclick={() => setPlaybackSpeed(1)}>
              1x
            </button>
            <button class="button button-secondary small-button" onclick={() => setPlaybackSpeed(1.25)}>
              1.25x
            </button>
            <button class="button button-secondary small-button" onclick={() => setPlaybackSpeed(1.5)}>
              1.5x
            </button>
            <button class="button button-secondary small-button" onclick={() => setPlaybackSpeed(2)}>
              2x
            </button>
          </div>
        </div>
      </div>
      
      <div class="control-group">
        <h3>Delay Between Loops</h3>
        <div class="time-inputs">
          <input 
            bind:value={delayTime}
            type="number" 
            class="delay-input" 
            min="0" 
            step="0.5"
          />
          <span>seconds</span>
        </div>
      </div>
      
      <div class="control-group">
        <h3>Playback</h3>
        <div class="quick-buttons">
          <button class="button button-secondary small-button" onclick={jumpToStart}>
            Jump to Start
          </button>
          <button class="button button-secondary small-button" onclick={jumpToEnd}>
            Jump to End
          </button>
        </div>
        <button 
          class="button {isLooping ? 'button-secondary' : 'button-primary'}" 
          onclick={toggleLoop}
          disabled={!isPlayerReady}
        >
          {isLooping ? 'Stop Loop' : 'Start Loop'}
        </button>
      </div>
    </div>
    
    <div class="status">
      <div class="current-time">{formattedCurrentTime}</div>
      <div class="loop-info">{loopInfo}</div>
      <div class="speed-info">{speedInfo}</div>
      {#if isLooping}
        <div class="loop-active">🔄 Looping active</div>
      {/if}
    </div>
  </div>
  
  <style>
    :global(body) {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
      background: #1a1a1a;
      color: #fff;
      margin: 0;
      padding: 20px;
    }
    
    .container {
      max-width: 800px;
      margin: 0 auto;
    }
    
    h1 {
      text-align: center;
      margin-bottom: 30px;
      color: #ff6b35;
    }
    
    .url-section {
      display: flex;
      gap: 10px;
      margin-bottom: 20px;
    }
    
    .url-input {
      flex: 1;
      padding: 12px;
      border: 2px solid #333;
      border-radius: 8px;
      background: #2a2a2a;
      color: #fff;
      font-size: 16px;
    }
    
    .url-input:focus {
      outline: none;
      border-color: #ff6b35;
    }
    
    .video-container {
      position: relative;
      width: 100%;
      height: 315px;
      margin-bottom: 20px;
      border-radius: 12px;
      overflow: hidden;
      background: #000;
    }
    
    .controls {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
      gap: 20px;
      margin-bottom: 20px;
    }
    
    .control-group {
      background: #2a2a2a;
      padding: 20px;
      border-radius: 12px;
      border: 1px solid #333;
    }
    
    .control-group h3 {
      margin-bottom: 15px;
      color: #ff6b35;
      font-size: 14px;
      text-transform: uppercase;
      letter-spacing: 1px;
    }
    
    .time-inputs {
      display: flex;
      gap: 10px;
      align-items: center;
      margin-bottom: 10px;
    }
    
    .time-input, .delay-input {
      padding: 8px;
      border: 1px solid #444;
      border-radius: 6px;
      background: #1a1a1a;
      color: #fff;
      text-align: center;
    }
    
    .time-input {
      width: 80px;
    }
    
    .delay-input {
      width: 100px;
    }
    
    .speed-controls {
      display: flex;
      flex-direction: column;
      gap: 15px;
    }
    
    .speed-slider-container {
      display: flex;
      align-items: center;
      gap: 15px;
    }
    
    .speed-slider {
      flex: 1;
      height: 6px;
      border-radius: 3px;
      background: #444;
      outline: none;
      -webkit-appearance: none;
    }
    
    .speed-slider::-webkit-slider-thumb {
      -webkit-appearance: none;
      appearance: none;
      width: 20px;
      height: 20px;
      border-radius: 50%;
      background: #ff6b35;
      cursor: pointer;
    }
    
    .speed-slider::-moz-range-thumb {
      width: 20px;
      height: 20px;
      border-radius: 50%;
      background: #ff6b35;
      cursor: pointer;
      border: none;
    }
    
    .speed-value {
      font-family: 'Courier New', monospace;
      font-weight: bold;
      color: #ff6b35;
      min-width: 40px;
      text-align: center;
    }
    
    .speed-presets {
      display: flex;
      gap: 8px;
      flex-wrap: wrap;
      justify-content: center;
    }
    
    .button {
      padding: 10px 20px;
      border: none;
      border-radius: 6px;
      font-weight: 600;
      cursor: pointer;
      transition: all 0.2s;
    }
    
    .button-primary {
      background: #ff6b35;
      color: white;
    }
    
    .button-primary:hover {
      background: #e55a2b;
    }
    
    .button-secondary {
      background: #333;
      color: white;
    }
    
    .button-secondary:hover {
      background: #444;
    }
    
    .button:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }
    
    .quick-buttons {
      display: flex;
      gap: 10px;
      margin-bottom: 15px;
      flex-wrap: wrap;
    }
    
    .small-button {
      padding: 6px 12px;
      font-size: 12px;
    }
    
    .status {
      text-align: center;
      padding: 20px;
      background: #2a2a2a;
      border-radius: 12px;
    }
    
    .current-time {
      font-family: 'Courier New', monospace;
      font-size: 24px;
      color: #ff6b35;
      margin-bottom: 10px;
    }
    
    .loop-info, .speed-info {
      color: #ccc;
      margin-bottom: 10px;
    }
    
    .loop-active {
      color: #4ade80;
      font-weight: 600;
      animation: pulse 2s infinite;
    }
    
    @keyframes pulse {
      0%, 100% { opacity: 1; }
      50% { opacity: 0.5; }
    }
    
    .toast {
      position: fixed;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      background: #4ade80;
      color: #000;
      padding: 12px 24px;
      border-radius: 8px;
      font-weight: 600;
      z-index: 1000;
      animation: slideIn 0.3s ease-out;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    }
    
    @keyframes slideIn {
      from {
        transform: translateX(-50%) translateY(-100%);
        opacity: 0;
      }
      to {
        transform: translateX(-50%) translateY(0);
        opacity: 1;
      }
    }
  </style>