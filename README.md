<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>數獨解算器 (顏色區分版)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* 根元素和背景 */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f7f3ed; /* 柔和的淺米色背景 */
        }
        
        /* 數獨網格容器 */
        .sudoku-grid {
            border-width: 4px; /* 外部總邊框 */
            border-color: #4A443E; /* 深棕色邊框 */
            background-color: #F8F4EE; /* 淺米色格盤背景 */
            display: grid;
            grid-template-columns: repeat(9, 1fr);
            aspect-ratio: 1 / 1;
            box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
        }

        /* 單元格樣式 */
        .cell {
            padding-bottom: 100%; /* 確保是正方形 */
            line-height: 0;
            font-size: clamp(1rem, 5vw, 1.8rem); /* 響應式字體大小 */
            font-weight: 700;
            display: flex;
            align-items: center;
            justify-content: center;
            user-select: none;
            cursor: pointer;
            
            /* 預設邊界：細線，淺色 */
            border: 1px solid #D9D2C6; 
            margin: -1px; /* 邊界重疊處理 */
            position: relative;
            z-index: 10;
        }

        /* 初始謎題數字 (手動輸入或圖片辨識載入的線索) */
        .fixed-cell {
            background-color: #fcf8f0; 
            color: #1a1a1a; /* 深黑色 */
            font-weight: 700;
            cursor: default;
        }
        
        /* 使用者輸入的數字 (在解算前，使用者填入的數字) */
        .user-input-cell {
             color: #1a1a1a; /* 深黑色 */
             font-weight: 700;
        }

        /* AI 運算結果 (解算後填入的數字) */
        .solved-cell {
             color: #2563eb; /* 亮藍色 */
             font-weight: 700;
        }

        /* 選中單元格 */
        .selected-cell {
            background-color: #ffe0b2 !important; /* 柔和的橘色選擇色 */
            z-index: 20;
            border-color: #f59e0b !important;
        }

        /* 錯誤單元格 */
        .error-cell {
            background-color: #f8c0c0 !important; /* 柔和的紅色錯誤色 */
            color: #cc0000 !important;
        }
        
        /* 強化 3x3 區塊的邊界 */
        .cell:nth-child(3n):not(:nth-child(9n)) { 
            border-right-width: 3px !important; 
            border-right-color: #4A443E !important;
        }
        
        .sudoku-grid > div:nth-child(3n) .cell { 
             border-bottom-width: 3px !important; 
             border-bottom-color: #4A443E !important;
        }
        
        /* 確保文字置中並可見 */
        .cell span {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            line-height: 1; 
        }

        /* 數字鍵盤按鈕樣式 */
        .num-pad-btn {
            font-size: 1.5rem;
            padding: 0.75rem 0;
            transition: transform 0.1s, box-shadow 0.1s;
        }
        .num-pad-btn:active {
            transform: scale(0.95);
        }
        /* 載入指示器樣式 */
        .spinner {
            border: 4px solid rgba(0, 0, 0, 0.1);
            border-top: 4px solid #4A443E;
            border-radius: 50%;
            width: 20px;
            height: 20px;
            animation: spin 1s linear infinite;
            display: inline-block;
            margin-right: 8px;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
    <script>
        // API 配置
        const apiKey = ""; 
        const modelName = "gemini-2.5-flash-preview-09-2025";
        const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/${modelName}:generateContent?key=${apiKey}`;

        // 初始完全空白的數獨謎題 (0 代表空單元格)
        const emptyPuzzle = [
            [0, 0, 0, 0, 0, 0, 0, 0, 0], [0, 0, 0, 0, 0, 0, 0, 0, 0], [0, 0, 0, 0, 0, 0, 0, 0, 0],
            [0, 0, 0, 0, 0, 0, 0, 0, 0], [0, 0, 0, 0, 0, 0, 0, 0, 0], [0, 0, 0, 0, 0, 0, 0, 0, 0],
            [0, 0, 0, 0, 0, 0, 0, 0, 0], [0, 0, 0, 0, 0, 0, 0, 0, 0], [0, 0, 0, 0, 0, 0, 0, 0, 0]
        ];
        
        let currentBoard = [];
        let initialBoard = []; // 用於儲存初始謎題狀態 (黑色數字)
        let solvedCells = {}; // 新增：追蹤哪些是 AI 運算出來的數字 (藍色數字)
        let selectedRow = -1;
        let selectedCol = -1;

        // --- 核心數獨解算邏輯 ---

        /**
         * 檢查將數字 num 放置在 (row, col) 是否有效 (用於 UI 衝突檢查和 Solver)。
         */
        function isValid(board, row, col, num) {
            // 1. 檢查行 (排除當前位置的檢查)
            for (let c = 0; c < 9; c++) {
                if (c !== col && board[row][c] === num) return false;
            }

            // 2. 檢查列 (排除當前位置的檢查)
            for (let r = 0; r < 9; r++) {
                if (r !== row && board[r][col] === num) return false;
            }

            // 3. 檢查 3x3 的九宮格
            const startRow = Math.floor(row / 3) * 3;
            const startCol = Math.floor(col / 3) * 3;
            for (let r = startRow; r < startRow + 3; r++) {
                for (let c = startCol; c < startCol + 3; c++) {
                    if (r === row && c === col) continue;
                    if (board[r][c] === num) return false;
                }
            }
            return true;
        }
        
        /**
         * 專門用於回溯解算。
         */
        function solveSudoku(board) {
            for (let row = 0; row < 9; row++) {
                for (let col = 0; col < 9; col++) {
                    if (board[row][col] === 0) {
                        for (let num = 1; num <= 9; num++) {
                            // 這裡的 isValidForSolver 和 isValid 邏輯相同，但我們使用更簡潔的寫法
                            if (isSafe(board, row, col, num)) {
                                board[row][col] = num;

                                if (solveSudoku(board)) {
                                    return true;
                                }

                                board[row][col] = 0; // 回溯
                            }
                        }
                        return false;
                    }
                }
            }
            return true;
        }

        /**
         * Solver 專用的 safety check (無需排除自身，因為當前位置為 0)
         */
        function isSafe(board, row, col, num) {
            for (let c = 0; c < 9; c++) {
                if (board[row][c] === num) return false;
            }
            for (let r = 0; r < 9; r++) {
                if (board[r][col] === num) return false;
            }
            const startRow = Math.floor(row / 3) * 3;
            const startCol = Math.floor(col / 3) * 3;
            for (let r = startRow; r < startRow + 3; r++) {
                for (let c = startCol; c < startCol + 3; c++) {
                    if (board[r][c] === num) return false;
                }
            }
            return true;
        }
        
        // --- API & 輔助函數 ---

        /**
         * 將 File 對象轉換為 Base64 字符串
         */
        function fileToBase64(file) {
            return new Promise((resolve, reject) => {
                const reader = new FileReader();
                reader.readAsDataURL(file);
                reader.onload = () => {
                    const base64String = reader.result.split(',')[1];
                    resolve(base64String);
                };
                reader.onerror = error => reject(error);
            });
        }

        /**
         * 呼叫 Gemini API 辨識圖片中的數獨並解析結果。
         */
        async function recognizeSudokuFromImage() {
            const fileInput = document.getElementById('image-upload');
            const recognizeBtn = document.getElementById('recognize-btn');

            if (fileInput.files.length === 0) {
                showMessage('請先選擇一個數獨圖片檔案。', 'bg-red-100 text-red-800');
                return;
            }

            const imageFile = fileInput.files[0];
            const mimeType = imageFile.type;
            
            if (!mimeType.startsWith('image/')) {
                 showMessage('請上傳有效的圖片檔案。', 'bg-red-100 text-red-800');
                 return;
            }
            
            recognizeBtn.disabled = true;
            recognizeBtn.innerHTML = '<span class="spinner"></span> 正在辨識...';

            try {
                showMessage('🧠 正在上傳圖片並使用 Gemini 辨識數獨謎題...', 'bg-yellow-100 text-yellow-800');
                
                const base64ImageData = await fileToBase64(imageFile);
                
                const prompt = "Please analyze the image containing a Sudoku puzzle. Return the state of the 9x9 grid as a single JSON array of arrays, where '0' represents an empty cell, and 1-9 represent filled cells. DO NOT include any text, explanation, or markdown formatting outside of the JSON structure. Example format: [[8,0,0,0,0,0,0,0,0],[0,0,3,6,0,0,0,0,0],...].";
                
                const payload = {
                    contents: [
                        {
                            role: "user",
                            parts: [
                                { text: prompt },
                                {
                                    inlineData: {
                                        mimeType: mimeType,
                                        data: base64ImageData
                                    }
                                }
                            ]
                        }
                    ],
                    generationConfig: {
                        responseMimeType: "application/json",
                        responseSchema: {
                            type: "ARRAY",
                            items: {
                                type: "ARRAY",
                                items: {
                                    type: "INTEGER"
                                }
                            }
                        }
                    }
                };
                
                const maxRetries = 3;
                let response = null;
                for (let i = 0; i < maxRetries; i++) {
                    try {
                        const fetchResponse = await fetch(apiUrl, {
                            method: 'POST',
                            headers: { 'Content-Type': 'application/json' },
                            body: JSON.stringify(payload)
                        });
                        if (fetchResponse.ok) {
                            response = fetchResponse;
                            break;
                        }
                    } catch (error) {
                        console.error(`Attempt ${i + 1} failed (exponential backoff):`, error);
                        if (i < maxRetries - 1) {
                            await new Promise(resolve => setTimeout(resolve, Math.pow(2, i) * 1000));
                        }
                    }
                }
                
                if (!response || !response.ok) {
                    throw new Error("API 呼叫失敗或達到最大重試次數。");
                }

                const result = await response.json();
                
                const jsonText = result?.candidates?.[0]?.content?.parts?.[0]?.text;
                if (!jsonText) {
                    throw new Error("API 返回的 JSON 內容為空，請確認圖片中包含數獨。");
                }

                const puzzleArray = JSON.parse(jsonText);
                
                if (Array.isArray(puzzleArray) && puzzleArray.length === 9 && puzzleArray.every(row => Array.isArray(row) && row.length === 9)) {
                    // 驗證通過，更新數獨盤
                    const newPuzzle = puzzleArray.map(row => row.map(cell => Math.max(0, Math.min(9, parseInt(cell) || 0))));
                    
                    // 確保在載入新謎題時，solvedCells 被重置
                    solvedCells = {};
                    initializeBoard(newPuzzle);
                    showMessage('✅ 圖片中的數獨謎題已成功載入! 現在可以點擊解算。', 'bg-green-100 text-green-800');
                } else {
                    throw new Error("解析後的資料結構不符合 9x9 數獨格式。");
                }

            } catch (error) {
                console.error("圖像辨識或解析錯誤:", error);
                showMessage(`❌ 圖像辨識失敗: ${error.message}. 請檢查圖片是否清晰。`, 'bg-red-100 text-red-800');
            } finally {
                recognizeBtn.disabled = false;
                recognizeBtn.innerHTML = '辨識並載入謎題 📸';
            }
        }


        // --- UI 渲染與互動邏輯 ---

        /**
         * 初始化數獨板的資料和 UI。
         */
        function initializeBoard(puzzle) {
            // 深複製謎題作為當前狀態和初始狀態 (此時的謎題就是初始線索)
            currentBoard = puzzle.map(row => [...row]);
            initialBoard = puzzle.map(row => [...row]); 
            
            selectedRow = -1; 
            selectedCol = -1;
            renderBoard();
        }

        /**
         * 渲染數獨板 UI。
         */
        function renderBoard() {
            const gridContainer = document.getElementById('sudoku-grid');
            gridContainer.innerHTML = ''; // 清空舊內容
            
            for (let r = 0; r < 9; r++) {
                for (let c = 0; c < 9; c++) {
                    const cellValue = currentBoard[r][c];
                    const cellKey = `${r}-${c}`;

                    const cellDiv = document.createElement('div');
                    cellDiv.className = 'cell relative';
                    cellDiv.setAttribute('data-row', r);
                    cellDiv.setAttribute('data-col', c);
                    
                    // 判斷是否為初始謎題數字 (手動或圖片載入的)
                    const isInitialClue = initialBoard[r][c] !== 0; 
                    // 判斷是否為 AI 解算出來的數字
                    const isSolvedByAI = solvedCells[cellKey];

                    
                    if (isInitialClue || isSolvedByAI) {
                        // 初始謎題和 AI 解算結果都不可編輯
                        cellDiv.classList.add('fixed-cell');
                        cellDiv.onclick = () => showMessage('這是謎題的初始線索或 AI 解算結果，不能編輯!', 'bg-red-100 text-red-800');
                        
                        if (isSolvedByAI) {
                            // AI 解算結果使用藍色
                            cellDiv.classList.add('solved-cell');
                        } else {
                            // 初始線索使用深黑色
                            cellDiv.classList.add('user-input-cell');
                        }

                    } else {
                        // 空白或使用者在解算前填入的數字，可點擊選中
                        cellDiv.onclick = () => selectCell(r, c);
                        if (cellValue !== 0) {
                           // 尚未解算，使用者填入的數字也用深黑色
                           cellDiv.classList.add('user-input-cell'); 
                        }
                    }
                    
                    if (r === selectedRow && c === selectedCol) {
                         cellDiv.classList.add('selected-cell');
                    }
                    
                    if (cellValue !== 0) {
                        cellDiv.innerHTML = `<span>${cellValue}</span>`;
                    } else {
                        cellDiv.innerHTML = `<span></span>`; 
                    }
                    
                    // 檢查單元格是否在當前的板子上造成衝突
                    if (cellValue !== 0 && !isValid(currentBoard, r, c, cellValue)) {
                        cellDiv.classList.add('error-cell');
                    }
                    
                    gridContainer.appendChild(cellDiv);
                }
            }
        }
        
        /**
         * 處理單元格選擇
         */
        function selectCell(r, c) {
             const cellKey = `${r}-${c}`;
             
             // 確保選中的不是 AI 解算結果或初始線索
             if (initialBoard[r][c] !== 0 || solvedCells[cellKey]) {
                showMessage('這個單元格不能被編輯!', 'bg-red-100 text-red-800');
                return;
             }

             if (selectedRow === r && selectedCol === c) {
                selectedRow = -1;
                selectedCol = -1;
            } else {
                selectedRow = r;
                selectedCol = c;
            }
            renderBoard();
        }

        /**
         * 處理數字鍵盤輸入。
         */
        function handleNumberInput(num) {
            if (selectedRow === -1 || selectedCol === -1) {
                showMessage('請先點擊一個空白單元格!', 'bg-yellow-100 text-yellow-800');
                return;
            }

            const r = selectedRow;
            const c = selectedCol;
            const cellKey = `${r}-${c}`;
            
            // 檢查是否是初始線索或 AI 結果
            if (initialBoard[r][c] !== 0 || solvedCells[cellKey]) {
                showMessage('這是謎題初始數字或 AI 結果，不能修改!', 'bg-red-100 text-red-800');
                return;
            }
            
            // 0 代表清除
            if (num === 0) {
                currentBoard[r][c] = 0;
            } else {
                currentBoard[r][c] = num;
            }
            
            // 輸入後取消選擇，並重新渲染
            selectedRow = -1; 
            selectedCol = -1;
            renderBoard();
            
            showMessage('數字已輸入，您可以繼續或點擊解算。', 'bg-blue-100 text-blue-800');
        }

        /**
         * 執行解算。
         */
        function handleSolve() {
            // 清除之前的錯誤標記
            selectedRow = -1;
            selectedCol = -1;
            renderBoard(); 

            // 檢查用戶輸入是否有衝突 
            for (let r = 0; r < 9; r++) {
                for (let c = 0; c < 9; c++) {
                    if (currentBoard[r][c] !== 0 && !isValid(currentBoard, r, c, currentBoard[r][c])) {
                        showMessage('❌ 您的數獨輸入存在規則衝突，請檢查紅色的錯誤單元格!', 'bg-red-100 text-red-800');
                        return; 
                    }
                }
            }

            // 複製一份板子進行解算
            const solvedBoard = currentBoard.map(row => [...row]);
            
            showMessage('🔍 正在使用回溯法解算中...', 'bg-yellow-100 text-yellow-800');
            
            // 輕微延遲以確保 UI 更新
            setTimeout(() => {
                const startTime = performance.now();
                if (solveSudoku(solvedBoard)) {
                    // 只有成功才更新 currentBoard
                    currentBoard = solvedBoard; 
                    
                    // 紀錄 AI 填入的數字
                    solvedCells = {};
                    for (let r = 0; r < 9; r++) {
                        for (let c = 0; c < 9; c++) {
                            // 如果 initialBoard 是 0，但 currentBoard 有數字，表示是 AI 填入的
                            if (initialBoard[r][c] === 0 && currentBoard[r][c] !== 0) {
                                solvedCells[`${r}-${c}`] = true;
                            }
                        }
                    }

                    renderBoard(true);
                    const endTime = performance.now();
                    const duration = (endTime - startTime).toFixed(2);
                    showMessage(`✅ 數獨已成功解開! 耗時 ${duration} 毫秒。`, 'bg-green-100 text-green-800');
                } else {
                    showMessage('❌ 這個數獨無解。', 'bg-red-100 text-red-800');
                }
            }, 10); 
        }

        /**
         * 重置數獨到初始狀態 (使用者輸入的謎題，清空 AI 結果)。
         */
        function handleReset() {
            // 重置 currentBoard 為 initialBoard (謎題狀態)
            currentBoard = initialBoard.map(row => [...row]);
            // 清空 AI 標記
            solvedCells = {};
            
            // 重新渲染並顯示訊息
            renderBoard();
            showMessage('🔄 數獨已重置為初始謎題狀態 (所有 AI 結果已清除)。', 'bg-blue-100 text-blue-800');
        }
        
        /**
         * 顯示訊息
         */
        function showMessage(text, className) {
            const messageDiv = document.getElementById('message');
            messageDiv.innerHTML = text;
            messageDiv.className = `p-3 mt-4 text-sm rounded-lg text-center shadow-md ${className}`;
        }


        // 頁面加載完成後執行初始化
        window.onload = () => {
            // 使用完全空白的謎題進行初始化
            initializeBoard(emptyPuzzle);
            showMessage('✏️ 請手動輸入謎題，或上傳圖片進行自動識別。', 'bg-blue-100 text-blue-800');

            // 設置數字鍵盤的點擊事件
            document.querySelectorAll('.num-pad-btn').forEach(button => {
                button.addEventListener('click', () => {
                    const num = parseInt(button.getAttribute('data-num'));
                    handleNumberInput(num);
                });
            });

            // 設置控制按鈕的點擊事件
            document.getElementById('solve-btn').addEventListener('click', handleSolve);
            document.getElementById('reset-btn').addEventListener('click', handleReset);
            
            // 設置圖像識別按鈕的點擊事件
            document.getElementById('recognize-btn').addEventListener('click', recognizeSudokuFromImage);
        };
    </script>
</head>
<body class="min-h-screen flex flex-col items-center p-4">
    
    <h1 class="text-3xl font-extrabold text-gray-800 mb-2">數獨解算器</h1>
    <p class="text-sm text-gray-600 mb-4">回溯演算法手機應用程式</p>

    <!-- 數獨網格容器 -->
    <div class="w-full max-w-sm">
        <div id="sudoku-grid" class="sudoku-grid rounded-lg">
            <!-- 數獨單元格將由 JavaScript 渲染 -->
        </div>
    </div>
    
    <!-- 訊息區域 -->
    <div id="message" class="w-full max-w-sm p-3 mt-4 text-sm rounded-lg text-center shadow-md bg-blue-100 text-blue-800">
        ✏️ 請手動輸入謎題，或上傳圖片進行自動識別。
    </div>
    
    <!-- 圖像識別與輸入 -->
    <div class="w-full max-w-sm mt-6 p-4 bg-white rounded-xl shadow-2xl">
        <h2 class="text-xl font-bold text-gray-800 mb-3">從圖像輸入數獨 (Gemini 驅動)</h2>
        <input type="file" id="image-upload" accept="image/*" class="w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-full file:border-0 file:text-sm file:font-semibold file:bg-indigo-50 file:text-indigo-700 hover:file:bg-indigo-100" />
        <button id="recognize-btn" class="w-full mt-3 px-4 py-3 bg-blue-500 hover:bg-blue-600 text-white font-bold rounded-xl shadow-lg transform transition duration-150 hover:scale-[1.02]">
            辨識並載入謎題 📸
        </button>
    </div>

    <!-- 數字輸入鍵盤 -->
    <div class="w-full max-w-sm mt-6 p-4 bg-white rounded-xl shadow-2xl">
        <div class="grid grid-cols-5 gap-3">
            <!-- 數字 1-9 -->
            <button class="num-pad-btn bg-indigo-500 hover:bg-indigo-600 text-white rounded-lg shadow-md" data-num="1">1</button>
            <button class="num-pad-btn bg-indigo-500 hover:bg-indigo-600 text-white rounded-lg shadow-md" data-num="2">2</button>
            <button class="num-pad-btn bg-indigo-500 hover:bg-indigo-600 text-white rounded-lg shadow-md" data-num="3">3</button>
            <button class="num-pad-btn bg-red-500 hover:bg-red-600 text-white rounded-lg shadow-md col-span-2" data-num="0">
                <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6 inline-block" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
                  <path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                </svg>
                清除
            </button>
            
            <button class="num-pad-btn bg-indigo-500 hover:bg-indigo-600 text-white rounded-lg shadow-md" data-num="4">4</button>
            <button class="num-pad-btn bg-indigo-500 hover:bg-indigo-600 text-white rounded-lg shadow-md" data-num="5">5</button>
            <button class="num-pad-btn bg-indigo-500 hover:bg-indigo-600 text-white rounded-lg shadow-md" data-num="6">6</button>

            <div class="col-span-2"></div>
            
            <button class="num-pad-btn bg-indigo-500 hover:bg-indigo-600 text-white rounded-lg shadow-md" data-num="7">7</button>
            <button class="num-pad-btn bg-indigo-500 hover:bg-indigo-600 text-white rounded-lg shadow-md" data-num="8">8</button>
            <button class="num-pad-btn bg-indigo-500 hover:bg-indigo-600 text-white rounded-lg shadow-md" data-num="9">9</button>

            <div class="col-span-2"></div>

        </div>

        <!-- 控制按鈕 -->
        <div class="mt-5 flex space-x-4">
            <button id="solve-btn" class="flex-1 px-4 py-3 bg-green-500 hover:bg-green-600 text-white font-bold rounded-xl shadow-lg transform transition duration-150 hover:scale-[1.02]">
                一鍵解算 🤖
            </button>
            <button id="reset-btn" class="flex-1 px-4 py-3 bg-gray-500 hover:bg-gray-600 text-white font-bold rounded-xl shadow-lg transform transition duration-150 hover:scale-[1.02]">
                重置 🔄
            </button>
        </div>
    </div>
    
    <div class="h-8"></div> <!-- 底部間距 -->
</body>
</html>
