
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>MP3 Converter </title>
<style>
  body{font-family:Inter,system-ui,Arial;margin:0;background:#0b1220;color:#e6eef6;display:flex;align-items:center;justify-content:center;height:100vh;padding:18px}
  .card{width:100%;max-width:820px;background:linear-gradient(180deg,#071025,#081426);padding:20px;border-radius:12px;box-shadow:0 10px 40px rgba(0,0,0,0.6)}
  h1{margin:0 0 8px 0;font-size:20px}
  p.small{color:#9aa4b2;margin:6px 0 18px 0}
  .row{display:flex;gap:10px;flex-wrap:wrap}
  input[type=file]{display:none}
  .drop{flex:1;padding:14px;border-radius:10px;border:1px dashed rgba(255,255,255,0.06);text-align:center;cursor:pointer;background:rgba(255,255,255,0.01)}
  select,input[type=number]{padding:8px;border-radius:8px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:inherit}
  button{padding:10px 14px;border-radius:10px;border:none;background:linear-gradient(90deg,#7c5cff,#5b45ff);color:white;cursor:pointer;font-weight:600}
  button:disabled{opacity:0.5;cursor:default}
  .progress{height:10px;background:rgba(255,255,255,0.04);border-radius:8px;overflow:hidden;margin-top:12px}
  .progress > i{display:block;height:100%;width:0;background:linear-gradient(90deg,#7c5cff,#5b45ff)}
  .log{max-height:160px;overflow:auto;background:#051021;padding:10px;border-radius:8px;margin-top:12px;font-size:13px;color:#9aa4b2}
  .links{margin-top:12px;display:flex;gap:8px;flex-wrap:wrap}
  a.download-link{padding:8px 12px;border-radius:8px;background:#0f1724;color:#9ae6b4;text-decoration:none}
  small.note{display:block;margin-top:10px;color:#8b98a8}
  .madeby{margin-top:14px;text-align:center;color:#9aa4b2;font-size:13px}
</style>
</head>
<body>
  <div class="card">
    <h1>MP3 Converter</h1>
    <p class="small">Convert video/audio files to MP3 right in your browser. Files do <strong>not</strong> leave your device.</p>

    <div class="row">
      <label class="drop" id="dropzone">Click to choose or drag & drop files here
        <input id="fileInput" type="file" accept="audio/*,video/*"/>
      </label>

      <div style="min-width:200px;">
        <label>Bitrate</label><br>
        <select id="bitrate">
          <option value="128k">128 kbps</option>
          <option value="192k" selected>192 kbps</option>
          <option value="256k">256 kbps</option>
          <option value="320k">320 kbps</option>
        </select>
      </div>

      <div style="min-width:180px;">
        <label>Sample rate</label><br>
        <select id="samplerate">
          <option value="44100" selected>44.1 kHz</option>
          <option value="48000">48 kHz</option>
        </select>
      </div>

      <div style="display:flex;align-items:flex-end;">
        <button id="convertBtn" disabled>Convert to MP3</button>
      </div>
    </div>

    <div class="progress" aria-hidden><i id="progressBar"></i></div>
    <div class="log" id="log"></div>
    <div class="links" id="links"></div>

    <small class="note">Tip: For large files (&gt;100MB) the browser approach may be slow or memory-heavy. Use a server-side ffmpeg for heavy jobs.</small>

    <!-- FOOTER --> 
    <div class="madeby">made with ❤ by armeen</div>
  </div>

<script src="https://unpkg.com/@ffmpeg/ffmpeg@0.11.8/dist/ffmpeg.min.js"></script>
<script>
  const { createFFmpeg, fetchFile } = FFmpeg;
  const ffmpeg = createFFmpeg({ log: false });
  const fileInput = document.getElementById('fileInput');
  const dropzone = document.getElementById('dropzone');
  const convertBtn = document.getElementById('convertBtn');
  const progressBar = document.getElementById('progressBar');
  const logEl = document.getElementById('log');
  const linksEl = document.getElementById('links');
  const bitrateEl = document.getElementById('bitrate');
  const samplerateEl = document.getElementById('samplerate');

  let currentFile = null;

  function log(msg){
    logEl.innerText += msg + '\n';
    logEl.scrollTop = logEl.scrollHeight;
  }

  dropzone.addEventListener('click', ()=> fileInput.click());
  dropzone.addEventListener('dragover', e => { e.preventDefault(); dropzone.style.borderColor='#7c5cff'; });
  dropzone.addEventListener('dragleave', e => { dropzone.style.borderColor=''; });
  dropzone.addEventListener('drop', e => {
    e.preventDefault(); dropzone.style.borderColor=''; const f = e.dataTransfer.files[0]; handleFile(f);
  });

  fileInput.addEventListener('change', e => { if(e.target.files && e.target.files[0]) handleFile(e.target.files[0]); });

  function handleFile(f){
    currentFile = f;
    dropzone.innerText = f.name + ' (' + Math.round(f.size/1024/1024) + ' MB)';
    convertBtn.disabled = false;
    linksEl.innerHTML = '';
    logEl.innerText = '';
  }

  async function ensureFFmpeg(){
    if(!ffmpeg.isLoaded()){
      log('Loading FFmpeg core (this may take a few seconds)...');
      await ffmpeg.load();
      log('FFmpeg loaded.');
    }
  }

  convertBtn.addEventListener('click', async ()=>{
    if(!currentFile) return alert('Choose a file first.');
    convertBtn.disabled = true;
    linksEl.innerHTML = '';
    progressBar.style.width = '0%';
    logEl.innerText = '';

    try {
      await ensureFFmpeg();
      const inName = 'input' + (currentFile.name.match(/\.[^.]+$/)?.[0] || '');
      const outName = 'output.mp3';
      log('Writing file: ' + inName);
      ffmpeg.FS('writeFile', inName, await fetchFile(currentFile));
      const br = bitrateEl.value;
      const sr = samplerateEl.value;
      log(`Converting → ${br}, ${sr} ...`);
      ffmpeg.setProgress(({ ratio }) => { progressBar.style.width = Math.min(1, ratio) * 100 + '%'; });
      await ffmpeg.run('-i', inName, '-vn', '-ar', sr, '-b:a', br, outName);
      const data = ffmpeg.FS('readFile', outName);
      const blob = new Blob([data.buffer], { type: 'audio/mpeg' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = (currentFile.name.replace(/\.[^.]+$/, '') || 'converted') + '.mp3';
      a.className = 'download-link';
      a.innerText = 'Download MP3 — ' + a.download;
      linksEl.appendChild(a);
      const player = document.createElement('audio');
      player.controls = true;
      player.src = url;
      player.style.marginLeft = '12px';
      linksEl.appendChild(player);
      log('Done. File ready for download.');
    } catch (err) {
      console.error(err);
      log('Error: ' + (err.message || err));
    } finally {
      convertBtn.disabled = false;
      ffmpeg.setProgress(() => {});
    }
  });
</script>
</body>
</html>
# MP3_Convertor
