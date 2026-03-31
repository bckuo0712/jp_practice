<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>50音 AI 練字王</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        html, body { position: fixed; overflow: hidden; width: 100%; height: 100%; touch-action: none; background-color: #020617; color: #f8fafc; font-family: sans-serif; }
        .canvas-container { position: relative; width: 320px; height: 320px; background-color: #1e293b; border: 2px solid #334155; border-radius: 28px; overflow: hidden; box-shadow: 0 0 20px rgba(59, 130, 246, 0.2); }
        #hint-text { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; color: #334155; font-size: 8rem; font-family: serif; font-weight: 900; z-index: 0; pointer-events: none; }
        canvas { position: absolute; inset: 0; touch-action: none; background-color: transparent; z-index: 10; cursor: crosshair; }
        .active-mode { background-color: #3b82f6; color: white; box-shadow: 0 0 15px rgba(59, 130, 246, 0.5); }
    </style>
</head>
<body class="flex flex-col items-center p-4">

    <div class="flex bg-slate-800 rounded-2xl p-1 mb-4 w-72 mt-4 z-50">
        <button id="btn-practice" onclick="setMode('practice')" class="flex-1 py-2 rounded-xl active-mode font-bold">練習版</button>
        <button id="btn-exam" onclick="setMode('exam')" class="flex-1 py-2 rounded-xl font-bold text-slate-400">實戰版</button>
    </div>

    <div class="text-center mb-4 z-50">
        <div id="target-romaji" class="text-4xl font-black text-blue-400 uppercase">A</div>
        <div class="text-[10px] text-slate-500 font-bold mt-1">NIGHT MODE ACTIVE</div>
    </div>

    <div class="canvas-container mb-6">
        <div id="hint-text">あア</div>
        <canvas id="mainCanvas" width="320" height="320"></canvas>
        <div id="loading" class="absolute top-4 right-4 hidden z-20">
            <div class="animate-spin rounded-full h-6 w-6 border-4 border-blue-400 border-t-transparent"></div>
        </div>
    </div>

    <div id="status-msg" class="h-8 text-lg font-bold mb-4 text-slate-500">READY</div>

    <div class="flex space-x-3 w-full max-w-[320px] z-50">
        <button onclick="resetCanvas()" class="flex-1 bg-slate-800 border border-slate-700 py-4 rounded-2xl font-bold text-slate-300">清除</button>
        <button onclick="showErrors()" class="px-5 bg-red-900/30 py-4 rounded-2xl font-bold text-red-400 border border-red-900/50">紀錄</button>
        <button onclick="nextQ()" class="flex-1 bg-blue-600 text-white py-4 rounded-2xl font-bold">跳過</button>
    </div>

    <div id="err-modal" class="fixed inset-0 bg-black/80 hidden flex items-center justify-center z-[100] p-6 backdrop-blur-md">
        <div class="bg-slate-900 border border-slate-800 rounded-3xl p-6 w-full max-w-sm max-h-[70vh] overflow-y-auto">
            <div class="flex justify-between items-center mb-6 border-b border-slate-800 pb-4">
                <span class="font-black text-xl">錯誤紀錄</span>
                <button onclick="closeErrors()" class="text-slate-500">✕</button>
            </div>
            <div id="err-list" class="space-y-3"></div>
            <button onclick="clearLogs()" class="mt-8 w-full text-slate-600 text-xs">清空數據</button>
        </div>
    </div>

    <script>
        const kanaData = [
            { h: 'あ', k: 'ア', r: 'a' }, { h: 'い', k: 'イ', r: 'i' },
            { h: 'う', k: 'ウ', r: 'u' }, { h: 'え', k: 'エ', r: 'e' },
            { h: 'お', k: 'オ', r: 'o' }, { h: 'か', k: 'カ', r: 'ka' },
            { h: 'き', k: 'キ', r: 'ki' }, { h: 'く', k: 'ク', r: 'ku' },
            { h: 'け', k: 'ケ', r: 'ke' }, { h: 'こ', k: 'コ', r: 'ko' }
        ];

        let curIdx = 0, mode = 'practice', drawing = false, timer = null, strokes = [];
        const canvas = document.getElementById('mainCanvas'), ctx = canvas.getContext('2d');

        function initBrush() { ctx.lineWidth = 8; ctx.lineCap = 'round'; ctx.lineJoin = 'round'; ctx.strokeStyle = '#ffffff'; }

        function render() {
            const q = kanaData[curIdx];
            document.getElementById('target-romaji').innerText = q.r;
            const hint = document.getElementById('hint-text');
            hint.innerText = q.h + q.k;
            hint.style.visibility = (mode === 'practice') ? 'visible' : 'hidden';
            resetCanvas();
            msg("請書寫...", "text-slate-500");
        }

        function setMode(m) {
            mode = m;
            document.getElementById('btn-practice').className = `flex-1 py-2 rounded-xl font-bold ${m==='practice'?'active-mode':'text-slate-400'}`;
            document.getElementById('btn-exam').className = `flex-1 py-2 rounded-xl font-bold ${m==='exam'?'active-mode':'text-slate-400'}`;
            render();
        }

        function getXY(e) {
            const rect = canvas.getBoundingClientRect();
            const cx = e.touches ? e.touches[0].clientX : e.clientX;
            const cy = e.touches ? e.touches[0].clientY : e.clientY;
            return { x: cx - rect.left, y: cy - rect.top };
        }

        const start = (e) => { clearTimeout(timer); drawing = true; const p = getXY(e); ctx.beginPath(); ctx.moveTo(p.x, p.y); strokes.push([[p.x], [p.y], []]); };
        const move = (e) => { if (!drawing) return; e.preventDefault(); const p = getXY(e); ctx.lineTo(p.x, p.y); ctx.stroke(); const last = strokes[strokes.length-1]; last[0].push(p.x); last[1].push(p.y); };
        const end = () => { if (!drawing) return; drawing = false; timer = setTimeout(apiRecognize, 500); };

        async function apiRecognize() {
            if (strokes.length === 0) return;
            document.getElementById('loading').classList.remove('hidden');
            msg("辨識中...", "text-blue-400");
            const q = kanaData[curIdx];
            try {
                const res = await fetch('https://www.google.com.tw/inputtools/request?ime=handwriting&app=mobilesearch&cs=1&oe=UTF-8', {
                    method: 'POST',
                    body: JSON.stringify({ options: "enable_pre_space", requests: [{ writing_guide: { writing_area_width: 320, writing_area_height: 320 }, ink: strokes, language: "ja" }]})
                });
                const data = await res.json();
                const candidates = data[1][0][1];
                if (candidates.includes(q.h) || candidates.includes(q.k)) {
                    msg("✅ 正確！", "text-green-400");
                    setTimeout(nextQ, 800);
                } else {
                    msg("❌ 再試一次", "text-red-400");
                    let logs = JSON.parse(localStorage.getItem('jp_err') || '[]');
                    if (!logs.find(x => x.r === q.r)) { logs.push({ h: q.h, k: q.k, r: q.r }); localStorage.setItem('jp_err', JSON.stringify(logs)); }
                }
            } catch (err) { msg("連線異常", "text-orange-400"); }
            finally { document.getElementById('loading').classList.add('hidden'); }
        }

        function msg(t, c) { const m = document.getElementById('status-msg'); m.innerText = t; m.className = `h-8 text-lg font-bold mb-4 ${c}`; }
        function resetCanvas() { ctx.clearRect(0, 0, canvas.width, canvas.height); strokes = []; clearTimeout(timer); }
        function nextQ() { curIdx = (curIdx + 1) % kanaData.length; render(); }
        function showErrors() {
            const logs = JSON.parse(localStorage.getItem('jp_err') || '[]');
            const list = document.getElementById('err-list');
            list.innerHTML = logs.length ? '' : '<p class="text-slate-500">尚無紀錄</p>';
            logs.forEach(x => { list.innerHTML += `<div class="flex justify-between p-4 bg-slate-800 rounded-2xl"><span>${x.h}${x.k}</span><span>${x.r}</span></div>`; });
            document.getElementById('err-modal').classList.remove('hidden');
        }
        function closeErrors() { document.getElementById('err-modal').classList.add('hidden'); }
        function clearLogs() { localStorage.removeItem('jp_err'); showErrors(); }

        canvas.addEventListener('touchstart', start, {passive:false});
        canvas.addEventListener('touchmove', move, {passive:false});
        window.addEventListener('touchend', end);
        canvas.addEventListener('mousedown', start);
        canvas.addEventListener('mousemove', move);
        window.addEventListener('mouseup', end);

        initBrush(); render();
    </script>
</body>
</html>
