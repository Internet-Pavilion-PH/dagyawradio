<script lang="ts">
  import { onMount, tick, onDestroy } from 'svelte';
  import Ably from 'ably';

  let started = false;
  let status = 'Waiting for audio...';
  let logs = '';
  let lastUrl = '';
  let connected = false;
  let player: HTMLAudioElement | undefined;
  let audioCtx: AudioContext | undefined;
  let pendingUrl = '';
  let bgColor = '#000000'; // initial black

  // Visualizer state
  let canvas: HTMLCanvasElement | null = null;
  let canvasCtx: CanvasRenderingContext2D | null = null;
  let analyser: AnalyserNode | null = null;
  let sourceNode: AudioNode | null = null;
  let dataArray: Uint8Array | null = null;
  let rafId: number | null = null;
  let visualsVisible = false;

  function log(msg: string) {
    const time = new Date().toLocaleTimeString();
    logs = `${time} — ${msg}\n${logs}`;
  }

  async function connectAbly() {
    console.log('connectAbly() called');
    try {
      status = 'Connecting to GhostWhisper server...';
      console.log('Status:', status);
      const realtime = new Ably.Realtime({
        authCallback: async (tokenParams, callback) => {
          try {
            const res = await fetch('https://kolown.net/api/ghost_auth', { credentials: 'include' });
            if (!res.ok) throw new Error('token request failed: ' + res.status);
            const tokenRequest = await res.json();
            callback(null, tokenRequest);
          } catch (err) {
            const message = (err instanceof Error) ? err.message : 'Unknown error';
            callback(message, null);
          }
        }
      });

      realtime.connection.on('connected', () => {
        connected = true;
        status = 'Connected to GhostWhisper server ';
        console.log('Ably connected!');
        log('GhostWhisper server connected');
      });

      const channel = realtime.channels.get('dropcall');
      channel.subscribe('url', async (msg) => {
        try {
          const url = (typeof msg.data === 'string') ? msg.data : (msg.data.url || JSON.stringify(msg.data));
          status = 'Playing audio';
          log('Received url: ' + url);
          lastUrl = url;
          await tick();
          attachPlayerHandlers();
          if (player) {
            player.src = url;
            player.play().then(() => {
              pendingUrl = '';
              setupVisualizer();
            }).catch((err: any)=>{
              log('play error: ' + (err && err.message ? err.message : err));
              pendingUrl = url;
            });
          } else {
            pendingUrl = url;
          }
        } catch (e: any) {
          log('Error handling message: ' + e.message);
        }
      });

    } catch (e: any) {
      status = 'GhostWhisper server connection failed';
      log('GhostWhisper server error: ' + e.message);
    }
  }

  function start() {
    console.log('start() called');
    started = true;
    bgColor = '#10b981'; // emerald-500
    console.log('Background set to blue');
    try {
      const AudioCtx = window.AudioContext || (window as any).webkitAudioContext;
      if (AudioCtx) {
        audioCtx = new AudioCtx();
        (window as any).__dagyaw_audio_context = audioCtx;
        audioCtx.resume().then(() => {
          try {
            if (!audioCtx) return;
            const o = audioCtx.createOscillator();
            const g = audioCtx.createGain();
            g.gain.value = 0;
            o.connect(g);
            g.connect(audioCtx.destination);
            o.start();
            o.stop(audioCtx.currentTime + 0.01);
          } catch (e) {}
        }).catch(()=>{});
      }
    } catch (e) {}
    setTimeout(() => connectAbly(), 350);
    setTimeout(() => {
      if (pendingUrl && player) {
        try { if (audioCtx) audioCtx.resume().catch(()=>{}); } catch(e){}
        attachPlayerHandlers();
        player.src = pendingUrl;
        player.play().then(()=>{ pendingUrl = ''; setupVisualizer(); }).catch((err: any)=>{ log('play error on start: ' + (err && err.message ? err.message : err)); });
      }
    }, 400);
  }

  function attachPlayerHandlers() {
    try {
      if (!player) return;
      if ((player as any)._handlersAttached) return;
      (player as any)._handlersAttached = true;
      player.addEventListener('playing', () => {
        bgColor = '#000000';
        console.log('playback started, bg changed to green');
        log('playback started');
        visualsVisible = true;
        if (!rafId) draw();
      });
      player.addEventListener('ended', () => {
        bgColor = '#10b981';
        status = 'Waiting for signal';
        log('playback ended');
        visualsVisible = false;
      });
      player.addEventListener('pause', () => {
        visualsVisible = false;
      });
    } catch (e) {}
  }

  async function setupVisualizer() {
    if (!player || !audioCtx) {
      console.warn('setupVisualizer: missing player or audioCtx', { player: !!player, audioCtx: !!audioCtx });
      return;
    }
    console.log('setupVisualizer() called');

    // Ensure AudioContext is resumed
    try {
      await audioCtx.resume();
      console.log('AudioContext resumed, state:', audioCtx.state);
    } catch (e) {
      console.warn('Failed to resume AudioContext', e);
    }

    // Check player state before connecting
    console.log('Player state:', { 
      muted: player.muted, 
      volume: player.volume, 
      paused: player.paused,
      src: player.src
    });

    // Ensure player isn't muted and has volume
    player.muted = false;
    player.volume = 1.0;

    // Reuse stored source if present
    if ((player as any).__dagyaw_media_source && (player as any).__dagyaw_media_source_context === audioCtx) {
      sourceNode = (player as any).__dagyaw_media_source as AudioNode;
      console.log('Reusing stored media source');
    } else {
      try {
        sourceNode = audioCtx.createMediaElementSource(player);
        (player as any).__dagyaw_media_source = sourceNode;
        (player as any).__dagyaw_media_source_context = audioCtx;
        console.log('Created new media source');
      } catch (err) {
        console.warn('createMediaElementSource failed, trying captureStream', err);
        const cap = (player as any).captureStream || (player as any).mozCaptureStream;
        if (cap) {
          try {
            const stream: MediaStream = cap.call(player);
            sourceNode = audioCtx.createMediaStreamSource(stream);
            console.log('Using captureStream fallback');
          } catch (e) {
            console.warn('captureStream failed', e);
            return;
          }
        } else {
          console.warn('captureStream not available');
          return;
        }
      }
    }

    if (!analyser) {
      analyser = audioCtx.createAnalyser();
      analyser.fftSize = 2048;
      analyser.smoothingTimeConstant = 0.75;
      dataArray = new Uint8Array(analyser.fftSize);
    }

    try {
      sourceNode.connect(analyser);
      analyser.connect(audioCtx.destination);
      console.log('Visualizer setup complete', {
        sourceConnected: true,
        analyserConnected: true,
        contextState: audioCtx.state,
        contextDestination: !!audioCtx.destination
      });
    } catch (e) {
      console.warn('Failed to connect nodes', e);
      return;
    }

    visualsVisible = true;
    if (!rafId) draw();
  }

  function resizeCanvas() {
    if (!canvas) return;
    const dpr = window.devicePixelRatio || 1;
    const w = Math.min(window.innerWidth - 40, 900);
    const h = 140;
    canvas.width = Math.floor(w * dpr);
    canvas.height = Math.floor(h * dpr);
    canvas.style.width = `${w}px`;
    canvas.style.height = `${h}px`;
    canvasCtx = canvas.getContext('2d');
    if (canvasCtx) canvasCtx.setTransform(dpr, 0, 0, dpr, 0, 0);
  }

  function draw() {
    if (!analyser || !dataArray || !canvas || !canvasCtx) {
      rafId = requestAnimationFrame(draw);
      return;
    }
    analyser.getByteTimeDomainData(dataArray as any);
    const w = canvas.width / (window.devicePixelRatio || 1);
    const h = canvas.height / (window.devicePixelRatio || 1);
    canvasCtx.clearRect(0, 0, w, h);
    canvasCtx.lineWidth = 2;
    canvasCtx.strokeStyle = '#10b981';
    canvasCtx.beginPath();
    const step = Math.max(1, Math.floor(dataArray.length / w));
    for (let i = 0, x = 0; i < dataArray.length; i += step, x++) {
      const v = dataArray[i] / 128.0;
      const y = (1 - v / 2) * h;
      if (x === 0) canvasCtx.moveTo(x, y); else canvasCtx.lineTo(x, y);
    }
    canvasCtx.stroke();
    rafId = requestAnimationFrame(draw);
  }

  onMount(() => {
    console.log('RadioVisuals mounted, started:', started);
    try { if (typeof document !== 'undefined') document.title = 'Dagyaw Radio'; } catch (e) {}
    if (started) connectAbly();
  });

  // Reactive: setup canvas when it becomes available
  $: if (canvas && typeof window !== 'undefined') {
    resizeCanvas();
  }

  onDestroy(() => {
    try { if (rafId) cancelAnimationFrame(rafId); } catch (e) {}
    try { analyser?.disconnect(); } catch (e) {}
    try { sourceNode?.disconnect(); } catch (e) {}
  });
</script>

<style>
  .bg-transition { transition: background-color 400ms ease, background-image 400ms ease; }
  .intro-fade { transition: opacity 300ms ease, transform 300ms ease; }
  .visuals-container { width:100%; max-width:900px; margin:12px auto; text-align: center; }
  canvas { display:block; margin: 0 auto; }

  /* reverted to full-screen layout */
  .radio-player {
    position: fixed;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    color: white;
    font-family: monospace;
    overflow: hidden;
    min-height: 100vh;
  }

  .player-content {
    text-align: center;
  }
</style>

<div class="radio-player bg-transition" style="background-color: {bgColor};">
  {#if !started}
    <button on:click={start} class="intro-fade focus:outline-none bg-emerald-600 hover:bg-emerald-700 text-white rounded-lg px-6 py-4">
      <div class="flex items-center justify-center">
        <span class="text-3xl sm:text-4xl font-semibold select-none">Tap to start</span>
      </div>
    </button>
  {/if}

  {#if started}
    <div class="player-content">
      <div class="text-white text-center">
        <h1 class="text-2xl font-bold">Dagyaw Radio</h1>
        <p class="mt-2 opacity-80">{status}</p>

        {#if lastUrl}
          <div class="mt-6">
            <audio bind:this={player} class="hidden" autoplay crossorigin="anonymous"></audio>
          </div>
        {/if}
      </div>

      {#if visualsVisible}
        <div class="visuals-container">
          <canvas bind:this={canvas} width="800" height="140"></canvas>
        </div>
      {/if}
    </div>
  {/if}
</div>
