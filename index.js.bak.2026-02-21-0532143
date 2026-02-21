#!/usr/bin/env node
'use strict';

import blessed from 'blessed';
import fs from 'fs';
import path from 'path';
import { execFile, execFileSync } from 'child_process';
import os from 'os';
import { AgentChatClient } from '@tjamescouch/agentchat';

// --- CLI Args ---
const args = process.argv.slice(2);
function getArg(name, fallback) {
  const idx = args.indexOf(`--${name}`);
  if (idx >= 0 && idx + 1 < args.length) return args[idx + 1];
  return fallback;
}

// --- Config ---
const AGENTS_DIR = path.join(os.homedir(), '.agentchat', 'agents');
const IDENTITIES_DIR = path.join(os.homedir(), '.agentchat', 'identities');
const POLL_INTERVAL = 3000;
const LOG_TAIL_LINES = 100;
const DEBOUNCE_MS = 100;
const AGENTCHAT_PUBLIC = process.env.AGENTCHAT_PUBLIC === 'true';
// AgentTUI is an admin tool; default to local-only connectivity.
// Allowing remote servers requires an explicit opt-in.
const AGENTTUI_ALLOW_REMOTE =
  args.includes('--allow-remote') || (process.env.AGENTTUI_ALLOW_REMOTE === '1' || process.env.AGENTTUI_ALLOW_REMOTE === 'true');
const CHAT_SERVER = (() => {
  const explicit = getArg('server', process.env.AGENTCHAT_URL);
  if (explicit) {
    const parsed = new URL(explicit);
    const isLocal = parsed.hostname === 'localhost' || parsed.hostname === '127.0.0.1' || parsed.hostname === '::1';
    if (!isLocal && !(AGENTCHAT_PUBLIC || AGENTTUI_ALLOW_REMOTE)) {
      console.error(`ERROR: server points to remote host "${parsed.hostname}" but remote access is not enabled.`);
      console.error("Set AGENTTUI_ALLOW_REMOTE=1 (or pass --allow-remote). (AGENTCHAT_PUBLIC=true also allows remote servers.)");
      process.exit(1);
    }
    return explicit;
  }
  return AGENTCHAT_PUBLIC ? 'wss://agentchat-server.fly.dev' : 'ws://127.0.0.1:6667';
})();
const CHAT_NAME = getArg('name', process.env.AGENTCHAT_NAME || 'tui');
const IDENTITY_PATH = (() => {
  const explicit = getArg('identity', null);
  if (explicit) return explicit;
  // If a name was provided and a matching identity file exists, use it
  const name = getArg('name', process.env.AGENTCHAT_NAME || null);
  if (name) {
    const identityFile = path.join(IDENTITIES_DIR, `${name}.json`);
    if (fs.existsSync(identityFile)) return identityFile;
  }
  return null;
})();
const DEFAULT_CHANNELS = ['#general', '#engineering'];
const RECONNECT_DELAY = 5000;
const MAX_MSG_LEN = 4096;

// --- Find agentctl.sh ---
function findAgentctl() {
  // Check explicit env var first
  if (process.env.AGENTCTL_PATH && fs.existsSync(process.env.AGENTCTL_PATH)) {
    return process.env.AGENTCTL_PATH;
  }

  // Check common locations
  const candidates = [
    path.join(os.homedir(), 'dev/claude/agentchat/lib/supervisor/agentctl.sh'),
    path.join(os.homedir(), 'dev/claude/projects/agent5/agentchat/lib/supervisor/agentctl.sh'),
    path.join(os.homedir(), '.agentchat/agentctl.sh'),
  ];
  for (const p of candidates) {
    if (fs.existsSync(p)) return p;
  }

  // Try PATH lookup
  try {
    const result = execFileSync('which', ['agentctl.sh'], { encoding: 'utf8', timeout: 3000 }).trim();
    if (result && fs.existsSync(result)) return result;
  } catch {}

  return null;
}

const AGENTCTL = findAgentctl();

// --- Agent State ---

function resolveStatus(agentDir) {
  const pidFile = path.join(agentDir, 'supervisor.pid');
  const stopFile = path.join(agentDir, 'stop');
  const stateFile = path.join(agentDir, 'state.json');

  if (fs.existsSync(stopFile)) return 'stopping';

  if (fs.existsSync(pidFile)) {
    try {
      const pid = parseInt(fs.readFileSync(pidFile, 'utf8').trim(), 10);
      if (pid && !isNaN(pid)) {
        try {
          process.kill(pid, 0);
          return 'running';
        } catch {
          // PID is stale (process doesn't exist) — clean up
          try { fs.unlinkSync(pidFile); } catch {}
          return 'stopped';
        }
      }
    } catch {}
    try { fs.unlinkSync(pidFile); } catch {}
    return 'stopped';
  }

  if (fs.existsSync(stateFile)) {
    try {
      const state = JSON.parse(fs.readFileSync(stateFile, 'utf8'));
      if (state.status === 'stopped') return 'stopped';
    } catch {}
  }

  return 'stopped';
}

function readAgent(name) {
  const dir = path.join(AGENTS_DIR, name);
  const status = resolveStatus(dir);

  let pid = null;
  const pidFile = path.join(dir, 'supervisor.pid');
  if (fs.existsSync(pidFile)) {
    try {
      pid = parseInt(fs.readFileSync(pidFile, 'utf8').trim(), 10);
      if (isNaN(pid)) pid = null;
    } catch {}
  }

  let mission = '';
  const missionFile = path.join(dir, 'mission.txt');
  if (fs.existsSync(missionFile)) {
    try { mission = fs.readFileSync(missionFile, 'utf8').trim(); } catch {}
  }

  let uptime = null;
  if (status === 'running' && pid) {
    try {
      const stat = fs.statSync(pidFile);
      uptime = Date.now() - stat.mtimeMs;
    } catch {}
  }

  return { name, dir, status, pid, mission, uptime };
}

function scanAgents() {
  if (!fs.existsSync(AGENTS_DIR)) return [];
  try {
    return fs.readdirSync(AGENTS_DIR)
      .filter(name => {
        try { return fs.statSync(path.join(AGENTS_DIR, name)).isDirectory(); } catch { return false; }
      })
      .map(readAgent)
      .sort((a, b) => a.name.localeCompare(b.name));
  } catch {
    return [];
  }
}

function formatUptime(ms) {
  if (!ms || ms < 0) return '-';
  const s = Math.floor(ms / 1000);
  const m = Math.floor(s / 60);
  const h = Math.floor(m / 60);
  const d = Math.floor(h / 24);
  if (d > 0) return `${d}d ${h % 24}h`;
  if (h > 0) return `${h}h ${m % 60}m`;
  if (m > 0) return `${m}m ${s % 60}s`;
  return `${s}s`;
}

// --- Log Streaming ---

function createLogStreamer() {
  let watcher = null;
  let filePos = 0;
  let debounceTimer = null;

  function tail(filePath, nLines) {
    try {
      const content = fs.readFileSync(filePath, 'utf8');
      const lines = content.split('\n');
      filePos = Buffer.byteLength(content, 'utf8');
      return lines.slice(-nLines).filter(Boolean);
    } catch {
      return [];
    }
  }

  function readNew(filePath, onLines) {
    try {
      const stat = fs.statSync(filePath);
      if (stat.size < filePos) filePos = 0;
      if (stat.size > filePos) {
        const fd = fs.openSync(filePath, 'r');
        const buf = Buffer.alloc(stat.size - filePos);
        fs.readSync(fd, buf, 0, buf.length, filePos);
        fs.closeSync(fd);
        filePos = stat.size;
        const newLines = buf.toString('utf8').split('\n').filter(Boolean);
        if (newLines.length > 0) onLines(newLines);
      }
    } catch {}
  }

  function start(logFile, onLines) {
    stop();
    if (!fs.existsSync(logFile)) {
      onLines(['{grey-fg}No logs yet. Watching...{/grey-fg}']);
      const parentDir = path.dirname(logFile);
      if (fs.existsSync(parentDir)) {
        try {
          watcher = fs.watch(parentDir, (event, filename) => {
            if (filename === path.basename(logFile) && fs.existsSync(logFile)) {
              start(logFile, onLines);
            }
          });
        } catch {}
      }
      return;
    }

    const initialLines = tail(logFile, LOG_TAIL_LINES);
    if (initialLines.length === 0) {
      onLines(['{grey-fg}Waiting for output...{/grey-fg}']);
    } else {
      onLines(initialLines);
    }

    try {
      watcher = fs.watch(logFile, () => {
        if (debounceTimer) clearTimeout(debounceTimer);
        debounceTimer = setTimeout(() => readNew(logFile, onLines), DEBOUNCE_MS);
      });
    } catch {}
  }

  function stop() {
    if (watcher) { try { watcher.close(); } catch {} watcher = null; }
    if (debounceTimer) { clearTimeout(debounceTimer); debounceTimer = null; }
    filePos = 0;
  }

  return { start, stop };
}

// --- Chat Client (using @tjamescouch/agentchat) ---

function createChatClient(onMessage, onStatus) {
  let client = null;
  let channels = new Set(DEFAULT_CHANNELS);
  let activeChannel = DEFAULT_CHANNELS[0];
  let reconnectTimer = null;
  let intentionalClose = false;
  let agentId = null;

  async function connect() {
    if (client && client.connected) return;
    intentionalClose = false;
    onStatus('connecting');

    try {
      const options = {
        server: CHAT_SERVER,
        name: CHAT_NAME,
      };

      if (IDENTITY_PATH) {
        options.identity = IDENTITY_PATH;
      }

      // Clean up old client before creating new one
      if (client) {
        try { client.removeAllListeners(); client.disconnect(); } catch {}
        client = null;
      }

      client = new AgentChatClient(options);

      // Wire up event handlers BEFORE connect to avoid race conditions
      client.on('message', (msg) => {
        if (msg.from === agentId) return; // skip own messages
        const displayName = escapeTags(msg.from_name || msg.name || msg.from || '?');
        const channelLabel = msg.to?.startsWith('#') ? msg.to : 'DM';
        onMessage({
          type: 'msg',
          from: displayName,
          channel: channelLabel,
          content: escapeTags(msg.content),
          isSelf: false,
        });
      });

      client.on('agent_joined', (msg) => {
        const ch = msg.channel;
        if (channels.has(ch)) {
          onMessage({ type: 'system', text: `${msg.name || msg.agent} joined ${ch}` });
        }
      });

      client.on('agent_left', (msg) => {
        const ch = msg.channel;
        if (channels.has(ch)) {
          onMessage({ type: 'system', text: `${msg.name || msg.agent} left ${ch}` });
        }
      });

      client.on('joined', (msg) => {
        onMessage({ type: 'system', text: `Joined ${msg.channel}` });
      });

      client.on('left', (msg) => {
        onMessage({ type: 'system', text: `Left ${msg.channel}` });
      });

      client.on('error', (msg) => {
        const text = msg.message || 'Unknown error';
        // Suppress noisy channel join errors (channel not found, already joined, etc.)
        const suppress = /channel.*not found|not a member|already/i.test(text);
        if (!suppress) {
          onMessage({ type: 'error', text });
        }
      });

      client.on('disconnect', () => {
        onStatus('disconnected');
        if (!intentionalClose) scheduleReconnect();
      });

      await client.connect();
      agentId = client.agentId;
      onStatus('connected');

      // Join default channels
      for (const ch of channels) {
        try {
          await client.join(ch);
        } catch {}
      }

    } catch (err) {
      onStatus('error');
      const hint = CHAT_SERVER.includes('localhost') || CHAT_SERVER.includes('127.0.0.1')
        ? ` (connecting to ${CHAT_SERVER} — did you mean to set AGENTCHAT_PUBLIC=true?)`
        : '';
      onMessage({ type: 'error', text: `Connection failed: ${err.message}${hint}` });
      scheduleReconnect();
    }
  }

  async function joinChannel(ch) {
    channels.add(ch);
    if (client && client.connected) {
      try { await client.join(ch); } catch {}
    }
  }

  async function leaveChannel(ch) {
    channels.delete(ch);
    if (client && client.connected) {
      try { await client.leave(ch); } catch {}
    }
  }

  function switchChannel(newChannel) {
    activeChannel = newChannel;
    if (!channels.has(newChannel)) {
      joinChannel(newChannel);
    }
  }

  function send(text) {
    if (!text || !text.trim()) return;
    if (text.length > MAX_MSG_LEN) {
      text = text.slice(0, MAX_MSG_LEN);
      onMessage({ type: 'system', text: `Message truncated to ${MAX_MSG_LEN} chars` });
    }
    if (client && client.connected) {
      client.send(activeChannel, text.trim());
      // Show own message locally (server may or may not echo)
      onMessage({
        type: 'msg',
        from: CHAT_NAME,
        channel: activeChannel,
        content: text.trim(),
        isSelf: true,
      });
    } else {
      onMessage({ type: 'error', text: 'Not connected' });
    }
  }

  function dm(target, text) {
    if (!text || !text.trim()) return;
    if (text.length > MAX_MSG_LEN) {
      text = text.slice(0, MAX_MSG_LEN);
      onMessage({ type: 'system', text: `Message truncated to ${MAX_MSG_LEN} chars` });
    }
    if (client && client.connected) {
      client.send(target, text.trim());
      onMessage({
        type: 'msg',
        from: CHAT_NAME,
        channel: 'DM',
        content: `→ ${target}: ${text.trim()}`,
        isSelf: true,
      });
    } else {
      onMessage({ type: 'error', text: 'Not connected' });
    }
  }

  function scheduleReconnect() {
    if (reconnectTimer) return;
    onMessage({ type: 'system', text: `Reconnecting in ${RECONNECT_DELAY / 1000}s...` });
    reconnectTimer = setTimeout(() => {
      reconnectTimer = null;
      connect();
    }, RECONNECT_DELAY);
  }

  function disconnect() {
    intentionalClose = true;
    if (reconnectTimer) { clearTimeout(reconnectTimer); reconnectTimer = null; }
    if (client) { try { client.disconnect(); } catch {} client = null; }
  }

  function getChannel() { return activeChannel; }
  function getChannels() { return [...channels]; }
  function isConnected() { return client && client.connected; }
  function getAgentId() { return agentId; }

  return { connect, disconnect, send, dm, switchChannel, joinChannel, leaveChannel, getChannel, getChannels, isConnected, getAgentId };
}

// --- Sanitize blessed tags in untrusted content ---

function escapeTags(str) {
  if (!str) return str;
  return str.replace(/\{/g, '\\{').replace(/\}/g, '\\}');
}

// --- Name Color ---

function nameColor(name) {
  const colors = ['red', 'green', 'yellow', 'blue', 'magenta', 'cyan', 'white'];
  let hash = 0;
  for (let i = 0; i < name.length; i++) {
    hash = ((hash << 5) - hash + name.charCodeAt(i)) | 0;
  }
  return colors[Math.abs(hash) % colors.length];
}

// --- Channel Color ---

function channelColor(channel) {
  const colors = ['cyan', 'magenta', 'yellow', 'green', 'blue'];
  let hash = 0;
  for (let i = 0; i < channel.length; i++) {
    hash = ((hash << 5) - hash + channel.charCodeAt(i)) | 0;
  }
  return colors[Math.abs(hash) % colors.length];
}

// --- TUI ---

function createUI() {
  const screen = blessed.screen({
    smartCSR: true,
    title: 'agentctl',
    fullUnicode: true,
  });

  // === Left column: agents + detail ===
  const leftCol = blessed.box({
    parent: screen,
    top: 0,
    left: 0,
    width: '25%',
    height: '100%-1',
  });

  const agentList = blessed.list({
    parent: leftCol,
    label: ' agents ',
    top: 0,
    left: 0,
    width: '100%',
    height: '55%',
    border: { type: 'line' },
    style: {
      border: { fg: 'cyan' },
      selected: { bg: 'blue', fg: 'white', bold: true },
      item: { fg: 'white' },
      label: { fg: 'cyan', bold: true },
    },
    keys: true,
    vi: true,
    mouse: true,
    scrollable: true,
    tags: true,
  });

  const detailBox = blessed.box({
    parent: leftCol,
    label: ' detail ',
    top: '55%',
    left: 0,
    width: '100%',
    height: '45%',
    border: { type: 'line' },
    style: {
      border: { fg: 'cyan' },
      label: { fg: 'cyan', bold: true },
    },
    tags: true,
    scrollable: true,
  });

  // === Center column: logs ===
  const logBox = blessed.log({
    parent: screen,
    label: ' logs ',
    top: 0,
    left: '25%',
    width: '40%',
    height: '100%-1',
    border: { type: 'line' },
    style: {
      border: { fg: 'green' },
      label: { fg: 'green', bold: true },
    },
    tags: true,
    scrollable: true,
    scrollback: 1000,
    mouse: true,
    keys: true,
  });

  // === Right column: chat ===
  const chatCol = blessed.box({
    parent: screen,
    top: 0,
    left: '65%',
    width: '35%',
    height: '100%-1',
  });

  const chatLog = blessed.log({
    parent: chatCol,
    label: ` ${DEFAULT_CHANNELS[0]} `,
    top: 0,
    left: 0,
    width: '100%',
    height: '100%-3',
    border: { type: 'line' },
    style: {
      border: { fg: 'magenta' },
      label: { fg: 'magenta', bold: true },
    },
    tags: true,
    scrollable: true,
    scrollback: 500,
    mouse: true,
    keys: true,
  });

  const chatInput = blessed.textbox({
    parent: chatCol,
    bottom: 0,
    left: 0,
    width: '100%',
    height: 3,
    border: { type: 'line' },
    style: {
      border: { fg: 'magenta' },
      fg: 'white',
    },
    keys: true,
    mouse: true,
  });

  // === Status bar ===
  const statusBar = blessed.box({
    parent: screen,
    bottom: 0,
    left: 0,
    width: '100%',
    height: 1,
    style: { bg: 'blue', fg: 'white' },
    tags: true,
  });

  function setStatus(msg) {
    statusBar.setContent(` ${msg}`);
    screen.render();
  }

  setStatus('[s]tart [x]stop [r]estart [K]ill [c]ontext [/]filter [tab]focus [M]copy [q]uit');

  return { screen, agentList, detailBox, logBox, chatLog, chatInput, statusBar, setStatus };
}

// --- Main ---

function main() {
  const ui = createUI();
  const logStreamer = createLogStreamer();

  let agents = [];
  let selectedIdx = 0;
  let filterText = '';
  let filteredAgents = [];
  let confirmAction = null;
  let userScrolledLog = false;
  let chatFocused = false;
  let focusPanel = 'agents'; // agents | logs | chat

  // --- Chat ---
  let chatStatus = 'disconnected';

  const chat = createChatClient(
    (msg) => {
      switch (msg.type) {
        case 'msg': {
          const color = msg.isSelf ? 'cyan' : nameColor(msg.from);
          const chColor = channelColor(msg.channel || '#general');
          const chPrefix = msg.channel && msg.channel !== chat.getChannel()
            ? `{${chColor}-fg}[${msg.channel}]{/${chColor}-fg} `
            : '';
          ui.chatLog.log(`${chPrefix}{${color}-fg}${msg.from}{/${color}-fg}: ${msg.content}`);
          break;
        }
        case 'system':
          ui.chatLog.log(`{grey-fg}--- ${msg.text}{/grey-fg}`);
          break;
        case 'error':
          ui.chatLog.log(`{red-fg}! ${msg.text}{/red-fg}`);
          break;
      }
      ui.screen.render();
    },
    (status) => {
      chatStatus = status;
      updateStatusBar();
      ui.screen.render();
    }
  );

  function updateStatusBar() {
    const chatIndicator = chatStatus === 'connected'
      ? `{green-fg}●{/green-fg} ${chat.getChannel()}`
      : chatStatus === 'connecting'
        ? `{yellow-fg}◐{/yellow-fg} connecting`
        : `{red-fg}○{/red-fg} disconnected`;

    const identity = IDENTITY_PATH ? `{green-fg}${CHAT_NAME}{/green-fg}` : `{grey-fg}${CHAT_NAME}{/grey-fg}`;
    const focusIndicator = `[${focusPanel}]`;

    ui.setStatus(
      `[s]tart [x]stop [r]estart [K]ill [c]ontext [/]filter [tab]focus [q]uit  ${focusIndicator}  ${identity}  chat: ${chatIndicator}`
    );
  }

  // --- Agent helpers ---

  function getSelected() {
    return filteredAgents[selectedIdx] || null;
  }

  function statusIcon(status) {
    switch (status) {
      case 'running': return '{green-fg}●{/green-fg}';
      case 'stopping': return '{yellow-fg}◐{/yellow-fg}';
      case 'stopped': return '{grey-fg}○{/grey-fg}';
      case 'dead': return '{red-fg}✗{/red-fg}';
      default: return '{grey-fg}?{/grey-fg}';
    }
  }

  function statusColor(status) {
    switch (status) {
      case 'running': return 'green';
      case 'stopping': return 'yellow';
      case 'stopped': return 'grey';
      case 'dead': return 'red';
      default: return 'grey';
    }
  }

function renderAgentList() {
    // If no filter, show all agents.
    if (!filterText) {
      filteredAgents = agents.slice();
    } else {
      // Use a lightweight fuzzy matching algorithm to rank results.
      const q = filterText.toLowerCase();
      function fuzzyScore(needle, hay) {
        // Simple sequential fuzzy match. Returns score (higher is better) or 0 if no match.
        needle = needle.toLowerCase();
        hay = hay.toLowerCase();
        let n = 0;
        let lastIdx = -1;
        for (let i = 0; i < needle.length; i++) {
          const ch = needle[i];
          const idx = hay.indexOf(ch, lastIdx + 1);
          if (idx === -1) return 0;
          // reward for consecutive matches
          if (idx === lastIdx + 1) n += 5;
          else n += 1;
          lastIdx = idx;
        }
        // shorter haystack preferred
        n += Math.max(0, 10 - hay.length);
        return n;
      }

      const withScores = agents.map(a => ({
        a,
        score: Math.max(fuzzyScore(q, a.name), fuzzyScore(q, a.dir || ''))
      })).filter(x => x.score > 0);

      withScores.sort((x, y) => y.score - x.score || x.a.name.localeCompare(y.a.name));
      filteredAgents = withScores.map(x => x.a);
    }

    const items = filteredAgents.map(a => {
      const icon = statusIcon(a.status);
      const padded = a.name.padEnd(12);
      return `${icon} ${padded} {${statusColor(a.status)}-fg}${a.status}{/${statusColor(a.status)}-fg}`;
    });

    ui.agentList.setItems(items);
    if (selectedIdx >= filteredAgents.length) selectedIdx = Math.max(0, filteredAgents.length - 1);
    ui.agentList.select(selectedIdx);

    if (filterText) {
      ui.agentList.setLabel(` agents [/${filterText}] `);
    } else {
      ui.agentList.setLabel(` agents (${filteredAgents.length}) `);
    }
  }

  function renderDetail() {
    const agent = getSelected();
    if (!agent) {
      ui.detailBox.setContent('{grey-fg}No agent selected{/grey-fg}');
      return;
    }

    const lines = [
      `{bold}Name:{/bold}    ${agent.name}`,
      `{bold}Status:{/bold}  {${statusColor(agent.status)}-fg}${agent.status}{/${statusColor(agent.status)}-fg}`,
      `{bold}PID:{/bold}     ${agent.pid || '-'}`,
      `{bold}Uptime:{/bold}  ${formatUptime(agent.uptime)}`,
      '',
      `{bold}Mission:{/bold}`,
      `  ${agent.mission || '{grey-fg}(none){/grey-fg}'}`,
    ];

    ui.detailBox.setContent(lines.join('\n'));
  }

  function switchLogStream() {
    const agent = getSelected();
    userScrolledLog = false;
    ui.logBox.setContent('');

    if (!agent) {
      ui.logBox.setLabel(' logs ');
      logStreamer.stop();
      return;
    }

    ui.logBox.setLabel(` logs: ${agent.name} `);
    const logFile = path.join(agent.dir, 'supervisor.log');

    logStreamer.start(logFile, (lines) => {
      for (const line of lines) {
        ui.logBox.log(line);
      }
      if (!userScrolledLog) {
        ui.logBox.setScrollPerc(100);
      }
      ui.screen.render();
    });
  }

  function refresh() {
    agents = scanAgents();
    const prevSelected = getSelected()?.name;
    renderAgentList();

    if (prevSelected) {
      const idx = filteredAgents.findIndex(a => a.name === prevSelected);
      if (idx >= 0) {
        selectedIdx = idx;
        ui.agentList.select(selectedIdx);
      }
    }

    renderDetail();
    ui.screen.render();
  }

  function runAgentctl(command, agentName, extra, callback) {
    if (!AGENTCTL) {
      ui.setStatus('{red-fg}agentctl.sh not found{/red-fg}');
      return;
    }

    const args = [command, agentName];
    if (extra) args.push(extra);

    ui.logBox.log(`{yellow-fg}> agentctl ${args.join(' ')}{/yellow-fg}`);

    execFile('bash', [AGENTCTL, ...args], { timeout: 30000 }, (err, stdout, stderr) => {
      const output = (stdout || '') + (stderr || '');
      const lines = output.trim().split('\n').filter(Boolean);
      for (const line of lines) {
        ui.logBox.log(`{yellow-fg}> ${line}{/yellow-fg}`);
      }
      if (err && !stdout && !stderr) {
        ui.logBox.log(`{red-fg}> Error: ${err.message}{/red-fg}`);
      }
      updateStatusBar();
      ui.screen.render();
      setTimeout(refresh, 1000);
      if (callback) callback(err);
    });
  }

  // --- Focus management ---

  let chatReading = false;

  function setFocus(panel) {
    focusPanel = panel;
    chatFocused = panel === 'chat';

    // Reset all borders to default
    ui.agentList.style.border.fg = 'cyan';
    ui.detailBox.style.border.fg = 'cyan';
    ui.logBox.style.border.fg = 'green';
    ui.chatLog.style.border.fg = 'magenta';

    switch (panel) {
      case 'agents':
        ui.agentList.style.border.fg = 'white';
        ui.agentList.focus();
        break;
      case 'logs':
        ui.logBox.style.border.fg = 'white';
        ui.logBox.focus();
        break;
      case 'chat':
        ui.chatLog.style.border.fg = 'white';
        if (!chatReading) {
          chatReading = true;
          ui.chatInput.readInput(() => { chatReading = false; });
        }
        break;
    }

    updateStatusBar();
    ui.screen.render();
  }

  // --- Key bindings ---

  ui.agentList.on('select item', (item, idx) => {
    if (idx !== selectedIdx) {
      selectedIdx = idx;
      renderDetail();
      switchLogStream();
      ui.screen.render();
    }
  });

  // Chat input handling
  function rearmChatInput() {
    chatReading = true;
    process.nextTick(() => {
      ui.chatInput.readInput(() => { chatReading = false; });
      ui.screen.render();
    });
  }

  ui.chatInput.on('submit', (value) => {
    if (value && value.trim()) {
      const text = value.trim();

      // Handle /join command
      const joinMatch = text.match(/^\/join\s+(#\S+)/);
      if (joinMatch) {
        const newChannel = joinMatch[1];
        chat.switchChannel(newChannel);
        ui.chatLog.setLabel(` ${newChannel} `);
      } else if (text.startsWith('/leave ')) {
        const leaveMatch = text.match(/^\/leave\s+(#\S+)/);
        if (!leaveMatch) return;
        const ch = leaveMatch[1];
        chat.leaveChannel(ch);
      } else if (text === '/channels') {
        const chs = chat.getChannels();
        ui.chatLog.log(`{grey-fg}--- Channels: ${chs.join(', ')}{/grey-fg}`);
      } else if (text === '/id') {
        const id = chat.getAgentId();
        ui.chatLog.log(`{grey-fg}--- Agent ID: ${id || 'not connected'}{/grey-fg}`);
      } else if (text.match(/^\/dm\s+(@\S+)\s+(.+)/)) {
        const dmMatch = text.match(/^\/dm\s+(@\S+)\s+(.+)/);
        if (chat.isConnected()) {
          chat.dm(dmMatch[1], dmMatch[2]);
        } else {
          ui.chatLog.log(`{red-fg}! Not connected{/red-fg}`);
        }
      } else {
        chat.send(text);
      }
    }

    ui.chatInput.clearValue();
    // keep focus in chat after submit
    rearmChatInput();
  });

  ui.chatInput.on('cancel', () => {
    ui.chatInput.clearValue();
    chatReading = false;
    setFocus('agents');
  });

  // Global keys (only when chat is NOT focused)
  ui.screen.key(['j', 'down'], () => {
    if (chatFocused) return;
    if (focusPanel === 'agents') {
      if (selectedIdx < filteredAgents.length - 1) {
        selectedIdx++;
        ui.agentList.select(selectedIdx);
        renderDetail();
        switchLogStream();
        ui.screen.render();
      }
    } else if (focusPanel === 'logs') {
      ui.logBox.scroll(1);
      userScrolledLog = true;
      ui.screen.render();
    } else if (focusPanel === 'chat') {
      ui.chatLog.scroll(1);
      ui.screen.render();
    }
  });

  ui.screen.key(['k', 'up'], () => {
    if (chatFocused) return;
    if (focusPanel === 'agents') {
      if (selectedIdx > 0) {
        selectedIdx--;
        ui.agentList.select(selectedIdx);
        renderDetail();
        switchLogStream();
        ui.screen.render();
      }
    } else if (focusPanel === 'logs') {
      ui.logBox.scroll(-1);
      userScrolledLog = true;
      ui.screen.render();
    } else if (focusPanel === 'chat') {
      ui.chatLog.scroll(-1);
      ui.screen.render();
    }
  });

  ui.screen.key(['pageup'], () => {
    if (chatFocused) return;
    if (focusPanel === 'logs') { ui.logBox.scroll(-10); userScrolledLog = true; ui.screen.render(); }
    else if (focusPanel === 'chat') { ui.chatLog.scroll(-10); ui.screen.render(); }
  });

  ui.screen.key(['pagedown'], () => {
    if (chatFocused) return;
    if (focusPanel === 'logs') { ui.logBox.scroll(10); userScrolledLog = true; ui.screen.render(); }
    else if (focusPanel === 'chat') { ui.chatLog.scroll(10); ui.screen.render(); }
  });

  ui.screen.key(['s'], () => {
    if (chatFocused || confirmAction) return;
    const agent = getSelected();
    if (!agent) return;

    if (agent.status === 'running') {
      ui.logBox.log(`{yellow-fg}${agent.name} is already running{/yellow-fg}`);
      ui.screen.render();
      return;
    }

    if (agent.mission) {
      runAgentctl('start', agent.name, agent.mission);
    } else {
      const input = blessed.textbox({
        parent: ui.screen,
        top: 'center',
        left: 'center',
        width: '60%',
        height: 3,
        border: { type: 'line' },
        style: { border: { fg: 'yellow' }, fg: 'white', bg: 'black' },
        label: ` mission for ${agent.name} `,
        inputOnFocus: true,
      });
      input.focus();
      ui.screen.render();
      input.readInput((err, value) => {
        input.destroy();
        ui.screen.render();
        if (value && value.trim()) {
          runAgentctl('start', agent.name, value.trim());
        }
      });
    }
  });

  ui.screen.key(['x'], () => {
    if (chatFocused || confirmAction) return;
    const agent = getSelected();
    if (!agent) return;
    if (agent.status !== 'running') {
      ui.logBox.log(`{yellow-fg}${agent.name} is not running{/yellow-fg}`);
      ui.screen.render();
      return;
    }
    runAgentctl('stop', agent.name);
  });

  ui.screen.key(['r'], () => {
    if (chatFocused || confirmAction) return;
    const agent = getSelected();
    if (!agent) return;
    runAgentctl('restart', agent.name);
  });

  ui.screen.key(['K'], () => {
    if (chatFocused) return;
    const agent = getSelected();
    if (!agent) return;

    if (confirmAction) {
      confirmAction = null;
      updateStatusBar();
      ui.screen.render();
      return;
    }

    confirmAction = { action: 'kill', agent };
    ui.setStatus(`{red-fg}Kill ${agent.name}? [y]es [n]o{/red-fg}`);
    ui.screen.render();
  });

  ui.screen.key(['y'], () => {
    if (chatFocused || !confirmAction) return;
    const { action, agent } = confirmAction;
    confirmAction = null;
    if (action === 'kill') {
      runAgentctl('kill', agent.name);
    }
  });

  ui.screen.key(['n'], () => {
    if (chatFocused) return;
    if (confirmAction) {
      confirmAction = null;
      updateStatusBar();
      ui.screen.render();
    }
  });

  ui.screen.key(['c'], () => {
    if (chatFocused || confirmAction) return;
    const agent = getSelected();
    if (!agent) return;

    const contextFile = path.join(agent.dir, 'context.md');
    if (!fs.existsSync(contextFile)) {
      ui.logBox.log('{grey-fg}No context file{/grey-fg}');
      ui.screen.render();
      return;
    }

    try {
      const content = fs.readFileSync(contextFile, 'utf8');
      ui.logBox.setContent('');
      ui.logBox.setLabel(` context: ${agent.name} `);
      for (const line of content.split('\n')) {
        ui.logBox.log(line);
      }
      ui.screen.render();
    } catch {}
  });

  ui.screen.key(['/'], () => {
    if (chatFocused || confirmAction) return;
    const input = blessed.textbox({
      parent: ui.screen,
      bottom: 0,
      left: 0,
      width: '50%',
      height: 1,
      style: { fg: 'white', bg: 'blue' },
      inputOnFocus: true,
    });
    input.focus();
    input.setValue(filterText);
    ui.screen.render();

    input.readInput((err, value) => {
      input.destroy();
      filterText = (value || '').trim();
      renderAgentList();
      renderDetail();
      if (filteredAgents.length > 0) switchLogStream();
      updateStatusBar();
      ui.screen.render();
    });
  });

  // ? key shows help
  ui.screen.key(['?'], () => {
    if (chatFocused || confirmAction) return;
    const helpLines = [
      '{bold}Keybindings:{/bold}',
      '',
      '  {cyan-fg}Tab{/cyan-fg}         Cycle focus: agents → logs → chat',
      '  {cyan-fg}Shift+Tab{/cyan-fg}   Reverse cycle',
      '  {cyan-fg}j/k or ↑/↓{/cyan-fg} Navigate agent list (or scroll logs/chat)',
      '  {cyan-fg}PgUp/PgDn{/cyan-fg}   Scroll logs/chat by 10 lines',
      '  {cyan-fg}M{/cyan-fg}           Toggle mouse copy mode (lets you drag-select text)',
      '  {cyan-fg}s{/cyan-fg}           Start selected agent',
      '  {cyan-fg}x{/cyan-fg}           Stop selected agent',
      '  {cyan-fg}r{/cyan-fg}           Restart selected agent',
      '  {cyan-fg}K{/cyan-fg}           Kill selected agent (requires confirmation)',
      '  {cyan-fg}c{/cyan-fg}           View agent context.md',
      '  {cyan-fg}/{/cyan-fg}           Filter agents by name',
      '  {cyan-fg}Escape{/cyan-fg}      Clear filter / exit chat / cancel',
      '  {cyan-fg}?{/cyan-fg}           Show this help',
      '  {cyan-fg}q{/cyan-fg}           Quit',
      '',
      '{bold}Chat Commands:{/bold}',
      '',
      '  {magenta-fg}/join #channel{/magenta-fg}   Switch active channel',
      '  {magenta-fg}/leave #channel{/magenta-fg}  Leave a channel',
      '  {magenta-fg}/channels{/magenta-fg}        List joined channels',
      '  {magenta-fg}/id{/magenta-fg}              Show your agent ID',
      '',
      '{bold}CLI Args:{/bold}',
      '',
      '  --name <name>       Agent name (default: tui)',
      '  --server <url>      Server URL',
      '  --identity <path>   Identity file path',
      '',
      '{grey-fg}Press any key to close{/grey-fg}',
    ];

    ui.logBox.setContent('');
    ui.logBox.setLabel(' help ');
    for (const line of helpLines) {
      ui.logBox.log(line);
    }
    ui.screen.render();
  });

  ui.screen.key(['escape'], () => {
    if (chatFocused) {
      ui.chatInput.clearValue();
      setFocus('agents');
      return;
    }
    if (confirmAction) {
      confirmAction = null;
      updateStatusBar();
      ui.screen.render();
      return;
    }
    if (filterText) {
      filterText = '';
      renderAgentList();
      renderDetail();
      switchLogStream();
      ui.screen.render();
    }
  });

  // Tab cycles focus
  ui.screen.key(['tab'], () => {
    if (confirmAction) return;
    const order = ['agents', 'logs', 'chat'];
    const idx = order.indexOf(focusPanel);
    setFocus(order[(idx + 1) % order.length]);
  });

  // S-tab reverse cycles
  ui.screen.key(['S-tab'], () => {
    if (confirmAction) return;
    const order = ['agents', 'logs', 'chat'];
    const idx = order.indexOf(focusPanel);
    setFocus(order[(idx - 1 + order.length) % order.length]);
  });

  // Enter should open the chat input when chat panel is focused (usability fix)
  ui.screen.key(['enter'], () => {
    if (confirmAction) return;
    if (focusPanel === 'chat') {
      setFocus('chat');
    }
  });

  // Mouse copy mode — toggle blessed mouse capture off so terminal can select text natively
  let mouseCopyMode = false;
  ui.screen.key(['M'], () => {
    if (chatFocused) return;
    mouseCopyMode = !mouseCopyMode;
    if (mouseCopyMode) {
      ui.screen.program.disableMouse();
      ui.setStatus('{yellow-fg}COPY MODE — drag to select, M to exit{/yellow-fg}');
    } else {
      ui.screen.program.enableMouse();
      updateStatusBar();
    }
    ui.screen.render();
  });

  // Log scroll detection
  ui.logBox.on('scroll', () => {
    userScrolledLog = ui.logBox.getScrollPerc() < 100;
  });

  // Quit (only when not in chat input)
  ui.screen.key(['q'], () => {
    if (chatFocused) return;
    cleanup();
  });

  ui.screen.key(['C-c'], () => {
    cleanup();
  });

  function cleanup() {
    clearInterval(pollTimer);
    logStreamer.stop();
    chat.disconnect();
    process.exit(0);
  }

  // --- Init ---
  setFocus('agents');
  refresh();
  switchLogStream();
  chat.connect();

  const pollTimer = setInterval(refresh, POLL_INTERVAL);

  // Show connection info
  const identityInfo = IDENTITY_PATH
    ? `{green-fg}persistent identity:{/green-fg} ${CHAT_NAME}`
    : `{grey-fg}ephemeral:{/grey-fg} ${CHAT_NAME}`;
  ui.chatLog.log(`{grey-fg}--- Connecting to ${CHAT_SERVER} as ${identityInfo}{/grey-fg}`);
  ui.chatLog.log(`{grey-fg}--- Channels: ${DEFAULT_CHANNELS.join(', ')}{/grey-fg}`);
  ui.chatLog.log(`{grey-fg}--- Type /join #channel to switch, ? for help{/grey-fg}`);

  ui.screen.render();
}

main();
