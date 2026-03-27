import math
import sys
import pygame


WIDTH, HEIGHT = 640, 640
ROWS, COLS = 8, 8
SQ_SIZE = WIDTH // COLS
FPS = 60

LIGHT = (240, 217, 181)
DARK = (181, 136, 99)
HIGHLIGHT = (90, 180, 90)
SELECTED = (80, 160, 220)
TEXT_COLOR = (20, 20, 20)
BG_PANEL = (245, 245, 245)

# Unicode pieces (works with fonts that support chess glyphs)
PIECE_UNICODE = {
    "wK": "♔",
    "wQ": "♕",
    "wR": "♖",
    "wB": "♗",
    "wN": "♘",
    "wP": "♙",
    "bK": "♚",
    "bQ": "♛",
    "bR": "♜",
    "bB": "♝",
    "bN": "♞",
    "bP": "♟",
}

PIECE_VALUES = {"K": 0, "Q": 900, "R": 500, "B": 330, "N": 320, "P": 100}


def initial_board():
    return [
        ["bR", "bN", "bB", "bQ", "bK", "bB", "bN", "bR"],
        ["bP", "bP", "bP", "bP", "bP", "bP", "bP", "bP"],
        ["--", "--", "--", "--", "--", "--", "--", "--"],
        ["--", "--", "--", "--", "--", "--", "--", "--"],
        ["--", "--", "--", "--", "--", "--", "--", "--"],
        ["--", "--", "--", "--", "--", "--", "--", "--"],
        ["wP", "wP", "wP", "wP", "wP", "wP", "wP", "wP"],
        ["wR", "wN", "wB", "wQ", "wK", "wB", "wN", "wR"],
    ]


class GameState:
    def __init__(self):
        self.board = initial_board()
        self.white_to_move = True
        self.move_log = []
        self.game_over = False
        self.game_result = ""

    def in_bounds(self, r, c):
        return 0 <= r < 8 and 0 <= c < 8

    def current_color(self):
        return "w" if self.white_to_move else "b"

    def opponent_color(self):
        return "b" if self.white_to_move else "w"

    def make_move(self, move):
        (sr, sc), (er, ec), captured, promoted = move
        piece = self.board[sr][sc]
        self.board[sr][sc] = "--"
        self.board[er][ec] = promoted if promoted else piece
        self.move_log.append(move)
        self.white_to_move = not self.white_to_move

    def undo_move(self):
        if not self.move_log:
            return
        (sr, sc), (er, ec), captured, promoted = self.move_log.pop()
        moved_piece = self.board[er][ec]
        if promoted:
            moved_piece = moved_piece[0] + "P"
        self.board[sr][sc] = moved_piece
        self.board[er][ec] = captured
        self.white_to_move = not self.white_to_move

    def find_king(self, color):
        king = color + "K"
        for r in range(8):
            for c in range(8):
                if self.board[r][c] == king:
                    return r, c
        return None

    def square_under_attack(self, row, col, attacker_color):
        # Knights
        knight_dirs = [(-2, -1), (-2, 1), (-1, -2), (-1, 2),
                       (1, -2), (1, 2), (2, -1), (2, 1)]
        for dr, dc in knight_dirs:
            rr, cc = row + dr, col + dc
            if self.in_bounds(rr, cc):
                piece = self.board[rr][cc]
                if piece == attacker_color + "N":
                    return True

        # Kings
        king_dirs = [(-1, -1), (-1, 0), (-1, 1), (0, -1),
                     (0, 1), (1, -1), (1, 0), (1, 1)]
        for dr, dc in king_dirs:
            rr, cc = row + dr, col + dc
            if self.in_bounds(rr, cc):
                piece = self.board[rr][cc]
                if piece == attacker_color + "K":
                    return True

        # Pawns
        pawn_dr = -1 if attacker_color == "w" else 1
        for dc in (-1, 1):
            rr, cc = row + pawn_dr, col + dc
            if self.in_bounds(rr, cc) and self.board[rr][cc] == attacker_color + "P":
                return True

        # Sliding pieces
        line_dirs = [(-1, 0), (1, 0), (0, -1), (0, 1)]
        diag_dirs = [(-1, -1), (-1, 1), (1, -1), (1, 1)]

        for dr, dc in line_dirs:
            rr, cc = row + dr, col + dc
            while self.in_bounds(rr, cc):
                piece = self.board[rr][cc]
                if piece != "--":
                    if piece[0] == attacker_color and piece[1] in ("R", "Q"):
                        return True
                    break
                rr += dr
                cc += dc

        for dr, dc in diag_dirs:
            rr, cc = row + dr, col + dc
            while self.in_bounds(rr, cc):
                piece = self.board[rr][cc]
                if piece != "--":
                    if piece[0] == attacker_color and piece[1] in ("B", "Q"):
                        return True
                    break
                rr += dr
                cc += dc

        return False

    def in_check(self, color):
        king_pos = self.find_king(color)
        if king_pos is None:
            return False
        kr, kc = king_pos
        attacker = "b" if color == "w" else "w"
        return self.square_under_attack(kr, kc, attacker)

    def get_valid_moves(self):
        color = self.current_color()
        pseudo = self.get_all_pseudo_legal_moves(color)
        valid = []
        for mv in pseudo:
            self.make_move(mv)
            if not self.in_check("b" if self.white_to_move else "w"):
                valid.append(mv)
            self.undo_move()

        if not valid:
            if self.in_check(color):
                self.game_over = True
                self.game_result = ("Black wins by checkmate" if color == "w"
                                    else "White wins by checkmate")
            else:
                self.game_over = True
                self.game_result = "Draw by stalemate"
        else:
            self.game_over = False
            self.game_result = ""
        return valid

    def get_all_pseudo_legal_moves(self, color):
        moves = []
        for r in range(8):
            for c in range(8):
                piece = self.board[r][c]
                if piece == "--" or piece[0] != color:
                    continue
                kind = piece[1]
                if kind == "P":
                    self._pawn_moves(r, c, moves, color)
                elif kind == "N":
                    self._knight_moves(r, c, moves, color)
                elif kind == "B":
                    self._slider_moves(r, c, moves, color, [(-1, -1), (-1, 1), (1, -1), (1, 1)])
                elif kind == "R":
                    self._slider_moves(r, c, moves, color, [(-1, 0), (1, 0), (0, -1), (0, 1)])
                elif kind == "Q":
                    self._slider_moves(
                        r, c, moves, color,
                        [(-1, -1), (-1, 1), (1, -1), (1, 1), (-1, 0), (1, 0), (0, -1), (0, 1)]
                    )
                elif kind == "K":
                    self._king_moves(r, c, moves, color)
        return moves

    def _pawn_moves(self, r, c, moves, color):
        direction = -1 if color == "w" else 1
        start_row = 6 if color == "w" else 1
        promotion_row = 0 if color == "w" else 7
        enemy = "b" if color == "w" else "w"

        nr = r + direction
        if self.in_bounds(nr, c) and self.board[nr][c] == "--":
            promoted = color + "Q" if nr == promotion_row else None
            moves.append(((r, c), (nr, c), "--", promoted))
            if r == start_row:
                nr2 = r + 2 * direction
                if self.in_bounds(nr2, c) and self.board[nr2][c] == "--":
                    moves.append(((r, c), (nr2, c), "--", None))

        for dc in (-1, 1):
            nc = c + dc
            if self.in_bounds(nr, nc):
                target = self.board[nr][nc]
                if target != "--" and target[0] == enemy:
                    promoted = color + "Q" if nr == promotion_row else None
                    moves.append(((r, c), (nr, nc), target, promoted))

    def _knight_moves(self, r, c, moves, color):
        jumps = [(-2, -1), (-2, 1), (-1, -2), (-1, 2),
                 (1, -2), (1, 2), (2, -1), (2, 1)]
        enemy = "b" if color == "w" else "w"
        for dr, dc in jumps:
            rr, cc = r + dr, c + dc
            if not self.in_bounds(rr, cc):
                continue
            target = self.board[rr][cc]
            if target == "--":
                moves.append(((r, c), (rr, cc), "--", None))
            elif target[0] == enemy:
                moves.append(((r, c), (rr, cc), target, None))

    def _slider_moves(self, r, c, moves, color, directions):
        enemy = "b" if color == "w" else "w"
        for dr, dc in directions:
            rr, cc = r + dr, c + dc
            while self.in_bounds(rr, cc):
                target = self.board[rr][cc]
                if target == "--":
                    moves.append(((r, c), (rr, cc), "--", None))
                else:
                    if target[0] == enemy:
                        moves.append(((r, c), (rr, cc), target, None))
                    break
                rr += dr
                cc += dc

    def _king_moves(self, r, c, moves, color):
        enemy = "b" if color == "w" else "w"
        dirs = [(-1, -1), (-1, 0), (-1, 1), (0, -1),
                (0, 1), (1, -1), (1, 0), (1, 1)]
        for dr, dc in dirs:
            rr, cc = r + dr, c + dc
            if not self.in_bounds(rr, cc):
                continue
            target = self.board[rr][cc]
            if target == "--":
                moves.append(((r, c), (rr, cc), "--", None))
            elif target[0] == enemy:
                moves.append(((r, c), (rr, cc), target, None))


def evaluate_board(state):
    score = 0
    for r in range(8):
        for c in range(8):
            piece = state.board[r][c]
            if piece == "--":
                continue
            value = PIECE_VALUES[piece[1]]
            if piece[0] == "w":
                score += value
            else:
                score -= value
    return score


def minimax(state, depth, alpha, beta, maximizing_for_white):
    valid_moves = state.get_valid_moves()
    if depth == 0 or state.game_over:
        if state.game_over:
            if "White wins" in state.game_result:
                return 1000000, None
            if "Black wins" in state.game_result:
                return -1000000, None
            return 0, None
        return evaluate_board(state), None

    best_move = None
    if maximizing_for_white:
        max_eval = -math.inf
        for mv in valid_moves:
            state.make_move(mv)
            eval_score, _ = minimax(state, depth - 1, alpha, beta, False)
            state.undo_move()
            if eval_score > max_eval:
                max_eval = eval_score
                best_move = mv
            alpha = max(alpha, eval_score)
            if beta <= alpha:
                break
        return max_eval, best_move
    min_eval = math.inf
    for mv in valid_moves:
        state.make_move(mv)
        eval_score, _ = minimax(state, depth - 1, alpha, beta, True)
        state.undo_move()
        if eval_score < min_eval:
            min_eval = eval_score
            best_move = mv
        beta = min(beta, eval_score)
        if beta <= alpha:
            break
    return min_eval, best_move


def draw_board(screen):
    for r in range(ROWS):
        for c in range(COLS):
            color = LIGHT if (r + c) % 2 == 0 else DARK
            pygame.draw.rect(screen, color, (c * SQ_SIZE, r * SQ_SIZE, SQ_SIZE, SQ_SIZE))


def draw_pieces(screen, board, font):
    for r in range(8):
        for c in range(8):
            piece = board[r][c]
            if piece == "--":
                continue
            glyph = PIECE_UNICODE.get(piece, piece)
            text_surface = font.render(glyph, True, TEXT_COLOR)
            rect = text_surface.get_rect(center=(c * SQ_SIZE + SQ_SIZE // 2, r * SQ_SIZE + SQ_SIZE // 2))
            screen.blit(text_surface, rect)


def draw_selection(screen, selected_sq, valid_moves):
    if selected_sq is None:
        return
    sr, sc = selected_sq
    pygame.draw.rect(screen, SELECTED, (sc * SQ_SIZE, sr * SQ_SIZE, SQ_SIZE, SQ_SIZE), 4)
    for move in valid_moves:
        (r0, c0), (r1, c1), _, _ = move
        if r0 == sr and c0 == sc:
            pygame.draw.rect(screen, HIGHLIGHT, (c1 * SQ_SIZE, r1 * SQ_SIZE, SQ_SIZE, SQ_SIZE), 4)


def draw_status(screen, font_small, message):
    panel_h = 40
    pygame.draw.rect(screen, BG_PANEL, (0, HEIGHT - panel_h, WIDTH, panel_h))
    text = font_small.render(message, True, TEXT_COLOR)
    screen.blit(text, (10, HEIGHT - panel_h + 10))


def square_from_mouse(pos):
    x, y = pos
    if y >= HEIGHT:
        return None
    return y // SQ_SIZE, x // SQ_SIZE


def main():
    pygame.init()
    screen = pygame.display.set_mode((WIDTH, HEIGHT))
    pygame.display.set_caption("Chess - Pygame + Minimax AI")
    clock = pygame.time.Clock()

    piece_font = pygame.font.SysFont("Segoe UI Symbol", 54)
    small_font = pygame.font.SysFont("Calibri", 22)

    state = GameState()
    selected = None
    valid_moves = state.get_valid_moves()

    # Human plays white, AI plays black.
    human_color = "w"
    ai_depth = 3

    running = True
    while running:
        clock.tick(FPS)

        turn_color = state.current_color()
        is_human_turn = turn_color == human_color

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            elif event.type == pygame.KEYDOWN and event.key == pygame.K_r:
                state = GameState()
                selected = None
                valid_moves = state.get_valid_moves()
            elif is_human_turn and event.type == pygame.MOUSEBUTTONDOWN and not state.game_over:
                sq = square_from_mouse(pygame.mouse.get_pos())
                if sq is None:
                    continue
                r, c = sq
                piece = state.board[r][c]
                if selected is None:
                    if piece != "--" and piece[0] == human_color:
                        selected = (r, c)
                else:
                    move_done = False
                    for mv in valid_moves:
                        if mv[0] == selected and mv[1] == (r, c):
                            state.make_move(mv)
                            selected = None
                            valid_moves = state.get_valid_moves()
                            move_done = True
                            break
                    if not move_done:
                        if piece != "--" and piece[0] == human_color:
                            selected = (r, c)
                        else:
                            selected = None

        if not state.game_over and not is_human_turn:
            maximizing = state.current_color() == "w"
            _, best = minimax(state, ai_depth, -math.inf, math.inf, maximizing)
            if best is not None:
                state.make_move(best)
                valid_moves = state.get_valid_moves()
            else:
                state.get_valid_moves()

        draw_board(screen)
        draw_selection(screen, selected, valid_moves)
        draw_pieces(screen, state.board, piece_font)

        status = "White to move" if state.white_to_move else "Black (AI) to move"
        if state.game_over:
            status = state.game_result + " | Press R to restart"
        else:
            status += " | Press R to restart"
        draw_status(screen, small_font, status)

        pygame.display.flip()

    pygame.quit()
    sys.exit()


if __name__ == "__main__":
    main()
