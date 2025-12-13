<script lang="ts">
  import { onMount, tick } from 'svelte';
  // Import Ably from the package; Vite will resolve the browser entry
  import Ably from 'ably';

  let started = false;
  let status = 'Waiting for audio...';
  let logs = '';
  let lastUrl = '';
  let connected = false;
  let player: HTMLAudioElement | undefined;
  let audioCtx: AudioContext | undefined;
  let pendingUrl = '';
  let bgColor = '#1e3a8a'; // initial dark blue

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
          // wait for DOM to update so the <audio> element binds to `player`
          await tick();
          // attach player event handlers so background changes when playback starts/ends
          attachPlayerHandlers();
          if (player) {
            player.src = url;
            player.play().then(() => {
              pendingUrl = '';
            }).catch((err: any)=>{
              log('play error: ' + (err && err.message ? err.message : err));
              // Save pending URL so the user can tap to play when allowed
              pendingUrl = url;
            });
          } else {
            // no audio element yet; keep as pending
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
    bgColor = '#2563eb'; // blue-600
    console.log('Background set to blue');
    // attempt to unlock audio on first user gesture (silent oscillator)
    try {
      const AudioCtx = window.AudioContext || (window as any).webkitAudioContext;
      if (AudioCtx) {
        audioCtx = new AudioCtx();
        audioCtx.resume().then(() => {
          try {
            if (!audioCtx) return;
            const o = audioCtx.createOscillator();
            const g = audioCtx.createGain();
            g.gain.value = 0; // silent
            o.connect(g);
            g.connect(audioCtx.destination);
            o.start();
            o.stop(audioCtx.currentTime + 0.01);
          } catch (e) {
            // ignore
          }
        }).catch(()=>{});
      }
    } catch (e) {}
    // start server connection shortly after intro fades
    setTimeout(() => connectAbly(), 350);
    // if a URL arrived earlier and was pending, try to play it now
    setTimeout(() => {
      if (pendingUrl && player) {
        // ensure audio context resumed
        try { if (audioCtx) audioCtx.resume().catch(()=>{}); } catch(e){}
        attachPlayerHandlers();
        player.src = pendingUrl;
        player.play().then(()=>{ pendingUrl = ''; }).catch((err: any)=>{ log('play error on start: ' + (err && err.message ? err.message : err)); });
      }
    }, 400);
  }

  function playPending() {
    if (!pendingUrl) return;
    try { if (audioCtx) audioCtx.resume().catch(()=>{}); } catch(e){}
    if (player) {
      attachPlayerHandlers();
      player.src = pendingUrl;
      player.play().then(()=>{ pendingUrl = ''; }).catch((err: any)=>{ log('playPending error: ' + (err && err.message ? err.message : err)); });
    }
  }

  function attachPlayerHandlers() {
    try {
      if (!player) return;
      // avoid reassigning handlers repeatedly
      if ((player as any)._handlersAttached) return;
      (player as any)._handlersAttached = true;
      player.addEventListener('playing', () => {
        bgColor = '#10b981'; // green-500
        console.log('playback started, bg changed to green');
        log('playback started');
      });
      player.addEventListener('ended', () => {
        bgColor = '#2563eb'; // blue-600
        status = 'Waiting for signal';
        log('playback ended');
      });
    } catch (e) {}
  }

  onMount(() => {
    console.log('DropcallReceiver mounted, started:', started);
    // set the browser tab title
    try { if (typeof document !== 'undefined') document.title = 'Dagyaw Radio'; } catch (e) {}
    // if we land directly to this page after initial start, connect immediately
    if (started) connectAbly();
  });
</script>

<style>
  .bg-transition { transition: background-color 400ms ease, background-image 400ms ease; }
  .intro-fade { transition: opacity 300ms ease, transform 300ms ease; }
</style>

<div class="min-h-screen flex items-center justify-center text-white font-mono bg-transition" style="background-color: {bgColor};">
  {#if !started}
    <button on:click={start} class="intro-fade focus:outline-none">
      <div class="flex items-center justify-center p-8">
        <span class="text-3xl sm:text-4xl font-semibold select-none">Tap to start</span>
      </div>
    </button>
  {/if}

  {#if started}
    <div class="absolute inset-0 flex items-center justify-center">
      <div class="text-white text-center">
        <h1 class="text-2xl font-bold">Dagyaw Radio</h1>
        <p class="mt-2 opacity-80">{status}</p>

        {#if lastUrl}
          <div class="mt-6">
            <!-- hidden player only; UI text suppressed for security -->
            <audio bind:this={player} class="hidden" autoplay></audio>
          </div>
        {/if}
      </div>
    </div>
  {/if}
</div>

<!-- background change handled in start() -->
