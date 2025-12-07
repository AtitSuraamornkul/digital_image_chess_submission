# Chess Move Detection System

## 📋 Overview
This notebook implements a robust chess move detection system that processes videos of chess games and outputs moves in PGN format. It handles various real-world challenges like camera angles, piece occlusion, and fast moves.

## How It Works (Plain Language Explanation)
Imagine you're watching a chess game through a camera. Here's how the system figures out what moves are being made:

1. **Finding the Chess Board**
   - The system first looks for the 4 corners of the chess board in the video
   - It tries multiple frames at the beginning (up to 30 frames) to reliably find these corners
   - Once found, it "flattens" the board to a top-down view (like looking straight down)

2. **Spotting the Pieces**
   - The system then identifies all chess pieces on the board using YOLOv8s
   - It pays special attention to the "feet" of the pieces to accurately place them on squares
   - The system also figures out which way the board is facing (which side is white/black)

### 🔍 How Pieces Are Mapped to Squares (Grid System)
The system uses a precise grid system to map pieces to chess squares:

1. **Grid Creation**
   - Creates an 8x8 grid representing the chess board
   - Each square is mapped to its chess notation (a1, a2, ..., h8)
   - Accounts for board borders with a 35-pixel margin
   
   ```python
   # Example grid creation
   square_size = (800 - 2*35) / 8  # 35px margin on each side
   grid = {
       'a1': {'center': (50, 750), 'bbox': (0, 700, 100, 800)},
       'b1': {'center': (150, 750), 'bbox': (100, 700, 200, 800)},
       # ... etc
   }
   ```

2. **Piece Positioning**
   - For each piece, calculates its "foot" position (base of the piece)
   - Adjusts for board rotation:
     - 0° (white bottom): Feet above center
     - 180° (black bottom): Feet below center
     - 90°/270°: Feet left/right of center
   
   ```python
   # Foot position calculation
   if rotation == 0:   # White at bottom
       foot_y = piece.y_center - (piece.height * 0.25)
   elif rotation == 180: # Black at bottom
       foot_y = piece.y_center + (piece.height * 0.25)
   ```

3. **Square Assignment**
   - Transforms foot position to board coordinates
   - Finds nearest square center
   - Only accepts assignments within half a square width

   ```python
   min_distance = float('inf')
   closest_square = None
   for square, data in grid.items():
       distance = sqrt((foot_x - data['center'][0])**2 + 
                       (foot_y - data['center'][1])**2)
       if distance < min_distance:
           min_distance = distance
           closest_square = square
   ```

3. **Tracking Moves**
   - As the video plays, the system constantly checks the board state
   - It doesn't trust single frames - it looks at the last 25 frames to be sure about piece positions
   - When it's 90% confident about a board state, it "locks it in"

4. **Validating Moves**
   - When pieces move between locked states, the system checks if it's a legal chess move
   - It uses chess rules to handle special cases like castling (O-O) and pawn captures (exd5)
   - If a move seems illegal, it can "resync" to match what it sees on the board

5. **Game Recording**
   - All valid moves are recorded in standard chess notation (PGN format)
   - The system adds + for checks and # for checkmates
   - It properly formats moves whether white or black starts first

6. **Final Output**
   - After processing all videos, it creates a clean CSV file with results
   - Each video gets its own row with all moves in the required format
   - The system ensures no missing data (NULL values)

This process allows the system to handle real-world challenges like:
- Camera angles and lighting changes
- Pieces being temporarily hidden by hands
- Fast moves between squares
- Different board orientations

## � Key Components

### 1. Chess Board Detection (`ChessBoardSystem`)
- **Corner Detection**: Uses YOLO model to find board corners
- **Piece Detection**: Identifies chess pieces and their positions
- **Perspective Transform**: Converts angled view to top-down perspective
- **Rotation Handling**: Detects board orientation (0°, 90°, 180°, 270°)

### 2. Move Detection (`MoveDetectorV4`)
- **Voting Stability System**: Uses sliding window consensus
- **Buffer Size**: 25 frames (configurable)
- **Confidence Threshold**: 90% agreement required
- **Failure Handling**: Resets after 10 consecutive failures

### 3. Move Validation (`ChessLogicValidator`)
- **python-chess Integration**: Validates moves using chess rules
- **Resync Capability**: Recovers from detection errors
- **Special Moves**: Handles castling, en passant, promotion

### 4. PGN Generation
- **Proper Formatting**: `1. e4 e5 2. Nf3 Nc6`
- **Check/Checkmate**: `Qh4+`, `Qxf7#`
- **Color Handling**: Supports both white-first and black-first games

## 🛠 Edge Case Handling

### 1. Initial Frame Issues
- **Multi-Frame Corner Detection**: Tries up to 30 frames to find corners
- **Consensus Requirement**: Needs 3 consistent detections
- **Fallback**: Uses best available detection if consensus not reached

```python
# Tries multiple frames for reliable corner detection
board_init = self.system.initialize_board_detection(
    cap, 
    max_attempts=30,
    required_detections=3
)
```

### 2. Board Rotation
- **Per-Video Reset**: `system.unlock_orientation()` between videos
- **Automatic Detection**: Uses piece positions to determine orientation
- **Locking**: Stops re-detection after first stable state

### 3. Move Validation Failures
- **Resync System**: Overrides engine state with visual detection
- **Forced Moves**: Uses `_uci_to_basic_san` when validation fails
- **Check Detection**: Verifies checks using current board state

### 4. Special Moves
| Move Type       | Handling                          |
|-----------------|-----------------------------------|
| Castling        | `O-O` (kingside), `O-O-O` (queenside) |
| En Passant      | Automatically validated by chess engine |
| Promotion       | `e8=Q` format                    |
| Check/Checkmate | `+` and `#` symbols              |

### 5. Video Variability
- **Bonus Videos**: Handles `(Bonus)Long_video_student.mp4`
- **File Types**: Processes both `.mp4` and `.mov`
- **Empty Output**: Returns empty string (not NULL) on errors

## ⚙ Processing Pipeline
1. **Initialize**: Detect corners and rotation
2. **Process Frames**:
   - Detect pieces and board state
   - Maintain state buffer (25 frames)
   - Find consensus state
3. **Track Moves**:
   - Calculate move between consensus states
   - Validate with chess engine
   - Resync if validation fails
4. **Generate Output**:
   - Convert moves to PGN format
   - Create CSV submission file

## 📊 Performance Optimization
- **Cached Corners**: Reuses initial corner detection
- **State Hashing**: Uses JSON hashes for efficient comparison
- **Early Termination**: Stops processing after consistent failures

## 🚀 Usage
```python
# Process single video
pgn = chess_move("video.mp4")

# Batch process
results = process_all_videos_to_csv("video_directory")
```

## ✅ Verification Checks
1. CSV header: `row_id,output`
2. No NULL values in output
3. Proper PGN formatting
4. Bonus video detection

This system robustly handles various chess game recordings while maintaining accurate move detection and PGN generation.
