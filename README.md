<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>P2P オンライン人狼ゲーム</title>
    <!-- PeerJS Library (P2P通信用) -->
    <script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>
    <!-- QRコード生成ライブラリ -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
    <style>
        :root {
            --bg-color: #1a1a2e;
            --panel-color: #16213e;
            --accent-color: #e94560;
            --text-color: #ffffff;
            --sub-text: #a0a0a0;
        }

        body {
            font-family: 'Helvetica Neue', Arial, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            margin: 0;
            padding: 20px;
            display: flex;
            justify-content: center;
        }

        .container {
            width: 100%;
            max-width: 800px;
            background: var(--panel-color);
            padding: 25px;
            border-radius: 12px;
            box-shadow: 0 8px 24px rgba(0,0,0,0.5);
        }

        h1, h2, h3 { text-align: center; color: #fff; }

        .section {
            background: rgba(255, 255, 255, 0.05);
            padding: 15px;
            margin-bottom: 20px;
            border-radius: 8px;
        }

        input[type="text"], button {
            padding: 10px 15px;
            margin: 5px;
            border-radius: 5px;
            border: none;
            font-size: 16px;
        }

        input[type="text"] {
            background: #0f3460;
            color: #fff;
            width: calc(100% - 30px);
        }

        button {
            background-color: var(--accent-color);
            color: white;
            cursor: pointer;
            transition: 0.2s;
            font-weight: bold;
        }

        button:hover { opacity: 0.8; }
        button:disabled { background: #555; cursor: not-allowed; }

        .hidden { display: none !important; }

        /* ロビー・プレイヤーリスト */
        .player-list {
            list-style: none;
            padding: 0;
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(140px, 1fr));
            gap: 10px;
        }

        .player-card {
            background: #0f3460;
            padding: 10px;
            border-radius: 6px;
            text-align: center;
            border: 2px solid transparent;
        }

        .player-card.is-me { border-color: #00fff5; }

        /* 役職表示カード */
        .role-card {
            background: #533483;
            padding: 20px;
            border-radius: 8px;
            text-align: center;
            font-size: 20px;
            font-weight: bold;
            margin: 15px 0;
        }

        /* ログエリア */
        #logArea {
            height: 150px;
            background: #000;
            overflow-y: scroll;
            padding: 10px;
            font-family: monospace;
            font-size: 14px;
            border-radius: 5px;
            color: #00ff66;
        }

        #qrcode {
            display: flex;
            justify-content: center;
            margin: 15px 0;
            background: white;
            padding: 10px;
            width: fit-content;
            margin-left: auto;
            margin-right: auto;
            border-radius: 8px;
        }
    </style>
</head>
<body>

<div class="container">
    <h1>🐺 オンライン P2P 人狼ゲーム</h1>

    <!-- 参加登録セクション -->
    <div id="setupSection" class="section">
        <h2>1. プレイヤー登録</h2>
        <input type="text" id="playerNameInput" placeholder="あなたの名前を入力..." maxlength="10">
        <div style="text-align: center; margin-top: 15px;">
            <button id="createRoomBtn">部屋を作成する (ホスト)</button>
        </div>
    </div>

    <!-- ホスト用：部屋共有セクション -->
    <div id="shareSection" class="section hidden">
        <h2>2. 友達を招待する (最大10人)</h2>
        <p>以下のURL共有またはQRコードを友達に送ってください：</p>
        <input type="text" id="shareUrlInput" readonly>
        <button onclick="copyShareUrl()">URLをコピー</button>
        <div id="qrcode"></div>
    </div>

    <!-- ゲームロビーセクション -->
    <div id="lobbySection" class="section hidden">
        <h2>参加者一覧 (<span id="playerCount">0</span>/10)</h2>
        <ul id="playerList" class="player-list"></ul>
        
        <div id="hostControls" class="hidden" style="text-align: center; margin-top: 20px;">
            <button id="startGameBtn" disabled>ゲームを開始する (3名以上)</button>
        </div>
    </div>

    <!-- ゲームメインエリア -->
    <div id="gameSection" class="section hidden">
        <h2>ゲーム進行中</h2>
        <div id="myRoleCard" class="role-card">あなたの役職: ???</div>
        <p id="gamePhaseText" style="text-align:center; font-size:18px; font-weight:bold;"></p>
        
        <div id="actionArea" style="text-align: center; margin: 15px 0;"></div>
    </div>

    <!-- システムログ -->
    <div class="section">
        <h3>通信/ゲームログ</h3>
        <div id="logArea"></div>
    </div>
</div>

<script>
    // --- グローバル状態管理 ---
    let peer = null;
    let myId = '';
    let myName = '';
    let isHost = false;
    let hostConn = null; // ゲスト用：ホストとの接続
    const connections = {}; // ホスト用：接続している全ゲストのPeerID -> connection
    let players = []; // { id, name, isHost, role, alive }

    // --- DOM要素 ---
    const setupSection = document.getElementById('setupSection');
    const shareSection = document.getElementById('shareSection');
    const lobbySection = document.getElementById('lobbySection');
    const gameSection = document.getElementById('gameSection');
    const playerNameInput = document.getElementById('playerNameInput');
    const createRoomBtn = document.getElementById('createRoomBtn');
    const shareUrlInput = document.getElementById('shareUrlInput');
    const playerList = document.getElementById('playerList');
    const playerCount = document.getElementById('playerCount');
    const hostControls = document.getElementById('hostControls');
    const startGameBtn = document.getElementById('startGameBtn');
    const logArea = document.getElementById('logArea');

    // --- 初期化処理 ---
    window.onload = () => {
        const urlParams = new URLSearchParams(window.location.search);
        const hostIdParam = urlParams.get('host');

        if (hostIdParam) {
            // URLにhostパラメータがある場合はゲストとして接続準備
            createRoomBtn.innerText = "部屋に参加する (ゲスト)";
            createRoomBtn.onclick = () => joinRoom(hostIdParam);
        } else {
            // パラメータがない場合はホストとして部屋作成
            createRoomBtn.onclick = createRoom;
        }
    };

    function log(msg) {
        const time = new Date().toLocaleTimeString();
        logArea.innerHTML += `[${time}] ${msg}<br>`;
        logArea.scrollTop = logArea.scrollHeight;
    }

    // --- ホスト：部屋作成 ---
    function createRoom() {
        myName = playerNameInput.value.trim();
        if (!myName) return alert("名前を入力してください");

        isHost = true;
        peer = new Peer(); // ランダムIDを生成

        peer.on('open', (id) => {
            myId = id;
            log(`部屋を作成しました (ID: ${id})`);
            
            // 共有用URL生成
            const shareUrl = `${window.location.origin}${window.location.pathname}?host=${id}`;
            shareUrlInput.value = shareUrl;
            
            // QRコード生成
            new QRCode(document.getElementById("qrcode"), { text: shareUrl, width: 128, height: 128 });

            setupSection.classList.add('hidden');
            shareSection.classList.remove('hidden');
            lobbySection.classList.remove('hidden');
            hostControls.classList.remove('hidden');

            // ホスト自身をプレイヤーに追加
            players.push({ id: myId, name: myName, isHost: true, role: '', alive: true });
            updateLobbyUI();
        });

        // ゲストからの接続要求の待受
        peer.on('connection', (conn) => {
            if (players.length >= 10) {
                conn.send({ type: 'ERROR', msg: '部屋が満員です（最大10人）' });
                setTimeout(() => conn.close(), 500);
                return;
            }

            conn.on('data', (data) => handleHostReceiveData(conn, data));
            conn.on('close', () => handleGuestDisconnect(conn.peer));
        });
    }

    // --- ゲスト：部屋参加 ---
    function joinRoom(hostId) {
        myName = playerNameInput.value.trim();
        if (!myName) return alert("名前を入力してください");

        peer = new Peer();

        peer.on('open', (id) => {
            myId = id;
            log(`サーバー(Peer)に接続完了。ホストに接続中...`);

            hostConn = peer.connect(hostId);

            hostConn.on('open', () => {
                log("ホストへの接続に成功しました！");
                // ホストへ参加リクエストを送信
                hostConn.send({ type: 'JOIN', name: myName });
                
                setupSection.classList.add('hidden');
                lobbySection.classList.remove('hidden');
            });

            hostConn.on('data', (data) => handleGuestReceiveData(data));
            hostConn.on('close', () => {
                alert("ホストとの接続が切断されました");
                location.href = window.location.pathname; // リセット
            });
        });
    }

    // --- ホスト側のデータ処理メッセージ制御 ---
    function handleHostReceiveData(conn, data) {
        switch(data.type) {
            case 'JOIN':
                connections[conn.peer] = conn;
                players.push({ id: conn.peer, name: data.name, isHost: false, role: '', alive: true });
                log(`${data.name} が参加しました`);
                broadcastGameState();
                break;
        }
    }

    // --- ゲスト側のデータ処理メッセージ制御 ---
    function handleGuestReceiveData(data) {
        switch(data.type) {
            case 'SYNC_STATE':
                players = data.players;
                updateLobbyUI();
                break;
            case 'GAME_START':
                const myData = data.players.find(p => p.id === myId);
                document.getElementById('myRoleCard').innerText = `あなたの役職: ${myData.role}`;
                lobbySection.classList.add('hidden');
                shareSection.classList.add('hidden');
                gameSection.classList.remove('hidden');
                log("ゲームが開始されました！");
                break;
        }
    }

    // --- 全員へ状態同期(ホスト専用) ---
    function broadcastGameState() {
        updateLobbyUI();
        const payload = { type: 'SYNC_STATE', players: players };
        Object.values(connections).forEach(conn => conn.send(payload));
    }

    // --- 切断ハンドリング ---
    function handleGuestDisconnect(peerId) {
        const p = players.find(x => x.id === peerId);
        if (p) log(`${p.name} が退出しました`);
        players = players.filter(x => x.id !== peerId);
        delete connections[peerId];
        broadcastGameState();
    }

    // --- ロビーUI更新 ---
    function updateLobbyUI() {
        playerList.innerHTML = '';
        players.forEach(p => {
            const li = document.createElement('li');
            li.className = `player-card ${p.id === myId ? 'is-me' : ''}`;
            li.innerHTML = `<strong>${escapeHtml(p.name)}</strong><br><small>${p.isHost ? 'ホスト' : '参加者'}</small>`;
            playerList.appendChild(li);
        });

        playerCount.innerText = players.length;

        if (isHost) {
            // 人狼ゲームは最低3名からスタート可能とする設定
            startGameBtn.disabled = players.length < 3;
        }
    }

    // --- ゲーム開始 (ホスト専用) ---
    startGameBtn.addEventListener('click', () => {
        if (players.length < 3) return;

        // 役職割り当て (例: 10人なら 人狼2, 占1, 騎士1, 市民6 / 少量なら適宜配分)
        const count = players.length;
        let roles = [];

        if (count >= 6) {
            roles = ['人狼', '人狼', '占い師', '騎士'];
        } else if (count >= 4) {
            roles = ['人狼', '占い師', '騎士'];
        } else {
            roles = ['人狼', '占い師'];
        }

        while (roles.length < count) {
            roles.push('市民');
        }

        // シャッフル
        roles.sort(() => Math.random() - 0.5);

        // 配布
        players.forEach((p, idx) => {
            p.role = roles[idx];
            p.alive = true;
        });

        // ゲストに送信
        const payload = { type: 'GAME_START', players: players };
        Object.values(connections).forEach(conn => conn.send(payload));

        // ホスト自身のUI更新
        const myData = players.find(p => p.id === myId);
        document.getElementById('myRoleCard').innerText = `あなたの役職: ${myData.role}`;
        lobbySection.classList.add('hidden');
        shareSection.classList.add('hidden');
        gameSection.classList.remove('hidden');
        log("ゲームを開始しました！");
    });

    // --- ユーティリティ ---
    function copyShareUrl() {
        shareUrlInput.select();
        document.execCommand('copy');
        alert("招待URLをコピーしました！友達に送信してください。");
    }

    function escapeHtml(str) {
        return str.replace(/[&<>"']/g, function(m) {
            return { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[m];
        });
    }
</script>
</body>
</html>
