// PadaMaitri backend for Thunkable
// ------------------------------------------------------------
// Node.js + Express backend
// Features:
// 1. Create a new game board
// 2. Validate submitted word paths
// 3. Score words
// 4. Track used words per game
// 5. Simple in-memory storage for testing
//
// For production, replace in-memory maps with Firebase RTDB / Firestore.
// ------------------------------------------------------------

const express = require("express");
const cors = require("cors");
const crypto = require("crypto");

const app = express();
app.use(cors());
app.use(express.json());

// ------------------------------------------------------------
// Small demo dictionary.
// Replace this with a full dictionary file or database.
// ------------------------------------------------------------
const WORDS = new Set([
  "CAT", "CATS", "DOG", "DOGS", "TREE", "TREES", "READ", "DEAR", "DARE",
  "CARD", "CAR", "CARS", "STAR", "START", "ART", "TAR", "RAT", "RATS",
  "TEA", "ATE", "EAT", "TEAM", "MEAT", "MATE", "GAME", "GAMES",
  "WORD", "WORDS", "CODE", "CODES", "NOTE", "NOTES", "STONE", "TONE"
]);

// ------------------------------------------------------------
// PadaMaitri dice for 4x4 classic-style board.
// Q is represented as QU on the board.
// ------------------------------------------------------------
const DICE_4X4 = [
  "AAEEGN", "ABBJOO", "ACHOPS", "AFFKPS",
  "AOOTTW", "CIMOTU", "DEILRX", "DELRVY",
  "DISTTY", "EEGHNW", "EEINSU", "EHRTVW",
  "EIOSST", "ELRTTY", "HIMNQU", "HLNNRZ"
];

const games = new Map();

function randomInt(max) {
  return crypto.randomInt(0, max);
}

function shuffle(array) {
  const copy = [...array];
  for (let i = copy.length - 1; i > 0; i--) {
    const j = randomInt(i + 1);
    [copy[i], copy[j]] = [copy[j], copy[i]];
  }
  return copy;
}

function rollBoard(size = 4) {
  if (size !== 4) {
    throw new Error("Only 4x4 board is implemented in this starter backend.");
  }

  const shuffledDice = shuffle(DICE_4X4);
  const letters = shuffledDice.map(die => {
    const ch = die[randomInt(die.length)];
    return ch === "Q" ? "QU" : ch;
  });

  const board = [];
  for (let r = 0; r < size; r++) {
    board.push(letters.slice(r * size, r * size + size));
  }
  return board;
}

function normalizeWord(word) {
  return String(word || "").trim().toUpperCase().replace(/[^A-Z]/g, "");
}

function areAdjacent(a, b) {
  const dr = Math.abs(a.row - b.row);
  const dc = Math.abs(a.col - b.col);
  return dr <= 1 && dc <= 1 && !(dr === 0 && dc === 0);
}

function pathToWord(board, path) {
  return path.map(cell => board[cell.row][cell.col]).join("");
}

function validatePath(board, path) {
  if (!Array.isArray(path) || path.length === 0) {
    return { ok: false, reason: "Path is empty." };
  }

  const size = board.length;
  const visited = new Set();

  for (let i = 0; i < path.length; i++) {
    const cell = path[i];

    if (
      !Number.isInteger(cell.row) ||
      !Number.isInteger(cell.col) ||
      cell.row < 0 ||
      cell.row >= size ||
      cell.col < 0 ||
      cell.col >= size
    ) {
      return { ok: false, reason: "Path contains an invalid board position." };
    }

    const key = `${cell.row},${cell.col}`;
    if (visited.has(key)) {
      return { ok: false, reason: "A letter cube cannot be used more than once." };
    }
    visited.add(key);

    if (i > 0 && !areAdjacent(path[i - 1], cell)) {
      return { ok: false, reason: "Each selected cube must touch the previous cube." };
    }
  }

  return { ok: true };
}

function scoreWord(word) {
  const len = word.length;
  if (len < 3) return 0;
  if (len <= 4) return 1;
  if (len === 5) return 2;
  if (len === 6) return 3;
  if (len === 7) return 5;
  return 11;
}

function newGameId() {
  return crypto.randomUUID();
}

// ------------------------------------------------------------
// Health check
// GET /health
// ------------------------------------------------------------
app.get("/health", (req, res) => {
  res.json({ ok: true, service: "boggle-backend" });
});

// ------------------------------------------------------------
// Start a game
// POST /game/start
// Body: { "playerId": "abc123", "size": 4 }
// ------------------------------------------------------------
app.post("/game/start", (req, res) => {
  try {
    const playerId = String(req.body.playerId || "guest");
    const size = Number(req.body.size || 4);
    const board = rollBoard(size);
    const gameId = newGameId();

    const game = {
      gameId,
      playerId,
      board,
      size,
      usedWords: [],
      score: 0,
      createdAt: new Date().toISOString(),
      status: "active"
    };

    games.set(gameId, game);

    res.json({
      ok: true,
      gameId,
      board,
      score: 0,
      usedWords: []
    });
  } catch (err) {
    res.status(400).json({ ok: false, error: err.message });
  }
});

// ------------------------------------------------------------
// Submit a word by selected path
// POST /game/submit
// Body:
// {
//   "gameId": "...",
//   "word": "CAT",
//   "path": [ { "row": 0, "col": 0 }, { "row": 0, "col": 1 }, { "row": 1, "col": 1 } ]
// }
// ------------------------------------------------------------
app.post("/game/submit", (req, res) => {
  const gameId = String(req.body.gameId || "");
  const game = games.get(gameId);

  if (!game) {
    return res.status(404).json({ ok: false, error: "Game not found." });
  }

  if (game.status !== "active") {
    return res.status(400).json({ ok: false, error: "Game is not active." });
  }

  const submittedWord = normalizeWord(req.body.word);
  const path = req.body.path;

  const pathCheck = validatePath(game.board, path);
  if (!pathCheck.ok) {
    return res.json({
      ok: false,
      accepted: false,
      reason: pathCheck.reason,
      score: game.score,
      usedWords: game.usedWords
    });
  }

  const boardWord = normalizeWord(pathToWord(game.board, path));

  if (submittedWord !== boardWord) {
    return res.json({
      ok: false,
      accepted: false,
      reason: `Submitted word does not match selected path. Path spells ${boardWord}.`,
      pathWord: boardWord,
      score: game.score,
      usedWords: game.usedWords
    });
  }

  if (submittedWord.length < 3) {
    return res.json({
      ok: false,
      accepted: false,
      reason: "Word must be at least 3 letters.",
      score: game.score,
      usedWords: game.usedWords
    });
  }

  if (!WORDS.has(submittedWord)) {
    return res.json({
      ok: false,
      accepted: false,
      reason: "Word is not in dictionary.",
      score: game.score,
      usedWords: game.usedWords
    });
  }

  if (game.usedWords.includes(submittedWord)) {
    return res.json({
      ok: false,
      accepted: false,
      reason: "Word already used in this game.",
      score: game.score,
      usedWords: game.usedWords
    });
  }

  const points = scoreWord(submittedWord);
  game.usedWords.push(submittedWord);
  game.score += points;

  res.json({
    ok: true,
    accepted: true,
    word: submittedWord,
    points,
    score: game.score,
    usedWords: game.usedWords
  });
});

// ------------------------------------------------------------
// Get game state
// GET /game/:gameId
// ------------------------------------------------------------
app.get("/game/:gameId", (req, res) => {
  const game = games.get(req.params.gameId);
  if (!game) {
    return res.status(404).json({ ok: false, error: "Game not found." });
  }

  res.json({
    ok: true,
    gameId: game.gameId,
    playerId: game.playerId,
    board: game.board,
    score: game.score,
    usedWords: game.usedWords,
    createdAt: game.createdAt,
    status: game.status
  });
});

// ------------------------------------------------------------
// End game
// POST /game/end
// Body: { "gameId": "..." }
// ------------------------------------------------------------
app.post("/game/end", (req, res) => {
  const gameId = String(req.body.gameId || "");
  const game = games.get(gameId);

  if (!game) {
    return res.status(404).json({ ok: false, error: "Game not found." });
  }

  game.status = "ended";
  game.endedAt = new Date().toISOString();

  res.json({
    ok: true,
    gameId,
    finalScore: game.score,
    usedWords: game.usedWords,
    status: game.status
  });
});

// ------------------------------------------------------------
// Local server start
// ------------------------------------------------------------
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Boggle backend running on port ${PORT}`);
});
