# Multiplayer Implementation Specification (For AI Agents)

## Overview
This document outlines the technical specification for upgrading the `index.html` Timeline Discussion Canvas into a real-time, multi-player collaborative tool. 

The goal is to implement a **"Token Passing" (Mutex Lock)** multiplayer architecture using **Cloudflare Durable Objects (via PartyKit)**. This avoids complex CRDT algorithms by ensuring only one person has edit rights at a time, fitting the "professional engineering meeting" context perfectly.

## Architecture
- **Frontend**: Remains a single, static `index.html` file (hosted on GitHub Pages).
- **Backend**: A minimal PartyKit WebSocket server (Cloudflare Edge).
- **State Management**: The JSON `model` remains the single source of truth. The Backend acts as a dumb broadcaster.

## Core Mechanisms: Token Passing
To prevent Race Conditions and simplify `Undo/Redo`, we strictly use a "Driver/Viewer" roles system.
1. **Driver**: The single user holding the "Edit Token". Can drag, add, and modify the canvas. When they modify the canvas, they broadcast their updated JSON `model` to the room.
2. **Viewer**: Read-only users. Their toolbar is disabled. They receive the JSON `model` and re-render their SVG.

## URL & Room Routing Strategy
- **Frictionless Entry**: 
  - If a user opens `index.html` without a room parameter, the JS automatically generates a random 6-character room ID, appends it to the URL (`?room=room-xxxxx`) using `history.replaceState()`, and connects.
  - Users share the URL. When a colleague clicks it, the JS reads `?room=room-xxxxx` and connects to that existing WebSocket room.

## User Identity (Presence)
- On page load, check `localStorage.getItem('userName')`.
- If missing, prompt the user: `prompt("Please enter your display name:")` and save it.
- Connect to WebSocket and send a `JOIN` message containing the `userName`.

## UI / UX Specifications (English Only)

### 1. User Presence (Top Right MetaStrip)
- Replace static chips with dynamic online presence.
- Example: `Online (3): 🟢 Mortis (Driver), ⚪ Colleague_A, ⚪ Colleague_B`

### 2. Action Buttons & Toolbar States
- **If you are the Driver**:
  - Toolbar is active.
  - Status text next to toolbar: `You have control.`
  - Optional button: `[ Yield Control ]`
- **If you are a Viewer**:
  - Toolbar inputs/buttons are `disabled`. SVG canvas `pointer-events` should prevent dragging.
  - Status text next to toolbar: `Read-only. Mortis is driving.`
  - Action button: `[ Take Control ]` (Clicking this sends a `REQUEST_CONTROL` message to the server, transferring the Driver role to this user).

## Implementation Steps for the Agent
1. **Initialize PartyKit Client**: Import `partysocket` via CDN/ESM in `index.html`.
2. **Setup Room Logic**: Parse `window.location.search` for `?room=`.
3. **Setup Presence**: Handle the Username prompt and LocalStorage.
4. **WebSocket Event Listeners**:
   - `onMessage('state_update')`: Overwrite local `model` and call `scheduleRender()`.
   - `onMessage('presence_update')`: Update the top-right User Presence UI and check if `myUserId == currentDriverId`.
5. **Intercept Saves**: Modify the existing `saveLocal()` (or similar logic) so that if the user is the Driver, it also sends `socket.send({ type: 'state_update', model: model })`.
6. **Implement UI Locks**: Wrap the SVG drag/drop and Toolbar event listeners with a check: `if (!isDriver) return;`.

## Backend Logic (PartyKit Server)
The `server.ts` logic is extremely minimal:
- Maintain `onlineUsers = Map<ConnectionId, {name: string}>`
- Maintain `currentDriverId: string`
- Maintain `canvasState: any`
- On `REQUEST_CONTROL`: Update `currentDriverId`, broadcast new Presence state.
- On `STATE_UPDATE` (from Driver only): Update `canvasState`, broadcast to all other connections.
- On `CONNECT` / `DISCONNECT`: Update `onlineUsers`, broadcast Presence state.
