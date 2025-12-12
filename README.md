# Creating a ready-to-run minimal React project zip with the component and placeholder audio files.
import os, zipfile, wave, struct, math

base = "/mnt/data/geometry_dash_ready"
os.makedirs(base, exist_ok=True)

# Create src directory and App.jsx with the component (adjusted to use .wav files)
src = os.path.join(base, "src")
public = os.path.join(base, "public")
audio_dir = os.path.join(public, "audio")
os.makedirs(src, exist_ok=True)
os.makedirs(audio_dir, exist_ok=True)

app_jsx = r'''import React from "react";
import GeometryDashLite from "./GeometryDashLite";

export default function App(){
  return <GeometryDashLite />;
}
'''

component = r'''import React, { useEffect, useRef, useState } from "react";

// Geometry Dash - simplified single-file React component
// This demo uses small placeholder .wav audio files included in /public/audio/.
// You can replace them with your own royalty-free tracks.
// Controls: Space / ArrowUp = jump, R = stop/restart.

export default function GeometryDashLite() {
  const gravity = 0.75;
  const jumpVelocity = -11;
  const groundY = 260;

  const levels = [
    { id: 1, name: "Stereo Madness", audio: "/audio/stereo_madness.wav", speed: 4 },
    { id: 2, name: "Back on Track", audio: "/audio/back_on_track.wav", speed: 5 },
    { id: 3, name: "Polargeist (demo)", audio: "/audio/polargeist.wav", speed: 6 },
  ];

  const [levelIdx, setLevelIdx] = useState(0);
  const [running, setRunning] = useState(false);
  const [score, setScore] = useState(0);
  const canvasRef = useRef(null);
  const audioRef = useRef(null);
  const animationRef = useRef(null);

  const player = useRef({ x: 40, y: groundY - 40, w: 40, h: 40, vy: 0, onGround: true });
  const obstacles = useRef([]);
  const tickRef = useRef(0);

  useEffect(() => {
    const canvas = canvasRef.current;
    const ctx = canvas.getContext("2d");
    canvas.width = 800;
    canvas.height = 320;

    function resetLevel() {
      player.current = { x: 40, y: groundY - 40, w: 40, h: 40, vy: 0, onGround: true };
      obstacles.current = [];
      tickRef.current = 0;
      setScore(0);
      if (audioRef.current) audioRef.current.currentTime = 0;
    }

    function spawnObstacle() {
      const width = Math.random() > 0.6 ? 24 + Math.random() * 26 : 40;
      const height = 24 + Math.random() * 56;
      obstacles.current.push({ x: canvas.width + 20, y: groundY - height, w: width, h: height });
    }

    function rectsIntersect(a, b) {
      return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
    }

    function update() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "#0f172a";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      ctx.fillStyle = "rgba(255,255,255,0.05)";
      for (let i = 0; i < 40; i++) {
        const px = (i * 37 + tickRef.current * 0.2) % canvas.width;
        ctx.fillRect(px, 40 + (i % 6) * 12, 4, 4);
      }

      ctx.fillStyle = "#111827";
      ctx.fillRect(0, groundY, canvas.width, canvas.height - groundY);

      const p = player.current;
      p.vy += gravity;
      p.y += p.vy;
      if (p.y + p.h >= groundY) {
        p.y = groundY - p.h;
        p.vy = 0;
        p.onGround = true;
      }

      ctx.fillStyle = "#ef4444";
      roundRect(ctx, p.x, p.y, p.w, p.h, 6);
      ctx.fillStyle = "rgba(255,255,255,0.12)";
      roundRect(ctx, p.x + 6, p.y + 6, p.w - 12, p.h - 12, 4);

      const speed = levels[levelIdx].speed;
      for (let i = obstacles.current.length - 1; i >= 0; i--) {
        const o = obstacles.current[i];
        o.x -= speed;
        ctx.fillStyle = "#22c55e";
        ctx.fillRect(o.x, o.y, o.w, o.h);

        if (rectsIntersect(p, o)) {
          setRunning(false);
          if (audioRef.current) audioRef.current.pause();
          return;
        }

        if (o.x + o.w < 0) {
          obstacles.current.splice(i, 1);
          setScore((s) => s + 1);
        }
      }

      if (running && Math.random() < 0.016 + Math.min(0.02, 0.002 * tickRef.current / 60)) {
        spawnObstacle();
      }

      ctx.fillStyle = "#e5e7eb";
      ctx.font = "16px Inter, Arial";
      ctx.fillText(`Level: ${levels[levelIdx].name}`, 12, 22);
      ctx.fillText(`Score: ${score}`, 12, 42);

      tickRef.current += 1;

      if (running) animationRef.current = requestAnimationFrame(update);
    }

    function start() {
      resetLevel();
      setRunning(true);
      if (audioRef.current) audioRef.current.play().catch(() => {});
      if (!animationRef.current) animationRef.current = requestAnimationFrame(update);
    }

    const handleKey = (e) => {
      if (e.code === "Space" || e.code === "ArrowUp") {
        e.preventDefault();
        if (!running) start();
        jump();
      }
      if (e.code === "KeyR") {
        resetLevel();
        setRunning(false);
        if (audioRef.current) audioRef.current.pause();
      }
    };

    window.addEventListener("keydown", handleKey);
    const handleTouch = (e) => { if (!running) start(); jump(); };
    canvas.addEventListener("touchstart", handleTouch);

    function jump() {
      const p = player.current;
      if (p.onGround) {
        p.vy = jumpVelocity;
        p.onGround = false;
      }
    }

    function roundRect(ctx, x, y, w, h, r) {
      ctx.beginPath();
      ctx.moveTo(x + r, y);
      ctx.arcTo(x + w, y, x + w, y + h, r);
      ctx.arcTo(x + w, y + h, x, y + h, r);
      ctx.arcTo(x, y + h, x, y, r);
      ctx.arcTo(x, y, x + w, y, r);
      ctx.closePath();
      ctx.fill();
    }

    return () => {
      window.removeEventListener("keydown", handleKey);
      canvas.removeEventListener("touchstart", handleTouch);
      if (animationRef.current) cancelAnimationFrame(animationRef.current);
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [levelIdx, score, running]);

  const togglePlay = () => {
    if (running) {
      setRunning(false);
      if (audioRef.current) audioRef.current.pause();
      if (animationRef.current) cancelAnimationFrame(animationRef.current);
    } else {
      setRunning(true);
      if (audioRef.current) audioRef.current.play().catch(() => {});
      animationRef.current = requestAnimationFrame(() => {});
    }
  };

  const changeLevel = (dir) => {
    const next = (levelIdx + dir + levels.length) % levels.length;
    setLevelIdx(next);
    if (audioRef.current) {
      audioRef.current.pause();
      audioRef.current.currentTime = 0;
    }
    setRunning(false);
    setScore(0);
  };

  return (
    <div style={{minHeight: '100vh', display:'flex', alignItems:'center', justifyContent:'center', background: 'linear-gradient(#0b1220,#081028)'}}>
      <div style={{width:900, maxWidth:'100%', background:'rgba(15,23,42,0.6)', borderRadius:16, padding:16}}>
        <div style={{display:'flex', justifyContent:'space-between', marginBottom:12}}>
          <div>
            <h1 style={{color:'#fff', fontSize:22}}>Geometry Dash — Lite (Web)</h1>
            <p style={{color:'#cbd5e1', marginTop:4}}>Use Space / ↑ / touch to jump. Press R to stop/restart.</p>
          </div>
          <div style={{display:'flex', gap:8, alignItems:'center'}}>
            <button onClick={() => changeLevel(-1)}>◀</button>
            <div style={{padding:'6px 12px', background:'#0f172a', color:'#fff'}}>{levels[levelIdx].name}</div>
            <button onClick={() => changeLevel(1)}>▶</button>
            <button onClick={togglePlay} style={{marginLeft:10}}>{running ? 'Pause' : 'Play'}</button>
          </div>
        </div>

        <div style={{display:'flex', gap:12}}>
          <div style={{flex:1}}>
            <canvas ref={canvasRef} style={{width:'100%', borderRadius:8, background:'#071133'}} />
          </div>

          <div style={{width:200}}>
            <div style={{background:'#0f172a', padding:10, borderRadius:8, color:'#e5e7eb', fontSize:13}}>
              <div style={{fontWeight:600, marginBottom:6}}>Controls</div>
              <div>Space / ↑ / touch — Jump</div>
              <div>R — Restart / Stop</div>
              <hr style={{margin:'10px 0', borderColor:'#0b1220'}} />
              <div style={{fontWeight:600}}>Audio</div>
              <div style={{fontSize:12, marginTop:6}}>Included placeholder audio files (replace with your own):</div>
              <ul style={{fontSize:12}}>
                {levels.map((l) => <li key={l.id}>{l.name} — {l.audio}</li>)}
              </ul>
              <div style={{marginTop:8, color:'#fbbf24'}}>Note: Do not include copyrighted tracks without permission.</div>
            </div>

            <audio ref={audioRef} src={levels[levelIdx].audio} loop />
          </div>
        </div>

        <div style={{marginTop:12, textAlign:'right', color:'#9ca3af', fontSize:12}}>Demo — replace audio in /public/audio/</div>
      </div>
    </div>
  );
}
'''

with open(os.path.join(src, "App.jsx"), "w", encoding="utf-8") as f:
    f.write(app_jsx)
with open(os.path.join(src, "GeometryDashLite.jsx"), "w", encoding="utf-8") as f:
    f.write(component)

# package.json
pkg = {
  "name": "geometry-dash-lite",
  "version": "0.1.0",
  "private": True,
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-scripts": "5.0.1"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build"
  }
}
import json
with open(os.path.join(base, "package.json"), "w", encoding="utf-8") as f:
    json.dump(pkg, f, indent=2)

# Create three short WAV placeholder audio files (sine tones)
def write_wav(filename, freq=440.0, duration=1.0, volume=0.5, samplerate=22050):
    n_samples = int(samplerate * duration)
    wav = wave.open(filename, 'w')
    wav.setparams((1, 2, samplerate, n_samples, 'NONE', 'not compressed'))
    for i in range(n_samples):
        sample = volume * math.sin(2 * math.pi * freq * (i / samplerate))
        # 16-bit PCM
        wav.writeframes(struct.pack('<h', int(sample * 32767.0)))
    wav.close()

write_wav(os.path.join(audio_dir, "stereo_madness.wav"), freq=440.0, duration=1.2)
write_wav(os.path.join(audio_dir, "back_on_track.wav"), freq=554.37, duration=1.2)
write_wav(os.path.join(audio_dir, "polargeist.wav"), freq=659.25, duration=1.2)

# Create README
readme = """Geometry Dash — Lite (Web)
------------------------

This is a minimal React project (create-react-app style) with a lightweight Geometry Dash-like component
and three placeholder audio files in /public/audio/.

How to run:
1. npm install
2. npm start

Files of interest:
- src/App.jsx
- src/GeometryDashLite.jsx
- public/audio/*.wav

Note: Included audio files are short tone placeholders (royalty-free). Replace them with your own tracks,
but do NOT include copyrighted songs without permission.
"""
with open(os.path.join(base, "README.txt"), "w", encoding="utf-8") as f:
    f.write(readme)

# Zip the folder
zip_path = "/mnt/data/geometry_dash_ready.zip"
with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as z:
    for root, dirs, files in os.walk(base):
        for file in files:
            z.write(os.path.join(root, file), arcname=os.path.relpath(os.path.join(root, file), base))

print("Created zip at:", zip_path)

