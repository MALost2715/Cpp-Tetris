# Cpp-Tetris

'''
/*******************************************************************************************
*   raylib - classic game: tetris (C++ / colored blocks)
*   Original by Marc Palau and Ramon Santamaria
********************************************************************************************/

#include "raylib.h"
#include <array>

#if defined(PLATFORM_WEB)
#include <emscripten/emscripten.h>
#endif

constexpr int SQUARE_SIZE = 20, COLS = 12, ROWS = 20;
constexpr int LAT_SPEED = 10, TURN_SPEED = 12, FAST_FALL = 30, FADE_TIME = 33;
constexpr int SW = 800, SH = 450;
constexpr int NUM_PIECES = 7;

// 피스 종류별 색상 (O, L, J, I, T, S, Z)
static const Color PIECE_COLORS[NUM_PIECES] = {
    YELLOW,                         // O - 노랑
    {255,165,0,255},                // L - 주황
    BLUE,                           // J - 파랑
    SKYBLUE,                        // I - 하늘
    PURPLE,                         // T - 보라
    GREEN,                          // S - 초록
    RED,                            // Z - 빨강
};

// 셀 값: 0=EMPTY, 1~7=MOVING(피스종류), 8~14=FULL(피스종류), 15=BLOCK, 16=FADING
// 단순화를 위해 셀에 피스 ID를 인코딩
// cell =  0        : EMPTY
// cell =  1.. 7   : MOVING (piece id = cell)
// cell =  8..14   : FULL   (piece id = cell - 7)
// cell = 15        : BLOCK
// cell = 16        : FADING
using Cell = int;
constexpr Cell EMPTY = 0, BLOCK = 15, FADING = 16;
inline Cell movingCell(int id) { return id + 1; }          // 1..7
inline Cell fullCell(int id) { return id + 8; }          // 8..14
inline bool isMoving(Cell c) { return c >= 1 && c <= 7; }
inline bool isFull(Cell c) { return c >= 8 && c <= 14; }
inline int  pieceId(Cell c) { return isMoving(c) ? c - 1 : c - 8; } // 0..6

using Grid4 = std::array<std::array<Cell, 4>, 4>;
using Grid = std::array<std::array<Cell, ROWS>, COLS>;

// ── State ─────────────────────────────────────────────────────────────────────
static Grid  grid;
static Grid4 piece, next;
static int   currentId = 0, nextId = 0;
static int   px = 0, py = 0;
static int   lines = 0;
static int   gravCnt = 0, latCnt = 0, turnCnt = 0, fastCnt = 0, fadeCnt = 0;
static int   gravSpeed = 30;
static bool  gameOver = false, paused = false;
static bool  pieceActive = false, detection = false, lineToDelete = false;
static Color fadingColor = GRAY;

// ── Piece shapes ──────────────────────────────────────────────────────────────
static Grid4 emptyPiece() { Grid4 p{}; for (auto& r : p) r.fill(EMPTY); return p; }

static Grid4 makePiece(int id)
{
    Grid4 p = emptyPiece();
    Cell  m = movingCell(id);
    switch (id) {
    case 0: p[1][1] = p[2][1] = p[1][2] = p[2][2] = m; break; // O
    case 1: p[1][0] = p[1][1] = p[1][2] = p[2][2] = m; break; // L
    case 2: p[1][2] = p[2][0] = p[2][1] = p[2][2] = m; break; // J
    case 3: p[0][1] = p[1][1] = p[2][1] = p[3][1] = m; break; // I
    case 4: p[1][0] = p[1][1] = p[1][2] = p[2][1] = m; break; // T
    case 5: p[1][1] = p[2][1] = p[2][2] = p[3][2] = m; break; // S
    case 6: p[1][2] = p[2][2] = p[2][1] = p[3][1] = m; break; // Z
    }
    return p;
}

// ── Init ──────────────────────────────────────────────────────────────────────
static void InitGame()
{
    lines = 0; fadingColor = GRAY; px = py = 0;
    paused = false; pieceActive = false; detection = false; lineToDelete = false;
    gravCnt = latCnt = turnCnt = fastCnt = fadeCnt = 0; gravSpeed = 30;

    for (int i = 0; i < COLS; i++)
        for (int j = 0; j < ROWS; j++)
            grid[i][j] = (j == ROWS - 1 || i == 0 || i == COLS - 1) ? BLOCK : EMPTY;

    nextId = GetRandomValue(0, NUM_PIECES - 1);
    next = makePiece(nextId);
    piece = emptyPiece();
}

// ── Helpers ───────────────────────────────────────────────────────────────────
static void clearMoving() {
    for (int i = 1; i < COLS - 1; i++) for (int j = 0; j < ROWS - 1; j++)
        if (isMoving(grid[i][j])) grid[i][j] = EMPTY;
}

static void stampPiece() {
    for (int i = px; i < px + 4; i++) for (int j = py; j < py + 4; j++)
        if (isMoving(piece[i - px][j - py])) grid[i][j] = piece[i - px][j - py];
}

static bool createPiece()
{
    px = (COLS - 4) / 2; py = 0;
    currentId = nextId;
    piece = next;
    nextId = GetRandomValue(0, NUM_PIECES - 1);
    next = makePiece(nextId);
    for (int i = px; i < px + 4; i++) for (int j = 0; j < 4; j++)
        if (isMoving(piece[i - px][j])) grid[i][j] = piece[i - px][j];
    return true;
}

static void checkDetection() {
    for (int j = ROWS - 2; j >= 0; j--) for (int i = 1; i < COLS - 1; i++)
        if (isMoving(grid[i][j]) && (isFull(grid[i][j + 1]) || grid[i][j + 1] == BLOCK))
            detection = true;
}

static void resolveFalling()
{
    if (detection) {
        for (int j = ROWS - 2; j >= 0; j--) for (int i = 1; i < COLS - 1; i++)
            if (isMoving(grid[i][j])) {
                grid[i][j] = fullCell(pieceId(grid[i][j]));
                detection = false; pieceActive = false;
            }
    }
    else {
        for (int j = ROWS - 2; j >= 0; j--) for (int i = 1; i < COLS - 1; i++)
            if (isMoving(grid[i][j])) { grid[i][j + 1] = grid[i][j]; grid[i][j] = EMPTY; }
        py++;
    }
}

static bool resolveLateral()
{
    bool col = false;
    int dir = IsKeyDown(KEY_LEFT) ? -1 : IsKeyDown(KEY_RIGHT) ? 1 : 0;
    if (!dir) return false;
    for (int j = ROWS - 2; j >= 0; j--) for (int i = 1; i < COLS - 1; i++)
        if (isMoving(grid[i][j]))
            if ((i + dir == 0) || (i + dir == COLS - 1) || isFull(grid[i + dir][j])) col = true;
    if (!col) {
        if (dir == -1) for (int j = ROWS - 2; j >= 0; j--) for (int i = 1; i < COLS - 1; i++)
        {
            if (isMoving(grid[i][j])) { grid[i - 1][j] = grid[i][j]; grid[i][j] = EMPTY; }
        }
        else for (int j = ROWS - 2; j >= 0; j--) for (int i = COLS - 1; i >= 1; i--)
        {
            if (isMoving(grid[i][j])) { grid[i + 1][j] = grid[i][j]; grid[i][j] = EMPTY; }
        }
        px += dir;
    }
    return col;
}

static bool resolveTurn()
{
    if (!IsKeyDown(KEY_UP)) return false;
    auto mv = [](Cell c) {return isMoving(c); };
    auto blk = [](Cell c) {return c != EMPTY && !isMoving(c); };
    bool bad = false;
#define CHK(a,b,c,d) if(mv(grid[px+(a)][py+(b)])&&blk(grid[px+(c)][py+(d)])) bad=true;
    CHK(3, 0, 0, 0)CHK(3, 3, 3, 0)CHK(0, 3, 3, 3)CHK(0, 0, 0, 3)
        CHK(1, 0, 0, 2)CHK(3, 1, 1, 0)CHK(2, 3, 3, 1)CHK(0, 2, 2, 3)
        CHK(2, 0, 0, 1)CHK(3, 2, 2, 0)CHK(1, 3, 3, 2)CHK(0, 1, 1, 3)
        CHK(1, 1, 1, 2)CHK(2, 1, 1, 1)CHK(2, 2, 2, 1)CHK(1, 2, 2, 2)
#undef CHK
        if (!bad) {
            auto rot = [&](int r1, int c1, int r2, int c2, int r3, int c3, int r4, int c4) {
                Cell t = piece[r1][c1];
                piece[r1][c1] = piece[r2][c2]; piece[r2][c2] = piece[r3][c3];
                piece[r3][c3] = piece[r4][c4]; piece[r4][c4] = t;
                };
            rot(0, 0, 3, 0, 3, 3, 0, 3); rot(1, 0, 3, 1, 2, 3, 0, 2);
            rot(2, 0, 3, 2, 1, 3, 0, 1); rot(1, 1, 2, 1, 2, 2, 1, 2);
        }
    clearMoving(); stampPiece();
    return true;
}

static void checkCompletion() {
    for (int j = ROWS - 2; j >= 0; j--) {
        int cnt = 0;
        for (int i = 1; i < COLS - 1; i++) if (isFull(grid[i][j])) cnt++;
        if (cnt == COLS - 2) { lineToDelete = true; for (int i = 1; i < COLS - 1; i++) grid[i][j] = FADING; }
    }
}

static int deleteLines() {
    int del = 0;
    for (int j = ROWS - 2; j >= 0; j--) {
        while (grid[1][j] == FADING) {
            for (int i = 1; i < COLS - 1; i++) grid[i][j] = EMPTY;
            for (int j2 = j - 1; j2 >= 0; j2--)
                for (int i = 1; i < COLS - 1; i++)
                    if (isFull(grid[i][j2]) || grid[i][j2] == FADING)
                    {
                        grid[i][j2 + 1] = grid[i][j2]; grid[i][j2] = EMPTY;
                    }
            del++;
        }
    }
    return del;
}

// ── 블록 그리기 헬퍼 ─────────────────────────────────────────────────────────
static void DrawBlock(int x, int y, Color col)
{
    DrawRectangle(x, y, SQUARE_SIZE, SQUARE_SIZE, col);
    // 하이라이트 (위/왼쪽 밝은 테두리)
    DrawLine(x, y, x + SQUARE_SIZE - 1, y, ColorBrightness(col, 0.4f));
    DrawLine(x, y, x, y + SQUARE_SIZE - 1, ColorBrightness(col, 0.4f));
    // 그림자 (아래/오른쪽 어두운 테두리)
    DrawLine(x + SQUARE_SIZE - 1, y, x + SQUARE_SIZE - 1, y + SQUARE_SIZE - 1, ColorBrightness(col, -0.4f));
    DrawLine(x, y + SQUARE_SIZE - 1, x + SQUARE_SIZE - 1, y + SQUARE_SIZE - 1, ColorBrightness(col, -0.4f));
}

static void DrawEmpty(int x, int y) {
    DrawLine(x, y, x + SQUARE_SIZE, y, LIGHTGRAY);
    DrawLine(x, y, x, y + SQUARE_SIZE, LIGHTGRAY);
    DrawLine(x + SQUARE_SIZE, y, x + SQUARE_SIZE, y + SQUARE_SIZE, LIGHTGRAY);
    DrawLine(x, y + SQUARE_SIZE, x + SQUARE_SIZE, y + SQUARE_SIZE, LIGHTGRAY);
}

// ── Update ────────────────────────────────────────────────────────────────────
static void UpdateGame()
{
    if (gameOver) { if (IsKeyPressed(KEY_ENTER)) { InitGame(); gameOver = false; } return; }
    if (IsKeyPressed('P')) paused = !paused;
    if (paused) return;

    if (lineToDelete) {
        fadingColor = (++fadeCnt % 8 < 4) ? MAROON : GRAY;
        if (fadeCnt >= FADE_TIME) { lines += deleteLines(); fadeCnt = 0; lineToDelete = false; }
        return;
    }

    if (!pieceActive) { pieceActive = createPiece(); fastCnt = 0; return; }

    fastCnt++; gravCnt++; latCnt++; turnCnt++;
    if (IsKeyPressed(KEY_LEFT) || IsKeyPressed(KEY_RIGHT)) latCnt = LAT_SPEED;
    if (IsKeyPressed(KEY_UP))  turnCnt = TURN_SPEED;
    if (IsKeyDown(KEY_DOWN) && fastCnt >= FAST_FALL) gravCnt += gravSpeed;

    if (gravCnt >= gravSpeed) { checkDetection(); resolveFalling(); checkCompletion(); gravCnt = 0; }
    if (latCnt >= LAT_SPEED) { if (!resolveLateral()) latCnt = 0; }
    if (turnCnt >= TURN_SPEED) { if (resolveTurn()) turnCnt = 0; }

    for (int j = 0; j < 2; j++) for (int i = 1; i < COLS - 1; i++)
        if (isFull(grid[i][j])) gameOver = true;
}

// ── Draw ──────────────────────────────────────────────────────────────────────
static void DrawGame()
{
    BeginDrawing();
    ClearBackground({ 30,30,30,255 }); // 어두운 배경

    if (gameOver) {
        DrawText("PRESS [ENTER] TO PLAY AGAIN",
            SW / 2 - MeasureText("PRESS [ENTER] TO PLAY AGAIN", 20) / 2, SH / 2 - 50, 20, WHITE);
        EndDrawing(); return;
    }

    int ox = SW / 2 - (COLS * SQUARE_SIZE / 2) - 50;
    int oy = SH / 2 - ((ROWS - 1) * SQUARE_SIZE / 2) + SQUARE_SIZE * 2 - 50;

    // 메인 그리드
    for (int j = 0; j < ROWS; j++) for (int i = 0; i < COLS; i++) {
        int x = ox + i * SQUARE_SIZE, y = oy + j * SQUARE_SIZE;
        Cell c = grid[i][j];
        if (c == EMPTY)       DrawEmpty(x, y);
        else if (c == BLOCK)  DrawBlock(x, y, DARKGRAY);
        else if (c == FADING) DrawBlock(x, y, fadingColor);
        else if (isMoving(c)) DrawBlock(x, y, PIECE_COLORS[pieceId(c)]);
        else if (isFull(c))   DrawBlock(x, y, ColorBrightness(PIECE_COLORS[pieceId(c)], -0.15f));
    }

    // 다음 피스 미리보기
    int npx = 500, npy = 45;
    DrawText("INCOMING:", npx, npy - 15, 10, LIGHTGRAY);
    for (int j = 0; j < 4; j++) for (int i = 0; i < 4; i++) {
        int x = npx + i * SQUARE_SIZE, y = npy + j * SQUARE_SIZE;
        if (next[i][j] == EMPTY) DrawEmpty(x, y);
        else DrawBlock(x, y, PIECE_COLORS[nextId]);
    }
    DrawText(TextFormat("LINES: %04i", lines), npx, npy + 4 * SQUARE_SIZE + 10, 10, LIGHTGRAY);

    if (paused) DrawText("GAME PAUSED",
        SW / 2 - MeasureText("GAME PAUSED", 40) / 2, SH / 2 - 40, 40, WHITE);

    EndDrawing();
}

static void UpdateDrawFrame() { UpdateGame(); DrawGame(); }

int main()
{
    InitWindow(SW, SH, "classic game: tetris (color)");
    InitGame();
#if defined(PLATFORM_WEB)
    emscripten_set_main_loop(UpdateDrawFrame, 60, 1);
#else
    SetTargetFPS(60);
    while (!WindowShouldClose()) UpdateDrawFrame();
#endif
    CloseWindow();
}
'''
