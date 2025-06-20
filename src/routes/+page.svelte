<script>
    import { onMount } from 'svelte';
    
    // Svelte 5 runes for reactive state
    let videoUrl = $state('');
    let videoId = $state('');
    let startTime = $state(0);
    let endTime = $state(30);
    let delayTime = $state(1);
    let isLooping = $state(false);
    let currentTime = $state(0);
    let isPlayerReady = $state(false);
    
    let player = $state(null);
    let loopTimeout = $state(null);
    let timeUpdateInterval = $state(null);
    
    // YouTube API setup
    let YT = $state(null);
    let apiReady = $state(false);
    
    // Derived state using $derived rune
    let formattedCurrentTime = $derived(formatTime(currentTime));
    let loopDuration = $derived(endTime - startTime);
    let loopInfo = $derived(`Loop: ${formatTime(startTime)} - ${formatTime(endTime)} (${delayTime}s delay)`);
    
    onMount(() => {
      loadYouTubeAPI();
      
      return () => {
        if (loopTimeout) clearTimeout(loopTimeout);
        if (timeUpdateInterval) clearInterval(timeUpdateInterval);
      };
    });
    
    function loadYouTubeAPI() {
      if (window.YT) {
        YT = window.YT;
        apiReady = true;
        return;
      }
      
      const tag = document.createElement('script');
      tag.src = 'https://www.youtube.com/iframe_api';
      
      window.onYouTubeIframeAPIReady = () => {
        YT = window.YT;
        apiReady = true;
      };
      
      document.head.appendChild(tag);
    }
    
    function extractVideoId(url) {
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
    
    function onPlayerReady(event) {
      isPlayerReady = true;
      startTimeTracking();
    }
    
    function onPlayerStateChange(event) {
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
      const loopDurationMs = loopDuration * 1000;
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
    
    function formatTime(seconds) {
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
    <h1>🎷 Sax Practice Looper</h1>
    
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
    </div>
    
    <div class="video-container">
      <div id="youtube-player"></div>
    </div>
    
    <div class="controls">
      <div class="control-group">
        <h3>Loop Points</h3>
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
          <span>sec</span>
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
    
    .loop-info {
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
  </style>