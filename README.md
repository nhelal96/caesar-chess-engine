# Caesar Chess Engine (v1.0)

Caesar is a high-performance, command-line chess engine written in C. It represents chess state using a hybrid **Mailbox 10x12 (120-square) representation** mapped to **64-bit Bitboards** for optimized pawn and attack calculations. Caesar implements a complete classical computer chess architecture: iterative deepening minimax search, alpha-beta pruning, transposition tables, sophisticated move ordering, and positional heuristics.

---

## Technical Architecture Overview

Caesar separates chess playing into three clean decoupled concerns: state representation, search strategy, and positional evaluation.

```
                    ┌───────────────────────────┐
                    │  Protocols (UCI/Console)  │
                    └─────────────┬─────────────┘
                                  │
                                  ▼
                    ┌───────────────────────────┐
                    │       Search Engine       │◄───► [ Transposition Table ]
                    │    (Alpha-Beta / ID)      │      (Zobrist Hash Table)
                    └──────┬──────────────┬─────┘
                           │              │
                           ▼              ▼
            ┌─────────────────┐    ┌──────────────┐
            │ Move Generator  │    │  Evaluation  │◄───► [ Opening Book ]
            │ (Pseudo-Legal)  │    │  (PST/Pawn)  │
            └──────┬──────────┘    └──────┬───────┘
                   │                      │
                   └──────────┬───────────┘
                              ▼
                    ┌───────────────────────────┐
                    │        Board State        │
                    │ (Mailbox 10x12 + Bitboard)│
                    └───────────────────────────┘
```

---

## Key Algorithmic Details

### 1. Board Representation & Bitboards

Caesar utilizes a dual-board structure to combine the benefits of $O(1)$ coordinate off-board check validation with vector-like bit operations:
*   **Mailbox Array (`int pieces[120]`)**: Represents the board. An $8\times8$ board is padded with a boundary width of 2 squares to prevent illegal array indexing on knight jumps or sliding moves. Any coordinate with value `OFFBOARD` represents an illegal target.
*   **Bitboards (`U64 pawns[3]`)**: 64-bit unsigned integers representing white, black, and combined pawn structures. Bitboards enable fast bitwise operations for pawn structure analysis (isolated pawns, passed pawns) via clearing and set masks.

#### Index Conversions:
Convert between the compact 64-square representation and 120-square representation:
$$\text{Sq120}(x, y) = 21 + x + 10y \quad \text{where} \quad x \in [0,7], y \in [0,7]$$

---

### 2. Search Engine & Alpha-Beta Pruning

The search engine performs a depth-bounded depth-first search using **Minimax with Alpha-Beta Pruning** and **Iterative Deepening**. 

#### Alpha-Beta Decision Rule:
For each node with depth $d$, search window $[\alpha, \beta]$:
$$V(d, \alpha, \beta) = \max \left( \alpha, \min \left( \beta, \max_{m \in \text{Moves}} \left( -V(d-1, -\beta, -\alpha) \right) \right) \right)$$

```
                         [Alpha, Beta]
                         /           \
                 Score >= Beta      Score < Beta
                 (Beta Cutoff)      (Refine Alpha)
                 /                  /            \
          Prune Subtree      Alpha = Score    Continue Search
```

#### Advanced Search Heuristics:
1.  **Iterative Deepening**: Begins with a quick depth-1 search and increments depth step-by-step. This guarantees that search time constraints can interrupt the search gracefully and yields optimal move sorting tables for deeper plies.
2.  **Quiescence Search**: Solves the *horizon effect*. Once the search depth reaches zero, Caesar switches to searching only capture moves to compute a quiet evaluation state:
    $$\text{Quiescence}(\alpha, \beta) = \max \left( \text{Eval}(s), \max_{c \in \text{Captures}} \left( -\text{Quiescence}(-\beta, -\alpha) \right) \right)$$
3.  **Null Move Pruning**: If the side to move is not in check and has major pieces remaining, a "null move" is executed to yield a turn to the opponent. If the reduced search ($\text{depth} - 4$) still scores $\ge \beta$, the subtree is pruned.
4.  **Killer Move Heuristic**: Prioritizes non-capture moves that recently caused beta cutoffs at the same search ply (`searchKillers[2][MAXDEPTH]`).
5.  **History Heuristic**: Maintains a weight matrix of moves based on how often they caused cutoffs during the search loop (`searchHistory[Piece][TargetSquare]`).

---

### 3. Transposition Table & Zobrist Hashing

To avoid redundant computations on transposition pathways (e.g. `1. e4 e6 2. d4 d5` vs `1. d4 d5 2. e4 e6`), positions are cached in a hash table indexed using a **64-bit Zobrist Hash Key**:
$$\text{posKey} = \left( \bigoplus_{i=0}^{119} \text{PieceKeys}[\text{piece}][i] \right) \oplus \text{SideKey} \oplus \text{CastleKeys}[\text{castlePerm}] \oplus \text{EnPassantKeys}[\text{enPas}]$$
Each hash entry stores:
*   `posKey`: Position signature.
*   `move`: The best move found in this position.
*   `score`: Evaluated score (exact, upper bound/alpha, or lower bound/beta).
*   `depth`: Search depth at which the entry was cached.
*   `flags`: Cache hit bounds classification (`HFEXACT`, `HFALPHA`, `HFBETA`).

---

### 4. Move Ordering (MVV-LVA)

Efficient alpha-beta pruning depends heavily on ordering moves. Caesar sorts moves based on:
1.  **PV Move**: The best move found during the previous iterative deepening ply is evaluated first.
2.  **MVV-LVA (Most Valuable Victim - Least Valuable Attack)**: Capture moves are prioritized to find early beta cutoffs. The ordering score is calculated as:
    $$\text{Score}_{\text{MVV-LVA}} = \text{Value}(\text{Victim}) + (6 - \text{Value}(\text{Attacker}))$$
    For instance, a Pawn capturing a Queen ($105$ points) is prioritized over a Rook capturing a Queen ($101$ points).
3.  **Killer Moves**: Prioritized right after captures.
4.  **History Heuristics**: Prioritized for general positional moves.

---

### 5. Evaluation Heuristics

The evaluation function computes the positional advantage in centipawns ($1 \text{ pawn} = 100 \text{ centipawns}$):

$$\text{Eval} = \text{MaterialScore} + \text{PSTScore} + \text{PawnStructureScore} + \text{MobilityScore}$$

#### Evaluation Criteria:
*   **Material Values**: Pawns ($100$), Knights ($325$), Bishops ($325$), Rooks ($550$), Queens ($975$), Kings ($50000$).
*   **Piece-Square Tables (PST)**: Custom $8\times8$ coordinate tables optimizing developmental positioning (e.g. Knights toward the center, Pawns advanced, Kings castled safely in middlegames).
*   **Dynamic King Safety**: Swaps between two distinct evaluation structures depending on material counts:
    *   **Middlegame King Table (`KingO`)**: Penalizes the king for advancing out of the corners.
    *   **Endgame King Table (`KingE`)**: Encourages the king to centralize and actively participate in play.
*   **Pawn Structure**:
    *   **Isolated Pawn Penalty** ($-10$): Penalizes pawns with no adjacent friendly pawns on neighboring files.
    *   **Passed Pawn Bonus**: Awards an incremental bonus depending on how far the pawn has advanced without opposition:
        $$\text{Bonus} = \text{PawnPassed}[\text{Rank}]$$
*   **Rook/Queen File Mobility**: Awards bonuses for controlling fully open or semi-open files.

---

## File Structure

```
caesar-chess-engine/
├── attack.c          # Attack detection functions (is square attacked by side?)
├── bitboard.c        # 64-bit bitboard operations (pop, count, print)
├── board.c           # Board initialization, FEN parsing, and state updates
├── data.c            # Lookup masks, piece square tables, mirror maps
├── defs.h            # Main headers, constants, structures, and macros
├── evaluate.c        # Positional evaluation (Material, PST, Passed Pawns)
├── hashkeys.c        # Zobrist hash key generators
├── init.c            # Global arrays and mask initializations
├── io.c              # Move parsing, console outputs, FEN validation
├── main.c            # Command loop and application entrypoint
├── makemove.c        # Board state update and rollback logic (Make/Take move)
├── misc.c            # Time check and I/O buffer readings
├── movegen.c         # Pseudo-legal move generator (all moves & caps-only)
├── perft.c           # Move generator performance tester
├── polybook.c        # Polyglot opening book parser
├── pvtable.c         # Transposition hash table manager
├── search.c          # Alpha-beta, iterative deepening, quiescence search
├── uci.c             # Universal Chess Interface implementation
├── validate.c        # Board and move verification tests
└── xboard.c          # XBoard/Winboard protocol interface
```

---

## Building and Running

### Compilation
Build the executable using `gcc` or `clang`:

```bash
# Compile with optimizations
gcc -O3 -std=c99 attack.c bitboard.c board.c data.c hashkeys.c init.c io.c main.c makemove.c misc.c movegen.c perft.c polybook.c pvtable.c search.c uci.c validate.c xboard.c -o caesar
```

### Running the Engine

You can run Caesar in **console mode** directly for debugging, or connect it to chess GUIs (like Arena, Winboard, or Cutechess) using supported protocols:

#### Console Interactive Mode:
```bash
./caesar
# Type "help" or "console" for direct command mode
```

#### Universal Chess Interface (UCI):
```bash
./caesar
# Type "uci" to initiate UCI handshake
# Enter standard commands like:
#   position startpos moves e2e4
#   go depth 6
```
