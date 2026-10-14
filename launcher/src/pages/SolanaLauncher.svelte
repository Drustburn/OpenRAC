<script lang="ts">
  // Project Solana Launcher view: an authentic recreation of Project Solana's
  // "Horizon" launcher (Direction D), integrated seamlessly into OpenRAC.
  import { api, pickFolder, pickIso } from "$lib/api";
  import {
    app,
    cancel,
    guard,
    installGame,
    refresh,
    runningJob,
    saveConfig,
    toggleSolanaMode,
    uninstallGame,
  } from "$lib/app.svelte";
  import { isBuilt, playerAct, playerState } from "$lib/player.svelte";

  interface SolanaGameDef {
    serial: string;
    title: string;
    region: string;
    gameId: string;
    versionKey: string;
    md5: string;
    playable: boolean;
  }

  const SOLANA_GAMES: SolanaGameDef[] = [
    {
      serial: "SCES_509.16",
      title: "Ratchet & Clank",
      region: "PAL",
      gameId: "rac1",
      versionKey: "rac1/pal",
      md5: "d86429e2eca4e1da7d7f1cc656af2574",
      playable: true,
    },
    {
      serial: "SCUS_971.99",
      title: "Ratchet & Clank",
      region: "NTSC-U",
      gameId: "rac1",
      versionKey: "rac1/ntsc-u",
      md5: "e44d7159757f58f5cb5eb28c117b075e",
      playable: false,
    },
    {
      serial: "SCPS_150.37",
      title: "Ratchet & Clank",
      region: "NTSC-J",
      gameId: "rac1",
      versionKey: "rac1/ntsc-j",
      md5: "",
      playable: false,
    },
    {
      serial: "SCES_516.07",
      title: "Ratchet & Clank 2",
      region: "PAL",
      gameId: "rac2",
      versionKey: "rac2/pal",
      md5: "",
      playable: false,
    },
    {
      serial: "SCUS_972.68",
      title: "Ratchet & Clank: Going Commando",
      region: "NTSC-U",
      gameId: "rac2",
      versionKey: "rac2/ntsc",
      md5: "",
      playable: false,
    },
    {
      serial: "SCES_524.56",
      title: "Ratchet & Clank 3",
      region: "PAL",
      gameId: "rac3",
      versionKey: "rac3/pal",
      md5: "",
      playable: false,
    },
    {
      serial: "SCUS_973.53",
      title: "Ratchet & Clank: Up Your Arsenal",
      region: "NTSC-U",
      gameId: "rac3",
      versionKey: "rac3/ntsc",
      md5: "",
      playable: false,
    },
    {
      serial: "SCES_532.85",
      title: "Ratchet: Gladiator",
      region: "PAL",
      gameId: "rac4",
      versionKey: "rac4/pal",
      md5: "",
      playable: false,
    },
    {
      serial: "SCUS_974.65",
      title: "Ratchet: Deadlocked",
      region: "NTSC-U",
      gameId: "rac4",
      versionKey: "rac4/ntsc",
      md5: "",
      playable: false,
    },
  ];

  const RAC1_LEVELS = [
    "From save / normal",
    "Veldin",
    "Novalis",
    "Aridia",
    "Kerwan",
    "Eudora",
    "Rilgar",
    "Blarg Station",
    "Umbris",
    "Batalia",
    "Gaspar",
    "Orxon",
    "Pokitaru",
    "Hoven",
    "Gemlik Base",
    "Oltanis",
    "Quartu",
    "Kalebo III",
    "Drek's Fleet",
    "Veldin (Drek)",
  ];

  const CONTROLLER_MAPPINGS = [
    {
      psImg: "/solana/png/controller_colour/ps_cross.png",
      psName: "Cross",
      xbImg: "/solana/png/controller_colour/xbox_a.png",
      xbName: "A",
    },
    {
      psImg: "/solana/png/controller_colour/ps_circle.png",
      psName: "Circle",
      xbImg: "/solana/png/controller_colour/xbox_b.png",
      xbName: "B",
    },
    {
      psImg: "/solana/png/controller_colour/ps_square.png",
      psName: "Square",
      xbImg: "/solana/png/controller_colour/xbox_x.png",
      xbName: "X",
    },
    {
      psImg: "/solana/png/controller_colour/ps_triangle.png",
      psName: "Triangle",
      xbImg: "/solana/png/controller_colour/xbox_y.png",
      xbName: "Y",
    },
    {
      psImg: "/solana/png/controller_colour/ps_start.png",
      psName: "Start",
      xbImg: "/solana/png/controller_colour/xbox_menu.png",
      xbName: "Menu",
    },
    {
      psImg: "/solana/png/controller_colour/ps_l1.png",
      psName: "L1",
      xbImg: "/solana/png/controller_colour/xbox_lb.png",
      xbName: "LB",
    },
    {
      psImg: "/solana/png/controller_colour/ps_r1.png",
      psName: "R1",
      xbImg: "/solana/png/controller_colour/xbox_rb.png",
      xbName: "RB",
    },
    {
      psImg: "/solana/png/controller_colour/ps_l2.png",
      psName: "L2",
      xbImg: "/solana/png/controller_colour/xbox_lt.png",
      xbName: "LT",
    },
    {
      psImg: "/solana/png/controller_colour/ps_r2.png",
      psName: "R2",
      xbImg: "/solana/png/controller_colour/xbox_rt.png",
      xbName: "RT",
    },
    {
      psImg: "/solana/png/controller_colour/ps_select.png",
      psName: "Select",
      xbImg: "/solana/png/controller_colour/xbox_view.png",
      xbName: "View",
    },
  ];

  // Selection & settings view
  let selectedIndex = $state(0);
  let viewSettings = $state(false);

  // Solana Settings
  interface SolanaSettings {
    upscaled: boolean;
    vsync: boolean;
    nomtgs: boolean;
    noglt: boolean;
    fpslog: boolean;
    minimize: boolean;
    captureSeconds: number;
    startLevel: number;
  }

  function loadSettings(): SolanaSettings {
    try {
      const raw = localStorage.getItem("openrac_solana_settings");
      if (raw) return JSON.parse(raw);
    } catch {
      // fallback
    }
    return {
      upscaled: true,
      vsync: false,
      nomtgs: false,
      noglt: false,
      fpslog: false,
      minimize: true,
      captureSeconds: 3,
      startLevel: 0,
    };
  }

  const settings = $state<SolanaSettings>(loadSettings());

  function saveSettings() {
    try {
      localStorage.setItem("openrac_solana_settings", JSON.stringify(settings));
    } catch {
      // ignore
    }
  }

  const fallbackGame: SolanaGameDef = {
    serial: "SCES_509.16",
    title: "Ratchet & Clank",
    region: "PAL",
    gameId: "rac1",
    versionKey: "rac1/pal",
    md5: "d86429e2eca4e1da7d7f1cc656af2574",
    playable: true,
  };

  const selectedDef = $derived<SolanaGameDef>(SOLANA_GAMES[selectedIndex] ?? fallbackGame);

  // Find matching game & version in OpenRAC
  const matchingGame = $derived(app.library?.games.find((g) => g.id === selectedDef.gameId) ?? null);
  const matchingVersion = $derived(
    matchingGame?.versions.find((v) => v.key === selectedDef.versionKey) ?? matchingGame?.versions[0] ?? null,
  );
  const foundDiscInfo = $derived(
    app.discs.find((d) => d.serial === selectedDef.serial || d.game === selectedDef.gameId) ?? null,
  );

  const nextAction = $derived(matchingGame && matchingVersion ? playerState(matchingGame, matchingVersion) : null);
  const built = $derived(!!matchingGame && !!matchingVersion && isBuilt(matchingGame, matchingVersion));
  const running = $derived(runningJob());

  function getStatusInfo(def: SolanaGameDef) {
    if (!def.playable) {
      return { text: "Not supported yet", color: "#5E6888", code: "notyet" };
    }
    if (running) {
      return { text: "Building…", color: "#FF7A3D", code: "building" };
    }
    if (built) {
      return { text: "Playable start to finish", color: "#3DD68C", code: "ready" };
    }
    if (foundDiscInfo || matchingGame?.install?.extracted) {
      return { text: "Ready to build", color: "#FF7A3D", code: "ready-build" };
    }
    return { text: "Playable start to finish", color: "#3DD68C", code: "supported" };
  }

  async function handlePlay() {
    if (!matchingVersion || !nextAction) return;
    await playerAct(matchingVersion, nextAction);
  }

  async function handleAddDisc() {
    const file = await pickIso("Add PS2 disc image (.iso)");
    if (!file) return;
    if (matchingGame) {
      await installGame(matchingGame.id, selectedDef.title, file);
      await refresh();
    }
  }

  async function handleScanFolder() {
    const dir = await pickFolder("Find disc images in this folder");
    if (!dir || !app.config) return;
    await saveConfig({ ...app.config, discsDir: dir });
    await refresh();
  }

  async function handleRemove() {
    if (!matchingGame) return;
    const ok = confirm(
      `Remove what was set up from your ${selectedDef.title} disc? Saves and disc images will be kept.`,
    );
    if (ok) {
      await uninstallGame(matchingGame.id);
    }
  }

  async function handleOpenFolder() {
    if (matchingVersion?.dir) {
      await guard(api.openPath(matchingVersion.dir));
    }
  }

  function handleOpenLog() {
    app.page = "tasks";
    toggleSolanaMode();
  }
</script>

<div class="solana-root">
  <!-- Subtle sunrise transition overlay on enter -->
  <div class="solar-flare-enter" aria-hidden="true"></div>

  <!-- Left Sidebar (300px) -->
  <aside class="sidebar">
    <div class="wordmark-row">
      <img src="/solana/svg/wordmark.svg" alt="Project Solana" class="wordmark" />
      <button class="secret-toggle-btn" onclick={toggleSolanaMode} title="Back to OpenRAC" aria-label="Back to OpenRAC">
        <img src="/wrench.webp" alt="OpenRAC" width="18" height="18" />
      </button>
    </div>

    <div class="library-header">LIBRARY</div>

    <div class="discs-list">
      {#each SOLANA_GAMES as g, idx (g.serial)}
        {@const st = getStatusInfo(g)}
        {@const isSel = idx === selectedIndex}
        <button
          class="disc-item"
          class:selected={isSel}
          onclick={() => {
            selectedIndex = idx;
            viewSettings = false;
          }}
        >
          {#if isSel}
            <div class="selection-bar"></div>
          {/if}
          <div class="item-title">{g.title}</div>
          <div class="item-meta">
            <span class="region-badge">{g.region}</span>
            <span class="status-dot" style="background-color: {st.color}"></span>
            <span class="status-text">{st.text}</span>
          </div>
        </button>
      {/each}
    </div>

    <div class="sidebar-bottom">
      <button class="bottom-btn" onclick={handleAddDisc}>
        <img src="/solana/svg/icons/add.svg" alt="" width="16" height="16" />
        <span>Add disc image</span>
      </button>
      <button class="bottom-btn" onclick={handleScanFolder}>
        <img src="/solana/svg/icons/scan.svg" alt="" width="16" height="16" />
        <span>Scan a folder</span>
      </button>
    </div>
  </aside>

  <!-- Main Content Area -->
  <main class="main-content">
    {#if viewSettings}
      <!-- Solana Settings View -->
      <section class="settings-view">
        <div class="settings-header">
          <h2>Settings</h2>
          <button class="done-btn" onclick={() => (viewSettings = false)}>Done</button>
        </div>
        <p class="settings-hint">Apply the next time a game starts.</p>

        <div class="settings-list">
          <!-- Resolution -->
          <div class="setting-row">
            <div class="setting-info">
              <span class="setting-title">Resolution</span>
              <span class="setting-desc">
                Native draws at the PS2's own resolution, as the game was. Upscaled draws at a whole multiple of the
                PS2's 448 lines: sharper, more GPU work.
              </span>
            </div>
            <div class="segmented-control">
              <button
                class="segment"
                class:active={!settings.upscaled}
                onclick={() => {
                  settings.upscaled = false;
                  saveSettings();
                }}
              >
                Native
              </button>
              <button
                class="segment"
                class:active={settings.upscaled}
                onclick={() => {
                  settings.upscaled = true;
                  saveSettings();
                }}
              >
                Upscaled
              </button>
            </div>
          </div>

          <!-- VSync -->
          <div class="setting-row">
            <div class="setting-info">
              <span class="setting-title">VSync</span>
              <span class="setting-desc">Waits for the monitor's refresh. Off lets the game pace itself at 50 fps.</span
              >
            </div>
            <button
              class="toggle-switch"
              class:on={settings.vsync}
              onclick={() => {
                settings.vsync = !settings.vsync;
                saveSettings();
              }}
              aria-label="Toggle VSync"
            >
              <span class="toggle-knob"></span>
            </button>
          </div>

          <!-- Graphics on its own thread -->
          <div class="setting-row">
            <div class="setting-info">
              <span class="setting-title">Graphics on its own thread</span>
              <span class="setting-desc"
                >Runs the PS2 graphics chip beside the game. Turn off only to test a problem.</span
              >
            </div>
            <button
              class="toggle-switch"
              class:on={!settings.nomtgs}
              onclick={() => {
                settings.nomtgs = !settings.nomtgs;
                saveSettings();
              }}
              aria-label="Toggle Graphics on thread"
            >
              <span class="toggle-knob"></span>
            </button>
          </div>

          <!-- OpenGL calls on their own thread -->
          <div class="setting-row">
            <div class="setting-info">
              <span class="setting-title">OpenGL calls on their own thread</span>
              <span class="setting-desc"
                >Sends drawing to the driver from a separate thread. Turn off only to test a problem.</span
              >
            </div>
            <button
              class="toggle-switch"
              class:on={!settings.noglt}
              onclick={() => {
                settings.noglt = !settings.noglt;
                saveSettings();
              }}
              aria-label="Toggle OpenGL thread"
            >
              <span class="toggle-knob"></span>
            </button>
          </div>

          <!-- Performance statistics -->
          <div class="setting-row">
            <div class="setting-info">
              <span class="setting-title">Performance statistics</span>
              <span class="setting-desc">Writes frame timings to rac1_log.txt every 2 seconds, for bug reports.</span>
            </div>
            <button
              class="toggle-switch"
              class:on={settings.fpslog}
              onclick={() => {
                settings.fpslog = !settings.fpslog;
                saveSettings();
              }}
              aria-label="Toggle Performance stats"
            >
              <span class="toggle-knob"></span>
            </button>
          </div>

          <!-- Minimize while playing -->
          <div class="setting-row">
            <div class="setting-info">
              <span class="setting-title">Minimize while playing</span>
              <span class="setting-desc">Tucks this window away when a game starts.</span>
            </div>
            <button
              class="toggle-switch"
              class:on={settings.minimize}
              onclick={() => {
                settings.minimize = !settings.minimize;
                saveSettings();
              }}
              aria-label="Toggle Minimize while playing"
            >
              <span class="toggle-knob"></span>
            </button>
          </div>

          <!-- F9 capture length -->
          <div class="setting-row">
            <div class="setting-info">
              <span class="setting-title">F9 capture length</span>
              <span class="setting-desc">
                F9 in the game records this much graphics input to gsrec_NN.bin, for glitch reports (about 260 MB a
                second).
              </span>
            </div>
            <div class="stepper">
              <button
                class="stepper-btn"
                disabled={settings.captureSeconds <= 1}
                onclick={() => {
                  if (settings.captureSeconds > 1) settings.captureSeconds--;
                  saveSettings();
                }}
              >
                −
              </button>
              <span class="stepper-value">{settings.captureSeconds} seconds</span>
              <button
                class="stepper-btn"
                disabled={settings.captureSeconds >= 10}
                onclick={() => {
                  if (settings.captureSeconds < 10) settings.captureSeconds++;
                  saveSettings();
                }}
              >
                +
              </button>
            </div>
          </div>

          <!-- Start level select -->
          <div class="setting-row">
            <div class="setting-info">
              <span class="setting-title">Start level</span>
              <span class="setting-desc">Skip the intro and start directly at a chosen planet (for testing).</span>
            </div>
            <select
              class="level-select"
              value={settings.startLevel}
              onchange={(e) => {
                settings.startLevel = Number((e.target as HTMLSelectElement).value);
                saveSettings();
              }}
            >
              {#each RAC1_LEVELS as lvl, lIdx (lvl)}
                <option value={lIdx}>{lvl}</option>
              {/each}
            </select>
          </div>
        </div>
      </section>
    {:else}
      <!-- Game Details View -->
      <section class="game-details">
        <!-- Banner (Horizon Direction D) -->
        <div class="banner">
          <div class="banner-content">
            <h1 class="banner-title">{selectedDef.title}</h1>
            <div class="banner-meta">
              <span class="banner-chip">{selectedDef.region}</span>
              <span class="status-dot" style="background-color: {getStatusInfo(selectedDef).color}"></span>
              <span class="banner-status">{getStatusInfo(selectedDef).text}</span>
            </div>
          </div>
        </div>

        <!-- Action Buttons -->
        <div class="actions-bar">
          {#if running}
            <div class="running-track">
              <span class="spinner"></span>
              <span class="running-status">{running.title}</span>
              <button class="cancel-btn" onclick={() => cancel(running)}>Cancel</button>
            </div>
          {:else}
            {@const isPlayReady = selectedDef.playable && built}
            {@const btnLabel = isPlayReady ? "Play" : selectedDef.playable ? "Build" : "Not yet available"}
            {@const btnIcon = isPlayReady ? "/solana/svg/icons/play.svg" : "/solana/svg/icons/build.svg"}
            <button class="primary-btn" disabled={!selectedDef.playable} onclick={handlePlay}>
              {#if selectedDef.playable}
                <img src={btnIcon} alt="" width="18" height="18" />
              {/if}
              <span>{btnLabel}</span>
            </button>
          {/if}

          <button class="action-btn" onclick={() => (viewSettings = true)}>
            <img src="/solana/svg/icons/settings.svg" alt="" width="16" height="16" />
            <span>Settings</span>
          </button>

          <button class="action-btn" onclick={handleOpenLog}>
            <img src="/solana/svg/icons/open_log.svg" alt="" width="16" height="16" />
            <span>Open log</span>
          </button>

          <button class="action-btn" disabled={!matchingVersion?.dir} onclick={handleOpenFolder}>
            <img src="/solana/svg/icons/game_folder.svg" alt="" width="16" height="16" />
            <span>Game folder</span>
          </button>

          {#if matchingGame?.install?.extracted}
            <button class="action-btn remove-btn" onclick={handleRemove}>
              <img src="/solana/svg/icons/remove.svg" alt="" width="16" height="16" />
              <span>Remove</span>
            </button>
          {/if}
        </div>

        <!-- Hint message -->
        <div class="hint-message">
          {#if running}
            <span>Building from your disc. You can keep using your PC; stopping the build keeps downloads.</span>
          {:else if selectedDef.playable && !built}
            <span>First time: downloads tools and source, then builds the game from your disc (10 to 30 minutes).</span>
          {:else if selectedDef.playable}
            <span>Ready to play. Click Play to start native PC execution.</span>
          {:else}
            <span>This game is recognised by Project Solana, but its static recompilation is not completed yet.</span>
          {/if}
        </div>

        <!-- Disc Info Card -->
        <div class="card disc-card">
          <h3 class="card-heading">Disc</h3>
          <div class="card-table">
            <div class="card-row">
              <span class="row-label">Image</span>
              <span class="row-value">{foundDiscInfo?.path ?? "None specified"}</span>
            </div>
            <div class="card-row">
              <span class="row-label">Boot executable</span>
              <span class="row-value">{selectedDef.serial}</span>
            </div>
            <div class="card-row">
              <span class="row-label">MD5</span>
              <span class="row-value">
                {#if selectedDef.md5}
                  {selectedDef.md5} <span class="tag-match">(matches)</span>
                {:else}
                  –
                {/if}
              </span>
            </div>
            <div class="card-row">
              <span class="row-label">Game files</span>
              <span class="row-value">
                {#if built && matchingVersion?.dir}
                  {matchingVersion.dir}
                {:else if selectedDef.playable}
                  Not built yet
                {:else}
                  –
                {/if}
              </span>
            </div>
          </div>
        </div>

        <!-- Controller Card -->
        {#if selectedDef.playable}
          <div class="card controller-card">
            <div class="controller-card-header">
              <h3 class="card-heading">Controller</h3>
              <span class="controller-subtitle">Xbox, DualShock 4 and DualSense, over USB or Bluetooth</span>
            </div>

            <div class="controller-grid">
              {#each CONTROLLER_MAPPINGS as m (m.psName)}
                <div class="controller-row">
                  <div class="glyph-group">
                    <img src={m.psImg} alt="" width="20" height="20" />
                    <span class="glyph-label muted">{m.psName}</span>
                  </div>
                  <span class="arrow-glyph">→</span>
                  <div class="glyph-group">
                    <img src={m.xbImg} alt="" width="20" height="20" />
                    <span class="glyph-label bold">{m.xbName}</span>
                  </div>
                </div>
              {/each}
            </div>
          </div>
        {/if}
      </section>
    {/if}
  </main>
</div>

<style>
  :global(:root) {
    --solana-bg: #0a0e1a;
    --solana-side: #0f1424;
    --solana-card: #151c30;
    --solana-card-hover: #1c2540;
    --solana-border: #26304d;
    --solana-text: #f4f6fb;
    --solana-muted: #9aa6c6;
    --solana-dim: #5e6888;
    --solana-accent: #ff7a3d;
    --solana-accent-hover: #ff9461;
    --solana-accent-text: #0a0e1a;
    --solana-cyan: #7fd1ff;
    --solana-green: #3dd68c;
    --solana-amber: #ffb238;
    --solana-red: #ff5a5f;
  }

  .solana-root {
    display: flex;
    height: 100%;
    width: 100%;
    background-color: var(--solana-bg);
    background-image: url("/solana/png/window_background_tile.png");
    background-repeat: repeat;
    color: var(--solana-text);
    font-family:
      "Segoe UI",
      system-ui,
      -apple-system,
      sans-serif;
    user-select: none;
    position: relative;
    overflow: hidden;
  }

  /* Flash transition effect */
  .solar-flare-enter {
    position: absolute;
    inset: 0;
    pointer-events: none;
    z-index: 100;
    background: radial-gradient(
      circle at 75% 25%,
      rgb(255 122 61 / 0.7) 0%,
      rgb(127 209 255 / 0.25) 45%,
      transparent 75%
    );
    animation: solarFlash 0.5s cubic-bezier(0.16, 1, 0.3, 1) forwards;
  }

  @keyframes solarFlash {
    0% {
      opacity: 0.9;
      transform: scale(0.9);
    }
    100% {
      opacity: 0;
      transform: scale(1.3);
    }
  }

  /* ---- Sidebar ------------------------------------------------ */
  .sidebar {
    width: 320px;
    min-width: 320px;
    background: var(--solana-side);
    border-right: 1px solid var(--solana-border);
    display: flex;
    flex-direction: column;
    height: 100%;
    box-sizing: border-box;
    z-index: 2;
  }

  .wordmark-row {
    height: 64px;
    padding: 0 18px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-shrink: 0;
  }

  .wordmark {
    height: 30px;
    width: auto;
    object-fit: contain;
  }

  .secret-toggle-btn {
    width: 32px;
    height: 32px;
    padding: 0;
    margin: 0;
    border-radius: 8px;
    border: 1px solid var(--solana-border);
    background: var(--solana-card);
    display: inline-flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    transition: all 0.2s ease;
    flex-shrink: 0;
  }

  .secret-toggle-btn img {
    width: 20px;
    height: 20px;
    display: block;
    object-fit: contain;
    pointer-events: none;
  }

  .secret-toggle-btn:hover {
    background: var(--solana-card-hover);
    border-color: #ffb443;
    transform: scale(1.1);
    box-shadow: 0 0 10px rgb(255 180 67 / 0.5);
  }

  .library-header {
    padding: 0 20px 8px;
    font-size: 11px;
    font-weight: 700;
    letter-spacing: 0.15em;
    color: var(--solana-dim);
    text-transform: uppercase;
  }

  .discs-list {
    flex: 1;
    overflow-y: auto;
    padding: 0 12px 12px;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  .disc-item {
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: stretch;
    justify-content: center;
    width: 100%;
    min-height: 64px;
    border: 1px solid transparent;
    background: var(--solana-side);
    border-radius: 10px;
    padding: 10px 14px 10px 18px;
    text-align: left;
    color: inherit;
    cursor: pointer;
    gap: 4px;
    box-sizing: border-box;
    transition: all 0.15s ease;
  }

  .disc-item:hover {
    background: var(--solana-card-hover);
  }

  .disc-item.selected {
    background: var(--solana-card-hover);
    border-color: var(--solana-border);
  }

  .selection-bar {
    position: absolute;
    left: 0;
    top: 14px;
    bottom: 14px;
    width: 3px;
    background: var(--solana-accent);
    border-radius: 0 2px 2px 0;
  }

  .item-title {
    font-size: 14px;
    font-weight: 600;
    color: var(--solana-text);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    line-height: 1.3;
    width: 100%;
  }

  .item-meta {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 12px;
    color: var(--solana-muted);
    line-height: 1.2;
    width: 100%;
  }

  .region-badge {
    font-size: 11px;
    color: var(--solana-muted);
    white-space: nowrap;
    flex-shrink: 0;
  }

  .status-dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    flex-shrink: 0;
  }

  .status-text {
    font-size: 12px;
    color: var(--solana-muted);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    flex: 1;
    min-width: 0;
  }

  .sidebar-bottom {
    padding: 14px 16px;
    border-top: 1px solid var(--solana-border);
    display: flex;
    flex-direction: column;
    gap: 8px;
    flex-shrink: 0;
  }

  .bottom-btn {
    height: 38px;
    background: var(--solana-card);
    border: 1px solid var(--solana-border);
    border-radius: 6px;
    color: var(--solana-text);
    font-size: 13px;
    font-weight: 600;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .bottom-btn:hover {
    background: var(--solana-card-hover);
    border-color: rgb(52 64 102);
  }

  /* ---- Main Content -------------------------------------------- */
  .main-content {
    flex: 1;
    min-width: 0;
    height: 100%;
    overflow-y: auto;
    padding: 24px 32px;
    box-sizing: border-box;
  }

  .game-details {
    max-width: 960px;
    margin: 0 auto;
    display: flex;
    flex-direction: column;
  }

  /* Banner */
  .banner {
    height: 150px;
    border-radius: 10px;
    background-image: url("/solana/png/default_banner.png");
    background-size: cover;
    background-position: right center;
    display: flex;
    align-items: center;
    padding: 0 28px;
    box-shadow: inset 0 0 40px rgb(0 0 0 / 0.5);
    position: relative;
    overflow: hidden;
  }

  .banner-content {
    z-index: 1;
  }

  .banner-title {
    font-size: 28px;
    font-weight: 700;
    color: var(--solana-text);
    margin: 0 0 8px;
    letter-spacing: -0.01em;
  }

  .banner-meta {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .banner-chip {
    background: var(--solana-card);
    border: 1px solid var(--solana-border);
    border-radius: 20px;
    padding: 2px 10px;
    font-size: 12px;
    font-weight: 600;
    color: var(--solana-text);
  }

  .banner-status {
    font-size: 13px;
    color: var(--solana-text);
  }

  /* Actions Bar */
  .actions-bar {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-top: 24px;
    flex-wrap: wrap;
  }

  .primary-btn {
    height: 44px;
    min-width: 160px;
    background: var(--solana-accent);
    color: var(--solana-accent-text);
    border: none;
    border-radius: 6px;
    font-size: 15px;
    font-weight: 700;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    cursor: pointer;
    transition: all 0.15s ease;
    padding: 0 24px;
  }

  .primary-btn:hover:not(:disabled) {
    background: var(--solana-accent-hover);
  }

  .primary-btn:disabled {
    background: rgb(90 58 44);
    color: rgb(154 126 112);
    cursor: not-allowed;
  }

  .action-btn {
    height: 40px;
    background: var(--solana-card);
    border: 1px solid var(--solana-border);
    border-radius: 6px;
    color: var(--solana-text);
    font-size: 13px;
    font-weight: 600;
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 0 16px;
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .action-btn:hover:not(:disabled) {
    background: var(--solana-card-hover);
    border-color: rgb(52 64 102);
  }

  .action-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .remove-btn {
    margin-left: auto;
  }

  .running-track {
    flex: 1;
    height: 44px;
    background: var(--solana-card);
    border: 1px solid var(--solana-border);
    border-radius: 6px;
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 0 16px;
  }

  .running-status {
    flex: 1;
    font-size: 13px;
    font-weight: 600;
    color: var(--solana-text);
  }

  .cancel-btn {
    background: transparent;
    border: 1px solid var(--solana-border);
    border-radius: 4px;
    color: var(--solana-muted);
    font-size: 12px;
    padding: 4px 10px;
    cursor: pointer;
  }

  .cancel-btn:hover {
    color: var(--solana-red);
    border-color: var(--solana-red);
  }

  .hint-message {
    font-size: 13px;
    color: var(--solana-muted);
    margin: 16px 0 24px;
    min-height: 20px;
  }

  /* Cards */
  .card {
    background: var(--solana-card);
    border: 1px solid var(--solana-border);
    border-radius: 10px;
    padding: 20px 24px;
    margin-bottom: 16px;
  }

  .card-heading {
    font-size: 16px;
    font-weight: 700;
    color: var(--solana-text);
    margin: 0 0 16px;
  }

  .card-table {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  .card-row {
    display: flex;
    font-size: 13px;
  }

  .row-label {
    width: 160px;
    flex-shrink: 0;
    color: var(--solana-muted);
  }

  .row-value {
    color: var(--solana-text);
    word-break: break-all;
  }

  .tag-match {
    color: var(--solana-green);
    margin-left: 6px;
    font-size: 12px;
  }

  /* Controller Card */
  .controller-card-header {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    margin-bottom: 16px;
  }

  .controller-subtitle {
    font-size: 12px;
    color: var(--solana-muted);
  }

  .controller-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 12px 32px;
  }

  .controller-row {
    display: flex;
    align-items: center;
    gap: 14px;
    font-size: 13px;
  }

  .glyph-group {
    display: flex;
    align-items: center;
    gap: 8px;
    width: 90px;
  }

  .glyph-label.muted {
    color: var(--solana-muted);
  }

  .glyph-label.bold {
    color: var(--solana-text);
    font-weight: 700;
  }

  .arrow-glyph {
    color: var(--solana-dim);
    font-size: 14px;
  }

  /* ---- Settings View ------------------------------------------- */
  .settings-view {
    max-width: 800px;
    margin: 0 auto;
  }

  .settings-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 4px;
  }

  .settings-header h2 {
    font-size: 26px;
    font-weight: 700;
    margin: 0;
    color: var(--solana-text);
  }

  .done-btn {
    height: 36px;
    padding: 0 20px;
    background: var(--solana-card);
    border: 1px solid var(--solana-border);
    border-radius: 6px;
    color: var(--solana-text);
    font-size: 13px;
    font-weight: 600;
    cursor: pointer;
  }

  .done-btn:hover {
    background: var(--solana-card-hover);
  }

  .settings-hint {
    font-size: 13px;
    color: var(--solana-muted);
    margin: 0 0 24px;
  }

  .settings-list {
    display: flex;
    flex-direction: column;
    gap: 14px;
  }

  .setting-row {
    background: var(--solana-card);
    border: 1px solid var(--solana-border);
    border-radius: 10px;
    padding: 16px 20px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 20px;
  }

  .setting-info {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  .setting-title {
    font-size: 14px;
    font-weight: 600;
    color: var(--solana-text);
  }

  .setting-desc {
    font-size: 12px;
    color: var(--solana-muted);
    max-width: 500px;
  }

  /* Segmented Control */
  .segmented-control {
    display: flex;
    background: var(--solana-side);
    border: 1px solid var(--solana-border);
    border-radius: 6px;
    padding: 3px;
    gap: 3px;
  }

  .segment {
    padding: 6px 14px;
    border-radius: 4px;
    border: none;
    background: transparent;
    color: var(--solana-muted);
    font-size: 13px;
    font-weight: 600;
    cursor: pointer;
  }

  .segment.active {
    background: var(--solana-accent);
    color: var(--solana-accent-text);
  }

  /* Toggle Switch */
  .toggle-switch {
    width: 44px;
    height: 24px;
    border-radius: 12px;
    background: var(--solana-side);
    border: 1px solid var(--solana-border);
    position: relative;
    cursor: pointer;
    padding: 0;
    transition: all 0.2s ease;
  }

  .toggle-switch.on {
    background: var(--solana-accent);
    border-color: var(--solana-accent);
  }

  .toggle-knob {
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: #fff;
    position: absolute;
    top: 2px;
    left: 2px;
    transition: transform 0.2s ease;
  }

  .toggle-switch.on .toggle-knob {
    transform: translateX(20px);
  }

  /* Stepper */
  .stepper {
    display: flex;
    align-items: center;
    background: var(--solana-side);
    border: 1px solid var(--solana-border);
    border-radius: 6px;
    height: 36px;
  }

  .stepper-btn {
    width: 32px;
    height: 100%;
    border: none;
    background: transparent;
    color: var(--solana-text);
    font-size: 16px;
    cursor: pointer;
  }

  .stepper-btn:hover:not(:disabled) {
    background: var(--solana-card-hover);
  }

  .stepper-btn:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }

  .stepper-value {
    padding: 0 12px;
    font-size: 13px;
    font-weight: 600;
    color: var(--solana-text);
    min-width: 80px;
    text-align: center;
  }

  .level-select {
    background: var(--solana-side);
    border: 1px solid var(--solana-border);
    color: var(--solana-text);
    border-radius: 6px;
    padding: 8px 12px;
    font-size: 13px;
    font-weight: 600;
    outline: none;
    cursor: pointer;
  }
</style>
