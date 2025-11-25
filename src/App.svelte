<script lang="ts">
    import { onMount, onDestroy, tick } from "svelte";
    import Chart from "chart.js/auto";

    // --- Types ---
    type GamePhase =
        | "SETUP"
        | "PREMISE_MEMORIZE"
        | "INTERFERENCE"
        | "QUESTION"
        | "RESULT"
        | "SESSION_END";
    
    type RftMode = "LINEAR" | "SPATIAL_2D" | "SPATIAL_3D" | "HIERARCHY" | "DISTINCTION";
    type SymbolMode = "EMOJI" | "WORDS" | "VORONOI" | "MIXED";

    // Configuration
    interface GameSettings {
        activeModes: Record<RftMode, boolean>;
        numPremises: number;
        autoProgress: boolean;
        
        useQuestionTimer: boolean;
        questionTimeLimit: number;
        sessionLengthMinutes: number;
        disableSessionTimer: boolean;

        blindMode: boolean;
        symbolMode: SymbolMode;
        
        enableDeictic: boolean;        
        enableTransformation: boolean; 
        enableInterference: boolean;
        enableCipher: boolean;
        enableMovement: boolean;
    }

    interface GameHistory {
        date: string;
        timestamp: number;
        totalScore: number;
        accuracy: number;
        questionsAnswered: number;
        highestDepth: number;
        avgReactionTime: number; 
        reactionTimeByDepth: Record<number, number>; 
        sessionDurationSeconds: number;
        activeModes: string[]; 
        activeModifiers: string[];
    }

    interface SessionRecord {
        id: number;
        mode: string;
        premises: string[];
        question: string;
        userAnswer: string;
        correctAnswer: string;
        isCorrect: boolean;
        reactionTimeMs: number;
        modifiers: string[];
    }

    // --- Constants ---
    const EMOJIS = ["🚀", "💎", "🔥", "🌊", "⚡", "🍄", "👁️", "🎲", "🧬", "🔮", "⚓", "🪐", "🌋", "🦠", "🌌", "💊", "🧿", "🧩", "🧸", "💣"];
    const NONSENSE_CONSONANTS = "BCDFGHJKLMNPQRSTVWXYZ";
    const NONSENSE_VOWELS = "AEIOU";
    const CIPHER_WORDS = [
        "ZAX", "JOP", "KIV", "LUZ", "MEC", "VEX", "QOD", "WIB", "HAF", "GUK", 
        "YIN", "BEX", "DUB", "ROZ", "NIX", "POK", "VOM", "JEX", "KAZ", "QUZ", 
        "YEP", "WUX", "FIP", "GOZ"
    ];
    
    // --- State ---
    let phase: GamePhase = "SETUP";
    
    let settings: GameSettings = {
        activeModes: { LINEAR: true, DISTINCTION: true, SPATIAL_2D: true, SPATIAL_3D: false, HIERARCHY: true },
        numPremises: 2,
        autoProgress: true,
        useQuestionTimer: true,
        questionTimeLimit: 15,
        sessionLengthMinutes: 5,
        disableSessionTimer: false,
        blindMode: false,
        symbolMode: "EMOJI",
        enableDeictic: false,
        enableTransformation: false,
        enableInterference: false,
        enableCipher: false,
        enableMovement: false 
    };

    let premises: string[] = [];
    let currentQuestion = "";
    let expectedAnswer = false;
    let currentRoundMode: RftMode = "LINEAR";
    
    let contextIsNight = false; 
    let activeModifiers: string[] = [];
    
    // Interference State
    let interferenceTargetColor = "";
    let interferenceCurrentColor = "";
    let interferenceStatus = "WAIT"; // WAIT, HIT, MISS
    
    let currentCipherMap: Record<string, string> = {};
    let cipherHasChanged = false;

    let currentScore = 0;
    let currentStreak = 0;
    let maxStreak = 0;
    let questionsAttempted = 0;
    let questionsCorrect = 0;
    let maxDepthReached = 2;
    
    let questionStartTime = 0;
    let totalReactionTime = 0;
    let reactionTimesByDepth: Record<number, number[]> = {}; 

    let history: GameHistory[] = [];
    let sessionLog: SessionRecord[] = [];

    let sessionSecondsRemaining = 0;
    let sessionSecondsElapsed = 0;
    let sessionInterval: any;
    let questionSecondsRemaining = 0;
    let questionInterval: any;
    let interferenceInterval: any;

    let isYesRight = true;
    let feedbackMsg = "";
    
    let showSettings = false;
    let showReviewModal = false;
    let showHistoryModal = false;

    let chartCanvas: HTMLCanvasElement;
    let chartInstance: Chart | null = null;

    onMount(() => {
        const storedHist = localStorage.getItem("rft_architect_history");
        if (storedHist) { try { history = JSON.parse(storedHist); } catch (e) { console.error(e); } }

        const storedSettings = localStorage.getItem("rft_architect_settings");
        if (storedSettings) {
            try { 
                const parsed = JSON.parse(storedSettings);
                const defaultModes = { LINEAR: true, DISTINCTION: true, SPATIAL_2D: true, SPATIAL_3D: false, HIERARCHY: true };
                settings = { ...settings, ...parsed, activeModes: { ...defaultModes, ...(parsed.activeModes || {}) } };
            } catch (e) { console.error(e); }
        }
    });

    onDestroy(() => {
        clearInterval(sessionInterval);
        clearInterval(questionInterval);
        clearInterval(interferenceInterval);
        if (chartInstance) chartInstance.destroy();
    });

    function saveSettings() { localStorage.setItem("rft_architect_settings", JSON.stringify(settings)); }

    // --- CHARTING ---
    async function renderChart() {
        await tick();
        if (!chartCanvas) return;
        if (chartInstance) chartInstance.destroy();
        const sortedHistory = [...history].sort((a, b) => (a.timestamp || 0) - (b.timestamp || 0));
        const chartData = sortedHistory.slice(-20);
        if (chartData.length === 0) return;

        const labels = chartData.map(h => h.date.split(" ")[0]);
        const accuracies = chartData.map(h => h.accuracy);
        const scores = chartData.map(h => h.totalScore);

        chartInstance = new Chart(chartCanvas, {
            type: 'line',
            data: {
                labels: labels,
                datasets: [
                    {
                        label: 'Accuracy (%)',
                        data: accuracies,
                        borderColor: '#00ff9d',
                        backgroundColor: 'rgba(0, 255, 157, 0.1)',
                        borderWidth: 2,
                        tension: 0.3,
                        yAxisID: 'y',
                        pointBackgroundColor: '#fff',
                        pointRadius: 4
                    },
                    {
                        label: 'Score',
                        data: scores,
                        borderColor: '#00f2ff',
                        backgroundColor: 'rgba(0, 242, 255, 0.1)',
                        borderWidth: 2,
                        borderDash: [5, 5],
                        tension: 0.3,
                        yAxisID: 'y1',
                        pointRadius: 0
                    }
                ]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                interaction: { mode: 'index', intersect: false },
                plugins: {
                    legend: { labels: { color: '#fff' } },
                    tooltip: {
                        backgroundColor: 'rgba(0,0,0,0.9)',
                        titleColor: '#00ff9d',
                        bodyColor: '#fff'
                    }
                },
                scales: {
                    x: { grid: { color: '#444' }, ticks: { color: '#ccc' } },
                    y: { type: 'linear', display: true, position: 'left', min: 0, max: 100, grid: { color: '#333' }, ticks: { color: '#ccc' } },
                    y1: { type: 'linear', display: true, position: 'right', grid: { drawOnChartArea: false }, ticks: { color: '#ccc' } }
                }
            }
        });
    }

    $: if (showHistoryModal && history.length > 0) renderChart();

    // --- GENERATORS ---
    function generateWord(): string { 
        const c1 = NONSENSE_CONSONANTS[Math.floor(Math.random() * NONSENSE_CONSONANTS.length)];
        const v1 = NONSENSE_VOWELS[Math.floor(Math.random() * NONSENSE_VOWELS.length)];
        const c2 = NONSENSE_CONSONANTS[Math.floor(Math.random() * NONSENSE_CONSONANTS.length)];
        const v2 = NONSENSE_VOWELS[Math.floor(Math.random() * NONSENSE_VOWELS.length)];
        return `<span class="symbol-text">${c1}${v1}${c2}${v2}</span>`;
    }
    function generateVoronoi(): string { 
         const hue = Math.floor(Math.random() * 360);
        const points = [];
        const numPoints = 5 + Math.floor(Math.random() * 4);
        for(let i=0; i<numPoints; i++) {
            const angle = (i / numPoints) * Math.PI * 2;
            const r = 30 + Math.random() * 20;
            const x = 50 + Math.cos(angle) * r;
            const y = 50 + Math.sin(angle) * r;
            points.push(`${x},${y}`);
        }
        return `<span class="symbol-wrap"><svg viewBox="0 0 100 100" class="voronoi-svg"><polygon points="${points.join(" ")}" fill="hsl(${hue}, 70%, 60%)" stroke="white" stroke-width="2"/></svg></span>`;
    }
    function generateSymbols(count: number): string[] {
        const symbols: string[] = [];
        const shuffledEmojis = [...EMOJIS].sort(() => Math.random() - 0.5);
        for(let i=0; i<count; i++) {
            let mode = settings.symbolMode;
            if (mode === "MIXED") {
                const roll = Math.random();
                if (roll < 0.33) mode = "EMOJI"; else if (roll < 0.66) mode = "WORDS"; else mode = "VORONOI";
            }
            if (mode === "WORDS") symbols.push(generateWord());
            else if (mode === "VORONOI") symbols.push(generateVoronoi());
            else symbols.push(`<span class="symbol-emoji">${shuffledEmojis[i % shuffledEmojis.length]}</span>`);
        }
        return symbols;
    }
    
    function shuffleArray<T>(array: T[]): T[] { return array.sort(() => Math.random() - 0.5); }
    function getDistinctRandomIndices(count: number): [number, number] {
        let a = Math.floor(Math.random() * count);
        let b = Math.floor(Math.random() * count);
        while (a === b) b = Math.floor(Math.random() * count);
        return [a, b];
    }

    // --- CIPHER LOGIC ---
    function generateCipherKey() {
        const relations = [
            "GREATER", "LESS", "SAME", "DIFFERENT", "CONTAINS", "INSIDE", 
            "NORTH", "SOUTH", "EAST", "WEST", "ABOVE", "BELOW",
            "LEFT", "RIGHT", "FRONT", "BEHIND",
            "START", "FACE", "WALK", "TURN"
        ];
        const shuffledWords = shuffleArray([...CIPHER_WORDS]);
        currentCipherMap = {};
        relations.forEach((rel, i) => {
            currentCipherMap[rel] = shuffledWords[i % shuffledWords.length];
        });
    }

    function makePremise(a: string, relation: string, b: string): string {
        if (settings.enableCipher) {
             const lookup = relation.replace("is ", "").replace(" than", "").replace(" of", "").replace("the ", "").replace(" as", "").replace("from", "").trim();
             if (currentCipherMap[lookup]) return `${a} <b>${currentCipherMap[lookup]}</b> ${b}`;
        }
        return `${a} ${relation} ${b}`;
    }

    // --- ENGINES (Logic Modules) ---
    function generateLinear() {
        const count = settings.numPremises + 1;
        const items = generateSymbols(count);
        const generatedPremises: string[] = [];
        for (let i = 0; i < count - 1; i++) {
            const a = items[i];
            const b = items[i + 1];
            if (Math.random() > 0.5) generatedPremises.push(makePremise(a, "is GREATER than", b));
            else generatedPremises.push(makePremise(b, "is LESS than", a));
        }
        const [idxA, idxB] = getDistinctRandomIndices(count);
        const qType = Math.random() > 0.5 ? "GREATER" : "LESS";
        const qRel = settings.enableCipher ? currentCipherMap[qType] : qType;
        currentQuestion = `Is ${items[idxA]} ${qRel} ${settings.enableCipher ? '' : 'than'} ${items[idxB]}?`;
        const actuallyGreater = idxA < idxB;
        if (qType === "GREATER") expectedAnswer = actuallyGreater; else expectedAnswer = !actuallyGreater;
        premises = shuffleArray(generatedPremises);
    }

    function generateDistinction() {
        const count = settings.numPremises + 1;
        const items = generateSymbols(count);
        const values: number[] = [];
        const generatedPremises: string[] = [];
        values[0] = Math.random() > 0.5 ? 1 : 0;
        for (let i = 0; i < count - 1; i++) {
            const isSame = Math.random() > 0.5;
            values.push(isSame ? values[i] : 1 - values[i]);
            const relText = isSame ? "SAME" : "DIFFERENT";
            generatedPremises.push(makePremise(items[i], relText, items[i + 1]));
        }
        const [idxA, idxB] = getDistinctRandomIndices(count);
        const areActuallySame = values[idxA] === values[idxB];
        const qType = Math.random() > 0.5 ? "SAME" : "DIFFERENT";
        const qRel = settings.enableCipher ? currentCipherMap[qType] : (qType === "SAME" ? "the SAME as" : "DIFFERENT from");
        currentQuestion = `Is ${items[idxA]} ${qRel} ${items[idxB]}?`;
        expectedAnswer = qType === "SAME" ? areActuallySame : !areActuallySame;
        premises = shuffleArray(generatedPremises);
    }

    function generateHierarchy() {
        const count = settings.numPremises + 1;
        const items = generateSymbols(count);
        const generatedPremises: string[] = [];
        for (let i = 0; i < count - 1; i++) { generatedPremises.push(makePremise(items[i], "CONTAINS", items[i+1])); }
        const [idxA, idxB] = getDistinctRandomIndices(count);
        const actuallyInside = idxA > idxB; 
        if (Math.random() > 0.5) { 
            const qRel = settings.enableCipher ? currentCipherMap["INSIDE"] : "inside";
            currentQuestion = `Is ${items[idxA]} ${qRel} ${items[idxB]}?`; 
            expectedAnswer = actuallyInside; 
        } else { 
            const qRel = settings.enableCipher ? currentCipherMap["CONTAINS"] : "contain";
            currentQuestion = `Does ${items[idxA]} ${qRel} ${items[idxB]}?`; 
            expectedAnswer = idxA < idxB; 
        }
        premises = shuffleArray(generatedPremises);
    }

    function generateSpatial(is3D: boolean) {
        const items = generateSymbols(settings.numPremises + 1);
        const positions: {x:number, y:number, z:number}[] = [];
        const generatedPremises: string[] = [];
        let cx = 0, cy = 0, cz = 0;
        positions.push({ x: 0, y: 0, z: 0 });

        for (let i = 0; i < settings.numPremises; i++) {
            const a = items[i];
            const b = items[i + 1];
            const options = is3D ? 6 : 4;
            const dir = Math.floor(Math.random() * options);
            let dx = 0, dy = 0, dz = 0, text = "";
            switch (dir) {
                case 0: dy = 1; text = "NORTH"; break;
                case 1: dy = -1; text = "SOUTH"; break;
                case 2: dx = 1; text = "EAST"; break;
                case 3: dx = -1; text = "WEST"; break;
                case 4: dz = 1; text = "ABOVE"; break;
                case 5: dz = -1; text = "BELOW"; break;
            }
            generatedPremises.push(makePremise(b, text, a));
            cx += dx; cy += dy; cz += dz;
            positions.push({ x: cx, y: cy, z: cz });
        }
        premises = shuffleArray(generatedPremises);

        if (settings.enableMovement && !is3D && Math.random() > 0.3) {
            activeModifiers.push("MOVEMENT");
            const startIdx = Math.floor(Math.random() * items.length);
            let curX = positions[startIdx].x;
            let curY = positions[startIdx].y;
            let heading = Math.floor(Math.random() * 4);
            const headingNames = ["NORTH", "EAST", "SOUTH", "WEST"];
            let instructions = [];
            const cStart = settings.enableCipher ? currentCipherMap["START"] : "Start at";
            const cFace = settings.enableCipher ? currentCipherMap["FACE"] : "Face";
            const hName = settings.enableCipher ? currentCipherMap[headingNames[heading]] : headingNames[heading];
            instructions.push(`${cStart} ${items[startIdx]}. ${cFace} ${hName}.`);

            const moves = 2 + Math.floor(Math.random() * 2);
            const cWalk = settings.enableCipher ? currentCipherMap["WALK"] : "Walk";
            const cTurn = settings.enableCipher ? currentCipherMap["TURN"] : "Turn";
            const cLeft = settings.enableCipher ? currentCipherMap["LEFT"] : "LEFT";
            const cRight = settings.enableCipher ? currentCipherMap["RIGHT"] : "RIGHT";

            for(let m=0; m<moves; m++) {
                const action = Math.random();
                if (action < 0.5) {
                    if(heading === 0) curY++; else if(heading === 1) curX++; else if(heading === 2) curY--; else if(heading === 3) curX--;
                    instructions.push(`${cWalk} 1.`);
                } else {
                    if (Math.random() > 0.5) { heading = (heading + 1) % 4; instructions.push(`${cTurn} ${cRight}.`); } 
                    else { heading = (heading + 3) % 4; instructions.push(`${cTurn} ${cLeft}.`); }
                }
            }
            const targetIdx = Math.floor(Math.random() * items.length);
            const targetPos = positions[targetIdx];
            const relX = targetPos.x - curX;
            const relY = targetPos.y - curY;

            if (relX === 0 && relY === 0) {
                currentQuestion = `${instructions.join(" ")} <br><br> Are you at ${items[targetIdx]}?`;
                expectedAnswer = true;
                return;
            }
            let localX = 0, localY = 0;
            if (heading === 0) { localX = relX; localY = relY; }
            else if (heading === 1) { localX = relY; localY = -relX; }
            else if (heading === 2) { localX = -relX; localY = -relY; }
            else if (heading === 3) { localX = -relY; localY = relX; }

            let trueRel = "";
            if (localY > 0 && Math.abs(localX) <= localY) trueRel = "FRONT";
            else if (localY < 0 && Math.abs(localX) <= Math.abs(localY)) trueRel = "BEHIND";
            else if (localX > 0) trueRel = "RIGHT";
            else if (localX < 0) trueRel = "LEFT";
            
            const qTypeRaw = ["FRONT", "BEHIND", "LEFT", "RIGHT"][Math.floor(Math.random()*4)];
            const qType = settings.enableCipher ? currentCipherMap[qTypeRaw] : qTypeRaw;
            currentQuestion = `${instructions.join(" ")} <br><br> Is ${items[targetIdx]} to your ${qType}?`;
            expectedAnswer = (trueRel === qTypeRaw);

        } else if (settings.enableDeictic && !is3D && Math.random() > 0.4) {
            activeModifiers.push("DEICTIC");
            let indices = [0,0,0].map(() => Math.floor(Math.random() * items.length));
            while (indices[0] === indices[1] || indices[1] === indices[2] || indices[0] === indices[2]) {
                indices = [0,0,0].map(() => Math.floor(Math.random() * items.length));
            }
            const [uIdx, fIdx, tIdx] = indices;
            const uPos = positions[uIdx], fPos = positions[fIdx], tPos = positions[tIdx];
            const fVec = { x: fPos.x - uPos.x, y: fPos.y - uPos.y };
            const tVec = { x: tPos.x - uPos.x, y: tPos.y - uPos.y };
            const cross = fVec.x * tVec.y - fVec.y * tVec.x;
            const dot = fVec.x * tVec.x + fVec.y * tVec.y;
            let relation = "";
            if (cross > 0.1) relation = "LEFT"; else if (cross < -0.1) relation = "RIGHT"; else if (dot > 0) relation = "FRONT"; else relation = "BEHIND";
            if (relation === "") { generateSpatial(false); return; }
            const qTypeRaw = Math.random() > 0.5 ? "LEFT" : "RIGHT";
            const qRel = settings.enableCipher ? currentCipherMap[qTypeRaw] : qTypeRaw;
            currentQuestion = `If you are at ${items[uIdx]} facing ${items[fIdx]}, is ${items[tIdx]} to your ${qRel}?`;
            expectedAnswer = (relation === qTypeRaw);
        } else {
            const [idxA, idxB] = getDistinctRandomIndices(items.length);
            const diffX = positions[idxB].x - positions[idxA].x;
            const diffY = positions[idxB].y - positions[idxA].y;
            const diffZ = positions[idxB].z - positions[idxA].z;
            let parts = [];
            if (is3D) { if (diffZ > 0) parts.push("ABOVE"); else if (diffZ < 0) parts.push("BELOW"); }
            let compass = "";
            if (diffY > 0) compass += "NORTH"; else if (diffY < 0) compass += "SOUTH";
            if (diffX > 0) compass += (compass ? "-" : "") + "EAST"; else if (diffX < 0) compass += (compass ? "-" : "") + "WEST";
            if (compass) parts.push(compass);
            let trueRel = "at the SAME LOCATION as";
            if (parts.length > 0) {
                if (settings.enableCipher && parts.length === 1 && currentCipherMap[parts[0]]) trueRel = currentCipherMap[parts[0]];
                else trueRel = parts.join(" and ") + (parts[0].includes("SAME") ? "" : " of");
            }
            const fakeDirs = is3D ? ["NORTH", "SOUTH", "EAST", "WEST", "ABOVE", "BELOW"] : ["NORTH", "SOUTH", "EAST", "WEST"];
            const rawFake = fakeDirs[Math.floor(Math.random() * fakeDirs.length)];
            const fake = settings.enableCipher ? currentCipherMap[rawFake] : (rawFake + " of");
            if (Math.random() > 0.5) { currentQuestion = `Is ${items[idxB]} ${trueRel} ${items[idxA]}?`; expectedAnswer = true; } 
            else { const qText = fake === trueRel ? "NOT " + fake : fake; currentQuestion = `Is ${items[idxB]} ${qText} ${items[idxA]}?`; expectedAnswer = false; }
        }
    }

    // --- SESSION LOGIC ---
    function startSession() {
        currentScore = 0; currentStreak = 0; questionsAttempted = 0; questionsCorrect = 0;
        maxDepthReached = settings.numPremises; sessionLog = []; totalReactionTime = 0; reactionTimesByDepth = {};
        sessionSecondsElapsed = 0; sessionSecondsRemaining = settings.sessionLengthMinutes * 60;
        generateCipherKey(); 
        clearInterval(sessionInterval);
        sessionInterval = setInterval(() => {
            sessionSecondsElapsed++;
            if (!settings.disableSessionTimer) {
                sessionSecondsRemaining--; 
                if (sessionSecondsRemaining <= 0) endSession();
            }
        }, 1000);
        startRound();
    }

    function endSession() {
        clearInterval(sessionInterval); clearInterval(questionInterval); clearInterval(interferenceInterval);
        const avgRT = questionsAttempted > 0 ? Math.round(totalReactionTime / questionsAttempted) : 0;
        const rtByDepth: Record<number, number> = {};
        for (const [d, times] of Object.entries(reactionTimesByDepth)) {
            const sum = times.reduce((a,b) => a+b, 0);
            rtByDepth[parseInt(d)] = Math.round(sum / times.length);
        }
        const activeM = Object.keys(settings.activeModes).filter(k => settings.activeModes[k as RftMode]);
        const activeMods = [];
        if (settings.blindMode) activeMods.push("BLIND");
        if (settings.enableCipher) activeMods.push("CIPHER");
        if (settings.enableDeictic) activeMods.push("DEICTIC");
        if (settings.enableMovement) activeMods.push("MOVE");
        if (settings.enableInterference) activeMods.push("INTERFERE");

        const histRecord: GameHistory = {
            date: new Date().toLocaleDateString() + " " + new Date().toLocaleTimeString(),
            timestamp: Date.now(),
            totalScore: currentScore,
            questionsAnswered: questionsAttempted,
            accuracy: questionsAttempted > 0 ? Math.round((questionsCorrect / questionsAttempted) * 100) : 0,
            highestDepth: maxDepthReached,
            avgReactionTime: avgRT,
            reactionTimeByDepth: rtByDepth,
            sessionDurationSeconds: sessionSecondsElapsed,
            activeModes: activeM,
            activeModifiers: activeMods
        };
        history = [...history, histRecord];
        localStorage.setItem("rft_architect_history", JSON.stringify(history));
        phase = "SESSION_END";
    }

    function cancelSession() { clearInterval(sessionInterval); clearInterval(questionInterval); clearInterval(interferenceInterval); phase = "SETUP"; }

    function clearHistory() {
        if(confirm("Delete all history?")) {
            history = [];
            localStorage.removeItem("rft_architect_history");
            if (chartInstance) { chartInstance.destroy(); chartInstance = null; }
        }
    }

    function startRound() {
        if (!settings.disableSessionTimer && sessionSecondsRemaining <= 0) { endSession(); return; }
        const enabledModes = (Object.keys(settings.activeModes) as RftMode[]).filter((k) => settings.activeModes[k]);
        if (enabledModes.length === 0) { alert("Please enable at least one Logic Module in Config."); return; }
        
        currentRoundMode = enabledModes[Math.floor(Math.random() * enabledModes.length)];
        activeModifiers = []; contextIsNight = false; 
        cipherHasChanged = false;

        if (settings.enableCipher && Math.random() < 0.15) {
            generateCipherKey();
            cipherHasChanged = true;
            activeModifiers.push("KEY_CHANGE");
        }

        if (currentRoundMode === "LINEAR") generateLinear();
        else if (currentRoundMode === "DISTINCTION") generateDistinction();
        else if (currentRoundMode === "SPATIAL_2D") generateSpatial(false);
        else if (currentRoundMode === "SPATIAL_3D") generateSpatial(true);
        else if (currentRoundMode === "HIERARCHY") generateHierarchy();
        else { startRound(); return; }

        if (settings.enableTransformation) {
            activeModifiers.push("TRANSFORM");
            contextIsNight = Math.random() > 0.5;
            if (contextIsNight) expectedAnswer = !expectedAnswer;
        }
        
        questionSecondsRemaining = settings.questionTimeLimit; isYesRight = Math.random() > 0.5; feedbackMsg = "";
        
        if (settings.blindMode || cipherHasChanged) {
            phase = "PREMISE_MEMORIZE";
        } else {
            if (settings.enableInterference) startInterferencePhase(); 
            else startQuestionPhase();
        }
    }

    function finishMemorization() { if (settings.enableInterference) startInterferencePhase(); else startQuestionPhase(); }

    // --- ACTIVE INTERFERENCE LOGIC ---
    function startInterferencePhase() {
        phase = "INTERFERENCE"; 
        activeModifiers.push("INTERFERENCE");
        interferenceStatus = "WAIT";
        const colors = ["red", "blue", "green", "yellow"];
        interferenceTargetColor = colors[Math.floor(Math.random() * colors.length)];
        
        clearInterval(interferenceInterval);
        // Fast cycle (300ms)
        interferenceInterval = setInterval(() => {
            interferenceCurrentColor = colors[Math.floor(Math.random() * colors.length)];
            interferenceStatus = "WAIT"; // Reset status on new color
        }, 300);
    }

    // --- MAIN LOGIC ---
    function startQuestionPhase() {
        phase = "QUESTION"; questionStartTime = Date.now(); 
        if (settings.useQuestionTimer) {
            clearInterval(questionInterval);
            questionSecondsRemaining = settings.questionTimeLimit; 
            questionInterval = setInterval(() => { 
                questionSecondsRemaining--; 
                if (questionSecondsRemaining <= 0) handleAnswer(null); 
            }, 1000);
        }
    }

    function handleAnswer(userAnswer: boolean | null) {
        clearInterval(questionInterval);
        const endTime = Date.now(); const reactionTime = endTime - questionStartTime;
        let correct = false; let ansString = "TIMEOUT";
        if (userAnswer !== null) {
            correct = userAnswer === expectedAnswer; ansString = userAnswer ? "YES" : "NO";
            if (settings.enableTransformation && contextIsNight) feedbackMsg = correct ? "INVERSION SUCCESSFUL" : "FAILED TO INVERT";
            else feedbackMsg = correct ? "VERIFIED" : "COLLAPSE";
        } else feedbackMsg = "TIME OUT";

        questionsAttempted++;
        if (correct) {
            questionsCorrect++; currentScore += settings.numPremises * 10; currentStreak++; maxStreak = Math.max(currentStreak, maxStreak);
            if (settings.autoProgress && currentStreak > 0 && currentStreak % 3 === 0) {
                settings.numPremises++; maxDepthReached = Math.max(maxDepthReached, settings.numPremises); feedbackMsg += " | DEPTH +";
            }
        } else {
            currentScore = Math.max(0, currentScore - 20); currentStreak = 0;
            if (settings.autoProgress && settings.numPremises > 2) { settings.numPremises--; feedbackMsg += " | DEPTH -"; }
        }
        totalReactionTime += reactionTime;
        if (!reactionTimesByDepth[settings.numPremises]) reactionTimesByDepth[settings.numPremises] = [];
        reactionTimesByDepth[settings.numPremises].push(reactionTime);
        const sessionRec: SessionRecord = {
            id: Date.now(), mode: currentRoundMode, premises: [...premises], question: currentQuestion, userAnswer: ansString, correctAnswer: expectedAnswer ? "YES" : "NO", isCorrect: correct, reactionTimeMs: reactionTime, modifiers: [...activeModifiers, ...(contextIsNight ? ['NIGHT_MODE'] : []), ...(settings.enableCipher ? ['CIPHER'] : [])]
        };
        sessionLog = [sessionRec, ...sessionLog]; saveSettings(); phase = "RESULT";
    }
    
    // --- KEYBOARD & HELPERS ---
    function handleKeydown(e: KeyboardEvent) {
        if (showSettings || showReviewModal || showHistoryModal) { if (e.code === "Escape") { showSettings = false; showReviewModal = false; showHistoryModal = false; } return; }
        if (phase === "SETUP" && e.code === "Enter") startSession();
        if (phase === "RESULT" && (e.code === "Enter" || e.code === "Space")) startRound();
        if (phase === "SESSION_END" && e.code === "Enter") phase = "SETUP";
        if (phase === "PREMISE_MEMORIZE" && (e.code === "Space" || e.code === "Enter")) finishMemorization();
        
        // Active Interference Control
        if (phase === "INTERFERENCE" && e.code === "Space") {
            if (interferenceCurrentColor === interferenceTargetColor) {
                interferenceStatus = "HIT";
                clearInterval(interferenceInterval);
                setTimeout(startQuestionPhase, 200); // Short pause to show success
            } else {
                interferenceStatus = "MISS"; // Visual penalty
            }
        }

        if (phase === "QUESTION") {
            if (e.code === "ArrowLeft" || e.code === "KeyD") handleAnswer(!isYesRight);
            if (e.code === "ArrowRight" || e.code === "KeyJ") handleAnswer(isYesRight);
        }
    }
    function formatTime(s: number) { const m = Math.floor(s / 60).toString().padStart(2, "0"); const sec = (s % 60).toString().padStart(2, "0"); return `${m}:${sec}`; }
    function getDurationLabel(h: GameHistory) { return h.sessionDurationSeconds ? formatTime(h.sessionDurationSeconds) : "N/A"; }
    
    // Context-Aware Cipher Helper
    function getRelevantCipherKeys() {
        if (!settings.enableCipher) return [];
        // Scan Question and Premises for cipher keys active in this round
        const textToScan = (currentQuestion + premises.join(" ")).toUpperCase();
        return Object.entries(currentCipherMap).filter(([meaning, word]) => {
            // Check if the nonsense word (value) appears in the text
            return textToScan.includes(word);
        });
    }
    
    function getCipherList() {
        if (!settings.enableCipher) return [];
        return Object.entries(currentCipherMap).filter(([k,v]) => {
            if (currentRoundMode === "LINEAR" && (k === "GREATER" || k === "LESS")) return true;
            if (currentRoundMode === "DISTINCTION" && (k === "SAME" || k === "DIFFERENT")) return true;
            if (currentRoundMode === "HIERARCHY" && (k === "CONTAINS" || k === "INSIDE")) return true;
            if ((currentRoundMode === "SPATIAL_2D" || currentRoundMode === "SPATIAL_3D")) return true;
            return false;
        });
    }
</script>

<svelte:window on:keydown={handleKeydown} />

<main class="container">
    <header class="app-header">
        <div class="title-block">
            <p class="sub-info">
                {#if phase === "SETUP"} READY
                {:else if phase === "SESSION_END"} COMPLETE
                {:else} 
                    {#if settings.disableSessionTimer} 
                        DURATION: <span class="timer">{formatTime(sessionSecondsElapsed)}</span>
                    {:else} 
                        REMAINING: <span class="timer {sessionSecondsRemaining < 30 ? 'warn' : ''}">{formatTime(sessionSecondsRemaining)}</span>
                    {/if}
                    {#if phase === "QUESTION" && settings.useQuestionTimer}
                        | Q-TIMER: <span class="timer {questionSecondsRemaining < 5 ? 'warn' : ''}">{questionSecondsRemaining}s</span>
                    {/if}
                {/if}
            </p>
        </div>
        <div class="header-controls">
            {#if phase !== "SETUP" && phase !== "SESSION_END"} <button class="settings-btn restart" on:click={cancelSession}>ABORT</button>
            {:else} <button class="settings-btn" on:click={() => (showHistoryModal = true)}>STATS</button>
            {/if}
            <button class="settings-btn" on:click={() => (showReviewModal = true)}>LOG</button>
            <button class="settings-btn" on:click={() => (showSettings = !showSettings)}>CONFIG</button>
        </div>
    </header>

    <div class="sidebar {showSettings ? 'open' : ''}">
        <div class="sidebar-header">
            <h2>CONFIGURATION</h2>
            <button class="close-btn" on:click={() => { showSettings = false; saveSettings(); }}>X</button>
        </div>
        <div class="sidebar-content">
            <div class="sidebar-section">
                <h3>TIME CONTROLS</h3>
                <div class="control-group">
                    <label>Session Mode</label>
                    <div class="toggle-wrapper">
                        <button class="toggle-btn {settings.disableSessionTimer ? 'active' : ''}" on:click={() => settings.disableSessionTimer = true}>∞ Infinite</button>
                        <button class="toggle-btn {!settings.disableSessionTimer ? 'active' : ''}" on:click={() => settings.disableSessionTimer = false}>Timed</button>
                    </div>
                </div>
                {#if !settings.disableSessionTimer}
                <div class="control-group">
                    <label>Session Minutes</label>
                    <div class="number-input">
                        <button on:click={() => settings.sessionLengthMinutes = Math.max(1, settings.sessionLengthMinutes - 1)}>-</button>
                        <input type="number" bind:value={settings.sessionLengthMinutes} min="1">
                        <button on:click={() => settings.sessionLengthMinutes++}>+</button>
                    </div>
                </div>
                {/if}
                <div class="control-group">
                    <label>Question Timer (Sec)</label>
                      <div class="number-input">
                        <button on:click={() => settings.questionTimeLimit = Math.max(5, settings.questionTimeLimit - 5)}>-</button>
                        <input type="number" bind:value={settings.questionTimeLimit} min="5">
                        <button on:click={() => settings.questionTimeLimit += 5}>+</button>
                    </div>
                </div>
                <label class="check-row compact"><input type="checkbox" bind:checked={settings.useQuestionTimer} /> <span>Enable Question Timer</span></label>
            </div>
            <div class="sidebar-section">
                <h3>LOGIC MODULES</h3>
                <label class="check-row"><input type="checkbox" bind:checked={settings.activeModes["LINEAR"]} /> <span>Linear Relation</span></label>
                <label class="check-row"><input type="checkbox" bind:checked={settings.activeModes["DISTINCTION"]} /> <span>Distinction</span></label>
                <label class="check-row"><input type="checkbox" bind:checked={settings.activeModes["SPATIAL_2D"]} /> <span>Spatial 2D</span></label>
                <label class="check-row"><input type="checkbox" bind:checked={settings.activeModes["SPATIAL_3D"]} /> <span>Spatial 3D</span></label>
                <label class="check-row"><input type="checkbox" bind:checked={settings.activeModes["HIERARCHY"]} /> <span>Hierarchy</span></label>
                <div class="divider"></div>
                <label class="check-row highlight-check">
                    <input type="checkbox" bind:checked={settings.enableCipher} /> 
                    <div><span>Derived/Cipher Mode</span> <small class="desc">Uses unknown words for logic.</small></div>
                </label>
            </div>
            <div class="sidebar-section highlight-section">
                <h3>WALL BREAKERS</h3>
                <label class="check-row">
                    <input type="checkbox" bind:checked={settings.enableDeictic} /> 
                    <div><span>Deictic (Perspective)</span> <small class="desc">"You are here..."</small></div>
                </label>
                <label class="check-row">
                    <input type="checkbox" bind:checked={settings.enableMovement} /> 
                    <div><span>Path Integration (Movement)</span> <small class="desc">"Start here, walk 3, turn right..."</small></div>
                </label>
                <label class="check-row">
                    <input type="checkbox" bind:checked={settings.enableTransformation} /> 
                    <div><span>Transformation</span> <small class="desc">Sun/Moon Inversion</small></div>
                </label>
                <label class="check-row">
                    <input type="checkbox" bind:checked={settings.enableInterference} /> 
                    <div><span>Interference Task</span> <small class="desc">Active Distractor (Press Space)</small></div>
                </label>
            </div>
            <div class="sidebar-section">
                <h3>COMPLEXITY</h3>
                <div class="control-group">
                    <label>Depth (Premises)</label>
                    <div class="number-input">
                        <button on:click={() => (settings.numPremises = Math.max(2, settings.numPremises - 1))}>-</button>
                        <span>{settings.numPremises}</span>
                        <button on:click={() => settings.numPremises++}>+</button>
                    </div>
                </div>
                <label class="check-row compact"><input type="checkbox" bind:checked={settings.autoProgress} /> <span>Auto-Progress Difficulty</span></label>
                <label class="check-row compact"><input type="checkbox" bind:checked={settings.blindMode} /> <span>Blind Mode (Memory)</span></label>
                <div class="control-group" style="margin-top: 15px;">
                    <label>Symbol Type</label>
                    <select bind:value={settings.symbolMode} class="styled-select">
                        <option value="EMOJI">Standard Emojis</option>
                        <option value="WORDS">Meaningless Words</option>
                        <option value="VORONOI">Voronoi (Shapes)</option>
                        <option value="MIXED">Mixed Chaos</option>
                    </select>
                </div>
            </div>
             <div class="sidebar-footer">
                <button class="btn-large start" on:click={() => { showSettings = false; saveSettings(); }}>SAVE & CLOSE</button>
            </div>
        </div>
    </div>
    {#if showSettings}<div class="sidebar-backdrop" on:click={() => { showSettings = false; saveSettings(); }}></div>{/if}

    {#if showHistoryModal}
    <div class="modal-backdrop" on:click|self={() => (showHistoryModal = false)}>
        <div class="modal-content history-modal">
            <div class="modal-header">
                <h2>STATS & HISTORY</h2>
                <div class="header-actions">
                    <button class="tiny-btn" on:click={clearHistory}>CLEAR</button>
                    <button on:click={() => (showHistoryModal = false)}>X</button>
                </div>
            </div>
            <div class="history-content">
                <div class="chart-container"><canvas bind:this={chartCanvas}></canvas></div>
                <div class="chart-legend">Improvement over sessions (Acc vs Score)</div>
                <div class="log-list">
                    {#if history.length === 0}
                        <div class="empty-msg">No sessions recorded.</div>
                    {:else}
                        {#each [...history].reverse() as h}
                            <div class="log-item">
                                <div class="log-row top">
                                    <span class="log-date">{h.date}</span>
                                    <span class="log-score {h.totalScore > 0 ? 'good' : 'bad'}">{h.totalScore} pts</span>
                                </div>
                                <div class="log-row meta">
                                    <span>Depth: {h.highestDepth}</span>
                                    <span>Acc: {h.accuracy}% ({h.questionsAnswered} Qs)</span>
                                </div>
                                <div class="log-row footer">
                                    <div class="badges">
                                        {#if h.activeModes && h.activeModes.includes("SPATIAL_2D")}<span class="badge">2D</span>{/if}
                                        {#if h.activeModifiers && h.activeModifiers.includes("MOVE")}<span class="badge move">MOVE</span>{/if}
                                        {#if h.activeModifiers && h.activeModifiers.includes("CIPHER")}<span class="badge cipher">CIPHER</span>{/if}
                                    </div>
                                    <span class="duration-tag">⏱ {getDurationLabel(h)}</span>
                                </div>
                            </div>
                        {/each}
                    {/if}
                </div>
            </div>
        </div>
    </div>
    {/if}

    {#if showReviewModal}
        <div class="modal-backdrop" on:click|self={() => (showReviewModal = false)}>
            <div class="modal-content review-modal">
                <div class="modal-header"><h2>SESSION LOG</h2> <button on:click={() => (showReviewModal = false)}>X</button></div>
                <div class="review-list">
                    {#if sessionLog.length === 0}
                        <div class="empty-msg">No questions answered yet.</div>
                    {:else}
                        {#each sessionLog as log}
                            <div class="review-card {log.isCorrect ? 'correct' : 'incorrect'}">
                                <div class="review-top">
                                    <span class="review-mode">{log.mode}</span>
                                    <span class="review-rt">{log.reactionTimeMs}ms</span>
                                </div>
                                <div class="modifiers-row">
                                    {#each log.modifiers as m}<span class="mod-tag">{m}</span>{/each}
                                </div>
                                <div class="review-premises">
                                    {#each log.premises as p}<div class="rp-line">{@html p}</div>{/each}
                                </div>
                                <div class="review-q">Q: {@html log.question}</div>
                                <div class="review-ans">
                                    <span>You: <strong>{log.userAnswer}</strong></span>
                                    <span>Correct: <strong>{log.correctAnswer}</strong></span>
                                </div>
                            </div>
                        {/each}
                    {/if}
                </div>
            </div>
        </div>
    {/if}

    <div class="game-stage card">
        <div class="stage-header">
            <div class="mini-stat">SCORE: {currentScore}</div>
            <div class="mini-stat">DEPTH: {settings.numPremises}</div>
            {#if settings.enableTransformation && (phase === "QUESTION" || phase === "RESULT")}
                <div class="context-cue {contextIsNight ? 'night' : 'day'}">
                    {contextIsNight ? "🌙 INVERT" : "☀️ STANDARD"}
                </div>
            {/if}
        </div>

        <div class="content-area">
            {#if phase === "SETUP"}
                <div class="setup-msg">
                    <h2>SYSTEM READY</h2>
                    <p>Mode: {settings.symbolMode}</p>
                    <button class="btn-large" on:click={startSession}>START SESSION</button>
                </div>

            {:else if phase === "INTERFERENCE"}
                <div class="interference-box {interferenceStatus === 'MISS' ? 'shake-anim' : ''}">
                    <h3>ACTIVE INTERFERENCE</h3>
                    <div class="color-dot" style="background-color: {interferenceCurrentColor};"></div>
                    <p class="instruction-interfere">
                        Press <strong>SPACE</strong> when color is <span style="color:{interferenceTargetColor}; text-transform:uppercase; font-weight:bold;">{interferenceTargetColor}</span>
                    </p>
                    {#if interferenceStatus === "MISS"}
                        <div class="interfere-feedback fail">MISSED!</div>
                    {/if}
                </div>

            {:else if phase === "PREMISE_MEMORIZE"}
                {#if cipherHasChanged}
                    <div class="alert-banner">⚠️ CIPHER KEY CHANGED</div>
                {/if}
                <div class="premise-box">
                    <div class="mode-badge">{currentRoundMode}</div>
                    
                    {#if settings.enableCipher}
                        <div class="cipher-key-box">
                            <div class="cipher-title">CIPHER KEY</div>
                            <div class="cipher-grid">
                                {#each getCipherList() as [k, v]}
                                    <div class="cipher-item">
                                        <span class="cipher-word">{v}</span> = <span class="cipher-mean">{k}</span>
                                    </div>
                                {/each}
                            </div>
                        </div>
                    {/if}

                    {#each premises as p}<div class="premise-line">{@html p}</div>{/each}
                    <div class="instruction">Memorize structure...</div>
                </div>
                <button class="btn-action" on:click={finishMemorization}>READY (Space)</button>

            {:else if phase === "QUESTION"}
                <div class="active-game-layout">
                    {#if settings.enableCipher}
                        {@const relevantKeys = getRelevantCipherKeys()}
                        {#if relevantKeys.length > 0}
                            <div class="mini-cipher-hint">
                                {#each relevantKeys as [k, v]}
                                    <span class="mc-item"><b>{v}</b>={k}</span>
                                {/each}
                            </div>
                        {/if}
                    {/if}

                    {#if !settings.blindMode}
                        <div class="premise-box compact">
                            <div class="mode-badge">{currentRoundMode}</div>
                            {#each premises as p}<div class="premise-line small">{@html p}</div>{/each}
                        </div>
                    {/if}
                    <div class="question-box">
                        <div class="step-counter">QUERY</div>
                        <h2 class="question-text">{@html currentQuestion}</h2>
                    </div>
                </div>
                <div class="controls">
                    <button class="btn-ans {isYesRight ? 'no' : 'yes'}" on:click={() => handleAnswer(!isYesRight)}>
                        {isYesRight ? "NO" : "YES"} <span class="key-hint">D / ←</span>
                    </button>
                    <button class="btn-ans {isYesRight ? 'yes' : 'no'}" on:click={() => handleAnswer(isYesRight)}>
                        {isYesRight ? "YES" : "NO"} <span class="key-hint">J / →</span>
                    </button>
                </div>

            {:else if phase === "RESULT"}
                <div class="result-box">
                    <h2 class={feedbackMsg.includes("VERIFIED") || feedbackMsg.includes("SUCCESS") ? "success" : "fail"}>{feedbackMsg}</h2>
                      {#if settings.enableTransformation}
                        <div class="context-reveal">Condition: {contextIsNight ? "NIGHT (Inverted)" : "DAY (Normal)"}</div>
                    {/if}
                    <div class="recap">
                        <div class="recap-label">PREMISES:</div>
                        {#each premises as p}<div class="recap-line">{@html p}</div>{/each}
                        <div class="recap-q">Q: {@html currentQuestion}</div>
                        <div class="recap-a">A: {expectedAnswer ? "YES" : "NO"}</div>
                    </div>
                    <button class="btn-large" on:click={startRound}>NEXT (Enter)</button>
                </div>
            
            {:else if phase === "SESSION_END"}
                 <div class="result-box">
                    <h2 class="success">SESSION COMPLETE</h2>
                    <div class="recap center-text">
                        <div>Score: <strong>{currentScore}</strong></div>
                        <div>Avg Reaction: <strong>{questionsAttempted > 0 ? Math.round(totalReactionTime / questionsAttempted) : 0}ms</strong></div>
                        <div>Max Depth: <strong>{maxDepthReached}</strong></div>
                    </div>
                    <button class="btn-large" on:click={() => (phase = "SETUP")}>RETURN TO SETUP</button>
                </div>
            {/if}
        </div>
    </div>
</main>

<style>
    /* Global & Base */
    :global(body) { margin: 0; overflow-x: hidden; background-color: #1a1a1a; color: #e0e0e0; font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif; }
    :root { --bg-surface: #2d2d2d; --bg-secondary: #333; --color-solution: #00ff9d; --color-exclusion: #ff4d4d; --accent-primary: #00f2ff; --text-primary: #e0e0e0; --text-secondary: #aaa; --border-color: #444; }
    
    /* Symbols */
    :global(.symbol-emoji) { font-size: 1.2em; vertical-align: middle; }
    :global(.symbol-text) { font-weight: 900; color: var(--accent-primary); font-family: monospace; border: 1px solid var(--border-color); padding: 0 4px; border-radius: 3px; background: var(--bg-secondary); }
    :global(.symbol-wrap) { display: inline-block; vertical-align: middle; }
    :global(.voronoi-svg) { width: 30px; height: 30px; display: inline-block; vertical-align: middle; }

    /* Layout */
    .container { max-width: 800px; width: 95%; margin: 0 auto; padding: 10px; display: flex; flex-direction: column; min-height: 100vh; box-sizing: border-box; }
    .card { background: var(--bg-surface); border: 1px solid var(--border-color); border-radius: 8px; padding: 15px; margin-bottom: 15px; box-shadow: 0 4px 10px rgba(0,0,0,0.3); }
    
    /* Header */
    .app-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
    .timer { color: var(--accent-primary); font-weight: bold; font-variant-numeric: tabular-nums; } .timer.warn { color: var(--color-exclusion); }
    .header-controls { display: flex; gap: 8px; }
    .settings-btn { background: #333; border: 1px solid #444; color: #aaa; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-weight: bold; }
    .settings-btn:hover { background: #444; color: #fff; }
    .settings-btn.restart { border-color: var(--color-exclusion); color: var(--color-exclusion); }

    /* Sidebar */
    .sidebar { padding-top:55px; position: fixed; top: 0; right: -360px; width: 340px; height: 100%; background: var(--bg-surface); border-left: 2px solid var(--color-solution); z-index: 200; transition: right 0.3s; display: flex; flex-direction: column; box-shadow: -5px 0 15px rgba(0,0,0,0.5); }
    .sidebar.open { right: 0; }
    .sidebar-backdrop { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0, 0, 0, 0.5); z-index: 150; }
    .sidebar-header { padding: 15px; border-bottom: 1px solid #444; display: flex; justify-content: space-between; background: #252525; }
    .sidebar-content { padding: 15px; overflow-y: auto; flex-grow: 1; }
    .sidebar-footer { padding: 15px; border-top: 1px solid #444; background: #252525; }
    
    .sidebar-section { margin-bottom: 25px; } 
    .sidebar-section h3 { font-size: 0.75rem; color: var(--accent-primary); text-transform: uppercase; letter-spacing: 1px; border-bottom: 1px solid #444; padding-bottom: 8px; margin-bottom: 12px; }
    
    .check-row { display: flex; align-items: center; gap: 12px; margin-bottom: 12px; cursor: pointer; padding: 8px; background: #333; border-radius: 6px; border: 1px solid #444; transition: 0.2s; }
    .check-row:hover { background: #3a3a3a; border-color: #666; }
    .check-row.compact { padding: 0; background: transparent; border: none; margin-top: 10px; }
    .check-row.compact:hover { background: transparent; }
    
    .highlight-check { border-color: var(--accent-primary); background: rgba(0, 242, 255, 0.05); }
    .highlight-section { border: 1px solid var(--accent-primary); padding: 15px; border-radius: 6px; background: rgba(0, 242, 255, 0.05); }
    .divider { height: 1px; background: #444; margin: 15px 0; }
    .desc { display: block; font-size: 0.7rem; color: #888; margin-top: 2px; }
    
    /* Improved Controls */
    .control-group { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; }
    .control-group label { font-size: 0.9rem; color: #ccc; }
    
    .number-input { display: flex; align-items: center; background: #222; border-radius: 4px; border: 1px solid #555; overflow: hidden; }
    .number-input button { width: 32px; height: 32px; background: #333; border: none; color: #fff; cursor: pointer; font-size: 1.1rem; display: flex; align-items: center; justify-content: center; }
    .number-input button:hover { background: #444; }
    .number-input span, .number-input input { width: 40px; text-align: center; font-family: monospace; border: none; background: transparent; color: #fff; font-size: 1rem; -moz-appearance: textfield; }
    .number-input input::-webkit-outer-spin-button, .number-input input::-webkit-inner-spin-button { -webkit-appearance: none; margin: 0; }

    .toggle-wrapper { display: flex; background: #222; border-radius: 4px; padding: 2px; border: 1px solid #555; }
    .toggle-btn { background: transparent; border: none; color: #888; padding: 6px 10px; font-size: 0.8rem; cursor: pointer; border-radius: 2px; }
    .toggle-btn.active { background: var(--color-solution); color: #000; font-weight: bold; }

    .styled-select { width: 100%; padding: 8px; background: #333; color: #fff; border: 1px solid #555; border-radius: 4px; }

    /* Modals */
    .modal-backdrop { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.8); z-index: 300; display: flex; justify-content: center; align-items: center; }
    .modal-content { background: var(--bg-surface); width: 90%; max-width: 600px; max-height: 80vh; border-radius: 8px; display: flex; flex-direction: column; border: 1px solid var(--color-solution); }
    .modal-header { padding: 15px; border-bottom: 1px solid #444; display: flex; justify-content: space-between; }
    .history-content, .review-list { overflow-y: auto; padding: 15px; display: flex; flex-direction: column; gap: 10px; }
    
    /* Cipher Styles */
    .cipher-key-box { background: #111; border: 1px solid #444; padding: 10px; border-radius: 6px; margin-bottom: 15px; text-align: center; }
    .cipher-title { color: #888; font-size: 0.7rem; letter-spacing: 2px; margin-bottom: 8px; }
    .cipher-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(100px, 1fr)); gap: 8px; }
    .cipher-item { font-size: 0.8rem; background: #222; padding: 4px; border-radius: 3px; }
    .cipher-word { color: var(--color-solution); font-weight: bold; }
    .alert-banner { background: var(--color-exclusion); color: white; padding: 8px; text-align: center; font-weight: bold; margin-bottom: 10px; border-radius: 4px; animation: flash 1s infinite alternate; }
    
    /* MINI CIPHER HINT */
    .mini-cipher-hint { background: #1a1a1a; border-bottom: 1px solid #444; padding: 8px; font-size: 0.8rem; color: #888; display: flex; flex-wrap: wrap; gap: 10px; justify-content: center; margin-bottom: 10px; border-radius: 4px; }
    .mc-item b { color: var(--color-solution); }

    @keyframes flash { from { opacity: 0.8; } to { opacity: 1; } }
    @keyframes shake { 0% { transform: translateX(0); } 25% { transform: translateX(-5px); } 50% { transform: translateX(5px); } 75% { transform: translateX(-5px); } 100% { transform: translateX(0); } }
    .shake-anim { animation: shake 0.3s; border-color: red; }

    /* Chart & History */
    .chart-container { height: 200px; width: 100%; background: #222; margin-bottom: 10px; border-radius: 4px; }
    .chart-legend { font-size: 0.7rem; color: #888; text-align: center; margin-bottom: 15px; font-style: italic; }
    .log-item { border-bottom: 1px solid #444; padding-bottom: 8px; }
    .log-row { display: flex; justify-content: space-between; }
    .log-row.top { font-weight: bold; } .log-score.good { color: var(--color-solution); } .log-score.bad { color: var(--color-exclusion); }
    .log-row.meta { font-size: 0.8rem; color: #aaa; }
    .log-row.footer { margin-top: 5px; align-items: center; }
    .badge { font-size: 0.6rem; background: #444; padding: 2px 4px; border-radius: 2px; margin-right: 5px; }
    .badge.cipher { color: #e0e0e0; background: #673ab7; }
    .badge.move { color: #000; background: #00f2ff; }
    
    /* Review Cards */
    .review-card { background: #333; padding: 10px; border-radius: 4px; border-left: 4px solid transparent; }
    .review-card.correct { border-left-color: var(--color-solution); } .review-card.incorrect { border-left-color: var(--color-exclusion); }
    .review-top { display: flex; justify-content: space-between; font-size: 0.8rem; color: #aaa; }
    .modifiers-row { margin: 4px 0; } .mod-tag { font-size: 0.7rem; background: #555; padding: 2px 4px; border-radius: 2px; margin-right: 4px; }
    .review-premises { margin: 8px 0; padding: 5px; background: #222; border-radius: 4px; }
    .rp-line { font-size: 0.85rem; color: #ccc; border-bottom: 1px solid #333; }
    .review-q { color: var(--accent-primary); font-weight: bold; margin: 5px 0; }
    .review-ans { display: flex; justify-content: space-between; font-size: 0.9rem; }

    /* Game Stage */
    .game-stage { min-height: 450px; display: flex; flex-direction: column; }
    .stage-header { display: flex; justify-content: space-between; padding-bottom: 10px; border-bottom: 1px solid #444; }
    .mini-stat { font-family: monospace; font-weight: bold; color: #888; }
    .content-area { flex-grow: 1; display: flex; flex-direction: column; justify-content: center; align-items: center; text-align: center; }
    
    .context-cue { padding: 4px 8px; border-radius: 4px; font-weight: bold; font-size: 0.8rem; }
    .context-cue.day { background: #000; color: #fff; border: 2px solid orange; }
    .context-cue.night { background: #000; color: #fff; border: 2px solid purple; }

    .setup-msg h2 { color: var(--color-solution); }
    .btn-large { width: 100%; padding: 15px; background: var(--color-solution); border: none; color: #000; font-weight: bold; font-size: 1.1rem; border-radius: 6px; cursor: pointer; margin-top: 20px; }
    .btn-action { background: #00bcd4; color: #fff; border: none; padding: 15px 40px; font-size: 1.2rem; border-radius: 8px; cursor: pointer; margin-top: 20px; }
    
    .premise-box { width: 100%; margin-bottom: 20px; }
    .premise-line { font-size: 1.2rem; margin: 5px 0; } .premise-line.small { font-size: 0.9rem; opacity: 0.8; }
    .mode-badge { font-size: 0.7rem; background: #444; padding: 2px 6px; border-radius: 4px; display: inline-block; margin-bottom: 10px; }
    
    /* Interference */
    .interference-box { height: 300px; display: flex; flex-direction: column; align-items: center; justify-content: center; border: 1px solid #444; background: #222; border-radius: 8px; width: 100%; }
    .color-dot { width: 100px; height: 100px; border-radius: 50%; margin: 20px; border: 4px solid #fff; transition: background-color 0.1s; }
    .instruction-interfere { font-size: 1.1rem; margin-top: 10px; }
    .interfere-feedback { font-weight: bold; font-size: 1.5rem; margin-top: 10px; }
    .interfere-feedback.fail { color: var(--color-exclusion); }

    .question-text { font-size: 1.8rem; color: var(--accent-primary); margin: 20px 0; }
    .controls { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; width: 100%; max-width: 400px; }
    .btn-ans { padding: 20px; font-size: 1.5rem; border: none; border-radius: 8px; cursor: pointer; font-weight: bold; display: flex; flex-direction: column; align-items: center; color: #000; }
    .yes { background: var(--color-solution); } .no { background: var(--color-exclusion); }
    .key-hint { font-size: 0.8rem; opacity: 0.6; font-weight: normal; }

    .result-box h2 { font-size: 2rem; margin: 0 0 20px 0; } .success { color: var(--color-solution); } .fail { color: var(--color-exclusion); }
    .recap { text-align: left; background: #333; padding: 15px; border-radius: 6px; width: 100%; box-sizing: border-box; margin-bottom: 20px; }
    .recap-label { font-size: 0.7rem; color: #888; } .recap-line { border-bottom: 1px solid #444; padding: 4px 0; }
    .recap-q { color: var(--accent-primary); font-weight: bold; margin-top: 10px; }
</style>