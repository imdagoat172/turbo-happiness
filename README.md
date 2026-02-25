// gamesquareplane.js
// GitHub Copilot
// Expanded GameSquarePlane + Level and LevelManager supporting 1500 levels
// - 500 "overworld" story levels, then 1000 "nether" levels
// - tracking completion/stars, freeplay unlock, simple procedural generation
// - draw & collision helpers
//
// Exports:
//   default: GameSquarePlane
//   named: LevelManager

class GameSquarePlane {
    constructor(cols = 400, rows = 8, tileSize = 64) {
        this.cols = cols;
        this.rows = rows;
        this.tileSize = tileSize;
        this.grid = this._createGrid(cols, rows, 0);
    }

    _createGrid(cols, rows, fill = 0) {
        const g = new Array(rows);
        for (let r = 0; r < rows; r++) {
            g[r] = new Array(cols).fill(fill);
        }
        return g;
    }

    clear(fill = 0) {
        for (let r = 0; r < this.rows; r++) {
            this.grid[r].fill(fill);
        }
    }

    // Procedural simple platform generator tuned by difficulty seed
    // difficulty: 0..1 (higher = more obstacles and gaps)
    // groundRow: index of ground (defaults to bottom)
    generatePlatforms(chance = 0.12, groundRow = null, difficulty = 0.2, seed = null) {
        if (groundRow === null) groundRow = this.rows - 1;
        const rand = seed ? mulberry32(seed) : Math.random;
        for (let c = 0; c < this.cols; c++) {
            // base chance modulated by difficulty
            const p = Math.max(0.02, chance + difficulty * 0.25);
            if (rand() < p) {
                // add a column of blocks of random height (1..(2+difficulty*3))
                const maxH = Math.min(this.rows, 1 + Math.floor(2 + difficulty * 3));
                const height = 1 + Math.floor(rand() * maxH);
                for (let h = 0; h < height; h++) {
                    const r = groundRow - h;
                    if (r >= 0) this.grid[r][c] = 1;
                }
                // occasionally place a floating platform
                if (rand() < 0.08 + difficulty * 0.12) {
                    const floatRow = Math.max(0, groundRow - height - (1 + Math.floor(rand() * 2)));
                    const floatLen = 1 + Math.floor(rand() * Math.min(4, this.cols - c));
                    for (let fc = 0; fc < floatLen && c + fc < this.cols; fc++) {
                        this.grid[floatRow][c + fc] = 1;
                    }
                }
            } else {
                // create gaps: ensure that near the right side of the world we can create long gaps
                if (rand() < 0.02 + difficulty * 0.05) {
                    // leave column empty
                }
            }
        }

        // add a ground line occasionally (not for nether-ish levels)
        if (difficulty < 0.9) {
            for (let c = 0; c < this.cols; c++) {
                if (Math.random() < 0.005) continue;
                this.grid[groundRow][c] = 1;
            }
        }
    }

    setTile(col, row, value) {
        if (col >= 0 && col < this.cols && row >= 0 && row < this.rows) {
            this.grid[row][col] = value;
        }
    }

    getTile(col, row) {
        if (col >= 0 && col < this.cols && row >= 0 && row < this.rows) {
            return this.grid[row][col];
        }
        return 0;
    }

    isSolidAtPixel(x, y) {
        const col = Math.floor(x / this.tileSize);
        const row = Math.floor(y / this.tileSize);
        return this.getTile(col, row) > 0;
    }

    // rectangle collision with any solid tile (rect uses world pixels)
    rectCollides(rect) {
        const leftCol = Math.floor(rect.x / this.tileSize);
        const rightCol = Math.floor((rect.x + rect.w) / this.tileSize);
        const topRow = Math.floor(rect.y / this.tileSize);
        const bottomRow = Math.floor((rect.y + rect.h) / this.tileSize);

        for (let r = topRow; r <= bottomRow; r++) {
            for (let c = leftCol; c <= rightCol; c++) {
                if (this.getTile(c, r) > 0) return true;
            }
        }
        return false;
    }

    // draw only visible tiles based on cameraX, cameraY and canvas size
    draw(ctx, cameraX = 0, cameraY = 0, canvasWidth = ctx.canvas.width, canvasHeight = ctx.canvas.height) {
        const startCol = Math.max(0, Math.floor(cameraX / this.tileSize));
        const endCol = Math.min(this.cols - 1, Math.floor((cameraX + canvasWidth) / this.tileSize));
        const startRow = Math.max(0, Math.floor(cameraY / this.tileSize));
        const endRow = Math.min(this.rows - 1, Math.floor((cameraY + canvasHeight) / this.tileSize));

        for (let r = startRow; r <= endRow; r++) {
            for (let c = startCol; c <= endCol; c++) {
                const v = this.grid[r][c];
                if (v > 0) {
                    const sx = c * this.tileSize - cameraX;
                    const sy = r * this.tileSize - cameraY;
                    // tile color can depend on v (type)
                    ctx.fillStyle = v === 1 ? '#333' : '#555';
                    ctx.fillRect(Math.round(sx), Math.round(sy), this.tileSize, this.tileSize);
                    // optional border
                    ctx.strokeStyle = 'rgba(0,0,0,0.2)';
                    ctx.strokeRect(Math.round(sx), Math.round(sy), this.tileSize, this.tileSize);
                }
            }
        }
    }

    get worldWidth() { return this.cols * this.tileSize; }
    get worldHeight() { return this.rows * this.tileSize; }

    toJSON() {
        return {
            cols: this.cols,
            rows: this.rows,
            tileSize: this.tileSize,
            grid: this.grid
        };
    }

    static fromJSON(obj) {
        const p = new GameSquarePlane(obj.cols, obj.rows, obj.tileSize);
        p.grid = obj.grid;
        return p;
    }
}

// Lightweight deterministic RNG (seeded) helper
// returns function() => [0,1)
function mulberry32(a) {
    let t = a >>> 0;
    return function() {
        t += 0x6D2B79F5;
        let r = Math.imul(t ^ t >>> 15, 1 | t);
        r ^= r + Math.imul(r ^ r >>> 7, 61 | r);
        return ((r ^ r >>> 14) >>> 0) / 4294967296;
    };
}

// Level data container
class Level {
    constructor(id, name, type = 'overworld', difficulty = 0.2, seed = null, cols = 400, rows = 8, tileSize = 64) {
        this.id = id;
        this.name = name || `Level ${id}`;
        this.type = type; // 'overworld' or 'nether' or 'custom'
        this.difficulty = difficulty; // 0..1
        this.seed = seed != null ? seed : Math.floor(Math.random() * 1e9);
        this.cols = cols;
        this.rows = rows;
        this.tileSize = tileSize;

        this.planeJSON = null; // lazy-generate / store as JSON for serialization
        this.completed = false;
        this.stars = 0; // 0..5
        this.bestTime = null; // optional
        this.createdAt = Date.now();
    }

    generateIfNeeded() {
        if (!this.planeJSON) {
            const p = new GameSquarePlane(this.cols, this.rows, this.tileSize);
            const baseChance = this.type === 'nether' ? 0.18 : 0.12;
            const effectiveDifficulty = Math.min(1, this.difficulty + (this.type === 'nether' ? 0.25 : 0));
            p.generatePlatforms(baseChance, this.rows - 1, effectiveDifficulty, this.seed);
            // add nether style tiles type 2
            if (this.type === 'nether') {
                for (let r = 0; r < p.rows; r++) {
                    for (let c = 0; c < p.cols; c++) {
                        if (p.grid[r][c] > 0) p.grid[r][c] = 2; // different tile id
                    }
                }
            }
            this.planeJSON = p.toJSON();
        }
    }

    getPlane() {
        this.generateIfNeeded();
        return GameSquarePlane.fromJSON(this.planeJSON);
    }

    toJSON() {
        return {
            id: this.id,
            name: this.name,
            type: this.type,
            difficulty: this.difficulty,
            seed: this.seed,
            cols: this.cols,
            rows: this.rows,
            tileSize: this.tileSize,
            planeJSON: this.planeJSON,
            completed: this.completed,
            stars: this.stars,
            bestTime: this.bestTime,
            createdAt: this.createdAt
        };
    }

    static fromJSON(obj) {
        const lvl = new Level(obj.id, obj.name, obj.type, obj.difficulty, obj.seed, obj.cols, obj.rows, obj.tileSize);
        lvl.planeJSON = obj.planeJSON;
        lvl.completed = obj.completed;
        lvl.stars = obj.stars;
        lvl.bestTime = obj.bestTime;
        lvl.createdAt = obj.createdAt;
        return lvl;
    }
}

// LevelManager manages the 500 overworld + 1000 nether levels and player progress
class LevelManager {
    constructor(options = {}) {
        this.overworldCount = options.overworldCount || 500;
        this.netherCount = options.netherCount || 1000;
        this.total = this.overworldCount + this.netherCount;
        this.rows = options.rows || 8;
        this.tileSize = options.tileSize || 64;

        this.levels = []; // array of Level
        this.freeplayUnlocked = false;
        this.customLevels = []; // user created
        this.storageKey = options.storageKey || 'gd_level_manager_v1';

        // load or generate
        if (!this._load()) {
            this._generateAllLevels();
            this._save();
        } else {
            this._rebuildIndexes();
        }
    }

    // generate all levels but do not keep planes in memory until requested
    _generateAllLevels() {
        this.levels = [];
        let id = 1;
        // Overworld (story) first 500 levels
        for (let i = 0; i < this.overworldCount; i++, id++) {
            const difficulty = Math.min(0.9, 0.05 + (i / Math.max(1, this.overworldCount - 1)) * 0.7);
            const cols = 300 + Math.floor((i / this.overworldCount) * 400);
            const lvl = new Level(id, `Story ${i + 1}`, 'overworld', difficulty, 12345 + id, cols, this.rows, this.tileSize);
            this.levels.push(lvl);
        }
        // Nether 1000 levels (more difficult)
        for (let i = 0; i < this.netherCount; i++, id++) {
            const difficulty = Math.min(1, 0.4 + (i / Math.max(1, this.netherCount - 1)) * 0.6);
            const cols = 400 + Math.floor((i / this.netherCount) * 600);
            const lvl = new Level(id, `Nether ${i + 1}`, 'nether', difficulty, 987654 + id, cols, this.rows, this.tileSize);
            this.levels.push(lvl);
        }
    }

    _rebuildIndexes() {
        // ensure counts match if loaded data changed
        this.total = this.levels.length;
        this.overworldCount = this.levels.filter(l => l.type === 'overworld').length;
        this.netherCount = this.levels.filter(l => l.type === 'nether').length;
    }

    getLevelById(id) {
        return this.levels.find(l => l.id === id) || null;
    }

    markComplete(id, stars = 1, time = null) {
        const lvl = this.getLevelById(id);
        if (!lvl) return false;
        lvl.completed = true;
        lvl.stars = Math.max(lvl.stars, Math.min(5, Math.floor(stars)));
        if (time != null) {
            if (lvl.bestTime == null || time < lvl.bestTime) lvl.bestTime = time;
        }
        this._checkFreeplayUnlockConditions();
        this._save();
        return true;
    }

    setStars(id, stars) {
        const lvl = this.getLevelById(id);
        if (!lvl) return false;
        lvl.stars = Math.max(lvl.stars, Math.min(5, Math.floor(stars)));
        this._checkFreeplayUnlockConditions();
        this._save();
        return true;
    }

    _checkFreeplayUnlockConditions() {
        // freeplay unlocks only after finishing all nether levels and getting 5 stars on each nether level
        const netherLevels = this.levels.filter(l => l.type === 'nether');
        const allCompleted = netherLevels.every(l => l.completed);
        const allFiveStars = netherLevels.every(l => l.stars >= 5);
        this.freeplayUnlocked = allCompleted && allFiveStars;
    }

    isFreeplayUnlocked() {
        return this.freeplayUnlocked;
    }

    // Create and register a custom level (for editor/freeplay)
    registerCustomLevel(plane, name = 'Custom Level') {
        const newId = this.levels.length + this.customLevels.length + 1;
        const lvl = new Level(newId, name, 'custom', 0.5, Math.floor(Math.random() * 1e9), plane.cols, plane.rows, plane.tileSize);
        lvl.planeJSON = plane.toJSON();
        this.customLevels.push(lvl);
        this._save();
        return lvl;
    }

    removeCustomLevel(id) {
        const idx = this.customLevels.findIndex(l => l.id === id);
        if (idx >= 0) {
            this.customLevels.splice(idx, 1);
            this._save();
            return true;
        }
        return false;
    }

    // simple persistence using localStorage
    _save() {
        try {
            const payload = {
                levels: this.levels.map(l => l.toJSON()),
                customLevels: this.customLevels.map(l => l.toJSON()),
                freeplayUnlocked: this.freeplayUnlocked,
                meta: {
                    savedAt: Date.now()
                }
            };
            localStorage.setItem(this.storageKey, JSON.stringify(payload));
            return true;
        } catch (e) {
            console.warn('LevelManager save failed', e);
            return false;
        }
    }

    _load() {
        try {
            const raw = localStorage.getItem(this.storageKey);
            if (!raw) return false;
            const obj = JSON.parse(raw);
            this.levels = (obj.levels || []).map(Level.fromJSON);
            this.customLevels = (obj.customLevels || []).map(Level.fromJSON);
            this.freeplayUnlocked = !!obj.freeplayUnlocked;
            this._rebuildIndexes();
            return true;
        } catch (e) {
            console.warn('LevelManager load failed', e);
            return false;
        }
    }

    exportAllJSON() {
        return JSON.stringify({
            levels: this.levels.map(l => l.toJSON()),
            customLevels: this.customLevels.map(l => l.toJSON()),
            freeplayUnlocked: this.freeplayUnlocked
        });
    }

    importFromJSON(jsonStr) {
        try {
            const obj = JSON.parse(jsonStr);
            this.levels = (obj.levels || []).map(Level.fromJSON);
            this.customLevels = (obj.customLevels || []).map(Level.fromJSON);
            this.freeplayUnlocked = !!obj.freeplayUnlocked;
            this._rebuildIndexes();
            this._save();
            return true;
        } catch (e) {
            console.warn('import failed', e);
            return false;
        }
    }

    // Utility: returns a shallow list of level summaries (id, name, type, completed, stars)
    getSummaries() {
        return [
            ...this.levels.map(l => ({ id: l.id, name: l.name, type: l.type, completed: l.completed, stars: l.stars })),
            ...this.customLevels.map(l => ({ id: l.id, name: l.name, type: l.type, completed: l.completed, stars: l.stars }))
        ];
    }
}

// Default export for compatibility with previous code
export default GameSquarePlane;
export { LevelManager, Level };

/*
Example usage (separate file/module):

import GameSquarePlane, { LevelManager } from './gamesquareplane.js';

const canvas = document.querySelector('canvas');
const ctx = canvas.getContext('2d');
canvas.width = 1024; canvas.height = 480;

const manager = new LevelManager({ overworldCount: 500, netherCount: 1000, rows: 8, tileSize: 64 });

// load level 1
const lvl = manager.getLevelById(1);
const plane = lvl.getPlane();

let cameraX = 0;
function loop() {
    ctx.clearRect(0,0,canvas.width,canvas.height);
    plane.draw(ctx, cameraX, 0, canvas.width, canvas.height);
    cameraX += 4;
    if (cameraX > plane.worldWidth - canvas.width) cameraX = 0;
    requestAnimationFrame(loop);
}
loop();
*/
